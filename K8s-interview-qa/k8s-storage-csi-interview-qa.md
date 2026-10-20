# k8s-storage-csi-interview-qa

# Kubernetes 存储 / CSI 面试题全解析：原理篇 × 方案模式篇 × 大规模优化篇

> 适用角色：SRE / 云原生运维 / 平台工程师面试（社招 3 年起 + 偏高阶）
> 姊妹篇：《Scheduler 篇》《Kubelet 篇》《网络 / CNI 篇》《控制面篇》
> 版本约定：以 **1.28 \~**** 1.31 主线**为准；凡涉及默认值、feature gate、GA/Beta 状态、flag 与 sidecar 版本的地方，一律标注"以所用版本官方文档 / Release Notes 为准"。

---

## 0. 使用说明

### 0.1 存储方向的考察层次

| 层级 | 考点 | 表现 |
| --- | --- | --- |
| L1 会用 | 会写 PV/PVC/StorageClass YAML | 只会 apply |
| L2 懂原理 | 说得清 PV 绑定状态机、动态供给链路、CSI 三个 gRPC 服务、attach/stage/publish 三段 | 能画全链路图 |
| L3 懂选型 | 能比较块/文件/对象、各 CSI 驱动能力矩阵、AZ 拓扑、访问模式 | 能说 trade-off |
| L4 有事故经验 | 能讲"Pod 挂着卷 20 分钟起不来""有状态服务跨可用区调度失败""卷删不掉"的根因与治理 | 有排查命令、有灰度、有回滚 |

> **答题模板**：先说这条链路上的**四个参与者**（PV Controller / CSI Controller / kubelet / CSI Node），再说各自做的事，最后给证据（配置、对象字段、metric、日志/事件）。

### 0.2 一句话区分四个层次

| 层次 | 对象 | 谁定义 | 生命周期 |
| --- | --- | --- | --- |
| Pod 内的 Volume 声明 | `spec.volumes` | 应用 | 与 Pod 一致 |
| 用户申请 | **PVC** | 应用/平台 | 独立于 Pod（关键！） |
| 集群资源 | **PV** | 管理员/动态供给 | 独立于 Namespace |
| 供给模板 | **StorageClass** | 平台 | 长期 |
| 真实实现 | **CSI driver** | 存储厂商/社区 | 独立于以上全部 |

### 0.3 数值约定

- **源码/官方默认值**：标注字段，提示现场核对；
- **建议值**：标为 *建议*，必须压测校准；
- **实测值**：本文档不提供任何未经实测的性能数字（IOPS/吞吐/延迟），一律给出**测量方法**。

---

# Part A · 原理篇（24 题）

## A1. K8s 存储的四种抽象各自解决什么问题？

| 抽象 | 解决的问题 | 关键点 |
| --- | --- | --- |
| **Volume（Pod 内声明）** | 让 Pod 能用上各种"卷"（emptyDir/configMap/secret/csi/pvc…） | 与 Pod 生命周期一致 |
| **PV** | 把"具体的存储资源"抽象成集群对象（容量/访问模式/回收策略/后端信息） | **集群级资源**，不属于某个 namespace |
| **PVC** | 让应用"只要提需求"（多少 GiB、什么访问模式、哪个 SC），不用关心后端 | 与 PV **一对一绑定** |
| **StorageClass** | 定义"如何供给"（provisioner、参数、回收策略、绑定模式） | 支持**动态供给**，是平台能力的边界 |

**必答**：PV 与 PVC 的绑定是 **1:1 且独占** 的；PVC 的生命周期**不跟随 Pod**（这是有状态服务能重建的基础，也是"磁盘泄漏"的来源）。

## A2. PV 的状态机是怎么走的？各状态代表什么？

```
                 静态创建 or 动态供给
                        │
                        ▼
                  [Available] ────── 被 PVC 绑定 ─────► [Bound]
                        ▲                                 │
                        │                            删除 PVC
                        │                                 ▼
                        │                            [Released]
                        │                                 │
                   回收后变可用                    reclaimPolicy=Delete → 删除后端资产 → PV 消失
                   (Retain 需人工清理)             reclaimPolicy=Retain  → 保留 PV 与数据（人工处理）
                                                        │
                                                   自动回收失败 → [Failed]
```

| PV 状态 | 含义 |
| --- | --- |
| `Available` | 空闲，未被任何 PVC 绑定 |
| `Bound` | 已绑定到某个 PVC |
| `Released` | PVC 已删除，但后端存储资产还在回收中 |
| `Failed` | 自动回收失败（多为后端 API 报错/凭据失效） |

**PVC 侧状态**：`Pending`（未绑定，最常见）/ `Bound` / `Lost`（PV 丢失或异常）。

**必答的回收策略**：

- `Retain`：删除 PVC 后 PV 保留，**数据还在**（合规/防误删常选），但没人自动清理 ⇒ 容易堆积；
- `Delete`：删除 PVC ⇒ 删除 PV ⇒ 调 CSI `DeleteVolume` **真删数据**（默认行为要看 StorageClass 的 `reclaimPolicy`）；
- `Recycle`：**已废弃**，不要再说它是推荐做法。

## A3. StorageClass 里每个字段都会问什么？

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: fast-ssd
provisioner: ebs.csi.aws.com          # ① CSI 驱动名（必须与 CSIDriver 对象一致）
parameters:                            # ② 传给 CreateVolume 的参数（驱动自定义）
  type: gp3
  iops: "5000"
  throughput: "250"
  encrypted: "true"
  csi.storage.k8s.io/fstype: ext4
reclaimPolicy: Delete                  # ③ Delete | Retain
volumeBindingMode: WaitForFirstConsumer # ④ Immediate | WaitForFirstConsumer（★ 高频考点）
allowVolumeExpansion: true             # ⑤ 是否允许扩容
mountOptions:                          # ⑥ 挂载参数
  - noatime
  - nodiratime
allowedTopologies:                     # ⑦ 拓扑约束（配合 CSI 的 accessibleTopology）
  - matchLabelExpressions:
      - key: topology.ebs.csi.aws.com/zone
        values: ["cn-north-1a", "cn-north-1b"]
```

**必答**：

- `WaitForFirstConsumer`** 几乎是所有支持的 StorageClass 的推荐默认**，否则会出现"PV 先落在一个 AZ，Pod 却被调度到另一个 AZ" ⇒ 永久 Pending；
- StorageClass 是**不可变为主**的资源：修改它**不会影响已创建的 PV**（很多人误以为能"改参数生效"）；
- 想给不同租户不同配额 ⟹ 用不同 StorageClass + ResourceQuota 的 `xxx.storageclass.storage.k8s.io/persistentvolumeclaims`。

## A4. AccessMode 四种模式到底限制什么？

| 模式 | 含义 | 典型 |
| --- | --- | --- |
| `ReadWriteOnce` (RWO) | 可被**一个节点**读写挂载 | 云盘、块存储 |
| `ReadOnlyMany` (ROX) | 可被**多节点只读**挂载 | 只读数据集 |
| `ReadWriteMany` (RWX) | 可被**多节点读写**挂载 | NFS / CephFS / EFS / NAS |
| `ReadWriteOncePod` (RWOP) | **整个集群只能一个 Pod** 挂载（比 RWO 更严格） | 需要严格单写互斥的有状态服务（版本相关，请核对） |

**必答的两个真相**：

1. AccessMode 是**声明**，K8s **不强制**执行（除了 RWOP 会在 kubelet 侧做 Pod 级限制）：驱动不支持 RWX 你写了 RWX，结果就是**多节点挂载失败**或**静默数据损坏**；
2. **RWO ≠ 只能一个 Pod**：同一节点上的多个 Pod 可以同时挂载同一个 RWO PVC（因为都在一个节点）；要防止"第二个 Pod 也挂上来"，必须用 **RWOP**。

## A5. volumeMode：Filesystem 与 Block 怎么选？

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
spec:
  volumeMode: Block            # Filesystem（默认）| Block
  resources:
    requests:
      storage: 100Gi
---
# Pod 侧使用方式完全不同
spec:
  containers:
    - name: app
      volumeDevices:           # 注意：不是 volumeMounts
        - name: data
          devicePath: /dev/xvda
  volumes:
    - name: raw
      persistentVolumeClaim:
        claimName: block-pvc
```

| 模式 | 特点 | 场景 |
| --- | --- | --- |
| `Filesystem`（默认） | kubelet 做 mkfs + mount | 绝大多数应用 |
| `Block` | 把裸设备透传给容器 | 数据库自己管文件系统、需要 Direct IO、部分 RDMA/高性能场景 |

**必答**：Block 模式下 kubelet **不创建文件系统也不 mount**，由应用自己处理；扩容/快照行为也与文件系统模式不同。

## A6. 一次"动态供给"的完整链路是什么？（核心大题）

