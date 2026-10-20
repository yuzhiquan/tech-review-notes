# k8s-troubleshooting-60-cases

# Kubernetes 排障实战 60 例：现象 → 证据 → 止血 → 根治 → 防复发

> 定位：**现场手册**，不是教程。搜"现象关键词"直接命中案例。
> 来源：由《Scheduler》《Kubelet》《网络 / CNI》《控制面》《存储 / CSI》《Linux / 容器底层》《GPU / 大模型》七篇抽取、重组、去重而成。
> 引用格式：`【篇名 · 主题】`，方便回原篇深挖；不再重复展开原理。

---

## 0. 怎么用这本手册

### 0.1 万能三步（任何故障，先做这三步再往下翻）

```
第 1 步（30 秒）：看"对象自己怎么说"
    kubectl get pod/svc/pvc/node -o wide
    kubectl describe <obj>            # Events 是最高性价比的证据
    kubectl logs <pod> --previous     # CrashLoop 场景必看

第 2 步（3 分钟）：判断"卡在哪一层"
    调度层（Pending / FailedScheduling）→ 第 1 章
    容器层（ContainerCreating / CrashLoop / OOM）→ 第 2 章
    节点层（NotReady / PLEG / 驱逐）→ 第 3 章
    网络层（DNS / 不通 / 超时）→ 第 4 章
    控制面（kubectl 卡 / 写入失败）→ 第 5 章
    存储层（PVC Pending / FailedMount）→ 第 6 章
    GPU 层（训练 hang / MFU 低 / 推理抖动）→ 第 7 章

第 3 步（10 分钟）：取"权威证据"而不是猜
    调度：scheduler_pending_pods / schedule_attempts_total / 事件
    容器：crictl ps、容器退出码、cgroup memory.events、cpu.stat
    节点：kubelet 日志、PLEG relist 指标、dmesg
    网络：nsenter/NodeLocal/conntrack 计数、CoreDNS 指标
    控制面：apiserver_request_duration_seconds、etcd fsync/leader 指标
    存储：storage_operation_duration_seconds、kubelet_volume_stats_*
    GPU：DCGM 指标、XID、NCCL_DEBUG、nccl-tests busbw
```

### 0.2 止血的优先级（先别忙着"根治"）

```
① 止损：停止继续产生损失（停止发布、回滚、隔离坏节点、摘流量）
② 恢复：让业务可用（扩容、切流、重启、重试、从备份/快照恢复）
③ 保留现场：日志/快照/事件/指标截图 —— 这一步最容易被忘，但决定你能不能找到根因
④ 再谈根治：改配置、改代码、改架构
```

### 0.3 现象关键词速查表

| 关键词/现象 | 案例 |
| --- | --- |
| Pending 没有任何 Event | P01 |
| insufficient cpu/memory | P02 |
| untolerated taint | P03 |
| node affinity / selector 不匹配 | P04 |
| anti-affinity / topology spread 不满足 | P05 |
| exceeded quota（403 forbidden） | P06 |
| unbound PersistentVolumeClaims | P07 |
| preemption / 抢占 victims | P08 |
| ContainerCreating 卡住 | P09 / P10 / P11 |
| ImagePullBackOff | P12 |
| CrashLoopBackOff | P13 |
| OOMKilled / 137 | P14 |
| 优雅终止超时 / 143 | P15 |
| 探针失败、不 Ready | P16 |
| 发布窗口 502 | P17 |
| Terminating 卡死 | P18 |
| kill -TERM 1 没反应 / 僵尸 | P19 |
| No space left on device（但空间够） | P20 |
| too many open files | P21 |
| 时区错 | P22 |
| read-only file system | P23 |
| permission denied（挂载卷） | P24 |
| CPU 不高但 RT 抖动（throttling） | P25 |
| Node NotReady | P26 |
| PLEG is not healthy | P27 |
| kubelet 证书过期 | P28 |
| MemoryPressure / DiskPressure 驱逐 | P29 |
| 节点宕机后 Pod 迟迟不重建 | P30 |
| 节点磁盘满 | P31 |
| inode 耗尽 | P32 |
| kubelet 挂 / lease 争抢 | P33 |
| 时钟漂移 | P34 |
| Pod 全部 Unknown | P35 |
| DNS 间歇 5 秒超时 | P36 |
| DNS 慢 / QPS 高 | P37 |
| CoreDNS SERVFAIL / 不可用 | P38 |
| Service 有 endpoints 却不通 | P39 |
| conntrack table full | P40 |
| kube-proxy 规则同步慢 | P41 |
| IP 耗尽 / IPAM | P42 |
| 大包失败 / MTU | P43 |
| 跨可用区不通 / 延迟高 | P44 |
| 加 NetworkPolicy 后 DNS 挂 | P45 |
| 源 IP 丢失 / LB 不均 | P46 |
| kubectl 卡 / apiserver 慢 | P47 |
| 写操作全失败（etcd NOSPACE） | P48 |
| etcd 慢 / 频繁切主 | P49 |
| 409 Conflict | P50 |
| too old resource version | P51 |
| webhook 阻断全集群 | P52 |
| APF 拒绝请求 | P53 |
| 控制面证书过期 | P54 |
| PVC 一直 Pending | P55 |
| FailedAttachVolume / FailedMount | P56 |
| 卷挂载十几分钟 | P57 |
| PV/PVC 卡 Terminating / 孤儿盘 | P58 |
| 训练 hang 在 NCCL | P58 |
| MFU 低 / 训练变慢 / straggler / 坏卡 | P59 |
| 推理冷启动慢 / 副本负载不均 / KV cache 打满 | P60 |

---

# 第 1 章 · 调度与准入（P01–P08）

## P01 · Pod 一直 Pending，且**完全没有 Event**

| 项 | 内容 |
| --- | --- |
| **现象** | `kubectl describe pod` 的 Events 是空的，Pod 停在 Pending |
| **30 秒验证** | `kubectl get pod -o yaml ``\| grep schedulerName`，再查是否真有该调度器在跑：`kubectl -n kube-system get lease <name>` |
| **常见根因** | ① `spec.schedulerName` 拼错或无对应调度器运行（**最高频**）；② 调度器进程挂了 / lease 争抢失败；③ Pod 未进入任何调度器的 informer（选择器/字段问题） |
| **止血** | 有对应调度器就重启它；调度器名写错只能**删除重建 Pod**（`schedulerName` 不可变） |
| **根治** | 用 admission/校验 webhook 校验 schedulerName 白名单；CI 模板固定调度器名；调度器多副本 + lease 告警 |
| **防复发** | 巡检脚本：Pending 且 0 Event 的 Pod 列表；`absent(up{job="kube-scheduler"})` 告警 |
| **关联** | 【Scheduler 篇 · L5 排障路径】【Scheduler 篇 · 多 profile】 |

## P02 · `0/N nodes are available: N Insufficient cpu/memory`

| 项 | 内容 |
| --- | --- |
| **现象** | FailedScheduling，资源不足 |
| **30 秒验证** | `kubectl describe node ``\| grep -A8 Allocatable`；对比 `kubectl top node` |
| **常见根因** | ① requests 虚高（申请远大于实际用量）；② 节点规格/数量不足；③ 大规格 Pod（如 32C）在碎片化节点上放不下；④ DaemonSet/系统预留吃掉一部分 |
| **止血** | 扩容节点池；临时降低非关键副本数；把大 Pod 调度到专用大机型池 |
| **根治** | requests 校准（用实测量画像驱动）；装箱策略（离线用 MostAllocated）；节点池分层；Descheduler 整理碎片 |
| **防复发** | 资源分配率告警 + 单规格可调度性巡检 |
| **关联** | 【Scheduler 篇 · 打分与装箱】【Kubelet 篇 · 预留】 |

## P03 · `node(s) had untolerated taint`

