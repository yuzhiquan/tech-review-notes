# k8s-kubelet-interview-qa

# Kubernetes Kubelet 面试题全解析：原理篇 × 大规模集群篇

> 适用角色：SRE / 云原生运维 / 平台工程师面试（社招 3 年起 + 偏高阶）
> 姊妹篇：《Kubernetes Scheduler 面试题全解析》
> 版本约定：以 **1.28 \~**** 1.31 主线**为准；涉及默认值、feature gate、迁移状态的地方，均标注"以所用版本 kubelet --help / KubeletConfiguration 文档 / Release Notes 为准"。

---

## 0. 使用说明

### 0.1 kubelet 面试题的考察层级

| 层级 | 考点 | 表现 |
| --- | --- | --- |
| L1 会用 | 会 `kubectl logs/exec`，知道三种探针 | 只会 describe |
| L2 懂原理 | 说得清 SyncLoop / PLEG / CRI / CNI / CSI / EvictionManager / cgroup 分层 | 能画节点内架构图 |
| L3 有规模感 | 知道高密度节点下 PLEG、镜像拉取、心跳、驱逐谁先崩 | 能报 metric 名 |
| L4 有事故经验 | 能讲清"Pod 卡 ContainerCreating / 优雅下线 502 / 节点失联 5 分钟才驱逐"的根因与治理 | 有 PromQL、有灰度、有回滚 |

> **答题模板**：结论 → 组件/数据流 → 给证据（配置项 / metric 名 / 事件字段）→ 补坑或算时间账。

### 0.2 数值约定

- **源码/官方默认值**：标注字段与出处，提示用 `kubelet --help | grep <field>` 或 `/var/lib/kubelet/config.yaml` 核对；
- **建议值**：标为 *建议*，必须实测校准；
- **实测值**：本文档不提供任何未经实测的性能数字，一律给出**测量方法**。

---

# Part A · 原理篇（24 题）

## Q1. kubelet 负责什么？和 scheduler / controller-manager 的边界在哪？

**一句话**：kubelet 是**节点代理**——它把 API Server 里"应该在本节点运行的 Pod"翻译成真实的容器、网络、存储、资源限制，并持续汇报节点与 Pod 状态。

| 事项 | kubelet | 说明 |
| --- | --- | --- |
| 决定是否在本节点跑 | ❌（scheduler 决定 nodeName） | kubelet 只认账不管分配 |
| 创建/启动/停止容器 | ✅ | 通过 CRI |
| 挂载卷 / 卸载卷 | ✅（VolumeManager + CSI Node 插件调用） | attach/detach 由 controller-manager 的 AttachDetach controller 或 CSI 自己完成 |
| 配置容器网络 | ✅（调用 CNI 插件） | IPAM 由 CNI 负责 |
| 探针探测 | ✅ | liveness 失败则重启容器 |
| 节点/ Pod 状态上报 | ✅ | Node status + Lease |
| 驱逐 Pod | ✅ **节点压力驱逐** | 与 scheduler 抢占完全不同 |
| 准入 Pod（节点级） | ✅ `CanAdmit` 检查 | 资源、cgroup、拓扑亲和等，失败 → `UnexpectedAdmissionError` |
| 镜像/容器/日志 GC | ✅ | ImageGC / ContainerGC / ContainerLogManager |

**必答要点**：kubelet 是"**单点自治**"的——即使 API Server 不可达，它仍会继续维持现有 Pod 的运行（这是设计目标），只是无法获取新指令。

---

## Q2. kubelet 内部有哪些核心组件？画一下

```
                       ┌─────────────────────── kubelet ───────────────────────┐
 API Server ─watch──►  │ Pod source (apiserver / file(static) / http)          │
                       │        │                                              │
                       │        ▼                                              │
 local manifests ────► │  PodManager (期望状态缓存)                             │
                       │        │                                              │
 (container runtime) ─►│  PLEG (事件生成：ContainerStarted/Died/Changed)        │
 watchdog events       │        │                                              │
                       │        ▼                                              │
                       │  SyncLoop (4 路事件合并 → syncLoopIteration)           │
                       │        │  dispatch → podWorkers                       │
                       │        ▼                                              │
                       │  PodWorker(pod) → syncPod / SyncTerminatingPod        │
                       │        ├── VolumeManager (mount/unmount, CSI)          │
                       │        ├── ImageManager + ImageGC                      │
                       │        ├── CRI Runtime (PodSandbox + containers)       │
                       │        ├── CNI plugin (sandbox 网络 setup/teardown)    │
                       │        ├── ProbeManager (liveness/readiness/startup)   │
                       │        ├── StatusManager (同步状态 → apiserver)         │
                       │        ├── CgroupManager / QoS cgroups                 │
                       │        └── EvictionManager (节点压力驱逐)              │
                       │                                                        │
                       │  cadvisor（容器指标） / device plugin manager          │
                       │  Certificate manager / NodeLease controller / metrics  │
                       └────────────────────────────────────────────────────────┘
```

**高频追问：PodManager 里存的是什么？**
答：期望状态（来自 apiserver/static file/http），与实际 runtime 状态的"对账源"。PLEG 提供真实事件，两者在 SyncLoop 里汇合，驱动 `syncPod` 做**幂等收敛**。

**幂等性（加分点）**：`syncPod` 可以被反复调用且结果一致，这是 kubelet 允许"周期性重放 + 事件驱动"混合模式的基础。

---

## Q3. SyncLoop 的四个事件源是什么？为什么设计成这样？

`syncLoopIteration` 同时从 4 个 channel 收事件：

| Channel | 来源 | 作用 |
| --- | --- | --- |
| `configCh` | PodManager（apiserver watch / static file / http） | 期望状态变化（新 Pod、删除 Pod） |
| `plegCh` | PLEG（周期性 relist runtime） | 真实状态变化（容器死了/启动了/变了） |
| `syncCh` | 定时器 `--sync-frequency`（默认 1m） | **兜底全量重同步**，防事件丢失导致状态漂移 |
| `lifecycleCh` | 上次 syncPod 失败/需要重试的 Pod 回灌 | 失败重放 |
| `housekeepingCh` | 周期性清理任务 | GC / 卷清理等 |

**必答的设计思想**：

1. **事件驱动 + 周期兜底（"level-triggered 而非 edge-triggered"）**——丢事件不会导致永久错 상태，因为最多 1 分钟就被 sync 拉回来；
2. `--sync-frequency` 越小收敛越快，但 CPU/API 开销越大（ Node-agent 的内部 arning）。

---

## Q4. PLEG 是什么？为什么它一出问题，节点上所有 Pod 都跟着 Pending？(必考)

**PLEG = Pod Lifecycle Event Generator**。

**旧版（GenericPLEG）**：**定期（默认 1s）relist** —— 调用 CRI 列出节点上所有 sandbox/container，与上一次快照 diff，生成 `ContainerStarted / ContainerDied / ContainerRemoved / ContainerChanged` 事件，塞进容量为 1000 的 channel，供 SyncLoop 消费。

**为什么它是单点故障放大器**：

| 现象 | 根因 |
| --- | --- |
| `PLEG is not healthy: pleg was last seen active 3m ago; threshold is 3m` | relist 周期超过阈值（源码默认 3m），kubelet 直接把 Node 置 **NotReady** |
| 所有 Pod 瞬间变 `ContainerCreating`/`Unknown` | 拿不到 runtime 状态 ⇒ 无法收敛 |
| `Event channel is full, discard this relist()` | 一次 relist 事件超过 channel 容量（1000），**事件被丢弃** ⇒ 状态漂移，只能等 1 分钟兜底 sync |

**relist 为什么会变慢**：

- 容器密度高 ⇒ 单次 relist 要遍历成千上万个容器，且 **CNI/CSI/CRI 调用是串行的**；
- runtime（containerd/docker）本身卡顿（overlayfs IO、磁盘满、shim 泄漏）；
- 节点 CPU 被打满（含被低优任务抢占）。

**解法**：

1. **Evented PLEG**（新版本的 feature gate）：改为 runtime 主动推送事件 + 低频全量 relist 兜底，显著降低 relist 开销（**是否 GA、版本号以 Release Notes 为准**）；
2. 降 Pod 密度 / 调 `--pod-pids-limit`；
3. 治理 runtime 健康（containerd shim/fs/inode）；
4. 监控 `kubelet_pleg_relist_duration_seconds` 提前预警。

```promql
# PLEG relist 耗时 P99（阈值需实测校准，集群内 >5s 就要查）
histogram_quantile(0.99, sum by (le) (rate(kubelet_pleg_relist_duration_seconds_bucket[5m])))
# relist 间隔
histogram_quantile(0.99, sum by (le) (rate(kubelet_pleg_relist_interval_seconds_bucket[5m])))
```

---

## Q5. CRI 是什么？PodSandbox 又是什么？

**CRI = Container Runtime Interface**：kubelet 通过 gRPC（`RuntimeService` + `ImageService`）调用运行时，由运行时负责真正拉镜像、创建/删除容器。**kubelet 自己不直接操作 runc/容器进程**。

常见运行时：**containerd**（主流，内置 CRI 插件）、**CRI-O**、（Docker 经由 dockershim 的路径已在 1.24 起移除，必须转 containerd/CRI-O）。

**PodSandbox（Pause 容器）的作用（必答）**：