```
① 用户创建 PVC（指定 StorageClass）
② PV controller 发现有 StorageClass → 标记 PVC 为 Pending，等待 provisioner
③ external-provisioner sidecar watch 到 PVC
       → 调 CSI ControllerService.CreateVolume(capacity, parameters, accessibilityRequirements, secrets)
④ 存储后端真的创建磁盘；pv provisioner 创建 PV 对象并绑定 PVC
⑤ Pod 被调度到某个节点（若 WaitForFirstConsumer，则 CreateVolume 会带上该节点的拓扑信息）
⑥ 若驱动需要 attach：external-attacher 调 ControllerPublishVolume → 云上把盘挂给这台虚拟机
⑦ kubelet VolumeManager 发现卷已就绪
       → NodeStageVolume（格式化 + mount 到 staging 路径，仅支持 STAGE_UNSTAGE 能力的驱动）
       → NodePublishVolume（bind mount 到 /var/lib/kubelet/pods/<uid>/volumes/...）
⑧ 容器启动，卷可用
⑨ 删除 Pod → NodeUnpublishVolume → NodeUnstageVolume → ControllerUnpublishVolume
⑩ 删除 PVC → 依 reclaimPolicy → DeleteVolume → PV 消失
```

**必答**：这条路上有 **4 类故障点**：`provisioner 没起来 / 后端配额不足`（停在第 3-4 步）、`AZ 拓扑不匹配`（停在第 5 步）、`attach 失败`（第 6 步，`FailedAttachVolume`）、`mount 失败`（第 7 步，`FailedMount`）。**Events 里的字段名就是最好的定位线索**。

## A7. CSI 有哪些核心 sidecar？各自干什么？（必考）

| sidecar | 干什么 | 跑在哪 |
| --- | --- | --- |
| **external-provisioner** | watch PVC → CreateVolume / DeleteVolume | Controller（Deployment） |
| **external-attacher** | watch VolumeAttachment → ControllerPublish/UnpublishVolume | Controller |
| **external-resizer** | watch PVC 扩容请求 → ControllerExpandVolume | Controller |
| **external-snapshotter** | watch VolumeSnapshot → CreateSnapshot / DeleteSnapshot | Controller |
| **external-health-monitor**（controller/agent） | 卷健康检测 → 上报 Abnormal 事件 | Controller / Node |
| **node-driver-registrar** | 向 kubelet 注册 driver socket → 生成 CSINode 对象 | Node（DaemonSet） |
| **livenessprobe** | 探活 CSI socket，暴露健康状态 | Controller + Node |

**为什么需要这么多 sidecar**：CSI 只定义了 gRPC 接口，**观看 K8s 对象并调用 CSI 这部分"胶水"交给标准 sidecar**，各驱动只需实现存储语义。

## A8. ControllerPublish / NodeStage / NodePublish 三者必须分得清（必考）

| 阶段 | 调用者 | 含义 | 失败时的事件 |
| --- | --- | --- | --- |
| `ControllerPublishVolume` | external-attacher（控制面） | **把磁盘挂给这台虚拟机**（云 API 层） | `FailedAttachVolume` |
| `NodeStageVolume` | kubelet（节点） | **格式化 + 挂到节点 staging 路径**（一次操作，可被多 Pod 复用） | `FailedMount` |
| `NodePublishVolume` | kubelet（节点） | **bind mount 到具体 Pod 目录**（每 Pod 一次） | `FailedMount` |

**好处**：多个 Pod 共享同一个节点上的卷时，**stage 只做一次**，publish 多次 ⇒ 显著减少昂贵操作。

**必答**：并不是所有驱动都需要 stage/unstage（`STAGE_UNSTAGE_VOLUME` capability）；块存储通常需要，文件存储/NFS 可能跳过。

## A9. 三个 gRPC 服务（Identity / Controller / Node）各自的能力？

| Service | 主要 RPC | 部署位置 |
| --- | --- | --- |
| **Identity** | GetPluginInfo、GetPluginCapabilities、Probe | 两侧都有 |
| **Controller** | CreateVolume / DeleteVolume / ControllerPublishVolume / ControllerUnpublishVolume / ValidateVolumeCapabilities / ListVolumes / GetCapacity / ControllerExpandVolume / CreateSnapshot / ListSnapshots | 通常只有 Controller 实例 |
| **Node** | NodeStageVolume / NodeUnstageVolume / NodePublishVolume / NodeUnpublishVolume / NodeGetInfo / NodeGetCapabilities / NodeGetVolumeStats / NodeExpandVolume | 每个节点 |

**必考追问**：Node GetInfo 返回什么？
→ **nodeID**（存储视角的节点标识）与 **accessibleTopology**（拓扑段，如 `topology.ebs.csi.aws.com/zone=cn-north-1a`），它们会写进 **CSINode 对象**，是调度器做拓扑决策的依据。

## A10. WaitForFirstConsumer 与 CSI 拓扑的完整流程（和调度器联动）

```
① PVC 创建，StorageClass.volumeBindingMode = WaitForFirstConsumer
   → PV controller 不立即供给，只标记 PVC annotation "selected node pending"
② Pod 创建 → 调度器介入：
     - VolumeBinding 插件查：该 PVC 未绑定 ⇒ 进入"延迟绑定"分支
     - Filter 阶段：遍历节点，看每个节点的**拓扑 labels** 是否落在 CSI 驱动的
       accessibleTopology / StorageClass.allowedTopologies 内
     - 不满足 ⇒ 该节点被过滤 ⇒ Events: "node(s) had volume node affinity conflict"
③ 节点选定后 → provisioner 带上该节点拓扑去 CreateVolume
④ PV 创建并完成绑定 → kubelet 才开始 attach + mount
```

**必答**：这就是为什么 **延迟绑定 + 单可用区云盘** 是标准搭配；Immediate 模式下"PV 先落一 AZ、Pod 后落到另一 AZ"的经典事故就是这么来的。

## A11. CSINode 对象为什么重要？

```yaml
apiVersion: storage.k8s.io/v1
kind: CSINode
metadata:
  name: node-1
spec:
  drivers:
    - name: ebs.csi.aws.com
      nodeID: i-0123456789
      topologyKeys: ["topology.ebs.csi.aws.com/zone"]
      allocatable:
        count: 39            # 该节点还能挂多少卷（版本相关）
```

**作用**：

- 记录**该节点上注册了哪些驱动**、nodeID、拓扑键；
- 调度器用它判断节点能否提供该 CSI 的卷；
- **节点上的 node-driver-registrar 挂了 ⇒ CSINode 不更新 ⇒ 新卷挂载全部失败**（已有挂载通常不受影响）；
- 节点 drain / 重建时若 CSINode 与真实能力不一致，会造成调度误判。

## A12. 为什么 PVC / PV 会卡在 Terminating？（Finalizer 必考）

`StorageObjectInUseProtection`** 准入插件** 会自动给对象加 finalizer：

- PVC：`kubernetes.io/pvc-protection`（正在被 Pod 使用时不允许删除）
- PV：`kubernetes.io/pv-protection`

**因此**：

| 现象 | 原因 |
| --- | --- |
| PVC 一直 Terminating | 还有 Pod 在用（包括已 Terminating 的 Pod） |
| PV 一直 Terminating | PVC 还在，或 storage 后端删除卡住 |
| 强删后残留 | finalizer 被手动摘掉后，后端资产**不会自动清理** ⇒ 产生"孤儿磁盘"（费钱！） |

```bash
# 正确排障
kubectl get pvc pvc-1 -n prod -o jsonpath='{.metadata.finalizers}'
kubectl get pods --all-namespaces -o json | jq '..|.volumes[]?.persistentVolumeClaim.claimName' | grep pvc-1
# 极端情况（工单级操作，注意后果）
kubectl patch pvc pvc-1 -n prod -p '{"metadata":{"finalizers":[]}}' --type=merge
```

## A13. 卷快照是怎么实现的？

```
VolumeSnapshotClass（模板）
      +
VolumeSnapshot（用户申请）  ──► VolumeSnapshotContent（实际绑定）
      │
      └─ snapshot-controller + external-snapshotter
              ├─ 调 CSI CreateSnapshot → 返回 snapshotID
              └─ 创建 VolumeSnapshotContent 并与 VolumeSnapshot 绑定
```

```yaml
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshotClass
metadata:
  name: csi-snapclass
driver: ebs.csi.aws.com
deletionPolicy: Delete          # Delete | Retain
---
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshot
metadata:
  name: db-snap-20240101
  namespace: prod
spec:
  volumeSnapshotClassName: csi-snapclass
  source:
    persistentVolumeClaimName: db-data
---
# 从快照恢复：PVC 的 dataSource 指向 VolumeSnapshot
apiVersion: v1
kind: PersistentVolumeClaim
spec:
  dataSource:
    name: db-snap-20240101
    kind: VolumeSnapshot
    apiGroup: snapshot.storage.k8s.io
  storageClassName: fast-ssd
  accessModes: ["ReadWriteOnce"]
  resources:
    requests:
      storage: 100Gi
```

**必答**：