| 项 | 内容 |
| --- | --- |
| **现象** | FailedScheduling，污点不容忍 |
| **30 秒验证** | `kubectl get node -o json ``\| jq '.items[].spec.taints'`；`kubectl describe node ``\| grep -i taint` |
| **常见根因** | ① 专用节点池（GPU/在线/离线）没配 tolerations；② 节点被自动打了压力污点（MemoryPressure 等）；③ 节点 cordon（`node.kubernetes.io/unschedulable`）；④ 云/CNI 初始化污点未移除 |
| **止血** | 给 Pod 加 tolerations，或把 Pod 调度到正确池；若是压力污点 ⇒ 先解决节点压力 |
| **根治** | 节点池 + tolerations 纳入模板管理；对压力污点建告警（这是"节点快不行了"的信号） |
| **防复发** | \`kube\_node\_status\_condition{condition=\~"MemoryPressure |
| **关联** | 【Scheduler 篇 · 污点容忍】【Kubelet 篇 · 驱逐】 |

## P04 · `didn't match Pod's node affinity/selector`

| 项 | 内容 |
| --- | --- |
| **现象** | FailedScheduling，亲和/标签不匹配 |
| **30 秒验证** | `kubectl get node --show-labels ``\| grep <key>`；核对 Pod 的 `nodeSelector/nodeAffinity` |
| **常见根因** | ① 标签拼写/值不一致；② 目标节点池已被缩掉或改了标签；③ 用了 NFD/GFD 标签但节点上没打；④ 硬性 `requiredDuringScheduling` 太严格 |
| **止血** | 修正标签或改为软亲和（`preferredDuringScheduling`）让服务先起来 |
| **根治** | 标签治理（NFD/节点池模板统一）；关键亲和条件要有多可用区兜底 |
| **防复发** | 对"因亲和无法满足而 Pending 的 Pod 数"做监控 |
| **关联** | 【Scheduler 篇 · 亲和三兄弟】【GPU 篇 · NFD/GFD】 |

## P05 · 反亲和 / TopologySpread 不满足导致调度不出

| 项 | 内容 |
| --- | --- |
| **现象** | FailedScheduling：`didn't match pod anti-affinity rules`，或副本数上不去 |
| **30 秒验证** | 统计每个节点上该应用的副本分布：`kubectl get pod -l app=X -o wide` |
| **常见根因** | ① 打散维度太严（hostname 级 + 副本数 \> 节点数）；② `whenUnsatisfiable: DoNotSchedule` 且域不足；③ 与资源不足叠加 |
| **止血** | 放宽为 `ScheduleAnyway`；扩容节点；临时减少副本 |
| **根治** | 用 `topologySpreadConstraints` 替代 podAntiAffinity；`maxSkew` + `minDomains` 设计合理；打散维度与节点数匹配 |
| **防复发** | 对"因 spread 无法调度"的计数与副本就绪率告警 |
| **关联** | 【Scheduler 篇 · 打散用 TopologySpread】【Scheduler 篇 · 大规模亲和成本】 |

## P06 · `forbidden: exceeded quota`

| 项 | 内容 |
| --- | --- |
| **现象** | Pod 创建直接被拒，甚至不会进入调度队列 |
| **30 秒验证** | `kubectl get resourcequota -n X -o yaml`；看 `status.used` |
| **常见根因** | ① 命名空间配额打满（cpu/memory/pods/gpu/storage）；② 上层控制器不断重试导致大量失败事件；③ LimitRange 缺失导致 requests 被默认放大 |
| **止血** | 调整配额或清理无用对象（旧 ReplicaSet、失败 Job、孤儿 PVC） |
| **根治** | 配额 + LimitRange 默认值 + 容量规划；对 GPU 用 Kueue 做队列与配额 |
| **防复发** | 配额使用率告警（≥80%）；失败事件监控 |
| **关联** | 【控制面篇 · ResourceQuota controller】【GPU 篇 · Kueue】 |

## P07 · `pod has unbound PersistentVolumeClaims` / `volume node affinity conflict`

| 项 | 内容 |
| --- | --- |
| **现象** | Pod Pending，提示卷未绑定或卷拓扑不匹配 |
| **30 秒验证** | `kubectl get pvc -n X`；`kubectl describe pvc`（看事件与 selected-node） |
| **常见根因** | ① StorageClass 用 `Immediate` ⇒ PV 落在别的可用区；② `allowedTopologies` 过严；③ provisioner/后端配额不足；④ CSI 驱动 CSINode 未注册 |
| **止血** | 若是 AZ 问题：删除 PVC 让它在正确 AZ 重新供给；或把 Pod 调度到 PV 所在 AZ |
| **根治** | **云盘类 StorageClass 一律 **`WaitForFirstConsumer`；拓扑与副本分布一起设计 |
| **防复发** | PVC Pending 时长告警；StorageClass 基线校验 |
| **关联** | 【存储篇 · WaitForFirstConsumer 与拓扑】【调度器篇 · VolumeBinding】 |

## P08 · 抢占频繁 / 在线服务被挤掉

| 项 | 内容 |
| --- | --- |
| **现象** | 在线 Pod 被驱逐后重建，Events 显示 Preempted；或 `scheduler_preemption_victims` 增长 |
| **30 秒验证** | `kubectl get priorityclass`；查看被驱逐 Pod 的 priority 与抢占者 |
| **常见根因** | ① 高优先级任务过多且允许抢占；② 资源不足才"需要抢"；③ 默认 PriorityClass 抢占策略过激 |
| **止血** | 暂停批处理发布；给在线服务配 PDB |
| **根治** | **默认 PriorityClass 设 **`preemptionPolicy: Never`；在线/离线节点池隔离；离线用队列配额而不是抢占 |
| **防复发** | `scheduler_preemption_attempts_total` 告警（\>0 就应 Review） |
| **关联** | 【Scheduler 篇 · 抢占全流程】【Scheduler 篇 · 反模式 C3】 |

---

# 第 2 章 · Pod 与容器（P09–P25）

## P09 · ContainerCreating 卡住：镜像拉取慢 / 失败

| 项 | 内容 |
| --- | --- |
| **现象** | Pod 长时间 ContainerCreating，Events 有 Pulling / Failed to pull image |
| **30 秒验证** | `kubectl describe pod`（看 Events 的 image 与 reason）；节点上 `crictl images ``\| grep <img>` |
| **常见根因** | ① 镜像大 + 串行拉取（`serializeImagePulls` 默认 true）；② 仓库限流/认证失败；③ 跨区域拉取；④ 节点首次拉取无缓存 |
| **止血** | 临时降低并发发布批次；确认凭据；必要时手动 `crictl pull` |
| **根治** | `serializeImagePulls: false` + 仓库限流参数；P2P/预热；镜像瘦身；就近仓库 |
| **防复发** | `kubelet_runtime_operations_duration_seconds{operation_type="pull_image"}` P99 告警 |
| **关联** | 【Kubelet 篇 · L3 镜像拉取】 |

## P10 · ContainerCreating 卡住：Sandbox / CNI 失败

| 项 | 内容 |
| --- | --- |
| **现象** | `FailedCreatePodSandBox`，Pod 拿不到 IP |
| **30 秒验证** | `kubectl describe pod`；`kubectl -n kube-system get pod -l k8s-app=calico-node` 等；节点上 `nsenter -t <pid> -n ip a` |
| **常见根因** | ① CNI DaemonSet 在该节点未就绪；② IPAM 地址耗尽；③ CNI 配置错误（`Failed to find plugin ... in /opt/cni/bin`）；④ 节点刚扩容、网络未初始化 |
| **止血** | 重启 CNI Pod；检查 IP 池；必要时 cordon 该节点 |
| **根治** | CNI 健康与就绪纳入节点上线校验；IP 容量规划；新节点用 `network-unavailable` 污点防呆 |
| **防复发** | `kube_node_status_condition{condition="NetworkUnavailable"}` 告警 + IP 池水位告警 |
| **关联** | 【网络篇 · CNI/IPAM】【网络篇 · L7】 |

## P11 · ContainerCreating 卡住：卷挂载（FailedMount / FailedAttach）

> 与 P56/P57 同源，区别：这里从"Pod 视角"快速判断归属。

| 项 | 内容 |
| --- | --- |
| **现象** | Events 出现 `FailedAttachVolume`（控制面/云 API）或 `FailedMount`（kubelet/CSI 节点侧） |
| **30 秒验证** | 看事件字段名即可分界：`FailedAttach*` = attach 层；`FailedMount` = mount 层 |
| **止血** | attach 层：检查云配额/限流、AZ 一致性；mount 层：检查 node plugin 与文件系统 |
| **根治** | 按【存储篇 B18】的现象→原因表处理；限制单节点卷数；分批重建 |
| **防复发** | `storage_operation_duration_seconds` 与事件转指标告警 |
| **关联** | 【存储篇 · 故障速查】【存储篇 · C3】 |

## P12 · ImagePullBackOff

| 项 | 内容 |
| --- | --- |
| **现象** | Pod 反复重试拉镜像 |
| **30 秒验证** | `kubectl describe pod` 看具体 reason：`toomanyrequests`（限流）、`manifest unknown`（tag 不存在）、`no basic auth credentials`（凭据） |
| **常见根因** | ① imagePullSecrets 缺失/失效；② 私有仓库证书/域名不通；③ tag 写错；④ 仓库限流 |
| **止血** | 修正 secret 或 tag；必要时用已缓存节点 |
| **根治** | secret 统一注入（ServiceAccount imagePullSecrets）；镜像仓库鉴权与配额治理；关键镜像预热 |
| **防复发** | 事件转指标：ImagePullBackOff 计数告警 |
| **关联** | 【Kubelet 篇 · 镜像 GC 与拉取】 |