- 为同一 Pod 里的所有容器**共享 network namespace**（Pod IP 在 sandbox 上，容器加入同一个 netns）；
- 承担 PID namespace 共享、共享 volume 生命周期；
- **Sandbox 没起来 = Pod 绝不可能起来** ⇒ `ContainerCreating` 卡住最常见的就是 sandbox 创建失败（CNI 失败、runtime 失败、资源不足）。

**CRI 相关 metric（排障利器）**：

```promql
# 按操作类型看 runtime 调用耗时，能精确分离"拉镜像慢 / 创建容器慢 / 停容器慢"
histogram_quantile(0.99,
  sum by (operation_type, le) (rate(kubelet_runtime_operations_duration_seconds_bucket[5m])))

# runtime 错误率
sum by (operation_type) (rate(kubelet_runtime_operations_errors_total[5m]))

# 调用量（密度视角）
sum by (operation_type) (rate(kubelet_runtime_operations_total[5m]))
```

> `operation_type` 通常包含 `create_container`、`start_container`、`stop_container`、`pull_image`、`image_status`、`container_status`、`podsandbox_status` 等（**指标取值请以目标节点 **`curl -k https://127.0.0.1:10250/metrics`** 核对**）。

---

## Q6. kubelet 在网络上到底做了什么？CNI 是怎么被调用的？

kubelet **不实现任何网络**，它只是在 sandbox 创建/销毁时按 CNI 规范调用二进制插件：

```
SyncPod → CRI RunPodSandbox → CNI ADD(/opt/cni/bin/<plugin>, conf from /etc/cni/net.d)
                             → plugin 创建 veth / 调 IPAM 分配 IP / 写路由 / 返回 IP
删除 Pod → CRI StopPodSandbox → CNI DEL
```

**必答要点**：

- 网络插件（Calico/Cilium/Flannel 等）负责 **IPAM、跨节点路由、网络策略**，kubelet **只负责触发 CNI 调用并把结果写回 Pod status**；
- **CNI 失败 ⇒ **`FailedCreatePodSandBox`，典型错误：`failed to allocate for range` (IP 耗尽)、`plugin type="calico" failed (add): ...`、CNI DaemonSet 未就绪；
- `network-unavailable` taint：CNI 未就绪时由网络组件或云管控在节点上打污点，防止 Pod 被调度到无网络节点。

---

## Q7. CSI 与 kubelet 的存储挂载流程？

kubelet 侧是 **VolumeManager**，两个核心 reconciler：

1. **DesiredStateOfWorldPopulator**：算出本节点该挂哪些卷；
2. **Reconciler**：执行 mount/unmount。

对 CSI 卷的典型调用链：

```
kubelet VolumeManager
   ├─ NodeStageVolume（格式化 + 挂载到 staging 路径）—— 可选，取决于驱动
   ├─ NodePublishVolume（bind mount 到 Pod 目录 /var/lib/kubelet/pods/<uid>/volumes/...）
   └─ NodeExpandVolume（在线扩容时）
卸载时反向：NodeUnpublishVolume → NodeUnstageVolume
```

**attach/detach 不在 kubelet**：块存储的 Attach/Detach 由 **AttachDetach controller（controller-manager）** 或 CSI attacher 处理；有些 CSI 驱动走 `skip attach` 模式（由 kubelet 直接调用 CSI NodePublish）。

**高频坑**：

- in-tree 存储插件已全面**迁移到 CSI**（社区推动 CSI migration；具体哪些卷类型已下线请以版本 Release Notes 为准，旧参数/annotation 在新版本会失效）；
- CSINode 对象记录该节点上驱动信息与拓扑；**驱动 Pod 崩 ⇒ 卷挂不上 ⇒ Pod 卡 ContainerCreating**；
- subPath 挂载的 **ConfigMap/Secret 不支持热更新**。

---

## Q8. 一次 Pod 启动，从 bind 到 Ready，全链路经过哪些阶段？慢在哪？（必考综合题）

```
t0  scheduler 绑定 nodeName
t1  kubelet watch 到本节点 Pod → PodManager → SyncLoop → podWorker.syncPod
t2  WaitForAttachAndMount：等待卷 attach + mount
t3  CRI RunPodSandbox（创建 pause 容器）
t4  CNI ADD（分配 IP、建 veth、设路由）
t5  拉取镜像（可能串行排队！）
t6  CRI CreateContainer + StartContainer（含 postStart hook）
t7  startupProbe / readinessProbe 通过
t8  Pod Ready → Endpoint 更新 → kube-proxy/Ingress 生效（这一步还有额外传播延迟！）
```

**慢点定位顺序（可直接背）**：

| 现象 | 大概率阶段 | 证据 |
| --- | --- | --- |
| `ContainerCreating` 很久 | 镜像拉取 / 卷挂载 / CNI | Events + `kubelet_runtime_operations_duration_seconds{operation_type="pull_image"}` |
| Pod 起来了但很久才 Ready | 探针太保守 / 应用冷启动 | `kubelet_pod_start_duration_seconds`、探针 threshold |
| Ready 了但流量还没切过来 | Endpoint 异步传播 | Endpoint/EndpointSlice 更新延迟、ingress reload |
| Pending 卡在 kubelet 之前 | 调度或 admission | 上一篇 L5 的排障路径 |

**可观测的三个黄金 metric**：

- `kubelet_pod_start_duration_seconds`（kubelet 视角的 Pod 启动耗时）
- `kubelet_pod_worker_duration_seconds`（syncPod 本身耗时）
- `scheduler_pod_scheduling_duration_seconds`（前一段）

**注意**：这三者相加再加上"镜像未缓存时的下载时间"才接近体感冷启动时长。

---

## Q9. 容器重启策略与 CrashLoopBackOff 的退避规律？

| `restartPolicy` | 行为 |
| --- | --- |
| `Always`（默认） | 容器退出就重启 |
| `OnFailure` | 仅当退出码非 0 时重启 |
| `Never` | 从不重启 |

**CrashLoopBackOff 退避**：**指数退避**，`10s → 20s → 40s → 80s → 160s → 300s`，上限 **300s（5 分钟）**（源码默认；版本相关请核对）。**容器稳定运行一段时间后计数值会重置**。

**必答的坑**：

- CrashLoopBackOff 是**结果不是原因**，必须看 `kubectl logs --previous` 和退出码；
- 退出码 **137 = SIGKILL**（多半 OOMKill 或主动 kill 或蒸发）；**143 = SIGTERM**（优雅终止中）；**139 = 段错误**；
- 应用启动慢但 liveness 阈值太小 ⇒ **应用还没起来就被反复杀**，正确解法是配 `startupProbe`。

```yaml
livenessProbe:
  httpGet: {path: /healthz, port: 8080}
  periodSeconds: 10
  timeoutSeconds: 1
  failureThreshold: 3
startupProbe:            # 关键：给慢启动应用豁免窗口
  httpGet: {path: /healthz, port: 8080}
  periodSeconds: 5
  failureThreshold: 60   # 最多等 5 * 60 = 300s
```

---

## Q10. 三种探针的区别与常见误用？

| 探针 | 失败后果 | 典型误用 |
| --- | --- | --- |
| `livenessProbe` | **重启容器**（按 restartPolicy） | 依赖了外部中间件 ⇒ DB 抖动导致全站重启雪崩 |
| `readinessProbe` | **摘除 Endpoint**（不重启） | 忘了配 ⇒ 滚动更新期间流量打到未就绪实例 |
| `startupProbe` | 失败不重启但阻塞 liveness 启动 | 慢启动应用没配 ⇒ 被 liveness 反复杀 |

**必答两条铁律**：

1. **liveness 只应检查"进程自身是否死锁/失活"**，绝不能检查外部依赖（DB、下游服务）——否则外部抖动会造成**全集群级重启风暴**；
2. **readiness + 滚动更新 + **`minReadySeconds` 才能避免"流量打到还没预热的实例"。

**探针默认值**（不配时的默认，版本相关）：`periodSeconds=10`、`timeoutSeconds=1`、`failureThreshold=3`、`successThreshold=1`。

---

## Q11. Pod 优雅终止到底是怎么回事？为什么会出现 502？（超高频事故题）

```
1. API Server 收到删除请求 → Pod 进入 Terminating，设 deletionTimestamp
2. kubelet 发现 → 触发 SyncTerminatingPod
3. ① 执行 preStop hook（阻塞式，必须执行完）
   ② 随后发 SIGTERM 给容器主进程（PID 1）
4. 等待优雅退出，最多 terminationGracePeriodSeconds（默认 30s）
5. 超时 ⇒ SIGKILL
6. 卸载卷、CNI DEL、清理 cgroup
```

**为什么会出现 502 / 连接中断（必答）**：

> `deletionTimestamp`** 落地的同时，Endpoint controller 会把该 Pod 移出 Endpoint**——但这个过程是**异步**的，kubelet 侧几乎同时就开始发 SIGTERM。于是：**进程已经停了，但 Ingress/kube-proxy/客户端侧的负载均衡列表还没更新** ⇒ 这段时间内的新请求打到"正在死亡"的 Pod ⇒ 502/连接重置。

**标准解法（可直接套用）**：

```yaml
spec:
  terminationGracePeriodSeconds: 60    # 给足的总预算
  containers:
    - name: app
      lifecycle:
        preStop:
          exec:
            command: ["sh", "-c", "sleep 15"]   # 15s < 60s，用于等待 endpoint 传播
      readinessProbe:      # 提前把自己变成 notReady，让 LB 先摘
        httpGet: {path: /ready, port: 8080}
        periodSeconds: 2
```

配套：