- VolumeSnapshot 是**命名空间级**，VolumeSnapshotContent 是集群级；两者绑定语义与 PV/PVC 类似；
- `deletionPolicy: Retain`** 能防止"删 PVC 连带把快照也删了"的误伤**（备份场景强烈建议）；
- 快照能力依赖驱动 `CREATE_DELETE_SNAPSHOT` capability，不是所有驱动都支持增量快照。

## A14. 卷克隆（Clone）和快照恢复有什么区别？

| 维度 | Clone（`dataSource.kind: PersistentVolumeClaim`） | 快照恢复 |
| --- | --- | --- |
| 数据源 | 另一个 PVC（**同一命名空间、同一 SC**） | VolumeSnapshot |
| CSI 调用 | `CreateVolume` 带 `VolumeContentSource: VOLUME` | `CreateVolume` 带 `VolumeContentSource: SNAPSHOT` |
| 时间点 | **当前状态**（要考虑源卷正在写入 ⇒ 数据一致性风险） | 快照时间点一致 |
| 用途 | 快速造一份同数据环境 | 恢复/回溯 |

**必答**：克隆在用数据库时通常要在应用层做 checkpoint/quiesce，否则可能得到一份"崩溃一致性"数据。

## A15. PVC 扩容的正确姿势？为什么扩了 Pod 里没变大？

**前提条件**：

1. `StorageClass.allowVolumeExpansion: true`；
2. 驱动支持 `EXPAND_VOLUME` / `NodeExpandVolume` capability。

**流程**：

```
改 PVC.spec.resources.requests.storage
   → external-resizer watch 到 → ControllerExpandVolume（扩后端磁盘）
   → kubelet 在下一次 reconcile 发现 PV.NewSize > 当前 → NodeExpandVolume（扩文件系统）
   → resize2fs / xfs_growfs
```

**常见坑（必答）**：

- **只能扩不能缩**（spec 改小会被拒绝）；
- 若卷**没有被 Pod 挂载**，某些文件系统无法离线扩容 ⇒ 需要重新挂载才生效（视 FS 与驱动能力）；
- `PVC.status.capacity` 与 `conditions: FileSystemResizePending` 能告诉你卡在哪一步：
  - 条件是 `FileSystemResizePending` ⇒ 磁盘已扩，**文件系统还没扩**（通常因为没挂载）；
- 扩容后要确认 `df -h` 在容器视图里也变大了。

```bash
kubectl get pvc db -n prod -o jsonpath='{.status}'
kubectl describe pvc db -n prod | tail -20
```

## A16. 卷健康监控（Volume Health）能做什么？

- kubelet 侧周期性 `NodeGetVolumeStats` ⇒ 暴露 `kubelet_volume_stats_*` 指标（**容量/inode 使用率的真相来源**）；
- CSI 侧 `VolumeCondition`（Abnormal）⇒ 通过 `external-health-monitor` 上报到 PVC 事件（该功能处于演进阶段，**请以所用版本为准**）；
- 应用层仍应自己监控 IO 延迟/错误，**不能只依赖 K8s 层**。

```promql
# 卷使用率（最基础的容量告警）
kubelet_volume_stats_used_bytes / kubelet_volume_stats_capacity_bytes > 0.85
# inode 使用率（被忽视的高频事故源）
kubelet_volume_stats_inodes_used / kubelet_volume_stats_inodes > 0.9
```

## A17. emptyDir / hostPath / local PV 三者的区别？

| 类型 | 生命周期 | 调度感知 | 适用 |
| --- | --- | --- | --- |
| **emptyDir** | 与 Pod 同生共死 | 无 | 临时缓存、scratch 空间（`medium: Memory` 用内存，**会计入容器内存 limit**） |
| **hostPath** | 数据留在节点上 | **完全无感知**（Pod 漂到别的节点就"数据丢了"） | 仅用于读取节点文件（日志采集、CNI），**禁止承载业务数据** |
| **local PV**（local volume） | 跟随 PV（Retain/手工管理） | **有 nodeAffinity，调度器看得懂** | 本地 SSD / 高性能缓存类有状态服务 |

```yaml
# local PV 示例（由 local-static-provisioner 自动生成）
apiVersion: v1
kind: PersistentVolume
metadata:
  name: local-pv-nvme-1
spec:
  capacity: {storage: 1000Gi}
  volumeMode: Filesystem
  accessModes: ["ReadWriteOnce"]
  persistentVolumeReclaimPolicy: Retain
  storageClassName: local-nvme
  local:
    path: /mnt/disks/nvme1
  nodeAffinity:
    required:
      nodeSelectorTerms:
        - matchExpressions:
            - key: kubernetes.io/hostname
              operator: In
              values: ["node-7"]
```

**必答**：local PV 的高性能代价是 **Pod 被钉在该节点**（节点故障 ⟹ 数据暂时不可用），因此**必须配合应用层多副本或存储层副本**。

## A18. 投影卷（projected）与 Secret/ConfigMap 挂载的热更新？

```yaml
volumes:
  - name: all-in-one
    projected:
      defaultMode: 0400
      sources:
        - secret:
            name: db-cred
        - configMap:
            name: app-conf
        - downwardAPI:
            items:
              - path: labels
                fieldRef: {fieldPath: metadata.labels}
        - serviceAccountToken:
            path: token
            expirationSeconds: 3600
```

**必答**：

- projected 卷把多种来源合并到一个目录，且**更新是原子的**；
- ConfigMap/Secret 作为 volume 挂载会**定期更新**（非 subPath 时），但**环境变量注入不会更新**；
- **大量 Pod 挂载同一个 Secret ⇒ kubelet 会 watch 它 ⇒ apiserver 压力放大** ⇒ 大规模场景应使用 `immutable: true`（减少 watch/拉取成本）。

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: db-cred
immutable: true     # 大规模建议：不可变 Secret 可显著降低 kubelet 侧 watch 成本
type: Opaque
```

## A19. fsGroup 与权限：为什么 PVC 挂载后目录不可写？为什么首次挂载特别慢？

```yaml
securityContext:
  fsGroup: 2000
  fsGroupChangePolicy: OnRootMismatch   # ★ 关键优化项（版本相关，请核对）
```

- `fsGroup` 会让 kubelet 递归 **chown/chgrp + 设置 setgid**，让 Pod 有写权限；
- **痛点**：卷里文件非常多时，递归 chown 可能耗时**数十秒到数分钟**（这也是"Pod 首次挂大卷特别慢"的常见原因）；
- `fsGroupChangePolicy: OnRootMismatch` ⇒ 只有当**根目录权限不匹配时才做**，避免每次挂载都全量递归；
- 另一种做法：在 Pod 里用 `initContainer` 显式 `chown`，可控且可观测。

> 注意 fsGroup 配额 (` supplementalGroups` + `seLinuxOptions`) 也会影响。

## A20. subPath 有什么坑？

```yaml
volumeMounts:
  - name: data
    mountPath: /var/lib/mysql
    subPath: mysql        # 只挂载卷里的某个子目录
```

| 坑 | 说明 |
| --- | --- |
| **热更新失效** | subPath 挂载的 ConfigMap/Secret **不会热更新**（符号链接切换机制失效） |
| **扩容/ cyanide 行为差异** | 有些行为与整卷挂载不一致（例如某些检查依赖卷根路径） |
| 变量替换 | `subPathExpr` 支持从 downward API 取值（`$(POD_NAME)`），可做"每 Pod 独立子目录" |

## A21. 挂载传播（mountPropagation）什么时候必须配？

| 值 | 行为 | 典型用途 |
| --- | --- | --- |
| `None`（默认） | 完全私有 | 普通业务 |
| `HostToContainer` | 宿主机后续 mount 能传进容器 | 想看到节点新增的挂载 |
| `Bidirectional` | 容器里 mount 也能传回宿主 | **CSI node driver、块设备管理类 DaemonSet 必须** |

**必答**：忘记配 `Bidirectional` 是"CSI 节点插件在容器里看不到宿主 mount / 挂载不生效"的高频原因；因为它是特权能力，通常需要配合 privileged 或更细的能力授予（**注意安全风险，别全集群放开**）。

## A22. StatefulSet 与 PVC 的关系？磁盘为什么会泄漏？

```yaml
apiVersion: apps/v1
kind: StatefulSet
spec:
  volumeClaimTemplates:            # ① 每个副本自动创建 PVC
    - metadata:
        name: data
      spec:
        accessModes: ["ReadWriteOnce"]
        storageClassName: fast-ssd
        resources:
          requests: {storage: 100Gi}
