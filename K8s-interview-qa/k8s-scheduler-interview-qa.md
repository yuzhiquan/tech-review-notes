# k8s-scheduler-interview-qa

# Kubernetes Scheduler 面试题全解析：原理篇 × 大规模集群篇

> 适用角色：SRE / 云原生运维 / 平台工程师面试（社招 3 年起 + 偏高阶）
> 文档定位：既能"背答案"，也能直接落地到集群治理
> 版本约定：以 **1.28 \~**** 1.31 主线**为准；涉及具体版本号、源码常量、默认值的地方均标注"以所用版本源码 / Release Notes 为准"，避免版本漂移导致答错。

---

## 0. 使用说明与答题方法论

### 0.1 面试官真正在考什么

| 层级 | 考点 | 表现 |
| --- | --- | --- |
| L1 会用 | 知道 requests/limits、亲和、污点怎么写 | 能背 YAML |
| L2 懂原理 | 说得清 Scheduling Framework 12 个扩展点、队列三级结构、抢占流程 | 能画图、能讲为什么这样设计 |
| L3 有规模感 | 知道 1000/5000 节点下谁先崩、吞吐瓶颈在哪、怎么量化 | 能报指标名、说得清 trade-off |
| L4 有事故经验 | 能讲清"现象 → 证据 → 定位路径 → 止血 → 根治 → 防复发" | 有 PromQL、有灰度、有回滚 |

> **答题模板（强烈建议每道题都套）**：结论先行 → 讲清机制/数据流 → 给一个可落地证据（配置片段 / 指标名 / 事件字段）→ 补一个坑或边界条件。

### 0.2 本文档的数值约定

按照 SRE 规范，本文档严格区分三类数字：

- **官方/源码值**：标注出处（如 `KubeSchedulerConfiguration` 字段），并提示用 `kube-scheduler --write-config-to=conf.yaml` 现场核对；
- **建议值（推荐初值）**：标注为 *建议*，必须结合业务实测调整；
- **实测值**：本文档 **不提供任何未实测的性能数字**，凡涉及吞吐/耗时均给出**测量方法**，避免编造。

---

# Part A · 原理篇（22 题）

## Q1. kube-scheduler 到底负责什么？职责边界在哪？

**一句话**：kube-scheduler 只做一件事——**为 **`.spec.nodeName`** 为空的 Pod 选一个 Node，并把绑定结果写回 API Server**。它不运行容器，不保证 Pod 真的能跑起来。

**职责边界（高频送分/送命点）**：

| 事项 | 是否 scheduler 负责 | 实际负责人 |
| --- | --- | --- |
| 选节点 + 写绑定 | ✅ 是 | kube-scheduler |
| `ResourceQuota` 校验 | ❌ 否 | admission controller（ResourceQuota 插件） |
| `LimitRange` 注入默认值 | ❌ 否 | admission controller |
| 校验 requests ≤ limits | ❌ 否 | apiserver validation |
| Node 故障驱逐 Pod | ❌ 否 | NodeLifecycleController（taint-manager）+ kubelet |
| 资源不足时驱逐 Pod | ❌ 否 | kubelet eviction manager |
| 抢占低优先级 Pod | ✅ 是 | kube-scheduler（PostFilter / DefaultPreemption） |
| 容器层面能否起来 | ❌ 否 | kubelet admission + runtime |

**加分点**：正因为 scheduler 不校验 quota，**Pod 被 scheduler 成功绑定后，仍可能在 binding 阶段被 admission/Kubelet 拒绝**（例如 quota 超限、快照卷不满足），这就是 `FailedScheduling` 之外还能看到奇怪失败的原因。

---

## Q2. 讲一次完整调度循环（scheduleOne）的数据流

```
┌─────────────── 一个 profile = 一个 goroutine 串行跑 scheduleOne ───────────────┐
NextPod()  ← 阻塞地从 SchedulingQueue pop 一个 Pod
   │
   ├─ 若 Pod 已在 assumedPod 集合中（重复入队）→ 跳过
   │
   ├─【1】获取 Node 快照 Snapshot（从 scheduler cache 拷贝 NodeInfo 列表，不含已删除 Node）
   │
   ├─【2】SchedulingFramework.RunPreFilterPlugins  ← 单个 PreFilter 返回 NonSuccess 即失败
   │         └─ 典型：PodFitsResources(with PreFilter extenders)、NodePorts、VolumeRestrictions
   │
   ├─【3】findNodesThatFitPod
   │         └─ 并发（默认 parallelism=16）对所有可行节点跑 Filter 插件
   │         └─ percentageOfNodesToScore 生效：达到节点发现比例即提前停止
   │
   ├─【4】若可行节点列表为空 → RunPostFilterPlugins（DefaultPreemption 抢占）
   │         └─ 抢占成功 → nominatedNodeName，回队列；失败 → requeue（backoff）
   │
   ├─【5】若仅 1 个候选节点 → 直接选中（不再打分，快路径）
   │
   ├─【6】RunPreScorePlugins → RunScorePlugins（并发 16）→ RunNormalizeScorePlugins
   │         └─ 加权求和：nodeScore = Σ(pluginScore(plugin, node) * pluginWeight)
   │
   ├─【7】SelectHost：在所有"最高分"节点中**随机**取一个（避免总打偏同一个节点）
   │
   ├─【8】Reserve 阶段调用 **cache.assumePod()**（乐观假设绑定：先把 Pod 记入 NodeInfo 占用资源，仅写调度器本地 cache，不落 API Server）
   │
   ├─【9】RunPermitPlugins（可以让 Pod 进入"等待"状态，如 Gang Scheduling 配额等待、某些流量网关）
   │
   └─【10】异步 goroutine：Bind 阶段（PreBind → Bind → PostBind）
              └─ 调用 API Server 的 binding subresource（或新版直接更新 Pod），
                 失败则 Unreserve（回滚 assume）+ Pod 重新入队
```

**关键设计点（要说出来才加分）**：

1. **每个 profile 是单 goroutine 串行的**。不是"所有 Pod 并发调度"，而是"一次调度周期内部的**节点遍历**是并发的"。这是单机吞吐上限讨论的根。
2. **Assume/乐观绑定**：Reserve 之后到真正 Bind 成功之间有一段时间窗，此时调度器 cache 已"假装"占用了资源，让**下一个 Pod 不必等 API Server 往返**即可继续调度 —— 这是吞吐优化的核心手法，代价是短暂的 cache 与真实状态不一致。
3. **CycleState**：每个调度周期一个 pod-scoped 状态对象，插件之间传递中间计算结果；它随周期销毁，不跨 Pod。

---

## Q3. Scheduling Framework 有哪些扩展点？作用与典型插件是什么？

**这是必考题，表格要能默下来**：

| # | 扩展点 | 阶段 | 作用 / 特点 | 典型内置插件 |
| --- | --- | --- | --- | --- |
| 1 | `PreEnqueue` | 队列 | Pod 进 ActiveQ 前的前置门禁 | 部分版本结合别的项目用；自定义 gate 常用 |
| 2 | `QueueSort` | 队列 | 决定 pop 顺序 | `PrioritySort`（按 priority 排） |
| 3 | `PreFilter` | 调度周期 | 全局预检 & 预计算；**任何一个返回失败就整体失败** | `NodeResourcesFit`、`NodePorts`、`VolumeRestrictions`、`PodTopologySpread`、`InterPodAffinity` |
| 4 | `Filter` | 调度周期 | 逐节点过滤（可并发） | `NodeUnschedulable`、`TaintToleration`、`NodeAffinity`、`NodeResourcesFit`、`VolumeBinding`、`VolumeZone` |
| 5 | `PostFilter` | 调度周期 | **Filter 全失败后才执行**，做抢占 | `DefaultPreemption` |
| 6 | `PreScore` | 调度周期 | 打分前预计算 | `PodTopologySpread`、`InterPodAffinity` |
| 7 | `Score` | 调度周期 | 逐节点打分 0\~100 | `NodeResourcesFit`、`NodeResourcesBalancedAllocation`、`ImageLocality`、`PodTopologySpread` |
| 8 | `NormalizeScore` | 调度周期 | 分数归一化到 0\~100 | 多数 Score 插件自带 |
| 9 | `Reserve` | 绑定周期 | 预留资源 / assume | `VolumeBinding` |
| 10 | `Permit` | 绑定周期 | 批准或延迟绑定（可 `Wait`） | 自研 Gang/Coscheduling 插件 |
| 11 | `PreBind` | 绑定周期 | 绑定前准备（如 VolumeBinding 在这里真正下发 PVC/PV 的绑定请求） | `VolumeBinding` |
| 12 | `Bind` | 绑定周期 | 真正写 `nodeName` | `DefaultBinder` |
| 13 | `PostBind` | 绑定周期 | 绑定后通知（异步、信息性） | 自研 |