- 应用自身处理 **SIGTERM**：停止接收新连接、等待存量请求处理完再退出；
- `Deployment` 配 `minReadySeconds` + `maxUnavailable: 0`（或 1）降低滚动更新抖动；
- 灰度/缩容场景配 **PodDisruptionBudget**；
- 长连接场景要留意 `keepalive` 时间。

> **注意**：`preStop` 时长**占用** `terminationGracePeriodSeconds` 预算，不是额外加的；强行缩短 grace 也可能造成"业务没退干净就被 SIGKILL"。

---

## Q12. Eviction Manager 是怎么工作的？（驱逐）

**目标**：节点自身资源快撑不住时，**主动杀 Pod 保住节点**，与 scheduler 抢占无关。

| 概念 | 说明 |
| --- | --- |
| **Hard eviction** | 触及即杀，**没有宽限期**（用户可配，但立刻给容器发 SIGTERM，随后 SIGKILL） |
| **Soft eviction** | 触及阈值后等待 `evictionSoftGracePeriod`，期间仍超限才杀 |
| `evictionPressureTransitionPeriod` | 压力消失后维持 Node condition 的时长（默认 5m），**防止 condition 抖动导致调度器反复 taint 抖动** |
| `evictionMinimumReclaim` | 每次驱逐至少要回收多少，避免"刚够就停"反复触发 |
| `--eviction-max-pod-grace-period` | 软驱逐时给 Pod 的最大优雅期（超时即 SIGKILL） |

**默认阈值（社区默认，请以所用版本为准）**：

| 信号 | 默认 Hard 阈值 |
| --- | --- |
| `memory.available` | `< 100Mi` |
| `nodefs.available` | `< 10%` |
| `nodefs.inodesFree` | `< 5%` |
| `imagefs.available` | `< 15%`（存在独立 imagefs 时生效） |
| `pid.available` | 由 `--pod-max-pids` 决定 |

**驱逐顺序（必答）**：

1. 先看**是否超过 requests**（实际用量超出 requests 的 Pod 优先被驱逐）；
2. 再看 **QoS 等级**：`BestEffort` → `Burstable` → `Guaranteed`（Guaranteed 只在自身超 limit 或系统级紧缺时被逐）；
3. 同优先级内再按资源占用量排序。

**触发后同时会**：给节点打 `MemoryPressure`/`DiskPressure`/`PIDPressure` condition ⇒ controller-manager 打对应 `NoSchedule`** taint**（注意不是 NoExecute）⇒ 新 Pod 不再调度上来。

**可直接套用**：

```yaml
apiVersion: kubelet.config.k8s.io/v1beta1
kind: KubeletConfiguration
evictionHard:
  memory.available: "500Mi"        # 建议：大内存节点按比例给，别用默认 100Mi
  nodefs.available: "10%"
  nodefs.inodesFree: "5%"
  imagefs.available: "15%"
evictionSoft:
  memory.available: "1Gi"          # 更早介入，给 Pod 优雅退出的时间
evictionSoftGracePeriod:
  memory.available: "1m30s"
evictionMaxPodGracePeriod: 30      # 软驱逐时最多等 30s 就 SIGKILL
evictionMinimumReclaim:
  memory.available: "0Mi"
  nodefs.available: "500Mi"
  imagefs.available: "2Gi"
evictionPressureTransitionPeriod: "5m"
```

> **关键实践**：把 `memory.available` 的 hard 阈值从默认 100Mi 调大（*建议大内存节点至少 500Mi\~**1Gi，实测校准*），否则节点会在 OOM kill 的边缘才动手，来不及救。

---

## Q13. Allocatable 是怎么算出来的？为什么要配 kube/system reserved？

```
节点 Capacity（整机）
  - kubeReserved      （kubelet / 容器运行时 等节点守护进程）
  - systemReserved    （sshd / systemd-journald / 云厂商 agent 等系统进程）
  - evictionThreshold （硬性预留不被算进可用）
  = Node Allocatable（调度器能用的量）
```

**必答关键点（极易踩坑）**：

> **只写 **`-KubeReserved/systemReserved`** 只是"记账"，不生效！** 必须同时指定 `--kube-reserved-cgroup` / `--system-reserved-cgroup`，kubelet 才会真的在对应 cgroup 上设限制；否则 Pod 依然可能把系统组件挤到 OOM。

```yaml
apiVersion: kubelet.config.k8s.io/v1beta1
kind: KubeletConfiguration
kubeReserved:
  cpu: "500m"
  memory: "1Gi"
  ephemeral-storage: "1Gi"
systemReserved:
  cpu: "1000m"
  memory: "2Gi"
  ephemeral-storage: "2Gi"
# 真正生效的关键（cgroup v2 路径示例；v1 路径不同，需按版本确认）
kubeReservedCgroup: /runtime.slice
systemReservedCgroup: /system.slice
```

**为什么不足预留会出事**：节点上跑高密度 Pod 时，Pod 会把 CPU/内存/IO 吃光，导致 **kubelet 自身被 OOM / 抢不到 CPU ⇒ PLEG relist 变慢 ⇒ Node NotReady ⇒ 更大规模雪崩**。这是典型的"自杀式循环"。

---

## Q14. QoS 等级怎么判定？OOM 时谁先死？

| QoS | 判定条件 | 内存紧张时的表现 |
| --- | --- | --- |
| **Guaranteed** | 每个容器 CPU/内存都 `requests == limits`（都设且相等） | 最后才被杀（只在自身超 limit 或系统级紧缺时） |
| **Burstable** | 至少一个容器设了 requests 但不满足 Guaranteed | 中间梯队 |
| **BestEffort** | 所有容器都没设 requests/limits | **第一个被杀** |

**OOM 判定顺序（内核 oom\_****score\_****adj 由 kubelet 设置）**：BestEffort(`1000`) \> Burstable(`min(1000, 999-1000*request/allocatable)` 量级) \> Guaranteed(`-997`)。

**必答要点**：

- **"没设 limits" ≠ 无限制**：`limits.memory` 不设意味着容器可以用到节点内存上限，**极易被 OOMKill 且不受 cgroup 保护**；
- **CPU 不设 limits 不会 OOM，但可能被 throttling**（若设了 limits）；CPU 是不可压缩资源竞争中的"可压缩资源"，只影响延迟不影响存活；
- 想要"**稳定的节点级资源底线**" = `requests 合理 + QoS=Guaranteed + kube/system reserved 生效`。

---

## Q15. cgroup v1 vs v2、CPU Manager、Memory Manager、Topology Manager？

| 管理层 | 作用 | 常见策略 |
| --- | --- | --- |
| **cgroup driver** | kubelet 用什么接口管理 cgroup | `systemd`（推荐，和主机 systemd 一致）vs `cgroupfs` |
| **CPU Manager** | 独占核/绑核 | `none`（默认）/ `static`（需 CPU request 为整数且 Guaranteed） |
| **Memory Manager** | 内存 NUMA 亲和 / hugepages | `None` / `Static` |
| **Topology Manager** | 跨 CPU/设备 NUMA 对齐 | `none` / `best-effort` / `restricted` / `single-numa-node` |

```yaml
apiVersion: kubelet.config.k8s.io/v1beta1
kind: KubeletConfiguration
cgroupDriver: systemd
cpuManagerPolicy: static
# cpuManagerPolicyOptions:
#   full-pcpus-only: "true"      # 各版本实现差异较大，以 Release Notes 为准
reservedSystemCPUs: "0-1"        # 给系统/守护进程预留，绝不让 Pod 独占所有核
memoryManagerPolicy: Static
topologyManagerPolicy: single-numa-node
topologyManagerScope: pod        # container/pod，版本相关
```

**必答的坑**：

- `cpuManagerPolicy: static` 时必须配 `reservedSystemCPUs`，否则 kubelet/系统进程会被挤图层用一个核上抖动；
- **static 策略要求 CPU requests 为整数且 QoS=Guaranteed**，否则拿不到独占核，甚至报 `TopologyAffinityError`/拓扑亲和失败；
- GPU 场景常配 `single-numa-node`，但会导致**调度条件变苛刻、部分节点不可用**（这是可用性成本，要量化）；
- **cgroup v2 是新版本推荐路径**，v1 正在逐步退场；迁移前要确认 runtime/CNI/监控组件都支持。

---

## Q16. 节点心跳与 Node NotReady 的完整时间账（超级必考）

```
kubelet 每 node-status-update-frequency（默认 10s）          → 更新 Node status（条件变化时才更新）
kubelet 每 10s（lease duration 默认 40s）                     → 续约 Node Lease（轻量对象，专门给心跳用）
kube-controller-manager --node-monitor-grace-period（默认 50s）→ 超过则置 Node Ready=Unknown/False
       ↓ taint-manager 打 taint：node.kubernetes.io/not-ready 或 unreachable (NoExecute)
       ↓ Pod 默认 tolerationSeconds = 300s（toleration 是 kubelet/apiserver 给 Pod 注入的默认容忍）
       ↓ 5 分钟后才开始驱逐 Pod → 重新调度到其它节点
```

**结论（必须背下来）**：**节点宕机后，Pod 大约需要 50s + 300s = 约 5\~****6 分钟才会被真正重调度。** 业务上"节点挂了马上自愈"是不可能的，除非主动缩短 `tolerationSeconds`。

**缩短恢复时间的办法（要在 YAML 里显式写）**：