## P13 · CrashLoopBackOff

| 项 | 内容 |
| --- | --- |
| **现象** | 容器反复重启，退避 10s→20s→…→300s |
| **30 秒验证** | `kubectl logs <pod> --previous`；`kubectl get pod -o jsonpath='{.status.containerStatuses[0].lastState.terminated.exitCode}'` |
| **常见根因** | ① 应用异常退出（配置/依赖/证书）；② 探针过严（还没起来就被杀）；③ OOM（137，见 P14）；④ 依赖未就绪 |
| **止血** | 回滚版本；改探针；临时加 initContainer 等待依赖 |
| **根治** | 加 `startupProbe`；探针只查自身活性；依赖就绪检查前置；退出码语义纳入监控 |
| **防复发** | 重启次数告警 + 按 Deployment 聚合 |
| **关联** | 【Kubelet 篇 · Q9 退避】【Kubelet 篇 · Q10 探针】 |

## P14 · OOMKilled（退出码 137）

| 项 | 内容 |
| --- | --- |
| **现象** | 容器被杀，`reason=OOMKilled` |
| **30 秒验证** | `kubectl describe pod`；容器内 `cat /sys/fs/cgroup/memory.events`（看 oom 计数）；宿主 `dmesg -T ``\| grep -i "out of memory"` |
| **常见根因** | ① memory limit 小于真实峰值；② **page cache 计入 cgroup**（读写文件暴涨）；③ 节点内存压力导致全局 OOM 选中；④ 应用按宿主内存算参数（JVM 堆） |
| **止血** | 提高 limit；临时降低并发/批大小 |
| **根治** | 用实测峰值（含 cache）设 limit；容器感知 cgroup；QoS=Guaranteed；给大量 IO 临时目录单独卷 |
| **防复发** | `kube_pod_container_status_terminated_reason{reason="OOMKilled"}` 告警；working\_set/limit 水位告警 |
| **关联** | 【Linux 篇 · A10 memory cgroup】【Kubelet 篇 · QoS】 |

## P15 · 优雅终止超时（143 / 30 秒后被 SIGKILL）

| 项 | 内容 |
| --- | --- |
| **现象** | Pod 删除慢，容器退出码 143；日志显示被硬杀 |
| **30 秒验证** | `kubectl get pod -o yaml`（`terminationGracePeriodSeconds`）；看应用是否处理 SIGTERM |
| **常见根因** | ① 应用不处理 SIGTERM；② `preStop` 执行时间过长或卡住；③ grace 太短；④ 卷卸载/storage 慢 |
| **止血** | 增大 `terminationGracePeriodSeconds`；修复 preStop |
| **根治** | 应用处理 SIGTERM；preStop 用于等待 endpoint 传播；配 PDB 与滚动参数 |
| **防复发** | 发布窗口错误率/502 监控；优雅终止时长指标 |
| **关联** | 【Kubelet 篇 · Q11 优雅终止】【Linux 篇 · A3 PID1】 |

## P16 · 探针失败：不 Ready / 被反复重启

| 项 | 内容 |
| --- | --- |
| **现象** | Pod 不进入 Ready；或 liveness 失败导致重启 |
| **30 秒验证** | `kubectl describe pod` 看 Readiness/Liveness 失败信息；`kubectl get endpoints` |
| **常见根因** | ① 探针路径/端口错；② 阈值太严（`failureThreshold × periodSeconds` 小于应用启动时间）；③ liveness 检查了外部依赖（DB/下游）⇒ 依赖抖动引发雪崩重启 |
| **止血** | 放宽阈值；把依赖检查从 liveness 挪到 readiness |
| **根治** | liveness 只查自身活性；慢启动用 startupProbe；配 `minReadySeconds` |
| **防复发** | 探针失败率与重启率告警 |
| **关联** | 【Kubelet 篇 · Q10 探针铁律】 |

## P17 · 滚动发布 / 缩容瞬间出现 502

| 项 | 内容 |
| --- | --- |
| **现象** | 发布窗口错误率尖峰，连接重置 |
| **30 秒验证** | 看发布期间 Endpoint 变化时间 vs Pod 终止时间；抓一条失败请求看打到了谁 |
| **常见根因** | ① 进程已停但 endpoints/LB/Ingress 传播未完成；② preStop 与 grace 太短；③ 长连接未优雅关闭；④ 客户端无重试 |
| **止血** | 回滚或暂停发布；增大 grace |
| **根治** | preStop sleep + 放宽 grace + 应用处理 SIGTERM；`maxUnavailable: 0` + PDB；网关侧做优雅摘流 |
| **防复发** | 发布期间 SLO 守护（自动暂停发布） |
| **关联** | 【Kubelet 篇 · Q11】【网络篇 · 优雅下线】 |

## P18 · Pod 卡在 Terminating

| 项 | 内容 |
| --- | --- |
| **现象** | Pod 一直 Terminating，删不掉 |
| **30 秒验证** | `kubectl get pod -o yaml` 看 `metadata.finalizers` 与 `deletionTimestamp`；容器是否还在运行 |
| **常见根因** | ① **Finalizer 未清除**（控制器挂了/自定义 finalizer）；② preStop 卡住；③ 卷无法卸载（存储 hang）；④ 容器进程 D 状态 |
| **止血** | 恢复对应控制器；修复卡住的 preStop；若卷 hang ⇒ 先 cordon 节点评估 |
| **根治** | 自定义 finalizer 必须有超时/自愈；卷卸载失败要有告警；避免依赖阻塞式 preStop |
| **防复发** | "Terminating 超过 5 分钟"的 Pod 巡检 |
| **关联** | 【Linux 篇 · B10 D 状态】【存储篇 · C3】 |

## P19 · 容器内 `kill -TERM 1` 无反应 / 僵尸进程堆积

| 项 | 内容 |
| --- | --- |
| **现象** | 发信号没效果；`ps` 里出现 `<defunct>` |
| **30 秒验证** | `kubectl exec` 看 PID 1 是什么（`ps -p 1 -o comm=`）；`ps -ef ``\| grep defunct ``\| wc -l` |
| **常见根因** | ① PID 1 是 shell（信号不转发、不 reap 僵尸）；② 应用未注册 handler 且是 PID 1（终止信号被内核忽略） |
| **止血** | 重启 Pod；必要时用 `kubectl exec` 内部 kill 具体子进程 |
| **根治** | **exec 形式入口**或 `dumb-init/tini`；镜像规范检查 |
| **防复发** | 镜像 CI 校验入口形式；僵尸数量监控 |
| **关联** | 【Linux 篇 · A3 PID namespace】 |

## P20 · `No space left on device` 但 `df -h` 有空间

| 项 | 内容 |
| --- | --- |
| **现象** | 写入失败，但容量看起来够 |
| **30 秒验证** | `df -i`（inode）；`df -h /dev/shm`（共享内存，默认常很小） |
| **常见根因** | ① **inode 耗尽**（overlay 层 + 海量小文件）；② `/dev/shm` 太小（PyTorch/Chrome/Postgres 常见）；③ 删除的文件仍被进程持有（`lsof ``\| grep deleted`） |
| **止血** | 清理文件；重启持有 fd 的进程；临时扩 shm |
| **根治** | emptyDir(Memory) 挂载 `/dev/shm` 并同步提高内存 limit；日志轮转与清理；inode 监控 |
| **防复发** | `node_filesystem_files_free / node_filesystem_files < 0.1` 告警 |
| **关联** | 【Linux 篇 · A18 overlayfs / A19 shm】 |

## P21 · `too many open files`

| 项 | 内容 |
| --- | --- |
| **现象** | 应用报错；或 kubelet/agent 报同样错误导致挂载失败 |
| **30 秒验证** | `cat /proc/<pid>/limits ``\| grep "open files"`；`ls /proc/<pid>/fd ``\| wc -l`；`cat /proc/sys/fs/file-nr` |
| **常见根因** | ① 进程 fd 泄漏；② 系统 `fs.file-max` 太小；③ **inotify watches/instances 耗尽**（常被误报成这个） |
| **止血** | 重启泄漏进程；临时调大限制 |
| **根治** | 修复泄漏；调 `fs.file-max` 与 inotify 参数；Secret 用 `immutable: true` 减少 watch |
| **防复发** | fd 使用率告警；inotify 用量纳入监控 |
| **关联** | 【Linux 篇 · A20 ulimit / C5 inotify】 |