**必答的边界**：

- `PreFilter` 是**全局**的（一次整个周期只跑一次），`Filter` 是**逐节点**的；
- `Reserve` 失败会触发 `Unreserve` 回滚；
- `Permit` 与其他的区别：它可以返回 `Wait`/`Timeout`，把 Pod 挂起而不算失败。

**可直接套用的配置片段**（`KubeSchedulerConfiguration`）：

```yaml
apiVersion: kubescheduler.config.k8s.io/v1
kind: KubeSchedulerConfiguration
profiles:
  - schedulerName: default-scheduler
    plugins:
      filter:
        enabled:
          - name: NodeResourcesFit
          - name: NodePorts
          - name: NodeAffinity
          - name: TaintToleration
        disabled:
          - name: "*"      # 先全关再逐个启用，白名单模式更安全
      score:
        enabled:
          - name: NodeResourcesFit
            weight: 2
          - name: PodTopologySpread
            weight: 2
    pluginConfig:
      - name: NodeResourcesFit
        args:
          scoringStrategy:
            type: MostAllocated     # 装箱更紧，降低碎片；在线业务建议 LeastAllocated
            resources:
              - name: cpu
                weight: 1
              - name: memory
                weight: 1
```

> **验证方式**：改完配置后，先 `kube-scheduler --config=conf.yaml --dry-run`（若版本支持）与 `--write-config-to=dump.yaml` 对比默认导出，确认插件集合符合预期再上线。

---

## Q4. 调度队列为什么是三级的？Pod 什么时候会被搬移？

```
                    ┌────────────┐
   新 Pod / 重排   │  ActiveQ   │  ← heap，按 QueueSort（默认 priority + 时间戳）排序
                    └─────┬──────┘
                          │ pop → scheduleOne
                          ▼
                    ┌────────────┐
   调度失败 ───────► │  BackoffQ  │  ← 指数退避：init→max（指数递增，默认 1s→10s，见配置字段）
                    └─────┬──────┘
                          │ 退避到期
                          ▼
                    ┌──────────────────┐
  集群事件命不中 ──► │ UnschedulableQ   │  ← 没有正要去野的集群变化时停在这（不消耗 CPU）
                    └──────────────────┘
```

**搬移条件（核心考点）**：

1. **触发事件类型**：Node/PV/PVC/StorageClass/CSINode/Service 等 add/update 时，会调用 `MoveAllToActiveOrBackoffQueue`；
2. `schedulingCycle`** 机制**：记录在队列内的 cycle 计数。某 Pod 是在第 N 周期被判定 unschedulable，只有"周期 N 之后"发生的集群事件才算"新鲜事件"，才会把它搬回 ActiveQ —— 避免老事件反复触发无用调度；
3. **兜底定时器**：即使没有任何事件，也会周期性 flush 回来重排（源码常量 `podMaxInUnschedulablePodsDuration`，默认 `5m`，在你的版本里 `grep podMaxInUnschedulablePodsDuration pkg/scheduler/internal/queue` 核对）；
4. **QueueingHint（新版本）**：进一步精细化——只有"该 Pod 失败的某个插件认为这个事件可能让它重新可行"时才搬移，大幅减少大规模集群里无意义的 requeue。

**可直接套用**：

```yaml
apiVersion: kubescheduler.config.k8s.io/v1
kind: KubeSchedulerConfiguration
podInitialBackoffSeconds: 1      # 建议初值（源码默认 1）
podMaxBackoffSeconds: 10         # 建议初值（源码默认 10）
podMaxInUnschedulablePodsDuration: 5m  # 源码默认 5m
parallelism: 16                  # 节点遍历并发度，源码默认 16
```

> **大规模场景的坑**：大批 Job Pod pending + 集群频繁节点变更 ⇒ 事件一来就把成千上万 unschedulable pod 全部搬回 ActiveQ，造成**重调度风暴**。解法：开 QueueingHint（新版本）、pod 拆分 Queue（多 profile）、资源提前预留、以及见 L4。

---

## Q5. percentageOfNodesToScore 是什么？为什么要有它？

**问题**：5000 节点的集群，如果每个 Pod 都完整遍历 5000 个节点跑一遍 Filter，一次调度周期的成本与节点数线性相关——节点越多吞吐越低。

**机制**：只遍历一定比例（或一定数量）的节点，**找到足够多的可行节点后就停止**，从这些里打分选最优。

**判定逻辑**：

- 节点总数 ≤ 100 → **100% 遍历**（小规模无意义减小；源码里的 min 阈值 100）；
- 节点总数 \> 100 → 使用 `percentageOfNodesToScore`；
- **未显式配置（=0）时的自适应公式**：`50 - numAllNodes / 125`，并 clamp 到 `[5, 100]`。即 5000 节点时约 10%（*该公式为社区实现，以源码为准*）。

**风险（必答）**：这是**牺牲最优性换吞吐**的近似算法——可能导致 Pod 落到"次优"节点（碎片、镜像缺失、跨可用区）。

- **离线批处理**：可以设 5\~15%，换吞吐；
- **在线低延迟/亲和敏感业务**：设 100%（或保证国产/敏感业务量大在为数不多的节点池里，节点基数小本身就不触发近似）。

```yaml
apiVersion: kubescheduler.config.k8s.io/v1
kind: KubeSchedulerConfiguration
percentageOfNodesToScore: 30   # 建议：普通集群 20~30；CPU/GPU 贵资源集群考虑 100
```

**验证方式**：改前后对比 `scheduler_framework_plugin_duration_seconds{...Filter...}` 的 P99 与 `scheduler_pod_scheduling_duration_seconds` P99。

---

## Q6. 抢占（Preemption）的完整流程是什么？

**触发点**：`Filter` 后**一个可行节点都没有** → 进入 `PostFilter`。

**流程**：

```
1. 优先级判定：Pod 必须有更高 PriorityClass；victim 只能来自同/更低优先级
2. 对所有节点做"假设 + 试删"：
      └─ 逐步把最低优先级的 Pod 标记为 victim（"把资源挤出来"），直到本 Pod 能放下
      └─ 受 PodDisruptionBudget 保护：若删除会违反 victim 的 PDB，该候选节点会被**重罚分**（天然排在后面甚至被排除）
3. 选一个"代价最小"的节点：优先级越低越好、被抢 Pod 越少越好、违反 PDB 越少越好
4. 通过 Eviction/Delete 删除 victim Pod（victim 会进入优雅退出）
5. 给抢占者设置 `status.nominatedNodeName = <node>`，Pod 回队列等待
6. 下一轮调度周期：
      └─ PreFilter/Filter 前会先"把 victims 从 NodeInfo 里暂借移出"再试算，
         即 nominated pod 能"看到"即将释放的资源
7. victim 真的消失后（deletion timestamp → 对象移除），抢占者完成绑定
```

**四个必答细节**：

| 细节 | 说明 |
| --- | --- |
| `nominatedNodeName` | 只对 Terminating 的 victim 有意义；是"预约席"，不是绑定结果 |
| **抢占失败/虚假抢占** | victim 迟迟没消失（优雅退出期长、finalizer 卡住），抢占者会一直等待或按退避重新计算；社区称这种现象为 bogus preemption（虚假抢占），可通过缩短 victim 的优雅退出期、治理 finalizer 来规避 |
| `preemptionPolicy: Never` | 高优先级 Pod 声明"我不抢别人"，只作为 victim 候选外的旁观者 |
| `PodDisruptionBudget` | 保护的是 **victim**（`minAvailable`），含义是"不会为了抢占而毁掉已有的可用性承诺" |

**可直接套用**：

```yaml
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: prod-critical
value: 1000000
globalDefault: false
preemptionPolicy: PreemptLowerPriority
description: "核心在线业务，允许抢占低优离线任务"
---
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: offline-batch
value: 1000
globalDefault: true
preemptionPolicy: Never     # 关键：批处理永不抢占，避免抢占风暴
description: "离线批处理，可被抢占，但不抢别人"
```

> **注意**：`globalDefault: true` 全局只能有一个。给"默认优先级"设 NEVER 抢占，是防大规模抢占风暴最有效的一招。

---

## Q7. 抢占 vs 驱逐 vs PDB vs Descheduler：别再混为一谈

| 机制 | 谁触发 | 目标 | 是否考虑 PDB | 典型场景 |
| --- | --- | --- | --- | --- |
| **Scheduler Preemption** | kube-scheduler | 给 Pending 的高优 Pod 腾地方 | 是 | 高优在线挤走离线 |
| **Kubelet Eviction** | kubelet | 节点本身快不行了（内存/磁盘/inode/PID 不足） | 否（节点级自保） | node-pressure eviction |
| **Node Lifecycle Eviction** | taint-manager | 节点 NotReady/Unreachable 后被标记 | 部分版本/路径有限支持 | 节点宕机重调度 |
| **API-Initiated Eviction** | 外部调用 Eviction API | 运维手动搬迁 | **是** | 节点排水（drain）/灰度腾空 |
| **Descheduler** | 独立组件 | 平衡已经运行起来的集群（去碎片、去重、去违反亲和） | 是（按策略） | 长期均衡 |