```

**命名规则**：`<claim-template>-<statefulset>-<ordinal>`，例如 `data-mysql-0`。

**必答的三个泄漏来源**：

1. **删除 StatefulSet 不会删除 PVC**（设计如此，防撞误删）⇒ 需要人工清理或用 ownerReference 方案；
2. **缩容（scale down）也不会删 PVC**（保留数据以便扩容回来）⇒ 长期缩容会留一堆闲置盘；
3. `reclaimPolicy: Retain`** 的 Dead PV** 没人管 ⇒ 孤儿磁盘持续计费。

**治理**：

- 建立定期巡检：找出"未被任何 PVC/Pod 引用的 PV"与"后端的孤儿磁盘"；
- 给 PVC/PV 打 owner/team/cost-center 标签，做成本归因；
- 删除流程必须进工单，并核对是否有最近的快照。

## A23. CSI 驱动的"凭据"与轮换怎么做？

CreateVolume / NodePublish 常常需要凭据（通过 `csi.storage.k8s.io/node-publish-secret-name` 等参数引用 Secret）：

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: secure-rbd
provisioner: rbd.csi.ceph.com
parameters:
  clusterID: <ceph-fsid>
  pool: kubernetes
  csi.storage.k8s.io/provisioner-secret-name: csi-rbd-secret
  csi.storage.k8s.io/provisioner-secret-namespace: ceph-system
  csi.storage.k8s.io/node-stage-secret-name: csi-rbd-secret
  csi.storage.k8s.io/node-stage-secret-namespace: ceph-system
```

**必答**：

- 凭据泄漏风险 ⇒ 严格 RBAC，Secret 只给 CSI 相关 SA；
- **轮换要有"双凭据并存"窗口**，避免轮换瞬间所有挂载/供给失败；
- 凭据失效的典型表现：`Failed to provision volume ... AuthFailure` / 已有卷无法 mount。

## A24. CSI driver 对象（`CSIDriver`）有什么作用？

```yaml
apiVersion: storage.k8s.io/v1
kind: CSIDriver
metadata:
  name: ebs.csi.aws.com
spec:
  attachRequired: true              # 是否需要 ControllerPublishVolume（attach）
  podInfoOnMount: true
  fsGroupPolicy: ReadWriteOnceWithFSType
  volumeLifecycleModes: ["Persistent"]
  storageCapacity: true             # 是否上报 CSIStorageCapacity
  requiresRepublish: true           # 是否需要周期性重新 publish（版本相关）
  tokenRequests: []
```

**必答**：`attachRequired` 决定 **是否走 attach 流程**（云盘 true；NFS/CephFS 通常 false）；`podInfoOnMount` 让驱动知道是哪个 Pod 在挂载（用于按 Pod 鉴权/审计）。

# Part B · 常见存储方案 / 模式专项篇（18 题）

> 这一篇按"方案能力矩阵 + 各种模式的取舍"出牌。

## B1. 先分清楚三类存储：块 / 文件 / 对象

| 类型 | 语义 | K8s 里的消费方式 | 典型用例 |
| --- | --- | --- | --- |
| **块（Block）** | 裸磁盘/裸设备，自己格式化 | PVC（RWO）为主，可 `volumeMode: Block` | 数据库、Kafka、etcd |
| **文件（File/NAS）** | 带文件系统的共享目录 | PVC（**RWX**） | 多 Pod 共享内容、CI 缓存、媒体处理 |
| **对象（Object）** | HTTP S3/OSS 语义，非 POSIX | **应用 SDK 直连**（最正统），或经 CSI/FUSE 挂载（性能与语义有损） | 图片/视频/备份归档/日志 |

**必答**：把对象存储当 PVC 挂载通常要点游览从来都不是最优解：POSIX 语义受限、ls 慢、追加写成本高。**选型铁律：能用 SDK 就别挂载**。

## B2. 主流 CSI 驱动能力矩阵（背这张表足够）

| 驱动 | 类型 | 典型访问模式 | AZ 约束 | 亮点 / 注意 |
| --- | --- | --- | --- | --- |
| **AWS EBS CSI** | 块 | RWO | **单可用区**（盘与 AZ 强绑定） | 快照、扩容、加密；跨 AZ 会挂不上 |
| **AWS EFS CSI** | 文件 | **RWX** | 跨可用区（Regional） | 多 Pod 共享；注意吞吐模式（bursting/provisioned/elastic） |
| **GCE PD CSI** | 块 | RWO | 单 zone（另有 regional PD 选项） | hyperdisk 等新形态请以厂商文档为准 |
| **Azure Disk / Files** | 块 / 文件 | RWO / RWX | Disk 受 zone 影响 | Files 走 SMB/NFS 协议 |
| **vSphere CSI (CNS)** | 块（VMDK/FCD） | RWO | 受 datastore/集群拓扑约束 | 与 vSphere 存储策略（SPBM）打通 |
| **Ceph CSI（RBD）** | 块 | RWO | 取决于 CRUSH/map 配置（常见按机架/主机） | 强一致、性能好；运维复杂度集中在 Ceph 侧 |
| **Ceph CSI（CephFS）** | 文件 | **RWX** | 同上 | 多 Pod 共享；MDS 是容量/性能关键 |
| **Longhorn** | 块（iSCSI 数据面） | RWO | 副本跨节点 | 轻量、易部署、适合边缘/中小；数据面成本在用户态 iSCSI |
| **OpenEBS Mayastor** | 块（NVMe-oF） | RWO | 取决于副本策略 | 面向 NVMe 高性能（容量陷阱：需要 HugePages/CPU 预留，请以版本为准） |
| **OpenEBS cStor / Jiva** | 块 | RWO | — | Jiva 已逐步退场（新版本请以官方说明为准），新部署建议 Mayastor/localPV |
| **NFS subdir external provisioner** | 文件 | **RWX** | 无（但依赖 NFS server 可用性） | 最简单；**官方 nfs-csi-driver 更推荐**，务必区分二者 |
| **JuiceFS CSI** | 文件（对象+元数据服务） | RWX | 跨域 | 挂载进程在 Pod 侧，POSIX 语义较好；注意元数据性能 |
| **Portworx** | 块/文件（商业） | RWO/RWX | 副本策略灵活 | 企业特性（加密、DR、auto-扩容），有 License |
| **local-static-provisioner** | 本地盘 | RWO | **节点强绑定** | 最便宜的高性能；故障域最大 |
| **GlusterFS** | 文件 | RWX | — | **社区已归档/不再推荐用于新集群**，面试时点明这一点很加分 |

## B3. 什么时候该选"本地盘"而不是"网络分布式存储"？

| 维度 | 本地盘（local PV / 实例 SSD） | 网络分布式（Ceph/Longhorn/云盘） |
| --- | --- | --- |
| 延迟/IOPS | **最好** | 受网络影响 |
| 单盘容量上限 | 受机型限制 | 更灵活 |
| 节点故障影响 | **数据暂时不可用**（需应用层多副本） | 存储层自动切副本 |
| 调度灵活性 | **Pod 被钉死在节点** | 可迁移（取决于实现） |
| 运维复杂度 | 低（但要管盘的健康/更换） | 高（集群、副本、恢复、容量） |
| 典型场景 | Kafka/ES/OLAP 这类**自带多副本**的系统 | 数据库主从、通用有状态服务 |

**标准答法**：

> "选本地盘的前提是**应用层已经提供了冗余**（Kafka replication、MySQL 半同步、ES replica）。如果指望存储层做冗余，那就别用本地盘；反过来，如果应用层本来就有副本，用网络盘就是给同一份数据付两次复制成本。"

## B4. 云盘最经典的坑：跨可用区挂载失败

**现象**：PVC 显示 Bound，Pod 一直 `ContainerCreating`，Events：`AttachVolume.Attach failed ... volume is in zone X, node is in zone Y`。

**成因**：云盘（EBS/云盘/Disk）是**可用区级资源**，只能挂给同 AZ 的虚拟机。

**治理清单**：

1. StorageClass 一定要 `volumeBindingMode: WaitForFirstConsumer`（**这是第一要务**）；
2. 需要保证副本分布 ⇒ 用 **TopologySpreadConstraints**（跨 zone）+ 允许等待也可以用 `allowedTopologies` 限定范围；
3. 恢复/迁移场景：**先给卷做快照再从快照创建新卷到目标 AZ**（这是标准迁移路径）；
4. 严禁跨 AZ 共享同一块的"土办法"（会数据损坏）。

## B5. 需要 RWX（多节点读写）时怎么做？

| 方案 | 说明 | 注意 |
| --- | --- | --- |
| NFS CSI | 最通用 | NFS server 自身要高可用，且要注意并发一致性语义 |
| CephFS | POSIX 语义较好 | MDS 需要容量规划与高可用；配额要提前配 |
| EFS / Azure Files / NAS | 云托管文件存储 | 关注吞吐模式与突发额度 |
| 应用层上传对象存储 | 最利于扩展 | 语义要改造 |

**必答警告**：

- **绝大多数 RWX 文件系统都没有跨节点强一致锁保证**，同一文件的并发写要靠应用层协议（如数据库绝不能放 RWX 卷上，除非该存储明确支持并完成验证）；
- 关系型数据库、etcd 这类组件**只用 RWO/RWOP**。

## B6. Rook-Ceph 面试会问什么？（分布式存储代表）

**核心组件**：