## P22 · 容器时区/时间不对

| 项 | 内容 |
| --- | --- |
| **现象** | 日志时间差 8 小时；定时任务跑错点 |
| **30 秒验证** | `kubectl exec -- date`；看 `/etc/localtime` 是否挂载；宿主 `timedatectl` |
| **常见根因** | ① 镜像未设 TZ 且未挂 `/etc/localtime`；② 节点时钟漂移（chrony 未同步） |
| **止血** | 挂载 hostsPath localtime 或设置 TZ 环境变量 |
| **根治** | 基础镜像统一时区；节点 NTP/chrony 标准化并监控 |
| **防复发** | `abs(node_timex_offset_seconds) > 0.05` 告警 |
| **关联** | 【Linux 篇 · A25 时钟】 |

## P23 · `Read-only file system`（容器内写不了）

| 项 | 内容 |
| --- | --- |
| **现象** | 应用写临时文件失败 |
| **30 秒验证** | `kubectl get pod -o yaml ``\| grep readOnlyRootFilesystem`；看挂载类型 |
| **常见根因** | ① 安全加固开启 `readOnlyRootFilesystem: true`；② 文件系统进入只读（磁盘错误）；③ 挂载卷为只读 |
| **止血** | 给需要写的目录挂 emptyDir（**不要为了省事关掉 readOnlyRootFilesystem**） |
| **根治** | 明确"临时目录/缓存目录"清单并挂载 writable 卷；应用配置指向这些目录 |
| **防复发** | 镜像规范 + 安全基线校验 |
| **关联** | 【Linux 篇 · B3 加固】【Linux 篇 · B4 #15】 |

## P24 · 挂载卷后 `permission denied`

| 项 | 内容 |
| --- | --- |
| **现象** | 容器无法写入挂载的卷 |
| **30 秒验证** | `kubectl exec -- ls -ld /data`；宿主上看卷目录属主；**注意不要直接 chmod 777** |
| **常见根因** | ① `fsGroup` 未配；② fsGroup 递归 chown 未完成/未配 `OnRootMismatch`；③ SELinux 标签不对；④ NFS/文件存储导出选项不匹配 |
| **止血** | 配 `securityContext.fsGroup`；核对导出选项 |
| **根治** | 统一 fsGroup 策略 + `fsGroupChangePolicy: OnRootMismatch`；SELinux 正确 relabel |
| **防复发** | 挂载耗时监控（大卷 chown 会拖慢启动） |
| **关联** | 【存储篇 · A19 fsGroup】【Linux 篇 · A17 SELinux】 |

## P25 · CPU 使用率不高但 RT/P99 抖动（CFS 限流）

| 项 | 内容 |
| --- | --- |
| **现象** | 平均 CPU 只有 30%\~50%，但延迟毛刺明显 |
| **30 秒验证** | cgroup：`cat /sys/fs/cgroup/.../cpu.stat`（`nr_throttled`/`throttled_usec`）；Prometheus：`container_cpu_cfs_throttled_periods_total / container_cpu_cfs_periods_total` |
| **常见根因** | CPU limit 设置过小（在 100ms 周期里早早用完配额）；突发流量；GC/批处理叠加 |
| **止血** | 临时放大 limit |
| **根治** | 延迟敏感服务**去掉 CPU limit 或放宽到 request 的 2\~****4 倍**；用节点配额与隔离兜底 |
| **防复发** | throttled 占比 \>10% 告警（**这是最容易漏的指标**） |
| **关联** | 【Linux 篇 · A8 CFS throttling】 |

# 第 3 章 · 节点（P26–P35）

## P26 · Node NotReady

| 项 | 内容 |
| --- | --- |
| **现象** | 节点 Ready=False/Unknown，Pod 开始被驱逐或重排 |
| **30 秒验证** | `kubectl describe node`（Conditions 与 Events）；节点上 `systemctl status kubelet`；`journalctl -u kubelet -n 200` |
| **常见根因** | ① kubelet 挂了/无法连 apiserver（证书、网络、LB）；② **PLEG 不健康**（见 P27）；③ 节点资源耗尽（内存/磁盘/PID）；④ 时钟漂移导致证书无效；⑤ 真的宕机/网络断开 |
| **止血** | 若节点可恢复：修 kubelet/磁盘/时钟；不可恢复：cordon + 等待 Pod 重排（**默认约 5\~****6 分钟**，见 P30） |
| **根治** | 心跳与 lease 监控；资源预留与驱逐阈值；证书/时钟治理；关键服务跨区打散 |
| **防复发** | `kube_node_status_condition{condition="Ready",status="false"}` 告警 + kubelet `up` 告警 |
| **关联** | 【Kubelet 篇 · Q16 时间账】【控制面篇 · C4】 |

## P27 · `PLEG is not healthy: pleg was last seen active 3m ago`

| 项 | 内容 |
| --- | --- |
| **现象** | 节点上所有 Pod 集体异常（ContainerCreating/Unknown），节点 NotReady |
| **30 秒验证** | `journalctl -u kubelet ``\| grep -i pleg`；指标 `kubelet_pleg_relist_duration_seconds`；`crictl ps ``\| wc -l` |
| **常见根因** | ① 容器密度过高导致 relist 慢；② runtime（containerd）卡顿（磁盘 IO、shim 泄漏）；③ 节点 CPU 被抢（kubelet 自己没保障） |
| **止血** | cordon 节点；必要时重启 kubelet/containerd（**会中断该节点新操作**）；严重时重启节点 |
| **根治** | 降 Pod 密度；kube/system reserved 与 cgroup 真正生效；治理 runtime；较新版本评估 Evented PLEG |
| **防复发** | relist P99 \> 5s 告警（**早于 NotReady 触发**） |
| **关联** | 【Kubelet 篇 · Q4 PLEG / L2】 |

## P28 · kubelet 证书过期 / 节点无法注册

| 项 | 内容 |
| --- | --- |
| **现象** | kubelet 日志大量 TLS/x509 错误；节点不注册或 status 不更新 |
| **30 秒验证** | `journalctl -u kubelet ``\| grep -i "certificate``\|x509"`；`openssl x509 -in /var/lib/kubelet/pki/kubelet-client-current.pem -noout -dates` |
| **常见根因** | ① 客户端证书轮换未生效；② `serverTLSBootstrap` 开启但 CSR 无人批准；③ 节点时钟漂移；④ CA 轮换后未同步 |
| **止血** | 手动批准 CSR 或重新 bootstrap 节点（必要时 drain 后重建） |
| **根治** | 启用 `rotateCertificates`；确保 CSR 审批通道可用；时钟同步；证书有效期告警 |
| **防复发** | 证书剩余有效期 \< 30 天告警（**这条告警能救一次 P0**） |
| **关联** | 【Kubelet 篇 · Q20 证书】【控制面篇 · B9】 |

## P29 · MemoryPressure / DiskPressure 导致 Pod 被驱逐

| 项 | 内容 |
| --- | --- |
| **现象** | 节点 condition 出现压力，Pod 状态 Evicted |
| **30 秒验证** | `kubectl describe node ``\| grep -i pressure`；`kubectl get pod -A ``\| grep Evicted`；`dmesg` / 磁盘 `df -h`、`df -i` |
| **常见根因** | ① requests 虚低导致严重超卖；② 日志/临时文件暴涨；③ 镜像堆积；④ `evictionHard.memory.available` 默认 100Mi 触发太晚 |
| **止血** | 清理日志/镜像；扩容节点；把离线任务挪走 |
| **根治** | requests 校准；预留 + cgroup 生效；调大阈值 + 软驱逐 + 优雅期；日志轮转与 GC |
| **防复发** | 压力 condition 告警 + Evicted 增长告警 + 磁盘/inode 水位告警 |
| **关联** | 【Kubelet 篇 · Q12 驱逐 / L5 风暴】【Linux 篇 · C8 磁盘】 |

## P30 · 节点宕机后，Pod 迟迟不重建（约 5\~6 分钟）

| 项 | 内容 |
| --- | --- |
| **现象** | 节点挂了，业务副本很久才在其它节点起来 |
| **30 秒验证** | 算一遍：lease 10s → `node-monitor-grace-period` 50s → taint → Pod 默认 `tolerationSeconds=300` |
| **常见根因** | **这是设计行为，不是故障**；若有状态服务被误驱逐则是容忍配置过短 |
| **止血** | 想快：显式设更短的 tolerationSeconds（无状态 30\~60s） |
| **根治** | 无状态服务快恢复、有状态服务放宽容忍；配合 PDB 与副本跨区分布 |
| **防复发** | 明确写入容量方案："节点故障恢复时间 ≈ 5\~6 分钟（默认）"，并据此设定副本数与超时 |
| **关联** | 【Kubelet 篇 · Q16】【控制面篇 · C4】【Scheduler 篇 · 排障】 |