**必答一句**：**抢占是"为了调度新 Pod"，驱逐是"为了保节点/回收坏节点"，Descheduler 是"为了均衡"**，三者目标不同、不可替代。

---

## Q8. 打分是怎么算的？默认插件和权重是多少？

**公式**：`nodeScore = Σ (normalize(pluginScore_i(node)) × weight_i)`，每个插件分数先归一化到 `[0, 100]`，再按插件权重加权求和。

**默认 profile 大致如下**（**请用 **`kube-scheduler --write-config-to=dump.yaml`** 现场核对，官方默认插件集与权重会随版本调整**）：

| 插件 | 默认权重 | 作用 |
| --- | --- | --- |
| `NodeResourcesFit` | 1 | 默认 `LeastAllocated`（越空闲越优） |
| `NodeResourcesBalancedAllocation` | 1 | CPU/内存/GPU 使用比例是否均衡（避免单一资源打满） |
| `ImageLocality` | 1 | 本地已有镜像加分（拉镜像快） |
| `InterPodAffinity` | 1 | 满足 Pod 亲和/反亲和程度 |
| `NodeAffinity` | 1 | 节点亲和匹配度 |
| `PodTopologySpread` | 2 | 拓扑分布均匀性（默认给的权重较大） |
| `TaintToleration` | 3 | 节点上污点越少越好；能容忍的污点越多则分数略降 |
| `VolumeBinding` | 1 | 卷等待节点亲和与否、存储拓扑 |

**常考追问**：怎么让调度"更紧"？→ `NodeResourcesFit.scoringStrategy.type: MostAllocated` + `NodeResourcesBalancedAllocation` 加权（给大数据离线/弹性批量用）。但要注意：**在线延迟敏感业务用 MostAllocated 会造成热点叠加**，通常在线用 `LeastAllocated`、离线 Mixed 才上 MostAllocated。

---

## Q9. Snapshot（NodeInfo 快照）与 scheduler cache 的一致性怎么保证？

**结构**：

- `SchedulerCache`（thread-safe，由 informer 更新）维护 `Nodes map[string]*NodeInfo`，每个 `NodeInfo` 维护该节点上 Pods 的资源聚合；
- 每次调度周期拿到的是**快照**（快照期间不再变），保证同周期内 Filter/Score 看到一致的视图。

**不一致来源与自愈**：

| 场景 | 处理 |
| --- | --- |
| Assume 之后还没 Bind 成功 | pod 在 `assumedPods` 里带 TTL；超时/绑定失败 → `ForgetPod` + 重新入队 |
| 实际 Pod 已删除但 cache 未更新 | informer 的 Delete event 修正 |
| Bind 失败 | `Unreserve` 回滚 assume |
| apiserver watch 断连 relist | 重新 list 重建 cache |

**必答**：正因为是"乐观 + 最终一致"，才会出现 **"调度器觉得有资源，Kubelet 起来时却发现不够"**（尽管很少），以及必须依赖 **informer 的修正**而不能靠分布式锁。

---

## Q10. 亲和性三兄弟 + 污点 + 拓扑分布怎么选？

| 手段 | 粒度 | 是硬约束还是软约束 | 规模风险 | 典型用法 |
| --- | --- | --- | --- | --- |
| `nodeSelector` | 节点标签 | 硬 | 无 | 最简单过滤 |
| `nodeAffinity` | 节点标签 | 硬 (`requiredDuringScheduling`) / 软 (`preferred`) | 低 | 机型/zone/GPU 型号 |
| `podAffinity` | Pod 间 | 硬/软 | **中高**（要扫同 topology 的所有 Pod） | 缓存靠近、数据本地性 |
| `podAntiAffinity` | Pod 间 | 硬/软 | **高** | 单实例打散、避免同机故障域 |
| `topologySpreadConstraints` | 拓扑域 | 硬 (`DoNotSchedule`) / 软 (`ScheduleAnyway`) | **中高**（大规模算 scrutiny 收益差） | 跨 zone/机架均匀分布 |
| `taints & tolerations` | 节点 | 排斥/容忍 + NoSchedule/NoExecute | 低 | 专用节点池、隔离 GPU/ARM |

**强烈建议的实践**：优先用 `topologySpreadConstraints` 而不是 `podAntiAffinity` 做"打散"。原因：

- `podAntiAffinity` 只区分"有没有满足条件的 Pod"，是**布尔**判定，无法表达 skew；
- TopologySpread 可以表达"每个 zone 之间副本数差不超过 1"，语义精确且能软着落（`ScheduleAnyway` 保证不会因为打散而调度不出去）。

**可直接套用（推荐组合拳）**：

```yaml
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
          - matchExpressions:
              - key: topology.kubernetes.io/zone
                operator: In
                values: ["cn-north-1a", "cn-north-1b", "cn-north-1c"]
  topologySpreadConstraints:
    - maxSkew: 1
      topologyKey: topology.kubernetes.io/zone
      whenUnsatisfiable: ScheduleAnyway     # 宁可挤一点也不能调度不出去
      minDomains: 3                          # 保证至少覆盖 3 个 zone
      labelSelector:
        matchLabels:
          app: web
    - maxSkew: 1
      topologyKey: kubernetes.io/hostname
      whenUnsatisfiable: DoNotSchedule       # 单机最多 1 副本
      labelSelector:
        matchLabels:
          app: web
  tolerations:
    - key: dedicated
      operator: Equal
      value: online
      effect: NoSchedule
```

---

## Q11. VolumeBinding 插件与 `WaitForFirstConsumer` 为什么要存在？

**背景**：PVC 详情在下述 StorageClass 下会选择即刻分配 PV：

- `volumeBindingMode: Immediate` → PVC 一创建就 bind + 分配 PV，**PV 的拓扑（zone）可能与 Pod 最终落点不一致** → Pod Pending；
- `volumeBindingMode: WaitForFirstConsumer` → 延迟到 Pod 首次调度时才 bind，**让 PV 的 chosen zone 与调度结果一致**。

**调度器侧做了两件事**：

1. **Filter**：检查该节点能否满足 PV 的 `nodeAffinity`（`accessibleTopology`、CSI 写入的拓扑标签），过滤掉不可能节点；
2. **Reserve/PreBind**：真正触发 bind（`BindCSIStorage` 逻辑），并把结果写回 API Server。

**必答的坑**：

- WaitForFirstConsumer 让**调度需要考虑存储拓扑** ⇒ 节点更多/器件braska 拓扑复杂时，这一层会成为瓶颈之一（见大规模 L10）；
- 云盘一般只允许挂载到同 zone ⇒ 这是最常见的 "0/N nodes are available: N node(s) had volume node affinity conflict"。

---

## Q12. 为什么 requests 比 limits 更影响调度？

- 调度只看 `.spec.containers[].resources.requests`（以及 initContainers 的 max/update-strap gaps 取最大）；**limits 完全不参与调度**；
- CPU request → CFS shares（争抢时的权重）；CPU limit → CFS quota（硬上限，会造成 throttling）；
- Memory request → 调度占位 + QoS 分级；memory limit → OOMKill 阈值（超过即被杀）。

**QoS 与驱逐优先级**：Guaranteed（requests==limits，且 CPU/内存都设）\> Burstable \> BestEffort。**升级包: BestEffort Pod 最先被节点压力驱逐**。

**必答陷阱**：

- `requests` 不设 → Burstable/BestEffort，调度器认为**不用资源**，会造成节点过量售卖 + 后续 OOM 集中崩；
- CPU limit 设太低 → throttling 导致 P99 毛刺；
- 建议：CPU 的 request 按真实常驻设、limit 可不设（避免 throttling），内存的 request==limit（保稳定）。这是"经验建议"，需压测校准。

---

## Q13. PriorityClass / QoS / Quota 分别解决什么问题？

| 机制 | 解决 | 作用点 | 是否影响调度 |
| --- | --- | --- | --- |
| PriorityClass | 谁先调度 / 谁能抢谁 | 队列排序 + 抢占 | **是**（强影响） |
| QoS | 节点压力时谁先死 | kubelet | 否（但影响稳定性） |
| ResourceQuota | 命名空间能用多少总量 | admission | 否（admission 拒绝，pod 不会进队列） |
| LimitRange | 命名空间默认值/上下界 | admission | 否 |

---

## Q14. 多个调度器共存是怎么工作的？多 profile 与多调度器进程的区别？

**两种模式**：