```yaml
spec:
  tolerations:
    - key: node.kubernetes.io/not-ready
      operator: Exists
      effect: NoExecute
      tolerationSeconds: 30        # 建议：无状态快恢复业务设 30~60
    - key: node.kubernetes.io/unreachable
      operator: Exists
      effect: NoExecute
      tolerationSeconds: 30
```

**必答的 trade-off**：

- 缩短 `tolerationSeconds` 会让网络抖动/短时失联期间**误驱逐**有状态服务（数据库/分布式共识组件），可能引发数据风险；
- 有状态服务通常应**容忍更久**（甚至用 `tolerationSeconds` 更大值），并且必须有 PDB + 数据副本保护；
- 缩 `--node-monitor-grace-period` 同样有副作用：**控制面抖动会放大**。

---

## Q17. Static Pod vs DaemonSet？Mirror Pod 是什么？

| 维度 | Static Pod | DaemonSet |
| --- | --- | --- |
| 定义位置 | 节点磁盘上的 manifest 文件（`--pod-manifest-path`，常见 `/etc/kubernetes/manifests`） | API Server 对象 |
| 管理者 | **kubelet 自己**（file source） | DaemonSet controller |
| 能否被 API 修改删除 | 不能（删了 kubelet 会重建） | 可以 |
| 集群可见性 | 通过 **Mirror Pod**（只读镜像对象）在 apiserver 里可见 | 原生可见 |
| 典型用途 | **引导控制面**：apiserver/etcd/kube-scheduler/controller-manager | 日志采集、CNI、监控 agent |

**必答**：

- Static Pod **不会被调度器调度**，不受 schedulerName 影响，也不参与 ResourceQuota；
- Mirror Pod 是只读的，`kubectl delete` 会立刻被 kubelet 重建（这就是"删不掉某些 kube-system Pod"的原因）；
- 名字后缀是节点名，例如 `kube-apiserver-node01`。

---

## Q18. 镜像/容器/日志 GC 是怎么做的？磁盘怎么防爆？

| GC | 触发 | 默认值（请以版本为准） |
| --- | --- | --- |
| **Image GC** | 磁盘使用率超过 `imageGCHighThresholdPercent` | High=85，Low=80（清理到 80 为止）；`imageMinimumGCAge` 默认 2m |
| **Container GC** | 定期（默认 1m） | 每个 Pod 保留的已死容器数（`--maximum-dead-containers-per-container`，旧曾是 2）/ 全局总数（旧 `--maximum-dead-containers=100`，**新版本这些开关正在被逐步弃用/隐藏**） |
| **Container log rotation** | 按文件大小/个数 | `--container-log-max-size=10Mi`、`--container-log-max-files=5`（默认） |

```yaml
apiVersion: kubelet.config.k8s.io/v1beta1
kind: KubeletConfiguration
imageGCHighThresholdPercent: 85
imageGCLowThresholdPercent: 80
imageMinimumGCAge: "2m"
containerLogMaxSize: "50Mi"     # 建议：按业务日志量实测调整，别用默认 10Mi 硬扛重日志业务
containerLogMaxFiles: 5
```

**必答实践**：

- 生产上**建议把 **`/var/lib/containerd`**（imagefs）与 **`/var/lib/kubelet`**、**`/`**（nodefs）分离磁盘**，避免镜像把根盘撑爆导致 kubelet 本身挂掉；
- 日志要**同时施肥两层防护**：容器日志轮转（防单容器）+ 节点日志采集组件限流（防 agent 写爆）+ 节点 disk 告警；
- `imageMinimumGCAge` 太小会导致刚拉的镜像被清，**反复拉取**。

---

## Q19. Device Plugin 是怎么注册到 kubelet 的？

```
Device plugin Pod
   │ 1. 在宿主机 /var/lib/kubelet/device-plugins/<name>.sock 起 gRPC server
   ▼
kubelet PluginManager（watch socket 目录）
   │ 2. 调用 Register(gRPC) → kubelet 建立长连接
   │ 3. 周期性 ListAndWatch → kubelet 更新 Node allocatable：nvidia.com/gpu: 8
   ▼
kubelet /metrics? popularity: PLEGRESS/pod-resources
   │ 4. Pod 创建时 kubelet 调 Allocate() → 拿回注入 env/mounts/annotations
   ▼
容器真正拿到设备
```

**必答**：

- kubelet 提供 **PodResources API**（`/var/lib/kubelet/pod-resources/kubelet.sock`），让第三方组件（如 GPU 监控/共享方案）能查询"某个 Pod 分到了哪些设备"；
- kubelet 侧只做**数量 + 健康状态**管理，**不理解设备拓扑/型号**（与上一篇 Q21 呼应）；
- Device plugin 挂掉 ⇒ 资源上报停止 ⇒ 已有 Pod 通常继续跑，但新 Pod 可能因资源不存在而失败（取决于实现）。

---

## Q20. kubelet 的权限模型：Node Authorizer / NodeRestriction / TLS Bootstrap（安全必考）

**三件事必须一起配**：

1. **kubelet 认证**：TLS 客户端证书（`authentication.x509.clientCAFile`）+ Bootstrap kubeconfig；
2. **kubelet 授权**：`authorization.mode=Webhook`（走 SubjectAccessReview）；
3. **API Server 侧**：启用 `--authorization-mode=...,Node` + `NodeRestriction`** admission 插件**。

**Node authorizer 能做什么（必须背）**：

| 允许 | 禁止 |
| --- | --- |
| 读取**自身节点**绑定的 Pod（field selector `spec.nodeName=<self>`） | 读其它节点的 Pod |
| 读取自己 Pod **引用到的** Secret/ConfigMap/PVC | 读任意 Secret |
| 创建/更新自己的 Node status、Lease | 改其它 Node |
| 创建 events、创建 CSR | 创建普通 Pod（防止 kubelet 被利用做权限提升） |

> `NodeRestriction`** 的存在意义**：即使攻击者拿到某节点的 kubelet 证书，也只能操作该节点的资源，**不能借助 apiserver 横向读取全集群 Secret**——这是"Pod 逃逸后的第二道防线"。

**TLS Bootstrap 与证书轮换**：

```yaml
apiVersion: kubelet.config.k8s.io/v1beta1
kind: KubeletConfiguration
rotateCertificates: true          # 客户端证书轮换（默认 true 更有利）
serverTLSBootstrap: true          # serving 证书走 CSR 流程，需由 approve 通道批准（见下文说明）
authentication:
  anonymous:
    enabled: false                # 必须关！
  webhook:
    enabled: true
  x509:
    clientCAFile: /etc/kubernetes/pki/ca.crt
authorization:
  mode: Webhook
readOnlyPort: 0                   # 关闭 10255 只读端口，历史漏洞面
```

**必答**：

- **关闭 **`readOnlyPort`**（10255）**、**关闭匿名认证** 是两条硬基线；
- 证书轮转由 kubelet 的 certificate manager 负责；若 `serverTLSBootstrap: true`，serving CSR 需要 apiserver 侧有 **CSR approver**（具体 approve 方式因发行版/版本而异：有的靠内置 approver、有的靠 kubeadm/CCM 的控制器，也可能要手动 `kubectl certificate approve`），**这一点务必在自己的集群里验证过**；
- "证书过期导致 kubelet 无法注册"是经典事故，务必监控证书有效期（见 D3）。

---

## Q21. ConfigMap/Secret 挂载为什么有时不更新？

| 使用方式 | 是否热更新 | 机制 |
| --- | --- | --- |
| **环境变量注入** | ❌ 不更新 | Pod 创建时快照，必须重建 Pod |
| **Volume 挂载** | ✅ 会更新（有延迟） | kubelet 周期性同步（受 `--sync-frequency` 影响），通过符号链接**原子切换**；延迟一般是数十秒到 1 分钟级别，**需实测** |
| `subPath`** 挂载** | ❌ 不更新 | subPath 挂载的是具体文件路径，没有符号链接切换机制 |

**必答**：

- 想要"立即生效"，必须让应用**监听文件变化重载**或用 Reloader/argocd 之类方案触发滚动重启；
- Secret/ConfigMap 更新后容器进程不会自动重启，这一点常常误解。

---

## Q22. 节点关机时能优雅吗？Graceful Node Shutdown 与 out-of-service taint？

| 机制 | 作用 |
| --- | --- |
| **Graceful Node Shutdown**（新版本 GA，请以 Release Notes 为准） | kubelet 监听 systemd 的 inhibitor lock，收到关机信号时**主动进入节点优雅下线流程**，按优先级分阶段终止 Pod（普通 Pod → 关键 Pod，`shutdownGracePeriod` / `shutdownGracePeriodCriticalPods`） |
| **Non-graceful shutdown handling** | 节点**没来得及关机就掉电**的场景：给节点打 `node.kubernetes.io/out-of-service` taint ⇒ 强制把该节点上有状态卷的 Pod 删除，让它们能在别处重建（**必须人工确认节点真的不可恢复**，否则会造成双写/脑裂风险） |

```yaml
apiVersion: kubelet.config.k8s.io/v1beta1
kind: KubeletConfiguration
shutdownGracePeriod: 60s
shutdownGracePeriodCriticalPods: 10s
```

**必答 caveat**：`out-of-service` taint 是**破坏性操作**（强制删除带卷 Pod），生产上必须有"节点已确认下线/不可恢复"的工单与确认流程（某些云厂商已在其托管服务中自动处理）。

---

## Q23. kubelet 自身怎么加固？（生产基线）