## P31 · 节点磁盘被打满

| 项 | 内容 |
| --- | --- |
| **现象** | DiskPressure、Pod 被逐、kubelet/containerd 异常 |
| **30 秒验证** | `df -h`、`du -sh /var/lib/containerd /var/lib/kubelet /var/log/pods`；`lsof ``\| grep deleted` |
| **常见根因** | ① 容器日志无轮转；② 镜像堆积（GC 阈值/频率）；③ 已删除文件被进程持有；④ 业务写 emptyDir/宿主目录 |
| **止血** | 触发镜像 GC；清理日志；重启持有 fd 的进程 |
| **根治** | 日志轮转参数 + 采集侧限流；imagefs 与 nodefs 分盘；ephemeral-storage limits |
| **防复发** | 分盘容量告警（imagefs/nodefs 分别）+ inode 告警 |
| **关联** | 【Kubelet 篇 · Q18 GC】【Linux 篇 · A18 overlayfs】 |

## P32 · inode 耗尽

| 项 | 内容 |
| --- | --- |
| **现象** | 报 No space left on device，但容量充足；新建文件失败 |
| **30 秒验证** | `df -i`；`find /var/lib/containerd -xdev -type f ``\| wc -l` |
| **常见根因** | 镜像层多 + 海量小文件；overlay 白出/合并导致 inode 增长 |
| **止血** | 清理镜像与无用层；迁移部分 Pod |
| **根治** | 镜像瘦身与层数收敛；小文件打包；inode 容量纳入机型选型 |
| **防复发** | inode 使用率告警 |
| **关联** | 【Linux 篇 · A18】【存储篇 · D12】 |

## P33 · kubelet 进程挂掉 / 调度器 lease 争抢失败导致调度暂停

| 项 | 内容 |
| --- | --- |
| **现象** | 新 Pod 全 Pending；或节点状态停止更新 |
| **30 秒验证** | `kubectl -n kube-system get lease kube-scheduler kube-controller-manager`；进程状态 `systemctl status kubelet` |
| **常见根因** | ① 进程 OOM/被驱逐；② leader election 抖动（控制面压力、网络）；③ 配置错误导致启动失败 |
| **止血** | 重启进程；确认只有一个 leader；必要时固定到健康实例 |
| **根治** | 给控制面组件保障资源与 APF 优先级；lease 变更告警；配置变更走灰度 |
| **防复发** | `up{job="kube-scheduler"}`、lease 更新停滞告警、pending\_pods 堆积告警 |
| **关联** | 【Scheduler 篇 · Q16 HA】【控制面篇 · C3】 |

## P34 · 节点时钟漂移

| 项 | 内容 |
| --- | --- |
| **现象** | 证书校验失败、日志错序、Lease 异常、分布式组件报错 |
| **30 秒验证** | `timedatectl`；`chronyc sources` / `ntpq -p`；`node_timex_offset_seconds` |
| **常见根因** | chrony/ntp 未配置或不可达；虚拟机时钟源不稳定；容器共享宿主时钟 |
| **止血** | 手动同步（谨慎，避免时间跳变影响有状态服务）；必要时重启 chrony |
| **根治** | 节点镜像固化 NTP 配置；监控偏移；避免使用会导致跳变的同步方式 |
| **防复发** | `abs(node_timex_offset_seconds) > 0.05` 告警 |
| **关联** | 【Linux 篇 · A25】【控制面篇 · 证书】 |

## P35 · 节点上所有 Pod 变成 Unknown

| 项 | 内容 |
| --- | --- |
| **现象** | Pod status.phase=Unknown，节点可能 NotReady |
| **30 秒验证** | Node Ready 状态 + kubelet 是否上报；`kubectl get pod -o wide` 看是否集中在一个节点 |
| **常见根因** | 节点失联（kubelet 挂/网络断/PLEG 崩），apiserver 保留 Pod 直到驱逐完成 |
| **止血** | 恢复节点；若节点不可恢复 ⇒ 走"强制移除 Pod"流程（**有状态服务必须先确认卷已释放**） |
| **根治** | 见 P26/P27；有状态服务要配 PDB 与卷释放流程 |
| **防复发** | 节点健康 + Pod 未知状态巡检 |
| **关联** | 【Kubelet 篇 · Q16】【存储篇 · Multi-Attach】 |

---

# 第 4 章 · 网络（P36–P46）

## P36 · DNS 间歇性 5 秒超时（经典）

| 项 | 内容 |
| --- | --- |
| **现象** | 偶发请求卡 5 秒后失败，概率低但持续存在 |
| **30 秒验证** | CoreDNS 侧是否收到请求（对比发起侧与服务端计数）；节点 `conntrack -S` 看 `insert_failed` |
| **常见根因** | **内核 conntrack + UDP NAT 竞态** ⇒ 包被丢弃；UDP 不重传 + resolver 超时 5s；search 域多、并发查询放大冲突概率 |
| **止血** | 应用侧改用 TCP 或加重试；临时降低并发 DNS 查询 |
| **根治** | **NodeLocal DNSCache**（本地 link-local 缓存，绕开 conntrack 竞态）；FQDN 写法 + 降 ndots；必要时升级内核 |
| **防复发** | CoreDNS SERVFAIL 与 P99 告警；NodeLocal 缓存命中率看板 |
| **关联** | 【网络篇 · A8 5 秒超时】【网络篇 · C3 DNS 优化】 |

## P37 · DNS 慢 / CoreDNS QPS 奇高

| 项 | 内容 |
| --- | --- |
| **现象** | 解析延迟上升；CoreDNS CPU 高 |
| **30 秒验证** | 看 Pod 内 `/etc/resolv.conf`（`ndots:5` 与 search 域）；`coredns_dns_requests_total` |
| **常见根因** | `ndots:5` 导致外部域名被放大 4\~10 倍查询；副本数不足；缓存未命中率高 |
| **止血** | CoreDNS 扩容 |
| **根治** | NodeLocal DNSCache（**同时解决延迟与 QPS**）；外部域名写 FQDN；按需调 ndots；autoscaler |
| **防复发** | QPS/延迟/缓存命中率告警 |
| **关联** | 【网络篇 · A7 / C3】 |

## P38 · CoreDNS 不可用 / SERVFAIL 飙升

| 项 | 内容 |
| --- | --- |
| **现象** | 全集群解析失败 |
| **30 秒验证** | `kubectl -n kube-system get pod -l k8s-app=kube-dns`；从 Pod 内 `nslookup kubernetes.default` |
| **常见根因** | ① CoreDNS Pod 被驱逐/资源不足；② 上游 forward 不可达；③ 配置错误（Corefile）；④ 节点网络问题 |
| **止血** | 恢复 CoreDNS 副本；临时切换上游 |
| **根治** | 多副本跨区 + PDB + 资源保障；NodeLocal 缓存兜底；配置变更灰度 |
| **防复发** | `up{job="coredns"}`、SERVFAIL 率、解析成功率合成探测 |
| **关联** | 【网络篇 · C3】【网络篇 · E3】 |

## P39 · Service 有 endpoints，但访问不通

| 项 | 内容 |
| --- | --- |
| **现象** | `kubectl get endpointslices` 有后端，但 curl ClusterIP 失败 |
| **30 秒验证** | ① 直接 curl PodIP（排除 Service 层）② 另一节点 curl PodIP（排除 CNI 跨节点）③ 看规则：`iptables-save ``\| grep <clusterIP>` 或 `cilium service list` |
| **常见根因** | ① kube-proxy 未同步（进程挂/规则不同步）；② CNI 跨节点不通；③ NetworkPolicy 拦截；④ 目标端口/协议错；⑤ hairpin 未开（访问自身 Service） |
| **止血** | 重启 kube-proxy；临时绕过（用 PodIP） |
| **根治** | 按分层逐段验证；监控规则同步与网络策略命中 |
| **防复发** | `kubeproxy_sync_proxy_rules_last_timestamp_seconds` 停滞告警 + 业务合成探测 |
| **关联** | 【网络篇 · B17 排障套路】【控制面篇 · D5 kube-proxy】 |

## P40 · `nf_conntrack: table full, dropping packet`