```yaml
# 模式 A：一个 kube-scheduler 进程跑多个 profile（推荐）
apiVersion: kubescheduler.config.k8s.io/v1
kind: KubeSchedulerConfiguration
profiles:
  - schedulerName: default-scheduler      # 在线业务：均衡 + 严格
    plugins:
      score:
        enabled:
          - name: NodeResourcesFit
            weight: 1
  - schedulerName: batch-scheduler        # 离线批处理：装箱 + 快
    plugins:
      score:
        enabled:
          - name: NodeResourcesFit
            weight: 5
    pluginConfig:
      - name: NodeResourcesFit
        args:
          scoringStrategy:
            type: MostAllocated
```

```yaml
# Pod 侧选择（注意：schedulerName 不可变）
spec:
  schedulerName: batch-scheduler
```

| 对比 | 多 profile（同进程） | 多调度器进程 / 独立部署 |
| --- | --- | --- |
| 资源 | 共享同一进程，省内存 | 各自独立，隔离更好 |
| 队列 | **各自独立 ActiveQ** | 各自独立 |
| 故障域 | 一个进程挂全挂 | 隔离好（一个挂了不影响另一个） |
| 适用 | 绝大多数场景 | 极高隔离/独立演进诉求 |

**必答**：`spec.schedulerName`** 是不可变字段**，Pod 一旦创建就绑定了该名字；若没有对应调度器运行 → Pod 永远 Pending 且没人管（连 event 都没有，最难受的一类事故）。

---

## Q15. Scheduler Extender 为什么被废弃物这么多？

| 维度 | Extender（HTTP 外挂） | Framework 插件（编译/多 profile） |
| --- | --- | --- |
| 性能 | **每次调度都要 HTTP 交互，串行阻塞** | 进程内，无网络开销 |
| 可靠性 | extender 挂了 ⇒ 调度卡住/拒绝 | 同进程，故障域小 |
| 能力 | 只能做 Filter/Prioritize + 少量钩子 | 12+ 扩展点全支持 |
| 缓存 | 自己维护，易与 scheduler cache 不一致 | 共享 CycleState/框架状态 |

**面试口径**：Extender 已经不推荐作为主路径（社区倾向用框架插件或多 profile + 自定义插件），但在"已有系统难改动/跨语言"时仍可用，需要注意加**超时 + 降级 + 幂等**。

---

## Q16. kube-scheduler 如何做到高可用？双主会不会重复调度？

- 基于 `coordination.k8s.io/v1 Lease` 做 leader election（与 controller-manager 同一套机制），只有 leader 进程执行 `scheduleOne`；
- 默认 Lease/Renew/Retry（componentbaseconfig 常见默认 15s/10s/2s，**以运行版本为准**对应 `leaderElection` 字段）；
- 非 leader 进程不干事，只是等待抢锁。

```yaml
apiVersion: kubescheduler.config.k8s.io/v1
kind: KubeSchedulerConfiguration
leaderElection:
  leaderElect: true
  leaseDuration: 15s
  renewDeadline: 10s
  retryPeriod: 2s
  resourceLock: leases
  resourceName: kube-scheduler
  resourceNamespace: kube-system
```

**为什么必须单点 leader**：调度本质是对共享资源（全网 Node 资源视图）做决策，多主同时决策会产生**并发超卖**（两个 Pod 同时算账互相之间看不到对方 assume）。

**必答的异常**：

- Lease 抖动频繁 ⇒ 调度暂停窗口变长 ⇒ `scheduler_pending_pods` 突增 + metric 断更；
- 版本滚动升级时尤其要注意 election 参数一致性。

---

## Q17. Descheduler 的价值与常用策略？和调度器冲突吗？

调度器是"**一次性决策**"，不会回头。随着时间推移：节点标签变了、新节点加入、资源碎片多了 ⇒ 集群越来越不平衡，需要 Descheduler/重调度。

| 策略 | 作用 |
| --- | --- |
| `RemovePodsViolatingNodeAffinity` | 节点标签变了，Pod 不再满足 nodeAffinity |
| `RemovePodsViolatingInterPodAntiAffinity` | 违反 Pod 反亲和 |
| `RemovePodsViolatingTopologySpreadConstraint` | 违反拓扑分布（很重要，新增节点后常用） |
| `RemoveDuplicates` | 同 controller 下过多副本在同节点 |
| `LowNodeUtilization` | 节点利用率过低 ⇒ 迁出，便于缩容省钱 |
| `HighNodeUtilization` | 提高装箱密度（与 Evict 结合） |
| `RemovePodsHavingTooManyRestarts` | 老 CrashLoop Pod |
| `RemoveFailedPods`(aka RemovePodsViolatingPss/ExpiringPods) | 清理失败 Pod 等 |

**必答风险（这是高分点）**：

- Descheduler 通过 **Evict API** 驱逐，**严格遵守 PDB**，否则会引发可用性事故；
- 驱逐后由上层 controller 重建 ⇒ 会重新进入调度 ⇒ 要防止"**驱逐-重建-再驱逐**"的无限循环（对**无上层控制器管理的裸 Pod** 要特别小心，驱逐等于直接消失）；
- 必须配置 `--evict-local-storage-pods=false`、优先级门槛（`priorityThreshold`）避免驱逐系统组件。

---

## Q18. Gang Scheduling / Coscheduling 怎么做？为什么必须用 Permit 扩展点？

**背景**：AI 训练/大数据作业要求"要么全部 Pod 起来，要么一个都不要"；否则会**死锁占资源**（部分 Pod 起来了等不到同伴，占着 GPU 空转）。

**实现**：

- `Permit` 插件：当一组 Pod（通过 PodGroup CR 或 label 关联）达到满足条件的最小数量时才批准绑定；否则返回 `Wait` 让 Pod 挂起，并设超时回滚；
- `PostFilter`：超时/资源无法凑齐时，整组放弃（或抢占）；
- `QueueSort`：**必须**按 PodGroup 创建时间排序，避免"后来者先占资源"导致先来者饿死。

**必答的坑**：

- `Permit` 处于**绑定周期**，等待的 Pod 已经通过 Filter/Score 并 `Assume` 了资源 ⇒ **等待期间资源是被占住的**。这就是"占位等待"，需要配套超时和配额，否则变资源泄漏；
- 大规模下 coscheduling 的 wait 会显著抬升 `scheduler_pending_pods` 与绑定延迟。

---

## Q19. Kubernetes 原生支持多调度扩展点，那么"自定义调度器"有几种做法？

| 方案 | 复杂度 | 性能 | 维护成本 | 适用 |
| --- | --- | --- | --- | --- |
| 调 Redis 的参数/权重 + 多 profile | 低 | 好 | 低 | **首选** |
| 编译自定义 Framework 插件（in-tree 或 out-of-tree） | 中/高 | 最好 | 中 | 通用需求 |
| Scheduler Extender（HTTP） | 低 | 差 | 中 | 遗留/快速验证 |
| MutatingWebhook 改 nodeName（不推荐变相排 Secheduler） | 中 | 好 | 高（灾难） | 不建议 |
| 完全自研独立调度器 | 高 | 取决于实现 | 高 | 特殊领域（如 HPC） |

---

## Q20. 调度器如何降低对 API Server 的压力？

- **Pod Informer 带 field selector**：只 watch 未调度 Pod（`spec.nodeName` 为空），不加候选 Pod 的全量流；
- **Assume 乐观绑定**：减少等待 API Server 往返；
- **Bind 异步**：由独立 goroutine 执行绑定，不阻塞下一个 Pod 的调度周期；
- **client 侧限流**：
  ```yaml
  clientConnection:
    qps: 100      # 建议初值（源码常见默认 50/100 量级，请 --write-config-to 核对）
    burst: 150
  ```
- **事件侧**：大量 Pending 时会产生海量 `FailedScheduling` Event ⇒ 需限流/聚合，必要时调 `-v` 级别。

---

## Q21. Device Plugin / Extended Resource / NUMA 三者关系？

| 层 | 谁负责 | 关键字段 |
| --- | --- | --- |
| **上报资源量** | kubelet device plugin (`ListAndWatch`) | node.status.capacity/allocatable `nvidia.com/gpu: 8` |
| **调度** | scheduler `NodeResourcesFit`（只看数量） | `resources.limits["nvidia.com/gpu"]: 1` |
| **分配具体设备** | device plugin `Allocate()` 在 kubelet 侧 | 返回 envs/挂载/annotations |
| **亲和局部性** | kubelet **TopologyManager** + `CPUManager`（single-numa-node 策略） | `--topology-manager-policy` |
| **GPU 型号/拓扑标签暴露** | GPU Operator / NFD / Device plugin 自身或 DRA | 标签如 `nvidia.com/gpu.product` |

**必答认知**：\*\*原生调度器做 GPU 只认"数量"，不认"拓扑/型号细节"。\*\*想做 GPU 型号/连通感拓扑/显存共享，需要：