| 项 | 建议值 | 理由 |
| --- | --- | --- |
| `readOnlyPort` | `0` | 关闭 10255 未认证只读端口 |
| `authentication.anonymous.enabled` | `false` | 禁止匿名访问 10250 |
| `authorization.mode` | `Webhook` | 走 SubjectAccessReview |
| Node 授权 + NodeRestriction | 启用 | 限制横向权限 |
| `protectKernelDefaults` | `true` | 防止 kubelet 把内核 sysctl 改错/依赖未设置的值 |
| TLS cipher suites | 关闭弱套件 | 合规 |
| 10250 端口 | 仅限管控网段可达 | 安全组/主机防火墙 |
| `--streaming-connection-idle-timeout` | 设合理值（如 4h） | 防止 exec/port-forward 长连接无限挂，同时避免过早断开 |
| 版本 | **不高于 apiserver** | 版本倾斜策略要求 |

---

## Q24. kubelet 提供哪些排障入口？（面试最后往往会问"你怎么看"）

| 入口 | 内容 | 用途 |
| --- | --- | --- |
| `https://<node>:10250/metrics` | kubelet + runtime + PLEG 指标 | 性能定位 |
| `/metrics/cadvisor` | 容器 CPU/内存/网络/FS 指标 | 资源画像 |
| `/metrics/resource` | 资源相关（各 cAdvisor Rread） | 资源画像 |
| `/metrics/probes` | 探针相关 latency | 探针开销分析 |
| `/stats/summary` | Summary API | metrics-server 数据源 |
| `/pods` | kubelet 视角的本节点 Pod 期望清单 | **对比实际状态，判断 apiserver↔kubelet 是否同步** |
| `/var/lib/kubelet/pod-resources/kubelet.sock` | Pod↔设备映射 | GPU/DPDK 类排查 |
| `journalctl -u kubelet` | 日志 | 权威解释 |

**常用命令组合（可直接背）**：

```bash
# 看 Pod 在 kubelet 侧的描述 vendor vs api
curl -sk --cert /var/lib/kubelet/pki/kubelet-client-current.pem \
     --key /var/lib/kubelet/pki/kubelet-client-current-key.pem \
     https://127.0.0.1:10250/pods | jq '.items[].metadata.name'

# 看 runtime 调用最慢的操作
curl -sk ... https://127.0.0.1:10250/metrics | grep kubelet_runtime_operations_duration

# 容器运行时侧（containerd）
ctr -n k8s.io containers ls | wc -l
crictl ps -a | wc -l
crictl inspectp <sandbox-id>
```

# Part B · 大规模集群篇（16 题）

> 定义：**大规模 = ≥ 1000 节点 / 单节点 ≥ 50 Pod（高密度）**。

## L0. 规模约束与版本倾斜政策（先立基准）

| 维度 | 官方目标（以官方文档为准） |
| --- | --- |
| 每节点 Pod 数 | ≤ **110**（`maxPods` 默认即 110） |
| 节点数 | ≤ 5000 |
| 集群 Pod 总数 | ≤ 150000 |

**版本倾斜政策（必考）**：

- kubelet **不得比 apiserver 新**；
- kubelet **最多可以比 apiserver 低 3 个 minor 版本**；
- 升级顺序：**先 apiserver（及其它控制面组件）→ 再 kubelet**。

> 面试可以这么收尾："所以大规模集群的 kubelet 往往是**多版本共存**的，压测和灰度必须以最老的 kubelet 版本为准。"

---

## L1. 大机型（少节点高密度）还是小机型（多节点低密度）？

| 维度 | 大机型（如 128C/512G，跑 100 Pod） | 小机型（如 8C/32G，跑 10 Pod） |
| --- | --- | --- |
| 资源利用率 | 高（碎片少、削峰填谷） | 低 |
| 故障爆炸半径 | **大**（一台宕机影响 100 Pod） | 小 |
| 节点数 | 少 ⇒ 控制面压力小 | 多 ⇒ 心跳/watch 压力大 |
| PLEG/GC 压力 | **大**（单节点容器数多，relist 代价高） | 小 |
| 镜像缓存命中 | 高（同节点同镜像仅拉一次） | 低（每台都要拉） |
| 特殊资源（GPU/NUMA） | 更好排布 | 受限 |

**标准答法**：

> "这是**故障半径 vs 资源效率 vs 控制面压力**的三方权衡。实践做法是分层：**在线低延迟业务用中小机型控制爆炸半径，离线批处理用大机型提高装箱率**；单节点 Pod 数建议留 20% 余量，不要贴着 110 跑（要给 DaemonSet、系统组件与突发伸缩留位置）。"

---

## L2. 高密度节点为什么容易 PLEG 不健康？怎么治理？

**成本模型**：GenericPLEG 每 **1s** relist 全部容器，单次成本约 O(容器数)，期间还要串行查询 runtime 状态。单节点 100+ Pod × 每 Pod 2\~3 容器 ⇒ 每次 relist 要遍历数百个实体；一旦 runtime 变慢（创建/删除排队、磁盘 IO 高），relist 就可能超过 3m 阈值 ⇒ **Node NotReady，该节点所有 Pod 全受影响**。

**治理清单**：

| 手段 | 做法 | 代价 |
| --- | --- | --- |
| **Evented PLEG**（按需开启，版本相关） | 运行时主动推事件 + 低频全量兜底 | 需 runtime 支持、版本较新 |
| 降低单节点 Pod 数 | `maxPods` 下调到实际需要的 1.2 倍 | 利用率下降 |
| 节点池拆分 | 把"容器生命周期变更极频繁"的业务隔离到独立池 | 池化复杂度 |
| 治理 runtime | containerd shim 泄漏、overlayfs、inode、磁盘 IO | 需专项排查 |
| **CPU 保障** | `kubeReserved` + `reservedSystemCPUs` + cgroup 真正生效，确保 kubelet 不被抢 CPU | 通常需重启节点生效 |

**告警（要早于故障）**：

```promql
# relist 耗时异常（阈值为建议初值，需按集群实测校准）
histogram_quantile(0.99, sum by (node, le) (rate(kubelet_pleg_relist_duration_seconds_bucket[5m]))) > 5

# 结合 Node Ready 状态判断
kube_node_status_condition{condition="Ready",status="false"} == 1
```

---

## L3. 镜像拉取为什么会成为大规模瓶颈？怎么优化？

**问题**：大规模滚动发布时，成百上千节点同时拉同一个大镜像（AI/大数据镜像动辄 GB 级）⇒ **仓库出口带宽打满 + 节点被限流** ⇒ 大面积 `ImagePullBackOff`、Pod 冷启动暴涨。

**证据**：

- Events：`Failed to pull image ...: toomanyrequests`、`Error response from daemon: Get "https://...": net/http: TLS handshake timeout`；
- `kubelet_runtime_operations_duration_seconds{operation_type="pull_image"}` P99 暴涨。

**优化手段**：

| 手段 | 说明 |
| --- | --- |
| **并发拉取** | `serializeImagePulls: false`（**默认是串行的！**） |
| **并发上限** | `maxParallelImagePulls`（引入版本较晚，请以 `KubeletConfiguration` 文档为准） |
| 仓库 QPS 限流 | `registryPullQPS` / `registryBurst`（*建议初值 5/10*） |
| **P2P / 延迟加载** | Dragonfly、Nydus/erofs 快照器等降低冷启动 |
| 镜像瘦身 | 多阶段构建、统一基础镜像、去无用层 |
| 就近仓库 | 避免跨区域拉取；用私有仓库 + 缓存/CDN |
| DaemonSet 预热 | 批量发布前先预热镜像到目标节点 |

```yaml
apiVersion: kubelet.config.k8s.io/v1beta1
kind: KubeletConfiguration
serializeImagePulls: false       # 关键：允许并发拉取不同的镜像
registryPullQPS: 5               # 建议初值，按仓库容量实测调整
registryBurst: 10
```

---

## L4. Pod 冷启动慢的系统性治理清单？

按"这段开销能不能省掉"逐层砍：

| 环节 | 优化 | 备注 |
| --- | --- | --- |
| 镜像下载 | 见 L3（并发/P2P/预热/瘦身） | 通常是最大头 |
| 镜像解压 | Nydus/erofs 延迟加载 + 快照器 | 大镜像收益明显 |
| 卷挂载 | 减少不必要的卷；关注 CSI 驱动健康 | attach/detach 串行要警惕 |
| CNI | DaemonSet 就绪时间、IPAM 性能 | IP 耗尽会直接卡住 sandbox |
| initContainer | **串行执行**且各要拉镜像，能合并就合并 | — |
| sidecar | 每个 sidecar 都要拉镜像+探针，累积延迟 | 注意 sidecar 的重启语义 |
| 探针 | `startupProbe` 给足窗口 | 过严会导致"刚起就被重启" |
| 应用自身 | 预热、连接池初始化 | 业务侧优化 |
| 调度 | 见调度篇 | — |

**度量方法（必给）**：

```promql
# kubelet 视角 Pod 启动耗时（判断是不是 podWorker 本身慢）
histogram_quantile(0.99, sum by (le) (rate(kubelet_pod_worker_duration_seconds_bucket[5m])))
histogram_quantile(0.99, sum by (le) (rate(kubelet_pod_start_duration_seconds_bucket[5m])))
# runtime 分解（判断是 pull 慢还是 create 慢）
histogram_quantile(0.99, sum by (operation_type, le) (
  rate(kubelet_runtime_operations_duration_seconds_bucket[5m])))
```