| 项 | 内容 |
| --- | --- |
| **现象** | 新建连接失败/超时，短连接高峰明显 |
| **30 秒验证** | `cat /proc/sys/net/netfilter/nf_conntrack_count` vs `_max`；`conntrack -S` 看 `insert_failed`/`drop` |
| **常见根因** | ① 表太小（默认值常不够）；② 短连接并发高 + timeout 过长；③ 大量 SNAT（NodePort/masquerade） |
| **止血** | 调大 `nf_conntrack_max`（注意 buckets 与内存）；缩短 TIME\_WAIT/CLOSE\_WAIT timeout |
| **根治** | 按并发实测规划容量；高并发短连接考虑 eBPF socket-LB 路径（绕开 conntrack） |
| **防复发** | 使用率 \> 75% 告警 + insert\_failed \> 0 告警 |
| **关联** | 【网络篇 · A18 / C5】【Linux 篇 · C2】 |

## P41 · kube-proxy 规则同步慢（Service 变更后不生效）

| 项 | 内容 |
| --- | --- |
| **现象** | Service/Endpoint 改了，但访问行为没变；节点 CPU 尖峰 |
| **30 秒验证** | `kubeproxy_sync_proxy_rules_duration_seconds`；`iptables-save ``\| wc -l`；`up{job="kube-proxy"}` |
| **常见根因** | ① iptables 模式规则数线性膨胀（Service/Endpoint 多）；② `minSyncPeriod` 设置不合理；③ kube-proxy 卡住/挂了 |
| **止血** | 重启 kube-proxy（已有规则不受影响） |
| **根治** | 切 IPVS 或 Cilium socket-LB；收敛 Service 数量 |
| **防复发** | 同步耗时 P99 告警 + 最后同步时间停滞告警 |
| **关联** | 【控制面篇 · D1–D7】【网络篇 · C2】 |

## P42 · Pod 拿不到 IP / IP 池耗尽

| 项 | 内容 |
| --- | --- |
| **现象** | `FailedCreatePodSandBox: failed to allocate for range`；或云 CNI 报 VPC IP 不足 |
| **30 秒验证** | CNI 侧地址池使用情况；节点 Pod 数 vs maxPods；云侧子网剩余 IP |
| **常见根因** | ① Pod CIDR/子网规划不足；② IPAM 泄漏（Pod 删除未释放）；③ 滚动更新期间新旧并存导致峰值翻倍 |
| **止血** | 回收泄漏 IP；临时减少副本或扩容网段（云上可加 CIDR/前缀委派） |
| **根治** | 容量规划（预期 Pod 数 × 2）；定期 IPAM 对账；开启前缀委派等提升密度的能力 |
| **防复发** | 地址池水位告警；`kubelet_running_pods / maxPods` 告警 |
| **关联** | 【网络篇 · L7】【Kubelet 篇 · maxPods】 |

## P43 · 大包失败、TLS 握手卡住（MTU 黑洞）

| 项 | 内容 |
| --- | --- |
| **现象** | ping 通、小请求正常，大文件/大 body 传输卡死 |
| **30 秒验证** | 从 Pod 内 `ping -M do -s <size>` 逐步找临界值；对比宿主与 Pod 的 MTU |
| **常见根因** | ① 隧道封装（VXLAN/IPIP/Geneve）后 MTU 未同步下调；② PMTUD 被 ICMP 过滤阻断；③ 配置漂移（部分节点不一致） |
| **止血** | 统一 MTU；开启 MSS clamp |
| **根治** | MTU 写入节点池基线；放通必要的 ICMP；变更后复测大包 |
| **防复发** | 大包连通性巡检 |
| **关联** | 【网络篇 · A11 / C10】 |

## P44 · 跨可用区 Pod 不通 / 延迟明显更高

| 项 | 内容 |
| --- | --- |
| **现象** | 同区正常，跨区失败或延迟高 |
| **30 秒验证** | 分别 curl 同区与跨区 PodIP；看路由与 CNI 模式（Overlay vs 路由） |
| **常见根因** | ① 纯路由模式（host-gw/BGP）跨三层不通；② 安全组/ACL 未放通 Pod CIDR；③ 跨区流量成本与延迟（架构问题） |
| **止血** | 改为 Overlay 或 CrossSubnet 模式；放通安全组 |
| **根治** | 按底层网络能力选型；用拓扑感知路由降低跨区流量；副本按区打散 |
| **防复发** | 跨区流量占比监控 + 连通性合成探测 |
| **关联** | 【网络篇 · A10 / C9】【网络篇 · B2】 |

## P45 · 加了 default-deny NetworkPolicy 后 DNS 全挂

| 项 | 内容 |
| --- | --- |
| **现象** | 策略上线后所有域名解析失败、健康检查挂 |
| **30 秒验证** | 删除策略是否恢复；看策略是否放行 kube-system 的 53/UDP+TCP |
| **常见根因** | default-deny 未放行 DNS 出口与关键探测来源；或 CNI 根本不支持 NetworkPolicy（策略"看起来生效"其实没生效） |
| **止血** | 立即删除策略 |
| **根治** | 分层策略：先 default-deny，再显式放行 DNS/探测/依赖；确认 CNI 支持策略 |
| **防复发** | 策略变更必须先在灰度 Namespace 验证 |
| **关联** | 【网络篇 · A9】【网络篇 · B14】 |

## P46 · 拿不到客户端源 IP / LB 流量不均

| 项 | 内容 |
| --- | --- |
| **现象** | 应用看到的是节点 IP 或 Pod IP；或流量集中到少数 Pod |
| **30 秒验证** | 看 Service 的 `externalTrafficPolicy`；看 LB 后端健康检查状态 |
| **常见根因** | ① `externalTrafficPolicy: Cluster` 会做额外 SNAT ⇒ 丢源 IP；② `Local` 模式下副本未按区打散 ⇒ 部分区无后端；③ gRPC/HTTP2 长连接导致连接级不均衡 |
| **止血** | 按需切换 `Local` 并补齐多区副本 |
| **根治** | 源 IP：Local + 多副本打散，或网关注入 header/PROXY protocol；均衡：限制 `MAX_CONCURRENT_STREAMS` 或用 per-request 网关 |
| **防复发** | 各副本请求量/连接数看板 |
| **关联** | 【网络篇 · A19】【GPU 篇 · C10 / E4】 |

# 第 5 章 · 控制面（P47–P53）

## P47 · kubectl 卡住 / API 延迟高 / 偶发 5xx

| 项 | 内容 |
| --- | --- |
| **现象** | 所有操作变慢；控制器大量重试；偶发 timeout |
| **30 秒验证** | `apiserver_request_duration_seconds` 按 verb/resource 分解；`apiserver_current_inflight_requests`；客户端侧 `rest_client_request_duration_seconds` |
| **常见根因** | ① **etcd 慢**（最常见真因）；② 某客户端大量 list（看 `sum by (user,resource)`）；③ APF 排队；④ 准入 webhook 慢；⑤ LB 把流量导到坏副本 |
| **止血** | 找出吵闹邻居并限流；摘掉坏 apiserver；必要时临时提高 inflight |
| **根治** | 治理大 list（informer + selector + 分页）；APF 给关键组件留额度；webhook 短超时与作用域；etcd 磁盘与容量 |
| **防复发** | 延迟 P99 + 5xx 比例 + APF 拒绝 + inflight 告警 |
| **关联** | 【控制面篇 · A14 / E1】【控制面篇 · E2】 |

## P48 · 写操作全部失败：`mvcc: database space exceeded`（etcd NOSPACE）

| 项 | 内容 |
| --- | --- |
| **现象** | 创建/删除对象失败，但读通常还能进行 |
| **30 秒验证** | `etcdctl endpoint status --write-out=table --cluster`；`etcdctl alarm list` |
| **常见根因** | 配额满（未压缩/未 defrag）⇒ 进入只读；Event 或高频更新放大 |
| **止血** | 依次执行：`compact <rev>` → `defrag --cluster`（逐成员）→ `alarm disarm` |
| **根治** | 自动压缩 + `event-ttl` + 独立 events etcd；调大配额；容量与增长监控 |
| **防复发** | `etcd_db_total_size_in_bytes / quota > 0.8` 告警 |
| **关联** | 【控制面篇 · B3 / E6】 |

## P49 · etcd 慢 / 频繁切主