| 组件 | 作用 | 规模要点 |
| --- | --- | --- |
| **MON** | 集群元数据脑核（维护 cluster map），**依赖 quorum（奇数）** | 通常 3 或 5 |
| **MGR** | 对外展示指标/管理通道（dashboard/prometheus 模块） | 至少 2（active/standby） |
| **OSD** | 真正存数据的进程（一块盘一个 OSD） | 数量决定容量/性能/故障域 |
| **MDS** | CephFS 元数据服务 | RWX 场景必需 |
| **RGW** | 对象网关 | 走 S3 协议场景 |

**运维要点（必答）**：

- **PG 状态是排障第一语言**：`active+clean` 正常；`degraded`（副本缺失）/`incomplete`（PG 不完整）/`inconsistent`（scrub 发现不一致）要立即处理；
- **副本 vs 纠删码**：副本简单但成本高（3x），EC 省空间但 CPU/恢复成本和读放大更高；
- **failure domain**：按 host/rack 分布，别让副本全在一台机器；
- **恢复与再平衡会引发 IO 风暴** ⇒ 要控制回填/恢复的并发与优先级（`osd_max_backfills`、`osd_recovery_*` 等参数，**请以所用版本 official doc 为准**）；
- **scrub/deep-scrub** 会消耗 IO，要错峰；
- 网络建议 **public network 与 cluster network 分离**（南北向客户端流量 vs 东西向复制流量）。

```bash
ceph -s                     # 一眼看 health / mon quorum / osd 数
ceph osd tree               # 拓扑与权重
ceph pg stat / ceph pg dump_stuck
ceph health detail
```

## B7. Longhorn 适合谁？它的短板在哪？

**适合**：中小集群/边缘/测试环境，追求"部署简单、能看到 GUI、社区活跃"，单盘容量需求不高。

**短板（必答）**：

- 数据面走 **iSCSI over TCP（用户态 tgt）**，高并发下的 CPU 开销与延迟明显高于 NVMe-oF/原生块；
- **大规模下会出现 iSCSI login/session 抖动**（成百上千卷时尤其要注意）；
- 副本重建会占满网络。
⇒ 结论：**不建议在大规模高 IOPS 的核心生产上作为主力**，更适合边缘与中等负载。

## B8. OpenEBS 各模式怎么选？

| 模式 | 数据面 | 适用 | 备注 |
| --- | --- | --- | --- |
| **localPV-hostpath** | 本地目录 | 追求简单、可接受节点绑定 | 常用于快速搭建 |
| **localPV-device**（LVM/ZFS） | 本地设备 | 需要本地裸盘 | TopoLVM 类方案也在此生态位 |
| **Mayastor** | NVMe-oF（用户态 SPDK） | 高性能本地网络存储 | 需要 CPU/HugePages 预留与节点能力校验（按版本确认） |
| **Jiva / cStor** | 早期分布式引擎 | **新建集群不再推荐** | 请查看官方当前推荐路线 |

## B9. 云盘的"性能档"怎么选？（比如 IOPS/吞吐突发）

**要点**：

- 云盘通常按"**容量越大、配额越高**"或"**可购买 IOPS/吞吐**"两种方式给配额（gp3/io2 之类的命名因厂商而异）；
- **突发型（bursting）盘有积分机制**：小容量盘扛不住持续高 IO，积分耗尽后 IOPS 直降到基线（典型"跑一阵子突然变慢"）；
- **治理**：
  1. 用 StorageClass parameters 固化 IOPS/吞吐档位，避免有人手抖建低配额盘；
  2. 监控卷的 IOPS/吞吐/延迟（云监控或节点层），对"接近配额"做告警；
  3. 压测要覆盖"稳态"而非仅看突发值。

```promql
# 卷接近 IOPS/吞吐配额（具体来源依赖云 exporter 或节点侧采集）
# 通用做法：采集 /var/lib/kubelet/pods 对应设备的 IOPS 并与云商配额对比做阈值告警
```

## B10. 卷加密有几种做法？

| 做法 | 层 | 说明 |
| --- | --- | --- |
| 云厂商静态加密 | 存储后端 | 最简单，通常默认开启，靠 KMS |
| CSI 层（LUKS/dm-crypt） | 节点 | 云无关的通用做法，注意 key 管理与性能损耗 |
| 应用层加密 | 应用 | 最细粒度，成本最高 |
| Secret 静态加密（etcd） | 控制面 | **这是保护 etcd 里的 Secret，与卷数据加密完全是两回事**（面试常混） |

## B11. 备份：Velero vs CSI 快照 vs 存储侧原生快照？

| 方案 | 粒度 | 恢复方式 | 适用 |
| --- | --- | --- | --- |
| **CSI 快照** | 单卷，速度快 | 快照→新卷→重建 PVC | 快速回滚、同集群/同区域 |
| **Velero**（支持 CSI 卷快照 + 文件系统备份两种模式） | 命名空间/标签/整集群（含对象清单） | 恢复到原集群或新集群 | **异地容灾、集群迁移、误删恢复** |
| **存储侧原生快照**（集群自带的定时快照） | 存储层 | 视产品 | 与 K8s 对象状态不一定一致，单独用风险大 |

**必答**：**Velero 的价值在于"对象清单 + 卷"一起备份**，纯 CSI 快照恢复时缺少 Deployment/StatefulSet/Service 等对象，两者应该配合而不是二选一。并且要定期做**恢复演练**（没有演练的备份等于没有备份）。

## B12. VolumeGroupSnapshot 是干什么的？（多卷一致性）

一次给**多个卷**同时打快照，保证跨卷的时间一致性（常见于"日志卷+数据卷"或分片数据库）。**该能力为较新特性，请以所用版本的 Beta/GA 状态为准**。

## B13. ReadWriteOncePod 的真实用途？

```
RWO：同一节点上可以有多个 Pod 挂载同一个 PVC
RWOP：整个集群仅限一个 Pod（kubelet 强制）
```

用途：防止"运维误操作把第二个 Pod 挂到同一个数据卷"导致文件损坏（比如手工 debug Pod 挂了生产盘）。**注意它不等于 fencing**——存储侧未必知道谁拿到了锁，只是 K8s 侧的单写保护。

## B14. 有状态应用上线的标准清单？

```
1) Headless Service + StatefulSet（稳定网络身份）
2) volumeClaimTemplates + 合适的 StorageClass（带 allowVolumeExpansion）
3) PDB + TopologySpreadConstraints（跨可用区/机架）
4) terminationGracePeriodSeconds + preStop（优雅终止，避免脏数据）
5) 探针：liveness 只查自身，readiness 区分"在同步/在 serving"
6) requests/limits 与 QoS：数据库类建议 Guaranteed
7) 资源隔离：专用节点池（taint）避免吵闹邻居抢 IO
8) 备份：定时快照 + 跨区域复制 + 定期恢复演练
9) 监控：卷容量/inode/延迟/IOPS + 应用层 db metrics
10) 升级策略：逐副本滚动 + 先扩勿删 + 明确回滚（含数据回滚）路径
```

## B15. 每个节点能挂多少卷？谁在限制？

- 云商限制 **每台虚拟机可挂磁盘数**，K8s 会把它体现为节点的 **allocatable 资源**，形如 `attachable-volumes-aws-ebs: 39`（数字随实例类型而变）；
- 调度侧有相应的 **NodeVolumeLimits**（versions and csi-specific limits 以版本为准）做前置检查；
- **超限表现**：Pod Pending，Events 会明确写出该节点挂载数已达上限。

```promql
# 节点已挂载卷数 / 上限（依赖 kube-state-metrics / 云 exporter）
kube_node_status_allocatable{resource="attachable_volumes_aws_ebs"}
```

## B16. CSI driver 自身要不要 HA？

| 部署 | HA 做法 |
| --- | --- |
| **Controller**（Deployment） | 多副本 + **leader election**（sidecar 内部选主）+ PDB；跨 AZ |
| **Node**（DaemonSet） | 每节点一份；**本身无 HA**，故障只能影响单节点 ⇒ 用节点池/灰度发布降低爆炸半径 |
| **livenessprobe** | 探活 socket，暴露健康状态，便于告警与自动重启 |

**必答**：Controller 挂 ⇒ **全集群新卷供给/attach/扩容全部停摆**（存量读写通常不影响）；Node 挂 ⇒ **单节点新挂载失败**（已挂载的卷通常不受影响）。

## B17. CSI 驱动部署的实例配置（一个通用骨架）