- 用 label/taint 做粗粒度隔离（NFD）；
- 或上 **DRA（Dynamic Resource Allocation）**；
- 或用离屏/第三方调度器（Koordinator、Volcano、HAMi 等）。

---

## Q22. DRA（Dynamic Resource Allocation）解决了 Device Plugin 什么痛点？

| 痛点（Device Plugin） | DRA 的解决方式 |
| --- | --- |
| 资源模型只有"整数计数"，无法表达"显存 40G / 特定型号 / 特定拓扑" | ResourceClaim + DeviceClass + ResourceSlice，**结构化参数** |
| 只有 kubelet 参与，调度时无法提前参与设备选择 | 调度器介入 ResourceClaim 分配（structured parameters 下），避免"调度完才发现不匹配" |
| 无法共享同一设备给多 Pod，也无法网络挂载设备 | 支持 shareable / 网络附加设备声明 |
| 资源发现与属性难以表达 | `ResourceSlice` 由 DRA driver 发布，带属性选择器 |

> **注意**：DRA 的 alpha→beta 演进较快，**是否已 GA、structured parameters 版本请以 Release Notes 为准**；面试时重点说清"它要解决的是表达力与调度介入，不是简单替代 device plugin"。

---

# Part B · 大规模集群篇（16 题）

> 定义：**大规模 = ≥ 1000 节点 / ≥ 15000 Pod**。以下每道题都按"现象 → 根因 → 量化方法 → 解法 → 验证"给。

## L0. 官方宣称的规模上限是什么？（必须先有基准）

Kubernetes 官方提出的扩展目标（**1.x 后期版本，以官方文档为准**）：

| 维度 | 上限 |
| --- | --- |
| 单节点 Pod 数 | 110 |
| Node 数 | 5000 |
| 集群总 Pod 数 | 150000 |
| 总容器数 | 300000 |

**SLO 口径（SIG Scalability 定义，大规模集群的性能验收）**：

- **Pod startup latency**（含镜像拉取与否两种口径）的分位数；
- **API Server 调用延迟**；
- 调度侧：Scheduler 使用 `scheduler_pod_scheduling_sli_duration_seconds` 作为 SLI（**该统计过滤掉多轮重试/抢占路径的 attempt，口径以版本为准**）。

> **重要提醒**：本文档**不提供"某集群实测 X pods/s"这种数字**——它强依赖节点规模、插件集、亲和约束复杂度、存储拓扑。**你的答案应该是：给出测量方法 + 给出你自己的实测数字。**

**如何压测（必须会答）**：社区做法是用 **Kubemark（hollow node）** 或 **KWOK** 模拟 N 千节点 + clusterloader2 / scheduler-perf 套件，测 `PodScheduled` 吞吐。

```bash
# 1. 起一个 fake/kwok 集群并注入 5000 个 hollow node
# 2. 跑 scheduler-perf 套件（perf-tests/scheduler），产出 metrics 耗时分布
# 3. 关注：scheduler_pod_scheduling_duration_seconds 的 P50/P90/P99 +
#    scheduler_framework_plugin_duration_seconds 按 plugin 分解
```

---

## L1. 大规模集群调度吞吐的瓶颈到底在哪？（必考综述题）

**从数据流上拆，瓶颈永远落在这 5 个位置**：

| # | 瓶颈点 | 量化指标 | 典型解法 |
| --- | --- | --- | --- |
| 1 | **单 profile 串行调度** | `scheduler_pod_scheduling_duration_seconds` 随 QPS 线性抬升 | 多 profile 拆流量 / 多进程拆分 / 业务独立集群 |
| 2 | **节点遍历成本 O(N)** | `scheduler_framework_plugin_duration_seconds{extension_point="Filter"}` | `percentageOfNodesToScore`；节点池化（taint + nodeAffinity 缩小候选集） |
| 3 | **亲和/Spread 计算 O(Pods)** | `scheduler_framework_plugin_duration_seconds{plugin="InterPodAffinity"}` | 限制 `topologyKey` 粒度、限制 labelSelector 基数、避免大量 Pod 在同一 domain |
| 4 | **存储拓扑参与调度** | `scheduler_framework_plugin_duration_seconds{plugin="VolumeBinding"}` | 减少涉及 WaitForFirstConsumer 的 Pod 比例 |
| 5 | **API Server 交互** | `apiserver_request_duration_seconds`、`scheduler` 侧 client 限流错误 | client qps/burst 调优、降低 Event 量、apiserver 扩容 |

**标准答法**：

> "大规模下调度器不是单一瓶颈，而是分层的。我会先看插件级耗时分解（`scheduler_framework_plugin_duration_seconds` 按 plugin/extension\_point），找出 Top 2 的插件，再决定是缩候选节点集（降低 N）还是简化约束（降低每节点成本）。这两条路径是乘法关系。"

---

## L2. 节点池化 / 隔离为什么是解决大规模调度的第一银弹？