| 项 | 内容 |
| --- | --- |
| **现象** | 全集群 API 延迟升高；日志 `apply request took too long`、`heartbeat took too long` |
| **30 秒验证** | `etcd_disk_wal_fsync_duration_seconds` P99（建议 \<10ms）、`etcd_disk_backend_commit_duration_seconds`（\<25ms）；`etcd_network_peer_round_trip_time_seconds`；`etcd_server_leader_changes_seen_total` |
| **常见根因** | ① 磁盘慢/共享盘/节流盘；② 网络 RTT 高（跨地域部署是硬伤）；③ 负载过高（大对象、频繁写） |
| **止血** | 降低写入量（限流吵闹客户端、清理无用对象）；必要时切换 leader 到健康成员 |
| **根治** | 独立 NVMe；同城低延迟组网；对象体积治理；成员数 3/5 且奇数 |
| **防复发** | fsync/commit P99 告警 + leader 变更次数告警 |
| **关联** | 【控制面篇 · B4 / B6】 |

## P50 · 409 Conflict（更新冲突）

| 项 | 内容 |
| --- | --- |
| **现象** | 控制器/CI 更新对象时报冲突 |
| **30 秒验证** | 看是否携带 `metadata.resourceVersion`；是否有多个 writer |
| **常见根因** | 乐观并发：**带上过期的 resourceVersion 更新**；或多个控制器同时写同一对象 |
| **止血** | 重试（重新 Get 再 Update） |
| **根治** | 客户端使用 client-go 的重试语义；避免多写者；用 patch/apply 而非全量 update |
| **防复发** | 控制器 409 计数监控 |
| **关联** | 【控制面篇 · A6 乐观锁 / C1 控制器范式】 |

## P51 · `too old resource version` / watch 频繁断开（410）

| 项 | 内容 |
| --- | --- |
| **现象** | 客户端报错 410 Gone，informer 不断 relist |
| **30 秒验证** | 客户端日志；etcd 压缩情况（revision 保留策略） |
| **常见根因** | ① watch 的 resourceVersion 已被压缩；② 客户端断线过久后从旧版本续接；③ 压缩保留设置过短 |
| **止血** | 客户端全量 relist（正常行为）；缩短重连间隔 |
| **根治** | 客户端正确处理 410；开启 watch bookmark；压缩保留策略与客户端行为匹配 |
| **防复发** | 客户端 relist 频率与 410 计数监控 |
| **关联** | 【控制面篇 · A6】【控制面篇 · E2】 |

## P52 · 准入 webhook 挂掉导致全集群变更失败

| 项 | 内容 |
| --- | --- |
| **现象** | 所有 create/update 被拒，报 webhook 连接失败 |
| **30 秒验证** | `kubectl get validatingwebhookconfigurations,mutatingwebhookconfigurations`；看 `failurePolicy` 与后端 Pod |
| **常见根因** | ① `failurePolicy: Fail` + 后端不可用；② webhook 作用于所有 namespace（含 kube-system）；③ 超时过长拖垮 apiserver |
| **止血** | **立即删除/禁用该 webhook 配置**（这是最快的恢复手段） |
| **根治** | `namespaceSelector` 排除系统 ns；多副本跨故障域；短超时；后端健康告警 |
| **防复发** | webhook 耗时与错误告警；变更前的故障演练 |
| **关联** | 【控制面篇 · A4 / F2】 |

## P53 · APF 拒绝请求 / 关键组件被饿死

| 项 | 内容 |
| --- | --- |
| **现象** | 部分请求被拒（429 或排队超时）；kubelet/scheduler 心跳异常 |
| **30 秒验证** | `apiserver_flowcontrol_rejected_requests_total` 按 priority\_level；`apiserver_flowcontrol_current_inqueue_requests` |
| **常见根因** | ① 未配 APF，总量限流被吵闹邻居占满；② FlowSchema 配错，把关键流量分到低优先级；③ 队列长度不足 |
| **止血** | 修正/删除错误的 FlowSchema；给关键组件临时保障 |
| **根治** | 为 leader-election、kubelet、scheduler、探针建专用优先级；对全集群 list 类请求降优先级 |
| **防复发** | APF 拒绝率与队列深度告警 |
| **关联** | 【控制面篇 · A7 / E4】 |

---

# 第 6 章 · 存储（P54–P57）

## P54 · PVC 一直 Pending / `ProvisioningFailed`

| 项 | 内容 |
| --- | --- |
| **现象** | PVC 不绑定，Pod 起不来 |
| **30 秒验证** | `kubectl describe pvc`（事件与 SC 名）；`kubectl -n <csi-ns> logs <csi-provisioner>`；`kubectl get sc` 确认 provisioner 名一致 |
| **常见根因** | ① StorageClass 名拼错或不存在；② provisioner 未运行/选主失败；③ 后端配额/凭据不足；④ 拓扑不满足（AZ） |
| **止血** | 修正 SC；恢复 provisioner；补配额 |
| **根治** | CSI Controller 多副本 + leader election + PDB；SC 模板化；后端容量告警 |
| **防复发** | PVC Pending 时长告警；事件转指标（ProvisioningFailed） |
| **关联** | 【存储篇 · A6 / B18 / C5】 |

## P55 · `FailedAttachVolume` 与 `FailedMount`

| 项 | 内容 |
| --- | --- |
| **现象** | Pod 卡 ContainerCreating，卷不可用 |
| **30 秒验证** | **看事件字段名即可分界**：`FailedAttach*` = attach 层（控制面/云 API）；`FailedMount` = kubelet/CSI 节点侧 |
| **常见根因** | attach：云配额/限流、AZ 不一致、节点挂载数上限、上次 detach 未完成；mount：node plugin 崩、文件系统损坏、mount 参数错 |
| **止血** | attach 层：核对 AZ 与配额、等待/清理残留 VolumeAttachment；mount 层：重启 node plugin、检查文件系统 |
| **根治** | WaitForFirstConsumer + 拓扑；分批重建有状态服务；node plugin 优雅发布 |
| **防复发** | `storage_operation_duration_seconds` 告警 + 事件转指标 |
| **关联** | 【存储篇 · B18 / C3】 |

## P56 · 卷挂载耗时十几分钟

| 项 | 内容 |
| --- | --- |
| **现象** | Pod 起来极慢，Events 时间戳显示 mount 阶段耗时巨大 |
| **30 秒验证** | 看 Events 时间戳差；`storage_operation_duration_seconds{operation_name=~"volume_mount``\|volume_attach"}`；节点 `journalctl -u kubelet` |
| **常见根因** | ① **fsGroup 递归 chown**（大卷文件多）；② 上一次 detach 未完成；③ 云 API 限流；④ 文件系统检查/修复；⑤ node plugin 崩 |
| **止血** | 等待或强制清理残留；分批重建 |
| **根治** | `fsGroupChangePolicy: OnRootMismatch`；规范 detach 流程（先确认 VolumeAttachment 消失）；限流调优 + 分批 |
| **防复发** | 挂载耗时 P99 告警 |
| **关联** | 【存储篇 · A19 fsGroup / C3】【Kubelet 篇 · 卷管理】 |

## P57 · PV/PVC 卡 Terminating / 孤儿磁盘（含扩容不生效）

| 项 | 内容 |
| --- | --- |
| **现象** | 删不掉；或扩容后容器内视图没变；或账单里躺着没人认的磁盘 |
| **30 秒验证** | `kubectl get pvc -o jsonpath='{.metadata.finalizers}'`；`kubectl get pv -o wide` 与后端磁盘列表对账；扩容场景看 PVC `status.conditions` 是否 `FileSystemResizePending` |
| **常见根因** | ① finalizer（`pvc-protection`/`pv-protection`）未释放；② **摘掉 finalizer 强删后后端资产变孤儿**；③ 扩容只完成磁盘未完成文件系统（卷未挂载）；④ StatefulSet 删除/缩容不删 PVC |
| **止血** | 释放使用方后删除；扩容场景重新挂载触发 NodeExpand；孤儿盘：对账后走工单清理 |
| **根治** | PV ↔ 后端磁盘定期双向对账；`reclaimPolicy: Retain` 要有巡检；StatefulSet 生命周期流程化；标签与成本归因 |
| **防复发** | 孤儿资源月度巡检；Dead PV 告警 |
| **关联** | 【存储篇 · A12 Finalizer / A15 扩容 / A22 泄漏 / C13】 |

---

# 第 7 章 · GPU 与大模型（P58–P60）

## P58 · 训练 hang 在 NCCL 初始化 / 第一步