```yaml
# Controller（示意，务必以驱动官方 Chart/ manifests 为准）
apiVersion: apps/v1
kind: Deployment
metadata:
  name: csi-provisioner
  namespace: csi-system
spec:
  replicas: 2
  strategy: {type: RollingUpdate, rollingUpdate: {maxUnavailable: 1}}
  template:
    spec:
      containers:
        - name: csi-provisioner
          image: registry.k8s.io/sig-storage/csi-provisioner:v3.6.2
          args:
            - "--csi-address=$(ADDRESS)"
            - "--timeout=60s"                 # 建议：明确超时，避免无限等待
            - "--leader-election"
            - "--leader-election-namespace=csi-system"
            - "--extra-create-metadata"
          env:
            - {name: ADDRESS, value: /csi/csi.sock}
        - name: csi-attacher
          image: registry.k8s.io/sig-storage/csi-attacher:v4.4.2
          args: ["--csi-address=$(ADDRESS)", "--leader-election", "--timeout=60s"]
        - name: csi-resizer
          image: registry.k8s.io/sig-storage/csi-resizer:v1.9.2
          args: ["--csi-address=$(ADDRESS)", "--leader-election", "--timeout=60s"]
        - name: csi-snapshotter
          image: registry.k8s.io/sig-storage/csi-snapshotter:v6.3.2
          args: ["--csi-address=$(ADDRESS)", "--leader-election", "--timeout=60s"]
        - name: liveness-probe
          image: registry.k8s.io/sig-storage/livenessprobe:v2.11.0
          args: ["--csi-address=$(ADDRESS)"]
        - name: driver
          image: <厂商 CSI 镜像>
          args: ["--endpoint=$(CSI_ENDPOINT)", "--logtostderr", "-v=5"]
```

> 镜像 tag 请务必按自身版本矩阵选择（**sidecar 与 K8s 版本有兼容性要求**，这是升级前必须核对的东西）。

## B18. 常见 CSI driver 故障的"现象→原因"速查

| 现象 | 常见原因 |
| --- | --- |
| PVC 一直 Pending，`no persistent volumes available` / `ProvisioningFailed` | provisioner 没起来、StorageClass 名字拼错、后端配额不足、凭据失效 |
| `FailedAttachVolume` | 云 API 限流、卷在别的 AZ、已达节点挂载上限、上一次 detach 未完成 |
| `FailedMount` | kubelet 侧 mount 失败（fs 损坏、mount 参数错、node plugin 崩） |
| `VolumeAttachment` 卡住 | ControllerPublish 超时重试/VolumeAttachment 残留未清理 |
| `node(s) had volume node affinity conflict` | WaitForFirstConsumer + AZ 拓扑不匹配 |
| `Multi-Attach error` | 上一个 Pod 还没完全释放卷（周期性驱逐/节点故障常见），或有第二个 Pod 非法挂载 |
| 卷挂载后只读 | 文件系统损坏进入 read-only、或两个 Pod 同时写 |
| 扩容后容器视图不变 | NodeExpandVolume 未执行/卷未挂载 |

# Part C · 大规模集群存储优化篇（14 题）

> 定义：**大规模 = ≥ 1000 节点 / PVC 数万 / 单集群有状态服务上百**。

## C1. 大规模下最先出问题的四个地方

| # | 位置 | 症状 | 治理方向 |
| --- | --- | --- | --- |
| 1 | **对象数量（PVC/PV/VolumeAttachment）** | apiserver/etcd 压力、list 慢 | 控制 PVC 数量、定期回收孤儿、审视 VolumeAttachment 残留 |
| 2 | **调度侧的 VolumeBinding** | 调度变慢、pending PVC 堆积 | 收敛挂载卷 Pod 的比例、减少等待绑定的 PVC |
| 3 | **Attach/Detach 串行 + 云 API 限流** | Pod 卡 ContainerCreating 十几分钟 | 分批创建、限流调优、避免大规模同时重建有状态服务 |
| 4 | **节点挂载数上限与单/多 SAN storm** | 挂载失败、iSCSI 会话抖动 | 节点池化、异构规格、控制单节点卷密度 |

## C2. PVC/PV 数量对控制面的影响怎么量化？

```promql
# PVC/PV 总数增长趋势（依赖 kube-state-metrics）
count(kube_persistentvolumeclaim_info)
count(kube_persistentvolume_info)
# VolumeAttachment 残留（不该长期存在）
count(kube_volumeattachment_info)
# 处于 Pending / Lost 的 PVC（异常信号）
count(kube_persistentvolumeclaim_status_phase{phase="Pending"})
```

**治理**：

- 建立 **PVC 生命周期归属**（owner/team 标签），定期清理无主 PVC/PV；
- **孤儿磁盘巡检**：平台侧（PV 列表）与云/存储侧（实际卷列表）**对账**，这是最省钱的一步；
- 对"每批次重建大量有状态 Pod"的动作做**分批限制**（否则瞬间 VolumeAttachment 洪峰会打爆 controller 与云 API）。

## C3. "Pod 挂着卷十几分钟起不来"怎么排查和根治？

**定位顺序（必会）**：

```bash
kubectl describe pod <pod>              # 看 Events 的时间戳，判断卡在哪一步
#   00:00  Scheduled
#   00:00  SuccessfulAttachVolume  ← attach 完成（或这里一直是 FailedAttachVolume）
#   00:01  FailedMount / MountVolume.SetUp failed   ← mount 阶段问题
#   00:15  MountVolume.SetUp succeeded
kubectl get volumeattachment | grep <pv-name>        # 有没有残留
kubectl -n <csi-ns> logs <csi-controller> -c csi-attacher
ssh node: journalctl -u kubelet | grep -i mount
```

**典型根因**：

1. **上一次 detach 没完成**（节点宕机/强制删除 Pod）⇒ 新 Pod 一直等待，直到 detach reconcile 超时解除；
2. **云 API 限流**（大规模同时 create/attach）⇒ 需要给 controller 侧 qps/burst 调优 + 业务侧分批；
3. **文件系统修复/check**（例如卷被脏卸载后 fsck）⇒ 时间不可预测；
4. **fsGroup 递归 chown**（大卷）⇒ 用 `fsGroupChangePolicy: OnRootMismatch`；
5. **Node plugin 崩** ⇒ 该节点所有新挂载失效。

**根治**：

- 有状态服务的重建**必须分批**（Deployment/StatefulSet 的滚动参数 + PDB）；
- 明确 clean detach 流程：**先删除 Pod 并确认 VolumeAttachment 消失，再重建**；
- 监控"卷操作耗时"并建告警（见 E 篇）。

## C4. 调度侧的 VolumeBinding 会成为瓶颈吗？

会。每个**处于等待绑定状态的 PVC** 都会让调度器在 Filter 阶段做拓扑匹配计算；大规模集群里"成千上万个未完成绑定的 PVC"会显著拉长调度周期。

**治理**：

- 用 `WaitForFirstConsumer` 但同时**避免积累大量无 Pod 消费的 PVC**（无用 PVC 直接删掉）；
- 尽量避免一个 Pod 挂**过多** PVC（每个都要参与调度计算）；
- 关注 `scheduler_framework_plugin_duration_seconds{plugin="VolumeBinding"}`（见调度篇 L1）。

## C5. CSI Controller 挂了怎么办？怎么预防？

| 影响面 | 说明 |
| --- | --- |
| 新卷供给 | 全部 Pending |
| attach/detach | 全部卡住（**会影响 Pod 重建**） |
| 扩容 | 全部卡住 |
| 存量读写 | **通常不受影响**（这是重要的冷静判断依据） |

**预防**：

- Controller 多副本（一般 ≥2）+ leader election + PDB + 跨 AZ；
- livenessprobe + 自动重启 + 告警；
- **给 CSI Controller 单独稳定的 machine/cgroup 资源保障**（它被系统/OOM 干掉会很惨）；
- 变更（升级/配置）**必须灰度**：先在测试集群或单 StorageClass 上验证。

## C6. 单节点挂载卷数量的治理

- 通过 Scheduling/allocatable 天然限制，但要**在容量规划时算进去**：`节点数 × 单节点可挂卷数 ≥ 期望 PV 总数`；
- 有状态服务的 **Pod 密度要低于无状态**（给 DaemonSet、临时 Pod、扩容留余量）；
- 倾向于"少量大卷"而不是"海量小卷"（对象数、iSCSI session、attach API 调用都会减少）；
- 对"每个 Pod 一个 PV 的分库分表"型场景要做好上限评估。

## C7. 大规模 Secret/ConfigMap 挂载怎么优化？

- **不变即 **`immutable: true`（kubelet 不再频繁 watch，显著降低 apiserver 压力）；
- 避免把大量数据塞进单个 Secret（会拖慢 list/watch 与 etcd）；
- 避免"成千上万个 Pod 挂载同一个 Secret"这种放大结构（改用投射卷/环境变量/启动时读取方案视场景而定）；
- 定期审计未被引用的 ConfigMap/Secret。

## C8. fsGroup 与大卷挂载的性能问题

见 A19。**重复强调因为它是最容易被忽略的性能杀手**：

```yaml
securityContext:
  fsGroup: 2000
  fsGroupChangePolicy: OnRootMismatch    # 大卷场景强烈建议（版本相关，请核对）
```

**验证方式**：对比开启前后同一卷的**挂载耗时**（用 `kubelet_pod_start_duration_seconds` 或 storage operation 指标），而不是凭感觉。

## C9. 卷容量平滑治理（Storage Capacity Tracking）

- CSI provisioner 可以上报 `CSIStorageCapacity` 对象，让调度器在"后端容量不足"时提前避开/做出更好决策（**特性的成熟度与开启方式请以版本为准**）；
- 即使不启用该特性，SRE 侧也应做 "**容量余量告警**"（见 E 篇），避免出现"PVC Pending 且原因是没空间"的被动局面。