---

## L5. 驱逐风暴怎么防？

**现象**：一批节点同时进入 `MemoryPressure`/`DiskPressure` ⇒ 大量 Pod 被逐 ⇒ 重建后又落到压力节点/其它压力节点 ⇒ 抖动扩散。

**根因**：

- requests 严重虚低（实际用量远超 requests）⇒ 节点严重超卖；
- 突发批处理、大内存任务叠加；
- 日志/临时文件暴涨把磁盘打满；
- kube/system reserved 没做 ⇒ kubelet 自身资源被抢，形成恶性循环。

**治理组合拳**：

1. **requests 校准**：用实测用量（Prometheus / VPA recommendation）收敛 requests，让超卖率可控；
2. **预留与阈值**：`kubeReserved`/`systemReserved` 且 **cgroup 真正生效**；`evictionHard.memory.available` 提到 500Mi\~1Gi（*建议，需实测*）；
3. **软驱逐 + 优雅期**：给 Pod 体面退出的机会，而不是直接 SIGKILL；
4. **在线/离线分节点池**（taint 隔离），避免离线流量冲击在线；
5. **PDB + 拓扑打散**，保证始终有可用实例；
6. **告警前置**：压力 condition 一出现就介入，而不是等真的驱逐。

```promql
# 节点压力 condition（依赖 kube-state-metrics）
kube_node_status_condition{condition=~"MemoryPressure|DiskPressure|PIDPressure",status="true"} == 1

# 被驱逐的 Pod 数量增长 => 风暴信号
increase(kube_pod_container_status_terminated_reason{reason="Evicted"}[15m])
```

## L6. 磁盘/inode 治理：为什么节点磁盘最容易出事？

**四类消耗**：

1. **镜像**（imagefs）：滚动发布留下的旧镜像层；
2. **容器可写层 + 容器内临时文件**：业务写 `/tmp`、线程 dump、heap dump；
3. **容器日志**（`/var/log/pods/...`）：最容易忽视的一类，日志量暴涨直接撑爆根盘；
4. **kubelet/运行时元数据**：`/var/lib/kubelet/pods/...`、`/var/lib/containerd`（含 volume 挂载残留）。

**治理清单**：

| 手段 | 配置/做法 |
| --- | --- |
| **磁盘分离** | imagefs（容器运行时）与 nodefs（根分区）使用不同磁盘/分区 |
| 容器日志轮转 | `containerLogMaxSize` / `containerLogMaxFiles`（建议按业务实测调整） |
| 镜像 GC 阈值 | `imageGCHighThresholdPercent: 85` / Low 80；必要时下调 High 让 GC 更早介入 |
| 业务侧约束 | 通过 `ephemeral-storage` requests/limits 限制容器可写层 + emptyDir |
| 残留清理 | 排查孤儿目录：`/var/lib/kubelet/pods/` 里已删除 Pod 的残留卷 |
| 监控 | 分别对 imagefs/nodefs 做容量告警，并监控 inode |

```yaml
# 给业务 Pod 加临时存储配额（防止单 Pod 写爆节点）
apiVersion: v1
kind: Pod
metadata:
  name: app
spec:
  containers:
    - name: app
      image: app:1.0
      resources:
        requests:
          ephemeral-storage: "1Gi"
        limits:
          ephemeral-storage: "2Gi"   # 超限即被驱逐
```

```promql
# 容器日志占用的文件系统字节数（指标存在则说明版本支持）
sum by (node) (kubelet_container_log_filesystem_used_bytes)
```

---

## L7. CNI/IPAM 在大规模下的坑？

| 坑 | 现象 | 解法 |
| --- | --- | --- |
| **IP 耗尽** | `FailedCreatePodSandBox: failed to allocate for range` | 扩容 Pod CIDR / 调整 per-node IP 配额 / 回收泄漏 IP |
| **IP 泄漏** | Pod 已删除但 IPAM 未释放（`crictl` 与 IPAM 账不一致） | 升级 CNI、检查 kubelet/runtime 异常退出导致的 CNI DEL 丢失 |
| **CNI 串行调用** | 节点批量起 Pod 时 sandbox 创建排队 | 选择支持并发/更高性能的 CNI，控制单节点 Pod 密度 |
| **CNI DaemonSet 未就绪** | 新扩容节点上的 Pod 全部卡 ContainerCreating | 用 cloud provider/CNI 的 `network-unavailable` taint 做防呆 |
| **网络策略/转发规则膨胀** | eBPF map 或 iptables 规则量过大导致 CPU 升高 | 规则治理、选型评估（见 L14） |

---

## L8. 节点心跳对控制面的压力模型？

**账要这么算**：

| 机制 | 频率 | 对象 | 成本 |
| --- | --- | --- | --- |
| **Node Lease**（推荐） | 每 10s 一次 | `Lease` 轻量对象 | 低（设计目的就是把心跳从 Node 对象里剥离出来） |
| Node status 更新 | 状态变化时才更新，另有强制上报周期（默认 5m） | Node 对象（大） | 高 |
| Pod status 更新 | 状态变化时 | Pod 对象 | 中（高密度/探针频繁时会放大） |

**5000 节点 ⇒ 仅心跳就约 500 QPS 的 Lease 更新**，所以必须用 Lease 而不是 Node 对象。

**关键参数**：

```yaml
apiVersion: kubelet.config.k8s.io/v1beta1
kind: KubeletConfiguration
nodeStatusUpdateFrequency: 10s     # 默认值；调小会增加写放大，仅在强检测需求时调整
nodeStatusReportFrequency: 5m      # 默认强制上报周期
```

**必答 trade-off**：把心跳频率调低（省控制面）必然会**拉长故障检测时间**，因为 `node-monitor-grace-period` 必须 ≥ 数倍心跳周期。所以这条只能配合平台侧的 SLO 来说。

---

## L9. 证书过期 / 轮换风暴是经典事故

**典型表现**：某天一批节点同时 `Unable to register node ... x509: certificate has expired or is not yet valid`，节点被集群踢出，Pod 开始漂移。

**原因**：

- `rotateCertificates` 没开或 apiserver 侧 CSR approver 缺失 ⇒ 客户端证书过期；
- `serverTLSBootstrap` 开启但 **serving CSR 无人 approve** ⇒ kubelet serving 证书过期（注意：这会让 `kubectl logs/exec` 失败）；
- 节点时钟漂移 ⇒ 证书"尚未生效"或"已过期"误判。

**治理**：

1. 确认 `rotateCertificates: true` 且 apiserver 侧有可用的 CSR 审批机制（**必须在自己环境验证，不同发行版/版本实现方式不同**）；
2. 打开证书过期告警（见 D3）；
3. 所有节点强制 NTP/chrony 时钟同步并纳入监控；
4. 定期演练："把证书过期TTL临时调短 / 模拟 CSR 不批准"的 GameDay。

---

## L10. 高密度 + NUMA/CPU Manager 的陷阱

| 陷阱 | 现象 | 建议 |
| --- | --- | --- |
| `cpuManagerPolicy: static` 未配 `reservedSystemCPUs` | kubelet/系统守护进程与 Pod 争抢，节点抖动 | 必须预留 1\~2 核 |
| CPU request 非整数 / 非 Guaranteed | 拿不到独占核，行为与预期不符（`TopologyAffinityError`） | 绑核场景必须 Guaranteed + 整数 CPU |
| `topologyManagerPolicy: single-numa-node` | **调度变苛刻**，多 NUMA 机器上部分 Pod 无法放下 | 权衡后使用；在线延迟敏感可用，普通业务建议 `best-effort`/`restricted` |
| hugepages 与 memory manager 组合复杂 | 内存分配失败 | 先在非生产压测 |

---

## L11. kubelet 配置变更 / 升级怎么灰度与回滚？

**原则：kubelet 是节点级组件，变更必须"以节点池为单位"灰度，且必须排水 Drain。**

| 阶段 | 范围 | 观察（≥ 30min，覆盖一次业务高峰） | 回滚触发条件 |
| --- | --- | --- | --- |
| Step 1 | **1 个节点**（先在预发池） | Node Ready、PLEG relist、Pod 启动 P99、业务 P99 | 任一指标超基线 1.5× |
| Step 2 | **1% 节点**（同池内） | + 驱逐率、运行时错误率 | 出现 Evicted 增长或 runtime error 上升 |
| Step 3 | 10% → 50% | 全量 SLO Dashboard | 同上，或收到业务投诉 |
| Step 4 | 100% | 稳定 ≥ 24h | — |

**标准操作流程**：

```bash
# 1) 排水并隔离
kubectl cordon <node>
kubectl drain <node> --ignore-daemonsets --delete-emptydir-data --grace-period=60 --timeout=10m

# 2) 变更（示例：改 KubeletConfiguration 后重启）
vim /var/lib/kubelet/config.yaml
systemctl restart kubelet
kubectl get node <node> -w     # 等 Ready

# 3) 观察 30min ... 异常则回滚（恢复上个版本配置文件后再次排水重启）
cp /etc/kubernetes/kubelet-config.bak /var/lib/kubelet/config.yaml
systemctl restart kubelet

# 4) 恢复调度
kubectl uncordon <node>
```

**必答的风险点**：

- **需要先 **`cordon`** 再 **`drain`，并保证 PDB 允许排水（否则 drain 会挂住或违反可用性）；
- kubelet 版本**不能高于 apiserver**，升级顺序错了无法回退；
- `cpuManagerPolicy` 变更需要清理 `cpu_manager_state` 状态文件（带破坏性，**必须在排水后操作并写入工单**）；
- 回滚 kubelet 版本时若中间已发生过"静态 CPU 分配"等状态变更，务必确认状态一致性。