**原理**：`percentageOfNodesToScore` 是按"全部节点数"算比例的。**如果每个 Pod 通过 **`nodeAffinity`**/**`taints`** 先把候选集缩到 100\~****200 个节点，Filter 的成本就是常数级**，远强于调比例。

```
全集群 5000 节点
   ├── online-pool    (2000 nodes)  taint: dedicated=online:NoSchedule
   ├── gpu-pool       (300 nodes)   taint: nvidia.com/gpu=present:NoSchedule
   ├── arm-pool       (500 nodes)   label: kubernetes.io/arch=arm64
   └── batch-pool     (2200 nodes)  taint: workload=batch:NoSchedule
```

**配套做法**：

- 给业务 Pod 显式 `tolerations` + `nodeAffinity`；
- 给敏感/专用机型加 taint，防止普通 Pod 误调度（`NoSchedule` 是"防呆"）；
- 用 `resourceQuota` / `priorityClass` 做租户配额。

**验证指标**：改前后对比同一业务队列的 Filter P99 与整体 `scheduler_pod_scheduling_duration_seconds` P99。

---

## L3. 集群事件批量触发导致的"重调度风暴"怎么办？

**现象**：集群批量加节点 / CSInode 更新 / 大量 PVC 变更 ⇒ UnschedulableQ 里上万 Pod 一次性搬回 ActiveQ ⇒ `scheduler_pending_pods{queue="active"}` 暴涨，CPU 打满，正常 Pod 排队变慢。

**根因**：`MoveAllToActiveOrBackoffQueue` 是粗粒度搬移 + 大量 Pod 依赖同一批事件。

**解法组合**：

1. **开启 QueueingHint**（较新版本）：让"事件 → 重新入队"精细化，只有真正可能被救活的 Pod 才搬；
2. **调大退避参数**：确认抖动严重时，可适当加大退避上限（建议把 `podMaxBackoffSeconds` 从 10 调到 20\~30 观察），给集群自愈留窗口；
3. **按业务拆分 profile**：让离线风暴不要污染在线队列；
4. **不要一次性创建海量 Pod**：上层控制器加分批发放（batch sizing / Kueue quota 门控）；
5. **控制 Pending 总量**：给离线 Namespace 加 ResourceQuota，天然封顶。

**告警（可落地）**：

```promql
# 活跃队列堆积（阈值需按集群容量实测校准）
sum(scheduler_pending_pods{queue="active"}) > 500

# backoff 堆积，说明大量 Pod 在反复失败重试
sum(scheduler_pending_pods{queue="backoff"}) > 200

# 调度失败率（5m 窗口）
sum(rate(scheduler_schedule_attempts_total{result="unschedulable"}[5m]))
  / clamp_min(sum(rate(scheduler_schedule_attempts_total[5m])), 0.001) > 0.1
```

---

## L4. 抢占风暴怎么防？（大规模经典反模式）

**现象**：夜里批处理高峰，大量高优 Pod 抢低优 ⇒ `scheduler_preemption_victims` 飙升、在线 Available 抖动、API Server 压力涨。

**根因**：无节制使用高优先级 + `preemptionPolicy: PreemptLowerPriority` 滥用。

**解法（按优先级）**：

1. **默认 PriorityClass 设为 **`preemptionPolicy: Never`（强烈推荐），只有极少数真正核心业务才开抢占；
2. **给 victim 侧配 PDB**，让抢占无法突破可用性底线；
3. **离线队列用 ResourceQuota 封顶**，从源头减少"需要抢"的场景；
4. **错峰 + 弹性扩队长 Len Cleveland**：离线高峰时先扩 nodepool（Cluster Autoscaler / Karpenter），而不是抢在线；
5. **在线/离线分节点池**（taint 隔离），把抢占变成"物理隔离"而非"频率抢占"。

```promql
# 抢占活跃度告警（>0 就应该被人群 Review）
sum(rate(scheduler_preemption_attempts_total[5m])) > 0.1
sum(rate(scheduler_preemption_victims[5m])) > 0
```

---

## L5. Pending Pod 排障的标准路径（面试必背）

```
1. kubectl get pod -A --field-selector=status.phase=Pending -o wide | head
2. kubectl describe pod X        # 看 Events：FailedScheduling 的 reason 是关键
     常见原因串：
       "0/N nodes are available: N Insufficient cpu."          → 资源不够
       "N node(s) had untolerated taint {key: value}"          → 污点
       "N node(s) didn't match Pod's node affinity/selector"   → 亲和/标签
       "N node(s) had volume node affinity conflict"           → 存储拓扑
       "N node(s) didn't match pod anti-affinity rules"        → 反亲和
       "N Insufficient nvidia.com/gpu"                         → 扩展资源
3. 若 Pod 完全没有任何 Event → 极可能是：
       a) spec.schedulerName 写错 / 没有对应调度器在跑（最高频！）
       b) 调度器进程挂了或 lease 争抢失败
       c) 变更的选择器/field selector 出错导致 Pod 没进 informer
4. 查调度器进程：kubectl -n kube-system get lease kube-scheduler
5. 查 metrics：scheduler_pending_pods / scheduler_schedule_attempts_total
6. 必要时开 -v=5 看单个 Pod 的决策日志（临时操作，注意恢复日志级别）
```

> **金牌追问**：为什么有的 Pod "Pending 但没有事件"？答：`schedulerName` 错误/调度器未运行是最常见原因，因为**没有任何组件认领它**，自然就没有 event。

---

## L6. 大规模下 FailedScheduling Event 洪水怎么治？

**现象**：上万 Pending Pod 持续失败 ⇒ 调度器疯狂写 Event ⇒ API Server `apiserver_current_inflight_requests` 打满，`apiserver_request_duration_seconds{verb="create",resource="events"}` 恶化 ⇒ 连带影响其它组件。

**治理清单**：

| 手段 | 做法 |
| --- | --- |
| **Event 限流** | 调整调度器 client qps/burst；给 Event 单独的 client 参数（版本支持时） |
| **日志降级** | 恢复 `-v=2`（不要把 `-v=5` 常开） |
| **根因减量** | 治理 pending 本身（批处理配额、弹性扩容、调度整形） |
| **Event 保留策略** | apiserver `--event-ttl`（注意版本差异）、避免 Event 表膨胀影响 etcd |
| **新版本特性** | 部分版本引入 FailedScheduling 事件**聚合**能力（门控名/可用性以所用版本 Release Notes 为准），升级后可大幅降低 Event 数 |

**告警**：

```promql
sum(rate(apiserver_request_total{verb="create",resource="events",code=~"2.."}[5m])) > 500
```

---

## L7. 资源碎片导致"总量够但调度不出去"怎么办？

**现象**：`Free CPU: 200 cores`，但一个新的 16C Pod 就是调度不出去（每个节点余 4\~8C）。

**根因**：历史上用 LeastAllocated 打分 + 无装箱 ⇒ 资源被打散。

**解法**：

1. **打分策略改为 **`MostAllocated`（针对离线/批处理 profile），让新 Pod 优先填坑；
2. 开启/加权 `NodeResourcesBalancedAllocation`，避免 CPU 用完内存剩一堆（反之亦然）；
3. **Requests 治理**：治理"申请虚高"（requests 远大于实际用量），这是碎片的根本来源 —— 用量画像（VPA recommendation / Prometheus 实测）驱动 requests 收敛；
4. **引入 Descheduler**（`HighNodeUtilization` / `LowNodeUtilization`）做二次整理（严格守 PDB）；
5. **规格标准化**：限制 node 机型数量、限制 Pod 规格档位（如只允许 2C/4G、4C/8G、8C/16G、16C/32G），碎片概率大幅下降。

**验证**：用分配率指标看改善：

```promql
# 集群 CPU 分配率（request / allocatable）
sum(kube_pod_container_resource_requests{resource="cpu"})
  / sum(kube_node_status_allocatable{resource="cpu"}) > 0.75

# 但"碎片"更要看：无法满足某规格的节点占比（需用自定义计算）
```

---

## L8. 一个 5000 节点集群 vs 5 个 1000 节点集群，怎么选？

| 维度 | 单大集群 | 多小集群 |
| --- | --- | --- |
| 资源利用率 | 高（池化效应，削峰补谷） | 低（需预留 buffer） |
| 故障爆炸半径 | **大**（一次故障全站） | 小 |
| 运维复杂度 | 低（一套控制面） | **高**（多套升级/配置漂移） |
| etcd/API 压力 | 大（需要专职调优） | 分摊 |
| 成本 | 低（overhead 少） | 高 |

**标准答法**：

> "不要为了分区而分区。分区的真正理由是**故障域隔离与变更隔离**，而不是性能。当出现以下信号才考虑拆：控制面单点故障会造成跨业务不可用、升级必须全局暂停、不同业务 SLO 差异极大（延迟敏感 vs 批处理）、或租户隔离合规要求。性能问题优先用节点池化 + profile 拆分 + 参数调优解决。"

---

## L9. 大规模 GPU / AI 场景的调度难点？

| 难点 | 现象 | 解法 |
| --- | --- | --- |
| GPU 是"整块"资源 | 一张卡只能给一个 Pod（原生），利用率低（\<30%） | 分片方案（MIG / 时间片共享厂商方案 / DRA）；实则要在 device plugin/第三方层实现 |
| Gang Scheduling | 部分起来 ⇒ 死锁占卡 | coscheduling（Permit 插件）+ PodGroup + Kueue/Voice/Volcano |
| 拓扑亲和 | NVLink/NVSwitch 亲和，跨卡通信慢 | label/NFD + nodeAffinity；或用专业的网络拓扑感知调度 |
| 抢占 | 高优训练任务等不到卡 | 优先级 + 合理 Quota；谨慎用抢占（训练中断成本极高） |
| Extender 性能 | 自研调度逻辑 O(N) 串行 HTTP | 改为 in-tree/out-of-tree framework 插件 |

---

## L10. 弹性扩缩节点时调度侧会踩什么坑？

**场景**：CA/Karpenter 扩容 1000 节点 ⇒ 大量 Pod 立即被调度到新节点，但节点 CNI/CSI 还没就绪 ⇒ Pod `ContainerCreating` 长时间 / `FailedMount` / `ImagePullBackOff` / CNI not ready。

**解法**：

1. 依赖 `node.kubernetes.io/not-ready`** / **`network-unavailable`** / cloud provider uninitialized taint** 天然防呆（scheduler 不会往 NoSchedule 污点节点调）；
2. **启动污点**（如 `node.cilium.io/agent-not-ready`、`node.kubernetes.io/network-unavailable`）由 CNI 就绪后移除；
3. 关键业务加 `minReadySeconds` / readinessGate 防"节点起来就接流"；
4. **弹性节奏控制**：客户端侧的批次发放避免瞬时打爆调度器和 API Server；
5. 给新节点预留 / 扩容预热（提前创建 Warm Pool）。

**收缩反向问题**：缩容时大量 Pod 同时被驱逐 ⇒ 瞬时 ΔQPS 爆发 ⇒ 同时进入调度队列，需要注意 PDB + 分批。

---

## L11. 大规模下 Kubernetes Control Plane 与 Scheduler 的相互伤害？

- **Scheduler watch 未调度 Pod**：大批 Pod 瞬时创建 ⇒ informer 事件流爆发 ⇒ 调度器内存/CPU 涨；
- **Scheduler 写 Event / Bind**：反向给 API Server 增压；
- **etcd 延迟上涨** ⇒ informer relist / resourceVersion 过期 "too old resource version" ⇒ scheduler cache 需要重建 ⇒ 调度暂停。

**解法**：

- 拆分 informer（多 profile 会各自 habere 一份 pod informer，注意内存随 profile 数涨）；
- 控制 apiserver `--max-requests-inflight` / Priority & Fairness (APF) 给 scheduler 分配专用 FlowSchema，**保障调度器不被其它组件饿死**；
- etcd 独立盘 + 监控 fsync 延迟（`etcd_disk_wal_fsync_duration_seconds` P99）。

```yaml
# APF 给调度器保障配额（示意，字段以版本为准）
apiVersion: flowcontrol.apiserver.k8s.io/v1
kind: PriorityLevelConfiguration
metadata:
  name: system-scheduler
spec:
  type: Limited
  limited:
    assuredConcurrencyShares: 100
    limitResponse:
      type: Queue
      queuing:
        queueLengthLimit: 500
```

---

## L12. 多租户 / 多个产品共用集群怎么防止互相影響？

组合拳：

1. **节点池 + taint**（硬隔离）；
2. **profile 拆分**（策略隔离：在线/离线不同打分）；
3. **ResourceQuota**（总量封顶）+ **LimitRange**（默认 requests，防 Pod 开 0 request）；
4. **PriorityClass 分层** 且默认层 `preemptionPolicy: Never`；
5. **Priority & Fairness** 保证调度器的 apiserver 配额。

```yaml
# 防呆：给每个 NS 注入默认 requests（否则业务创建 0 request 的 Pod 会引发过度调度 + OOM）
apiVersion: v1
kind: LimitRange
metadata:
  name: defaults
  namespace: tenant-a
spec:
  limits:
    - type: Container
      defaultRequest:
        cpu: 500m
        memory: 512Mi
      default:
        cpu: "2"
        memory: 2Gi
      max:
        cpu: "32"
        memory: 64Gi
```

---

## L13. 怎么判断"scheduler 本身慢"还是"apiserver/etcd 慢"？

**三步定位法**：

```promql
# ① 调度器自身：插件级耗时分解（若是这里高 → 是 scheduler 侧算法/规模问题）
histogram_quantile(0.99,
  sum by (plugin, extension_point, le) (
    rate(scheduler_framework_plugin_duration_seconds_bucket[5m])
  )
)

# ② 到 apiserver 的往返（若这里高 → 是 apiserver/etcd 或客户端限流）
histogram_quantile(0.99,
  sum by (verb, resource, le) (
    rate(apiserver_request_duration_seconds_bucket[5m])
  )
)

# ③ etcd 自身（若这里高 → 根子在 etcd）
histogram_quantile(0.99,
  rate(etcd_request_duration_seconds_bucket[5m])
)
histogram_quantile(0.99, rate(etcd_disk_wal_fsync_duration_seconds_bucket[5m]))
```

**判定表**：

| 现象 | 结论 |
| --- | --- |
| ① 高、②③ 正常 | scheduler 算法/规模瓶颈 → 调 percentageOfNodesToScore、简化约束、节点池化 |
| ①②③ 都高 | etcd 慢，级联到 apiserver 再到 scheduler → 先治 etcd（磁盘/压缩/对象数） |
| ① 正常、② 高、③ 正常 | apiserver 侧：对象数过多 / Event 洪水 / 限流分流 → 看 APF、看 verb=create events |

---

## L14. 亲和/反亲和在大规模下为什么是性能杀手？怎么优化？

**成本模型**：

- `podAffinity/podAntiAffinity` 需要对每个候选节点，**遍历该 topology domain 下所有匹配 labelSelector 的 Pod**；
- 若 topologyKey=hostname 且 Pod 很多 ⇒ 每次要聚合全集群 Pod ⇒ 与 Pod 总数近似线性。

**社区优化**（答出来很加分）：

- PreFilter 阶段预先计算 matching pod 列表 / topology pairs，避免每个节点重复扫；
- 限制同一拓扑域内的候选数量；
- `topologyKey` 不要乱选 hostname（除非必要），用更粗的 topologyKey 能减少聚合开销。

**治理建议**：

1. **优先用 **`topologySpreadConstraints`** 取代 **`podAntiAffinity` 表达打散；
2. labelSelector **基数要小**（不要用它匹配全量 Pod）；
3. 周期性检查 spike：`scheduler_framework_plugin_duration_seconds{plugin=~"InterPodAffinity|PodTopologySpread"}` P99。

---

## L15. 调度策略变更如何灰度与回滚？（生产变更治理）

**任何调度器配置变更都必须走这条路**：

| 阶段 | 灰度比例 | 观察指标（≥ 30min） | 回滚触发条件 |
| --- | --- | --- | --- |
| Step 1 | **1%** Pod（通过独立 `schedulerName` 灰度，而不是直接改默认调度器） | P99 调度时延、失败率、Pending 数 | 失败率 \> 基线 +2pp，或 P99 \> 基线 ×1.5 |
| Step 2 | 10% | + 业务 P99 延迟、错误率 | 同上，或任一业务 SLO 告警触发 |
| Step 3 | 50% | 全量 SLO Dashboard | 同上 |
| Step 4 | 100% | 稳定 ≥ 24h | — |

**回滚要点**：

- **优先"改回 schedulerName"**（把 Pod 切回 default-scheduler），而不是回滚二进制；这是秒级、幂等的；
- 配置用 ConfigMap/文件 + 版本化管理，回滚就是切回上一个版本文件；
- *注意*：回滚旧 scheduling 配置**不会重新调度**已经在跑的 Pod ⇒ 已受影响的 Pod 需要靠 Descheduler 或人工滚动重启才能回到期望分布（这点必须在方案里写明）。

**示例：通过新 profile 灰度**

```yaml
# 1) 先在同一个 kube-scheduler 里加一个灰度 profile
apiVersion: kubescheduler.config.k8s.io/v1
kind: KubeSchedulerConfiguration
profiles:
  - schedulerName: default-scheduler        # 老策略，保持不变
  - schedulerName: default-scheduler-canary # 新策略
    plugins:
      score:
        enabled:
          - name: NodeResourcesFit
            weight: 2
    pluginConfig:
      - name: NodeResourcesFit
        args:
          scoringStrategy:
            type: MostAllocated
            resources:
              - {name: cpu, weight: 1}
              - {name: memory, weight: 1}
```

```bash
# 2) 把 1% 的 Deployment 切进去（用 patch，可观察、可秒回）
kubectl -n app patch deploy web -p '{"spec":{"template":{"spec":{"schedulerName":"default-scheduler-canary"}}}}'
# 3) 观察 30min ... 出问题：
kubectl -n app patch deploy web -p '{"spec":{"template":{"spec":{"schedulerName":"default-scheduler"}}}}'
```

---

## L16. 给你一个新集群，怎么设计其"调度治理"方案？（综合题/架构题）

按这个顺序答，面试官会认为你干过：

1. **先问 SLO**：业务的可用性目标是多少？P99 延迟目标？成本预算约束？（**不先问 SLO 就谈调度的都是耍流氓**）
2. **节点池化设计**：按机型/业务线/租户划 pool（taint + label），天然缩小调度候选集。
3. **优先级体系**：3\~5 个 PriorityClass（system \> prod \> default \> batch-low），默认层 `preemptionPolicy: Never`。
4. **requests/limits 治理**：LimitRange 给默认值；定期用实测用量做 requests 校准（VPA recommendation / Prometheus）。
5. **多 profile**：在线用 LeastAllocated + 均衡；离线用 MostAllocated + 装箱。
6. **可观测性**：scheduler metrics + 关键告警（Pending、失败率、P99、抢占、插件耗时 Top）。
7. **混沌演练**：定期 GameDay —— 杀调度器、模拟节点宕机、模拟 Pod 洪峰，验证 SLO 与 MTTR。
8. **变更治理**：灰度/回滚流程 + Descheduler 兜底。

---

# Part C · 反模式清单（SRE 视角，可直接用于生产巡检）

> 每条含：反模式 / 证据 / 风险等级 / 修复动作 / 验证方式

| # | 反模式 | 证据（怎么发现） | 等级 | 修复动作 | 验证方式 |
| --- | --- | --- | --- | --- | --- |
| C1 | 容器不设 requests / requests 远低于实际用量 | `kube_pod_container_resource_requests` vs 实际用量；节点 OOM 频繁 | 🔴 高 | LimitRange 注入默认；VPA recommend 校准 | 节点 OOMKill 数下降；`kube_pod_container_status_terminated_reason{reason="OOMKilled"}` 归零 |
| C2 | 只用 `podAntiAffinity` 做打散，且 labelSelector 基数巨大 | `scheduler_framework_plugin_duration_seconds{plugin="InterPodAffinity"}` P99 高 | 🔴 高 | 改用 `topologySpreadConstraints`；收敛 selector | 该插件 P99 下降 ≥ 30%（实测） |
| C3 | 默认 PriorityClass 允许抢占 | 存在多个 value \> 0 且 `preemptionPolicy` 非 Never 的默认类 | 🔴 高 | 默认层设 `preemptionPolicy: Never` | `scheduler_preemption_victims` 速率下降 |
| C4 | scheduleName 拼错 / 无对应调度器 | Pod Pending 且**无任何 Event** | 🟠 中 | 校验 webhook 检查 schedulerName 是否在白名单；巡检脚本 | 巡检脚本输出异常 Pod 数为 0 |
| C5 | 生产长期开 `-v=5` 调试日志 | scheduler 日志量巨大、CPU 高 | 🟠 中 | 恢复 `-v=2`；按需临时打开 | CPU 使用率回落；日志量下降 |
| C6 | 一个超大 RuntimeClass/Pool 无任何隔离 | `kube_node_info` 5000 节点单一 labels | 🟠 中 | 池化 + taint | Filter P99 下降 |
| C7 | 用 Scheduler Extender 做核心逻辑 | extender 依赖 + kube-scheduler 超时日志 | 🟠 中 | 迁移为 framework 插件 | 无 HTTP 调用；错误率归零 |
| C8 | Descheduler 未配 PDB / priorityThreshold | 驱逐关键 Pod、业务抖动 | 🔴 高 | 配 evict 策略 threshold + PDB | 演练：驱逐时段业务可用性不跌破 SLO |
| C9 | 集群无调度 SLO 与告警 | 无 PromQL / 无 Dashboard | 🟠 中 | 建立 Part D 的告警集 | 演练：注入 Pending 洪峰，告警 5min 内触发 |
| C10 | 一次性创建上万 Pod（无批次控制） | apiserver inflight 打满 + scheduler CPU 满 | 🟠 中 | 上层控制器分批 + Quota 封顶 | 创建速率平稳，无 QoS 挤兑 |
| C11 | MostAllocated 用于在线延迟敏感业务 | 同节点多实例叠加，P99 恶化 | 🟠 中 | 在线 profile 用 LeastAllocated | 业务 P99 回到基线 |
| C12 | 不监控 scheduler lease | 调度暂停无告警 | 🔴 高 | 加 lease 存活/频率告警 | 模拟抢主，告警触发 |

---

# Part D · 可观测性：SLO、指标与告警（可直接落地）

## D1. 建议的调度 SLO（**初始值，必须按业务实测校准**）

| SLI | 建议目标（起点） | 计算方式 |
| --- | --- | --- |
| 调度成功率（5min 窗口） | ≥ 99.9% | `1 - rate(result="unschedulable") / rate(total)` |
| P99 调度时延（首调度路径） | ≤ 2s | `histogram_quantile(0.99, scheduler_pod_scheduling_sli_duration_seconds_bucket)` |
| Pending \> 5min 的 Pod 占比 | ≤ 0.1% | 需自定义（Pod annotation 时间戳或审计） |
| 抢占事件频率 | 在线 profile ≈ 0 | `rate(scheduler_preemption_attempts_total[5m])` |

> 错误预算：以 30 天滚动窗口，消耗速率 \> 14.4×/6×（Google SRE burn rate）分别触发紧急/工单告警。

## D2. 核心指标清单（先 `curl :10259/metrics` 核对指标名）

| 指标 | 含义 | 用于 |
| --- | --- | --- |
| `scheduler_pending_pods{queue=...}` | 各级队列长度 | 堆积、风暴 |
| `scheduler_pod_scheduling_duration_seconds{attempts}` | 端到端调度耗时 | P99 时延、attempts 分层分析 |
| `scheduler_pod_scheduling_sli_duration_seconds` | SLI 口径耗时（排除复杂重试路径，口径以版本为准） | SLO 度量 |
| `scheduler_schedule_attempts_total{profile,result}` | 结果分布 | 成功率 |
| `scheduler_framework_plugin_duration_seconds{plugin,extension_point,status}` | **插件级耗时**（最有价值的分解指标） | 性能瓶颈定位 |
| `scheduler_preemption_attempts_total` / `scheduler_preemption_victims` | 抢占活动 | 抢占风暴 |
| `scheduler_queue_incoming_pods_total{event}` | 入队来源 | 队列抖动 |
| `scheduler_scheduler_cache_size{type}` | cache 对象数 | 内存/cahce 泄漏 |
| `scheduler_goroutines` | goroutine 数 | 泄漏/卡住 |

## D3. Prometheus 告警规则（可直接粘贴，阈值请实测校准）

```yaml
groups:
  - name: kube-scheduler
    rules:
      - alert: SchedulerNoMetrics
        expr: absent(up{job="kube-scheduler"} == 1)
        for: 3m
        labels: {severity: critical}
        annotations:
          summary: "kube-scheduler 指标消失，可能进程挂掉或 lease 争抢失败"

      - alert: SchedulerPendingPodsHigh
        expr: sum(scheduler_pending_pods{queue="active"}) > 500
        for: 10m
        labels: {severity: warning}
        annotations:
          summary: "调度活跃队列堆积 {{ $value }}，检查资源/亲和/配额"

      - alert: SchedulerUnschedulableRateHigh
        expr: |
          sum(rate(scheduler_schedule_attempts_total{result="unschedulable"}[5m]))
            / clamp_min(sum(rate(scheduler_schedule_attempts_total[5m])), 0.001) > 0.1
        for: 10m
        labels: {severity: critical}
        annotations:
          summary: "5 分钟内调度失败率 {{ $value | humanizePercentage }}"

      - alert: SchedulerSchedulingLatencyP99
        expr: |
          histogram_quantile(0.99,
            sum(rate(scheduler_pod_scheduling_duration_seconds_bucket[5m])) by (le)) > 5
        for: 10m
        labels: {severity: warning}
        annotations:
          summary: "P99 调度时延 {{ $value }}s"

      - alert: SchedulerPreemptionActive
        expr: sum(rate(scheduler_preemption_attempts_total[5m])) > 0.1
        for: 5m
        labels: {severity: warning}
        annotations:
          summary: "调度器正在频繁抢占，检查是否出现抢占风暴"

      - alert: SchedulerCacheTooLarge
        expr: scheduler_scheduler_cache_size{type="pods"} > 200000
        for: 15m
        labels: {severity: info}
        annotations:
          summary: "scheduler cache 中 Pod 数异常，检查是否有 assumed pod 泄漏"
```

> **10259 端口是 HTTPS**，抓取需要 `scheme: https` + `insecureSkipVerify` + bearer token（或 kubeconfig RBAC），CA/SAN 处理务必放到经审批的方案里，别口头 monkey-patch。

---

# Part E · 演练（GameDay）计划表

| 场景 | 注入方式 | 期望表现 | 验证的 SLO |
| --- | --- | --- | --- |
| 调度器主失联 | 删除 scheduler Pod / 强制抢 lease | 30s 内新主接管，Pending 短暂上涨后回落 | 调度可用性 |
| Pod 洪峰 | 一次性创建 3000 Pod | 无 unschedulable 溢出、apiserver 无 5xx | 调度成功率、时延 |
| 节点宕机 | Stop 一个 100 Pod 的节点 | 5min 内重调度完成，业务 P99 不破线 | 可用性 + MTTR |
| 资源碎片 | 人为造成每节点仅剩 4C 空闲 | 16C 的新 Pod 要么调度失败率超过阈值、要么被 MostAllocated 策略正确装箱；二者必有其一且结论可解释 | 装箱策略有效性 |
| 存储拓扑冲突 | 跨 zone PVC + Pod | 调度到正确 zone（或给出明确事件） | 存储亲和正确性 |
| 抢占演练 | 注入高优 Pod | Victim 被优雅终止、PDB 不被突破 | 抢占安全性 |

---

# Part F · 10 张面试速记卡（临阵磨枪）

1. **一次调度 = NextPod → 快照 → PreFilter → Filter(并发) → \[****PostFilter 抢占\] → Score → 随机选最高分 → assume(Reserve) → Permit → 异步 Bind**。每 profile 一个 goroutine 串行。
2. **扩展点 13 个**：PreEnqueue / QueueSort / PreFilter / Filter / PostFilter / PreScore / Score / NormalizeScore / Reserve / Permit / PreBind / Bind / PostBind（+ Unreserve 回滚）。
3. **队列三级**：ActiveQ / BackoffQ / UnschedulableQ；UnschedulableQ 靠集群事件 + `podMaxInUnschedulablePodsDuration` 兜底重排。
4. **抢占**：只对"Filter 全失败"生效；`nominatedNodeName` 是预约不是绑定；PDB 会压制；默认优先级应 `preemptionPolicy: Never`。
5. **调度只看 requests，不看 limits**。limits 是运行时的事。
6. **ResourceQuota 是 admission 的事，不是 scheduler 的事。**
7. **打散用 TopologySpread，不用 podAntiAffinity**；`maxSkew` + `whenUnschedulable` + `minDomains` 三件套。
8. **大规模第一银弹 = 节点池化**（缩小 N），第二才是 `percentageOfNodesToScore`（牺牲最优性）。
9. **排障第一问**：有没有 Event？没有 ⇒ 先看 `spec.schedulerName` 和调度器进程活着没。
10. **所有性能数字必须自己测**：用 Kubemark/KWOK + scheduler-perf，按 `scheduler_framework_plugin_duration_seconds` 分解找瓶颈。

---

## 附：文档自查清单

- [ ] 每条建议都给了可直接套用的配置片段或 PromQL
- [ ] 所有版本号/默认值均标注"以所用版本源码或 Release Notes 为准"
- [ ] 未提供任何未经实测的性能数字，全部给出**测量方法**
- [ ] 反模式清单含：证据 / 等级 / 修复 / 验证
- [ ] SLO 建议标注为"初始值，需实测校准"，并给出错误预算口径
- [ ] 生产变更给出灰度比例（1%→10%→50%→100%）、回滚触发条件与回滚操作