## C10. 有状态服务的存储 IO 隔离怎么做？

| 手段 | 说明 | 代价 |
| --- | --- | --- |
| **专用节点池**（taint + toleration） | 最彻底，避免 IO 争抢 | 利用率下降 |
| requests/limits + QoS Guaranteed | 保证资源不被挤 | 需要准确的画像 |
| `ephemeral-storage` limits | 限制容器可写层 | 主要防写爆 |
| cgroup IO 限制（`io.weight`/`blkio`） | **需要 cgroup v2 且场景验证** | 配置复杂需压测 |
| 存储的 QoS/限流（云盘 IOPS 配额） | 直接在存储层限制 | 受限于后端能力 |

**必答**：**数据库这种抖动敏感负载一定要做 IO 隔离**，否则"邻居跑个批处理就把 IOPS 吃满"会直接体现为业务 P99 恶化，而且很难定位。

## C11. 有状态服务跨可用区/跨集群的容灾怎么做？

| 方案 | RPO/RTO | 复杂度 | 说明 |
| --- | --- | --- | --- |
| 应用层复制（主从/半同步/Raft） | 取决于复制延迟 | 中 | **最推荐**（数据语义正确） |
| 存储层同步复制（跨 AZ 卷类型/存储集群多站点） | 接近 0 | 高 | 依赖后端能力，成本高、写延迟上升 |
| 快照 + 异地 Snapshot copy | 分钟\~小时级 | 低 | 定期快照 + 跨区域复制，作为兜底 |
| Velero 备份恢复 | 取决于备份频率 | 中 | 兜底 + 迁移 |

**必答**：不要把"备份"当成"容灾"。**任何方案都必须写明 RPO/RTO 并演练过**。

## C12. 集群升级时，存储相关的变更顺序与风险

```
① 先升级/验证 CSI driver 与 sidecar 的版本兼容矩阵（这是最容易翻车的环节）
② Controller 优先（滚动、观察新卷供给能力）
③ Node 插件按节点池灰度（DaemonSet 滚动）
④ 有状态服务的节点：优先使用"不驱逐"的方式（或确保 detach/attach 流程顺畅，分批进行）
⑤ 升级前务必确认快照/备份可用
```

**风险**：

- CSI Node 插件与 kubelet skews会导致全部挂载失败；
- 节点 drain 时若有大量卷 detach，容易触发限流；
- 注意 sidecar 的参数在跨大版本时会有改名/废弃（如超时、选举相关参数），升级前必须逐项比对新旧 manifests 差异。

## C13. 存储成本治理（面试会加分）

1. **孤儿资源对账**：PV ↔ 后端磁盘 双向核对；
2. **Retain 的 Dead PV 定期清理**（带审批）；
3. **快照保留策略**（太多旧快照往往比数据盘更贵）；
4. **性能档位治理**：把 burst 型盘升级/规划为 provisioned；不要把 gp2 之类的老一代默认盘常态化；
5. **本地盘 vs 网络盘**：按业务冗余能力选择，避免重复支付复制成本；
6. **标签与成本归因**：给 PVC/PV 打 team/app 标签，做 showback/chargeback。

## C14. 给你一个 2000 节点集群，怎么设计存储基线？（综合题）

按顺序答：

1. **问 SLO**：数据重要性？RPO/RTO？是否需要跨可用区/跨城容灾？单卷最大容量与快照频率？合规要求（加密、保留年限）？
2. **选型**：通用有状态 → 网络块（云盘/Ceph RBD）；共享读 → 文件存储；自带副本的系统 → 考虑本地盘降本；对象类数据 → 走 SDK。
3. **StorageClass 分层**：`fast-rwo`（高 IOPS，生产库）、`standard-rwo`（通用）、`rwx-shared`（文件）、`local-nvme`（自带副本的中间件），并统一 `WaitForFirstConsumer` + `allowVolumeExpansion`。
4. **拓扑**：AZ 感知 + TopologySpreadConstraints，明确 AZ 级套餐的支持范围。
5. **CSI 部署**：Controller ≥2 副本跨 AZ + PDB + leader election；Node 插件优雅发布。
6. **容量与配额**： ResourceQuota（`requests.storage` + per-SC PVC 数量）+ LimitRange + StorageCapacity 监控。
7. **备份**：快照 + Velero + 定期恢复演练，明确 retention 与异地复制。
8. **可观测**：卷容量/inode、attach/mount 耗时、CSI sidecar 健康、云配额 IOPS（见 E 篇）。
9. **变更治理**：CSI 升级灰度 + 分批 + 回滚（含数据回滚）+ 成本巡检月度化。
10. **演练**：GameDay（见 F 篇）。

# Part D · 反模式清单（可直接用于生产巡检）

| # | 反模式 | 证据（怎么发现） | 等级 | 修复动作 | 验证方式 |
| --- | --- | --- | --- | --- | --- |
| D1 | 云盘 StorageClass 用 `Immediate` 绑定 | 出现跨 AZ pending、`volume node affinity conflict` | 🔴 高 | 改 `WaitForFirstConsumer`（老 PV 需重建） | 跨 AZ 场景下 Pod 能正常绑定挂载 |
| D2 | 数据库放到 RWX 共享文件系统 | 多 Pod 并发写同一数据目录 | 🔴 高 | 改 RWO/RWOP 块存储 | 单写约束验证通过 |
| D3 | 无 storageclass 配额，谁都能建 10TiB 卷 | 账单异常 | 🟠 中 | ResourceQuota（`requests.storage` + per-SC pvc 数）+ LimitRange | 超额申请被拒绝且报错清晰 |
| D4 | `reclaimPolicy: Retain` 但无人巡检 | Dead PV/Dead 磁盘堆积 | 🟠 中 | 月度孤儿资源对账（PV↔后端）+ 清理工单 | 对账脚本输出为 0 |
| D5 | 手动删 finalizer 强删 PVC | 后端磁盘变孤儿 | 🔴 高 | 先释放使用方再删；强删必须关联"磁盘回收"动作 | 对账无残留 |
| D6 | 给容器配置 `fsGroup` 但不配 `OnRootMismatch` | 大卷挂载耗时分钟级 | 🔴 高 | 加 `fsGroupChangePolicy: OnRootMismatch` | 挂载耗时回到基线（实测） |
| D7 | 生产用 hostPath 承载业务数据 | Pod 漂移后数据丢失 | 🔴 高 | 换 local PV 或网络存储 | 漂移演练数据不丢 |
| D8 | hostPath 卷可被任意创建（无策略约束） | 存在逃逸/篡改节点风险 | 🔴 高 | PSA/PSS `baseline` 以上 + 策略引擎禁 hostPath | 策略拦截生效 |
| D9 | CSI Controller 单副本、无 PDB | Controller 挂 ⇒ 全集群供给/attach 停 | 🔴 高 | 多副本 + leader election + PDB + 跨 AZ | 杀一个副本不影响供给 |
| D10 | 大量 Pod 挂载同一 Secret 且非 immutable | apiserver Secret watch 放大 | 🟠 中 | `immutable: true` + 收敛用量 | apiserver Secret 请求 QPS 下降 |
| D11 | 一次性批量重建成百有状态 Pod | 云 API 限流、Attach 洪峰 | 🔴 高 | 分批 + PDB + 观察 VolumeAttachment | 批次窗口内无 FailedAttach |
| D12 | 未监控 inode | inode 耗尽导致"有空间却写不进" | 🟠 中 | 加 `kubelet_volume_stats_inodes_used` 告警 | 告警能在耗尽前触发 |
| D13 | 只有快照没有恢复演练 | 出事才发现恢复不了 | 🔴 高 | 定期 restore 演练并记录 RTO | 演练记录达标 |
| D14 | 新集群还在用 GlusterFS / 已归档方案 | 依赖无人维护组件 | 🟠 中 | 迁移到 CephFS/云文件/NFS CSI | 迁移收益（可维护性）书面化 |
| D15 | StorageClass 参数随意（无 IOPS 档位） | burst 积分耗尽后性能暴跌 | 🟠 中 | 在 SC 里固化性能档并监控配额使用率 | 稳态压测达标 |

---

# Part E · 可观测性：SLO、指标与告警

## E1. 建议的存储侧 SLO（**初始建议值，必须实测校准**）

| SLI | 建议目标（起点） | 度量 |
| --- | --- | --- |
| 卷供给成功率（PVC → Bound） | ≥ 99.9% | PVC Phase 与 ProvisioningFailed 事件计数 |
| 新卷供给时长 P99（含后端创建） | ≤ 30s | PV create 时间 − PVC create 时间（需自建观测或审计） |
| Attach + Mount 完成时长 P99 | ≤ 60s | 挂载相关指标 + `FailedMount` 计数 |
| 卷容量使用率（告警阈值） | ≤ 85% | `kubelet_volume_stats_used_bytes / capacity` |
| inode 使用率 | ≤ 90% | `kubelet_volume_stats_inodes_used / inodes` |
| 快照成功率 | ≥ 99% | VolumeSnapshot ready 状态 |
| 备份 RTO / RPO | 按演练实测填写 | 演练记录 |