| 项 | 内容 |
| --- | --- |
| **现象** | 任务启动后卡住不动，无报错或只报 NCCL timeout |
| **30 秒验证** | 打开 `NCCL_DEBUG=INFO NCCL_DEBUG_SUBSYS=INIT,ENV,GRAPH`；确认 `NCCL_SOCKET_IFNAME` / `NCCL_IB_HCA` 是否选对网卡；节点 `ibstat` / `rdma link` |
| **常见根因** | ① **选错网卡**（多网卡机器最常见）；② RDMA 链路/MTU/GID 配置问题；③ 未启用 GDR 但配置期望它；④ rank 数量与 rendezvous 不一致 |
| **止血** | 显式指定正确的网卡与环境变量；必要时先退回 hostNetwork/单节点验证 |
| **根治** | 把 NCCL 环境变量固化到作业模板；网络基线（nccl-tests busbw）上线前验收；CNI 方案固定（勿让 RDMA 走隧道） |
| **防复发** | 上线/变更后必跑 nccl-tests 基线；训练 step time 异常告警 |
| **关联** | 【GPU 篇 · C3 / C6 / C8】 |

## P59 · 训练变慢 / MFU 低 / 出现 straggler / 坏卡

| 项 | 内容 |
| --- | --- |
| **现象** | 整体吞吐下降；个别 rank 明显慢；或节点报 XID/ECC |
| **30 秒验证** | DCGM：`DCGM_FI_PROF_SM_ACTIVE`/`_PIPE_TENSOR_ACTIVE`、温度/功耗/SM clock、`DCGM_FI_DEV_XID_ERRORS`；网络：丢包与 nccl-tests；对比各 rank 的 step time |
| **常见根因** | ① 降频/限功率/温度墙；② NVLink/网卡链路降级；③ straggler 节点；④ 数据/存储 IO 拖累；⑤ 并行策略或 batch 配置不佳 |
| **止血** | cordon 可疑节点；把受影响作业重排；必要时重启驱动/节点 |
| **根治** | 坏卡自动检测 + 隔离 + 维护流程；网络与 IO 基线；并行策略与算子优化（FlashAttention、重计算权衡、通信 overlap） |
| **防复发** | XID/ECC 告警；SM/Tensor active 低水位告警；step time 分位数告警 |
| **关联** | 【GPU 篇 · A13 / B15 / C9 / F1–F4】 |

## P60 · 推理服务：冷启动太慢 / 副本负载不均 / KV cache 打满

| 项 | 内容 |
| --- | --- |
| **现象** | 扩容后很久才可用；某些副本被打爆；请求被拒或延迟飙升 |
| **30 秒验证** | 看 Pod 就绪时长（权重加载）；按副本统计请求数/连接数；推理引擎指标（排队长度、KV cache 使用率） |
| **常见根因** | ① **权重加载慢 + 探针阈值太严**（启动即被杀/迟迟不 Ready）；② gRPC/HTTP2 **连接级负载不均**；③ KV cache 打满导致拒绝/抢占；④ 网关超时与响应缓冲（SSE 被缓存） |
| **止血** | 放大 startupProbe 阈值与 grace；减少单连接并发（限制 `MAX_CONCURRENT_STREAMS`）或换 per-request 网关；限流降级 |
| **根治** | 权重预热（本地缓存/预取）+ 常驻最小副本；推理网关做模型/队列/缓存感知路由；网关禁用响应缓冲并放大超时；按 KV cache 与排队长度扩缩 |
| **防复发** | TTFT/TPOT 分桶 SLO；各副本请求量看板；排队长度与 KV cache 告警 |
| **关联** | 【GPU 篇 · C10 / C11 / E2–E5】 |

---

# 附录 A · 出事第一时间要抓的"证据清单"

```bash
# ① 对象侧（30 秒内）
kubectl get pod/pvc/node -o wide
kubectl describe pod <pod>              # Events 是最高性价比证据
kubectl logs <pod> --previous           # CrashLoop 必看
kubectl get events -n <ns> --sort-by=.lastTimestamp | tail -40

# ② 节点侧
systemctl status kubelet && journalctl -u kubelet -n 300
dmesg -T | tail -50                     # OOM / XID / 硬件
crictl ps -a | head -30
cat /sys/fs/cgroup/<path>/memory.events # cgroup OOM 计数
cat /sys/fs/cgroup/<path>/cpu.stat      # throttling

# ③ 网络侧
nsenter -t <pid> -n ip a / ss -lntp / tcpdump
cat /proc/sys/net/netfilter/nf_conntrack_count
kubectl -n kube-system logs -l k8s-app=kube-dns

# ④ 控制面
kubectl -n kube-system get lease
kubectl get --raw /metrics | grep apiserver_request_duration
etcdctl endpoint status --write-out=table --cluster ; etcdctl alarm list

# ⑤ 存储侧
kubectl get pvc,pv,volumeattachment -A
kubectl describe pvc <pvc>
kubectl -n <csi-ns> logs <csi-provisioner> <csi-attacher>

# ⑥ GPU 侧
nvidia-smi
nvidia-smi -q -d TEMPERATURE,POWER,UTILIZATION,MEMORY
dmesg -T | grep -i -E "NVRM|Xid"
NCCL_DEBUG=INFO ... （重跑一次最小复现）
```

**保留现场清单**：Events 截图/导出、Pod YAML（含 finalizers/status）、容器日志（含上一实例）、节点 dmesg、监控面板时间窗、etcd 快照（控制面类故障）。

---

# 附录 B · 最容易被误判的 10 件事

| # | 误判 | 真相 |
| --- | --- | --- |
| 1 | "Pending 就是资源不够" | 也可能是 **schedulerName 写错 / 调度器没跑**（完全没有 Event） |
| 2 | "GPU 利用率 100% 说明跑满了" | 可能是空转；要看 **SM/Tensor active + MFU** |
| 3 | "CPU 使用率不高说明没瓶颈" | 可能正被 **CFS throttling** 卡住 |
| 4 | "没超内存怎么会被 OOM" | **page cache 计入 cgroup memory** |
| 5 | "节点挂了 Pod 会马上重建" | 默认约 **5\~****6 分钟**（grace 50s + toleration 300s） |
| 6 | "删了容器磁盘就该释放" | 可能有进程持有已删除文件（`lsof ``\| grep deleted`），或 **inode 没释放** |
| 7 | "No space left 就是容量满了" | 可能是 **inode 耗尽** 或 `/dev/shm` 太小 |
| 8 | "NetworkPolicy 配了就一定生效" | **Flannel 不支持**；CNI 不实现就完全不生效 |
| 9 | "PV/PVC 删不掉是 bug" | 是 **finalizer** 在保护；强删会留下孤儿磁盘 |
| 10 | "apiserver 慢就是 apiserver 的问题" | 绝大多数是 **etcd 磁盘慢** 或某个客户端在疯狂 list |

---

# 附录 C · 复盘模板（一页纸）

```
1. 影响：时间窗、受影响服务/租户、SLO 消耗、错误预算消耗比例
2. 时间线：发现 → 定位 → 止血 → 恢复（关键动作与耗时）
3. 根因：直接原因 / 根本原因 / 为什么监控没提前发现
4. 为什么发生：流程（变更？容量？设计）+ 技术（代码/配置/依赖）
5. 改进项：
   - 检测（新增/优化告警，含阈值与验证方式）
   - 预防（配置校验、CI 门禁、策略、容量规划）
   - 恢复（手册/脚本/演练，明确 RTO）
   - 防扩散（灰度、限流、隔离、回滚）
6. 每项改进：负责人 + 截止时间 + 验收方式
7. 演练计划：如何验证改进真的有效（GameDay 场景）
```

---

# 附录 D · 最小可用告警集（先有这 20 条再谈治理）

| 领域 | 告警 |
| --- | --- |
| 节点 | Node NotReady、Node Pressure（Mem/Disk/PID）、kubelet `up`、PLEG relist P99、证书剩余有效期 |
| Pod/容器 | Pending 超时、CrashLoop/重启次数、OOMKilled、Evicted、Terminating 超时、CPU throttled 占比 |
| 调度 | `scheduler_pending_pods` 堆积、调度失败率、抢占次数、无 Event 的 Pending Pod |
| 网络 | CoreDNS 延迟/ SERVFAIL/ `up`、conntrack 使用率与 insert\_failed、kube-proxy 同步停滞、CNI NetworkUnavailable、IP 池水位 |
| 控制面 | apiserver 延迟 P99 / 5xx / `up`、etcd fsync+commit P99、etcd 容量比、etcd leader 变更、APF 拒绝 |
| 存储 | PVC Pending、卷容量/inode 水位、卷操作耗时 P99、VolumeAttachment 残留 |
| GPU | XID/ECC、SM active 低水位、显存水位、推理排队长度与 KV cache 使用率 |

> **原则**：告警要有明确"下一步动作"。没有行动项的告警等于噪音，最终会被静音——**静音的告警比没有告警更危险**。