---

## L12. 常见故障现象速查排障表（面试杀手锏）

| 现象 | 常见根因 | 第一步证据 |
| --- | --- | --- |
| **Pending** | 调度层问题（见调度篇） | Events 是否有 `FailedScheduling` |
| **ContainerCreating 卡住** | 镜像拉取慢/失败、CNI 失败、卷挂载失败、secret 缺失 | Events（`FailedCreatePodSandBox` / `FailedMount` / `FailedAttachVolume`）、`crictl ps -a` |
| **ImagePullBackOff** | 仓库认证/限流、镜像不存在、tag 错误 | `Failed to pull image` reason、`crictl pull <img>` 在节点上验证 |
| **CrashLoopBackOff** | 应用异常退出、探针过严、配置缺失退出码非 0 | `kubectl logs <pod> --previous`、退出码 137/139/143 |
| **OOMKilled** | memory limit 过小或节点压力 | `kube_pod_container_status_terminated_reason{reason="OOMKilled"}`、内核日志 |
| **Evicted** | 节点压力驱逐 | `kubectl describe pod` reason=Evicted、节点 condition |
| **Terminating 卡死** | Finalizer 卡住、preStop 阻塞、卷卸载失败、runtime 僵死 | `kubectl get pod -o yaml` 看 finalizers、`crictl` 状态 |
| **Unknown** | 节点失联后 kubelet 无法上报，Pod 被保留至驱逐完成 | Node Ready 状态、lease 心跳 |
| **CreateContainerConfigError** | ConfigMap/Secret key 不存在 | Events 明确指出缺失的 key |
| **整个节点 Pod 全unknown** | **PLEG 不健康 / kubelet 挂了 / 网络断** | `journalctl -u kubelet` + PLEG relist 指标 |

---

## L13. 版本治理：为什么必须盯 version skew？

- kubelet 可低于 apiserver 最多 3 个 minor，不可高于；
- **升级顺序**：控制面（含 apiserver）→ 其它 controller/kube-proxy → kubelet；
- 大规模集群常见问题是"**部分节点忘了升**"，导致行为不一致（如 GC 参数、cgroup v2 支持、feature gate 差异），要提供**版本台账 + 自动巡检**。

```promql
# 各节点的 kubelet 版本分布（kube-state-metrics）
count by (kubelet_version) (kube_node_info)
```

---

## L14. 大规模下 kubelet 侧可观测性的成本问题

| 项 | 风险 | 建议 |
| --- | --- | --- |
| `/metrics/cadvisor` 抓取 | 每指标维度大，抓取成本高 | 按需精简 metric relabeling，或降低抓取频率 |
| metrics-server → summary API | 每个节点调用 kubelet summary API，大规模下压力大 | 注意采样频率与副本数；新版倾向于 **kepler/cAdvisor 直采 + Prometheus** 组合 |
| `--enable-features` 打开过多 | 指标维度爆炸 | 按需开启 |
| 探针频率过高 | 每个探针都要发请求，Pod 多时开销明显 | `periodSeconds` 按 SLO 给，不是越小越好 |

> **经验建议（需实测校准）**：单节点 Pod 数 \> 50 时，应核算 `kubelet` 自身 CPU/内存占用，把"节点 agent 开销"计入集群容量规划（一般给节点预留数百 MiB 内存和 1\~2 核 CPU 给系统 + 守护进程，具体以实测为准）。

---

## L15. kubelet 安全加固清单（可直接用于巡检）

- [ ] `readOnlyPort: 0`（10255 关闭）
- [ ] `authentication.anonymous.enabled: false`
- [ ] `authorization.mode: Webhook`
- [ ] apiserver 侧启用 Node authorizer + `NodeRestriction` admission
- [ ] 10250 端口仅管控网段可达（安全组/iptables）
- [ ] `protectKernelDefaults: true`
- [ ] 禁用弱 TLS 套件
- [ ] 证书自动轮换开启，并有 CSR approve 通道
- [ ] 关闭 `privileged` Pod 的默认允许（上层策略：PSA/PSS、OPA/Kyverno）
- [ ] 节点 hostPath 挂载受控（Pod Security Admission 或策略引擎）
- [ ] 审计：谁在 exec/attach/port-forward（`audit-policy` + kubelet webhook）

---

## L16. 给你一个新节点池，怎么设计 kubelet 基线？（综合题）

按这个顺序答：

1. **先问 SLO 与工作负载画像**：在线/离线？有无绑核/大页/GPU？单节点跑多少 Pod？
2. **机型与密度**：确定机器规格（CPU/内存/磁盘/NUMA）与 `maxPods`（留 20% 余量）；
3. **资源预留**：`kubeReserved`/`systemReserved` + cgroup 真正生效 + `reservedSystemCPUs`；
4. **驱逐策略**：hard/soft 阈值 + 优雅期 + minimumReclaim；
5. **磁盘**：镜像/容器数据与根分区分离；日志轮转 + GC 阈值；
6. **拉取策略**：`serializeImagePulls: false` + 仓库 QPS + 是否需要 P2P；
7. **CPU/Memory/Topology manager**：按 NUMA 需求定策略；
8. **安全基线**：L15 清单全勾；
9. **可观测性**：kubelet + runtime + PLEG + 节点 condition 告警；
10. **变更治理**：镜像化 kubelet 配置 + 节点池灰度 + drain/cordon 流程 + 回滚工单；
11. **演练**：杀 kubelet、注内存压力、模拟列表 Large image pull，验证 SLO 与 MTTR。

# Part C · 反模式清单（可直接用于生产巡检）

> 每条含：反模式 / 证据 / 风险等级 / 修复动作 / 验证方式

| # | 反模式 | 证据（怎么发现） | 等级 | 修复动作 | 验证方式 |
| --- | --- | --- | --- | --- | --- |
| C1 | 容器不设 requests/limits | `kube_pod_container_resource_requests` 为空；节点频繁 OOM | 🔴 高 | LimitRange 默认值 + 用量画像校准 | OOMKill 数下降并稳定 |
| C2 | 只设 `kubeReserved`/`systemReserved` **没配对应 cgroup** | 节点上 kubelet/系统进程被 OOM，`config.yaml` 无 reservedCgroup | 🔴 高 | 补齐 `kubeReservedCgroup`/`systemReservedCgroup` 并重启 | 压力测试下 kubelet 进程不再被杀 |
| C3 | `livenessProbe` 检查外部依赖（DB/下游） | 依赖抖动时全集群 Pod 重启 | 🔴 高 | liveness 只查自身活性，依赖检查放 readiness | 依赖故障演练不再触发重启 |
| C4 | 不配 `preStop` / `terminationGracePeriodSeconds` 过短 | 发布窗口出现 502/连接重置 | 🔴 高 | preStop sleep + 放宽 grace + 应用处理 SIGTERM | 滚动更新期间错误率不上升 |
| C5 | 用默认 `evictionHard.memory.available=100Mi` | 触发过晚，来不及救 | 🔴 高 | 调大并按机型实测 | 内存压力下有足够时间优雅退出 |
| C6 | `readOnlyPort` 未关 / 匿名认证开启 | 安全扫描、10255 可访问 | 🔴 高 | `readOnlyPort: 0` + 关闭匿名 | 端口探测关闭 |
| C7 | `serializeImagePulls` 保持默认 true | 发布时间长、P99 启动高 | 🟠 中 | 设 false + 调整仓库限流 | 发布总时长下降（实测） |
| C8 | 日志无上限（用默认 10Mi/5 却发现写爆） | 节点 disk 打满、Evicted 增多 | 🟠 中 | 按业务调整轮转参数 + ephemeral-storage limit | DiskPressure condition 消失 |
| C9 | 单节点密度满载 110 Pod | PLEG relist 抖动、NotReady 事件 | 🟠 中 | 留 20% 余量；按业务画像下调 maxPods | `kubelet_pleg_relist_duration_seconds` 回归基线 |
| C10 | 静态绑核未配 `reservedSystemCPUs` | 节点抖动、kubelet 抢不到 CPU | 🟠 中 | 预留 1\~2 核 | 重启后资源画像稳定 |
| C11 | 证书轮换无 approver 通道 | CSR 长期 Pending、证书过期 | 🔴 高 | 完善 approve 机制 + 过期告警 | 演练临时证书过期可自动续签 |
| C12 | 无节点 PLEG/runtime 监控 | 故障才发现 relist 早就变慢 | 🟠 中 | 建立 D3 告警集 | 注入故障 5min 内告警 |
| C13 | kubelet 版本高于 apiserver / 长期不升 | `kube_node_info` 版本台账异常 | 🟠 中 | 遵循 skew 策略先升控制面 | 巡检台账无越界节点 |
| C14 | 接到态服务未设更长 `tolerationSeconds` | 网络抖动被误驱逐 | 🟠 中 | 有状态服务放宽容忍或固定节点 | 网络抖动演练不触发重调度 |

---

# Part D · 可观测性：SLO、指标与告警（可直接落地）

## D1. 建议的 kubelet 侧 SLO（**初始建议值，必须实测校准**）