## E2. 指标清单（**先 **`curl -k https://127.0.0.1:10250/metrics | grep volume`** 核对**）

| 指标 | 含义 | 用途 |
| --- | --- | --- |
| `kubelet_volume_stats_capacity_bytes` / `_used_bytes` | 卷容量 | 容量告警 |
| `kubelet_volume_stats_inodes` / `_inodes_used` / `_inodes_free` | inode 用量 | inode 告警 |
| `storage_operation_duration_seconds{operation_name=...}` | kubelet 卷操作耗时（`volume_mount`、`volume_attach`、`verify_volumes_are_attached` 等） | **挂载慢定位首选** |
| `volume_manager_total_volumes`（可用性与状态明细依版本） | 节点上卷数量 | 密度治理 |
| kube-state-metrics：`kube_persistentvolumeclaim_status_phase` / `_resource_requests_storage_bytes` / `kube_persistentvolume_status_phase` / `kube_volumeattachment_*` | 对象状态 | 规模与异常检测 |
| CSI sidecar 自身 metrics（各 sidecar 端口/路径不同） | 调用量与错误 | CSI 健康 |

## E3. Prometheus 告警规则（可直接粘贴，阈值需实测校准）

```yaml
groups:
  - name: k8s-storage
    rules:
      - alert: PvcPendingTooLong
        expr: |
          kube_persistentvolumeclaim_status_phase{phase="Pending"} == 1
        for: 15m
        labels: {severity: warning}
        annotations:
          summary: "PVC {{ $labels.namespace }}/{{ $labels.persistentvolumeclaim }} 长时间未绑定"

      - alert: PvcLost
        expr: kube_persistentvolumeclaim_status_phase{phase="Lost"} == 1
        for: 5m
        labels: {severity: critical}
        annotations:
          summary: "PVC {{ $labels.namespace }}/{{ $labels.persistentvolumeclaim }} 状态为 Lost"

      - alert: VolumeNearlyFull
        expr: |
          kubelet_volume_stats_used_bytes / kubelet_volume_stats_capacity_bytes > 0.85
        for: 30m
        labels: {severity: warning}
        annotations:
          summary: "卷 {{ $labels.namespace }}/{{ $labels.persistentvolumeclaim }} 使用率 {{ $value | humanizePercentage }}"

      - alert: VolumeInodeNearlyFull
        expr: kubelet_volume_stats_inodes_used / kubelet_volume_stats_inodes > 0.9
        for: 30m
        labels: {severity: warning}
        annotations:
          summary: "卷 inode 使用率过高：{{ $labels.namespace }}/{{ $labels.persistentvolumeclaim }}"

      - alert: VolumeOperationSlow
        expr: |
          histogram_quantile(0.99,
            sum by (operation_name, le) (rate(storage_operation_duration_seconds_bucket[5m]))) > 30
        for: 10m
        labels: {severity: warning}
        annotations:
          summary: "卷操作 {{ $labels.operation_name }} P99 达 {{ $value }}s"

      - alert: VolumeAttachmentStuck
        expr: kube_volumeattachment_created > 0
        for: 60m
        labels: {severity: info}
        annotations:
          summary: "存在长期残留的 VolumeAttachment，检查是否有异常卷"

      - alert: CsiControllerDown
        expr: up{job=~"csi-.*-controller"} == 0
        for: 5m
        labels: {severity: critical}
        annotations:
          summary: "CSI Controller 不可用，新卷供给/attach/扩容都会停摆"

      - alert: VolumeProvisionFailed
        expr: |
          sum by (namespace, reason) (
            increase(kubernetes_events_total{involved_object_kind="PersistentVolumeClaim",
                     reason=~"ProvisioningFailed|ExternalProvisioningFailure.*"}[15m])) > 0
        for: 10m
        labels: {severity: critical}
        annotations:
          summary: "出现 {{ $labels.reason }}：命名空间 {{ $labels.namespace }} 的卷供给失败"
          description: "本规则依赖 event exporter 将事件转成指标，请先确认其在你的集群可用"
```

> **事件监控补充**：强烈建议部署 **kube-event-exporter / kubernetes-event-exporter**，把 `ProvisioningFailed`、`FailedAttachVolume`、`FailedMount` 这些**带明确失败原因的事件**转成指标与日志，这是存储排障里性价比最高的一步。

```promql
# 思路：把事件转为计数器后按 reason 聚合
sum by (reason, namespace) (increase(kubernetes_events_total{involved_object_kind="Pod",
       reason=~"FailedMount|FailedAttachVolume|FailedScheduling"}[15m]))
```

---

# Part F · 演练（GameDay）计划表

| 场景 | 注入方式 | 期望表现 | 验证的 SLO |
| --- | --- | --- | --- |
| CSI Controller 全部挂掉 | 停 Controller 副本 | 存量读写不受影响；新卷 Pending 且告警及时 | 供给可用性 + MTTR |
| Node 插件崩（单节点） | 停 DaemonSet Pod | 单节点新挂载失败，已有卷可用；节点被隔离 | 爆炸半径可控 |
| 卷已满 | 写满一个卷 | 容量告警先触发；业务能优雅降级 | 容量治理 |
| inode 耗尽 | 狂写小文件 | inode 告警触发 | inode 监控有效 |
| detach 未完成 | 强制删节点 | 卷能在可接受时间内被强制回收（工单流程） | 卷回收流程 |
| 快照失败 | 让后端快照 API 失败 | 备份失败告警触发，且有重试/告警闭环 | 快照成功率 |
| 恢复演练 | 用快照/Velero 恢复服务到隔离环境 | 数据一致、服务可起来、耗时达标 | RTO/RPO |
| 跨可用区重建 | 模拟 AZ 故障 | 有状态服务能在其它 AZ 恢复（应用层复制有效） | 容灾目标 |
| 大规模重建 | 一次性重建 200 有状态 Pod | 分批次 + 无限流（考察治理是否生效） | 变更稳定性 |
| 孤儿资源巡检 | 对账 PV vs 后端磁盘 | 能发现并清理孤儿资产 | 成本治理 |

---

# Part G · 10 张面试速记卡（临阵磨枪）

1. **四层抽象**：Pod 内 volume（跟 Pod）→ PVC（独立）→ PV（集群级）→ StorageClass（模板）；PV 与 PVC **1:1 独占绑定**。
2. **三种策略**：`reclaimPolicy`（Delete/Retain）、`volumeBindingMode`（**强烈建议 WaitForFirstConsumer**）、`allowVolumeExpansion`（只能扩不能缩）。
3. **访问模式是"声明不强制"**：RWO ≠ 只能一个 Pod（同节点多 Pod 也能挂）；要严格单写用 **RWOP**；数据库别放 RWX。
4. **attach vs stage vs publish**：`ControllerPublishVolume`（云盘挂给虚拟机）→ `NodeStageVolume`（格式化挂载到 staging，可被多 Pod 复用）→ `NodePublishVolume`（bind mount 到 Pod 目录）。对应事件：`FailedAttachVolume` / `FailedMount`。
5. **sidecar 四件套**：provisioner（增删改卷）、attacher（发布/解绑）、resizer（扩容）、snapshotter（快照）+ node-driver-registrar（注册）与 livenessprobe（探活）。
6. **拓扑决定生死**：云盘是 AZ 级资源 ⇒ `WaitForFirstConsumer` + `allowedTopologies`；报错关键字 `volume node affinity conflict`。
7. **卡 Terminating 看 finalizer**：`kubernetes.io/pvc-protection` / `pv-protection`；强删 finalizer 会留下**孤儿磁盘**（最贵的坑）。
8. **挂载慢的四大元凶**：fsGroup 递归 chown、上一次 detach 没完成、云 API 限流、Node plugin 崩。先看 `storage_operation_duration_seconds` 再细化。
9. **Controller 挂 vs Node 挂**：Controller 挂 ⇒ **全集群**新卷操作停（存量读写通常没事）；Node 挂 ⇒ **单节点**新挂载失败。
10. **StatefulSet 的 PVC 不会自动删**：删除/缩容都保留 ⇒ 磁盘泄漏的经典来源；必须做 monthly 对账 + 标签归因。

---

## 附：文档自查清单

- [ ] 每条建议配了可直接套用的 YAML / 部署骨架 / PromQL / 排障命令
- [ ] 所有默认值与 feature gate（Attach 相关 flag、RWOP、CSIStorageCapacity、VolumeGroupSnapshot 等）标注"以版本为准"
- [ ] 未提供任何未经实测的性能数字，均给出**对比测量方法**
- [ ] 反模式清单含：证据 / 等级 / 修复 / 验证
- [ ] SLO 标为建议初值，并含 RPO/RTO 与演练要求
- [ ] 变更（CSI 升级、有状态服务重建）给出分批、灰度与回滚（含数据回滚）路径