| SLI | 建议目标（起点） | 计算方式 |
| --- | --- | --- |
| 节点 Ready 率（按池） | ≥ 99.9% | `kube_node_status_condition{condition="Ready",status="true"}` |
| Pod 启动 P99（kubelet 视角） | ≤ 5s（不含镜像首次拉取） | `kubelet_pod_start_duration_seconds` |
| 节点压力驱逐频率 | 在线池 ≈ 0 | `increase(kube_pod_container_status_terminated_reason{reason="Evicted"}[1h])` |
| PLEG relist P99 | ≤ 5s | `kubelet_pleg_relist_duration_seconds` |
| 证书剩余有效期 | \> 30 天告警 | `kubelet_certificate_manager_*_expiration_seconds` |
| 容器重启率 | 单 Deployment \< 0.1%/h | `kube_pod_container_status_restarts_total` |

## D2. 核心指标清单（先在节点上 `curl -k https://127.0.0.1:10250/metrics` 核对）

| 指标 | 含义 | 用于 |
| --- | --- | --- |
| `kubelet_pod_start_duration_seconds` | Pod 从进入 kubelet 到所有容器 started 的耗时 | 冷启动 SLO |
| `kubelet_pod_worker_duration_seconds` | syncPod 执行耗时（按 operation 分） | 定位 kubelet 内部慢点 |
| `kubelet_pleg_relist_duration_seconds` / `..._interval_seconds` | PLEG relist 耗时/间隔 | PLEG 健康度 |
| `kubelet_runtime_operations_total` / `_duration_seconds` / `_errors_total` | CRI 调用量/耗时/错误（按 `operation_type`） | **runtime 排障首选** |
| `kubelet_running_pods` / `kubelet_running_containers` | 本机 Pod/容器数 | 密度与泄漏 |
| `kubelet_started_pods_total` / `kubelet_started_containers_total` | 累计启动数 | 变更/抖动频率 |
| `kubelet_container_log_filesystem_used_bytes` | 容器日志占用 | 磁盘治理 |
| `kubelet_cgroup_manager_duration_seconds` | cgroup 操作耗时 | 资源压力下的性能 |
| `kubelet_certificate_manager_client_expiration_seconds` / `_server_expiration_seconds` | 证书剩余有效期 | 过期事故预防 |
| `kubelet_node_name`（info 类） | 节点存在性/元信息 | up 检测 |
| `rest_client_request_duration_seconds` | kubelet 作为客户端访问 apiserver 的耗时 | 区分是 apiserver 慢还是本节点慢 |
| kube-state-metrics：`kube_node_status_condition` | Node Ready/Pressure condition | 节点可用性 |
| kube-state-metrics：`kube_pod_container_status_terminated_reason` | 终止原因（Evicted / OOMKilled / Error） | 驱逐与 OOM 治理 |

> **注意**：指标名在不同版本会有增删/改名，**上线前必须先在目标版本节点核对**。

## D3. Prometheus 告警规则（可直接粘贴，阈值请实测校准）

```yaml
groups:
  - name: kubelet
    rules:
      - alert: NodeNotReady
        expr: kube_node_status_condition{condition="Ready",status="false"} == 1
        for: 3m
        labels: {severity: critical}
        annotations:
          summary: "节点 {{ $labels.node }} NotReady"

      - alert: NodeUnderPressure
        expr: |
          kube_node_status_condition{condition=~"MemoryPressure|DiskPressure|PIDPressure",status="true"} == 1
        for: 5m
        labels: {severity: warning}
        annotations:
          summary: "节点 {{ $labels.node }} 处于 {{ $labels.condition }}，可能即将驱逐 Pod"

      - alert: PodEvictedSpike
        expr: increase(kube_pod_container_status_terminated_reason{reason="Evicted"}[15m]) > 10
        for: 2m
        labels: {severity: critical}
        annotations:
          summary: "15 分钟内 {{ $value }} 个容器被驱逐，疑似驱逐风暴"

      - alert: PlegRelistSlow
        expr: |
          histogram_quantile(0.99,
            sum by (instance, le) (rate(kubelet_pleg_relist_duration_seconds_bucket[5m]))) > 5
        for: 10m
        labels: {severity: warning}
        annotations:
          summary: "实例 {{ $labels.instance }} PLEG relist P99 达 {{ $value }}s"

      - alert: RuntimeOperationsError
        expr: |
          sum by (instance, operation_type) (rate(kubelet_runtime_operations_errors_total[5m])) > 0.1
        for: 10m
        labels: {severity: warning}
        annotations:
          summary: "{{ $labels.instance }} 的 {{ $labels.operation_type }} 持续报错"

      - alert: ContainerRestartSpike
        expr: increase(kube_pod_container_status_restarts_total[30m]) > 10
        for: 5m
        labels: {severity: warning}
        annotations:
          summary: "容器 {{ $labels.namespace }}/{{ $labels.pod }} 30 分钟内重启 {{ $value }} 次"

      - alert: KubeletCertExpiringSoon
        expr: kubelet_certificate_manager_client_expiration_seconds < 30 * 86400
        for: 1h
        labels: {severity: critical}
        annotations:
          summary: "kubelet 客户端证书剩余 {{ $value | humanizeDuration }}，请检查自动轮换"

      - alert: NodeDiskPressurePredictive
        expr: |
          sum by (node) (kubelet_container_log_filesystem_used_bytes) > 20 * 1024 * 1024 * 1024
        for: 30m
        labels: {severity: info}
        annotations:
          summary: "节点 {{ $labels.node }} 容器日志占用超 20GiB，检查日志轮转"
```

> 抓取注意：kubelet 的 `10250` 是 HTTPS 且需要认证，Prometheus 侧要配 `scheme: https` + bearer token/kubeconfig CA + RBAC（`system:kubelet-api-admin` 或自建最小权限 ClusterRole）。**权限要最小化，别图省事给 cluster-admin。**

---

# Part E · 演练（GameDay）计划表

| 场景 | 注入方式 | 期望表现 | 验证的 SLO |
| --- | --- | --- | --- |
| kubelet 进程挂掉 | `systemctl stop kubelet` | 节点在 grace period 后 NotReady，Pod 在默认 5min 容忍后被重调度 | MTTR、容错时间账 |
| PLEG 不健康 | 制造 runtime 卡顿/大量容器创建 | PLEG 告警先于 NotReady 触发 | 告警前置能力 |
| 内存压力 | 跑内存压测容器 | 先软驱逐再硬驱逐，PDB 不被突破 | 驱逐 SLO |
| 磁盘打满 | 写大文件至根分区 | DiskPressure + Pod 被逐，日志轮转生效 | 磁盘治理 |
| 镜像仓库限流 | 限制 registry 出口 | 并发拉取开关生效，P2P/预热兜底 | 冷启动 SLO |
| 优雅下线 | `kubectl delete pod` 观察 | 无 502、preStop 生效 | 发布可用性 |
| 证书过期 | 临时缩短 TTL / 不 approve CSR | 告警触发且轮换通道可用 | 证书治理 |
| 节点非优雅掉电 | 强制关机 | `out-of-service` 流程按工单执行，带卷 Pod 能重建 | 有状态容灾 |

---

# Part F · 10 张面试速记卡（临阵磨枪）

1. **kubelet 是节点代理**：负责建/启/停容器、挂卷、配网络、跑探针、节点压力驱逐、上报状态；**不负责调度，不管集群级编排**。
2. **SyncLoop 四路事件**：configCh（期望） / plegCh（真实） / syncCh（1m 兜底） / lifecycleCh（失败重放）+ housekeeping（GC）。
3. **PLEG 是节点故障放大器**：relist 每 1s 一次、channel 容量 1000、连续 3m 没成功 ⇒ Node NotReady；解法 Evented PLEG / 降密度 / 保 kubelet CPU。
4. **五分钟账**：10s 心跳 → 50s `node-monitor-grace-period` → NotReady/Unreachable taint → Pod 默认 `tolerationSeconds=300` → **约 5\~****6 分钟才真正重调度**。
5. **驱逐 ≠ 抢占**：驱逐是 kubelet 保节点（看 requests 超限 + QoS：BestEffort \> Burstable \> Guaranteed）；抢占是调度器为了新 Pod。
6. **预留只记账不生效**：`kubeReserved`/`systemReserved` 必须配对应 `xxxReservedCgroup` 才真正限住。
7. **优雅终止三件事**：`preStop`（等 endpoint 传播）+ `terminationGracePeriodSeconds` + 应用处理 SIGTERM；三者缺一就会 502。
8. **探针铁律**：liveness 只查自身活性（查依赖会引发重启雪崩）；慢启动应用必配 `startupProbe`。
9. **镜像拉取默认串行**：`serializeImagePulls` 默认 true，大规模必须 `false` + 仓库限流 + P2P/预热。
10. **版本 skew**：kubelet 不可高于 apiserver，最多低 3 个 minor；升级顺序先控制面后 kubelet，灰度按节点池 + drain/cordon + PDB。

---

## 附：文档自查清单

- [ ] 每条建议配了可直接套用的 `KubeletConfiguration` / Pod YAML / PromQL
- [ ] 所有默认值（1s relist、3m 阈值、100Mi、10%、110 Pods、300s toleration、10s/5m 心跳等）均标注"以所用版本为准"
- [ ] 未提供任何未经实测的性能数字，均给出**度量方法**
- [ ] 反模式清单含：证据 / 等级 / 修复 / 验证
- [ ] SLO 标为建议初值并附错误预算口径
- [ ] 变更给出灰度比例（1 台 → 1% → 10% → 50% → 100%）、回滚触发条件与操作步骤
