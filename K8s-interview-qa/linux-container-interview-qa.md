# linux-container-interview-qa

# Linux / 容器底层面试题全解析：隔离机制篇 × 运行时故障篇 × 内核调优篇

> 适用角色：SRE / 云原生运维 / 平台工程师面试（社招 3 年起 + 偏高阶）
> 姊妹篇：《Scheduler 篇》《Kubelet 篇》《网络 / CNI 篇》《控制面篇》《存储 / CSI 篇》
> 版本约定：以 **主流 5.x 内核 + systemd 245+ / cgroup v2 时代**为背景；内核参数、默认值、cgroup 文件名随内核与发行版差异很大，凡涉及具体取值处均标注"以所在环境 `sysctl -a` / `/sys/fs/cgroup` 实际内容为准"。

---

## 0. 使用说明

### 0.1 先建立一张"Linux 如何支撑容器"的全景图

```
┌──────────────────────── Linux Kernel ────────────────────────┐
│                                                               │
│  隔离（看得到什么）           限额（能用多少）                  │
│  ├─ namespace                 ├─ cgroup v2 (unified)          │
│  │   uts / pid / mount /      │   cpu / cpuset / memory /     │
│  │   net / ipc / user /       │   io / pids / hugetlb /       │
│  │   cgroup / time            │   rdma / misc                 │
│  │                            └─ (v1: 每 subsystem 独立层级)   │
│  ├─ capabilities（能做什么）：CAP_SYS_ADMIN / NET_RAW ...       │
│  ├─ seccomp（能调哪些 syscall）                                │
│  ├─ MAC：SELinux / AppArmor                                   │
│  ├─ dev cgroup + uevent（能看到/操作哪些设备）                  │
│  └─ overlayfs / tmpfs / bind mount（文件系统怎么给）            │
│                                                               │
│  网络数据面：netns + veth + bridge + netfilter(nft) + conntrack │
│              + TC/qdisc + XDP/eBPF                            │
└───────────────────────────────────────────────────────────────┘
```

**一句话总纲**：容器不是一个"东西"，而是 **namespace（看得见）+ cgroup（用多少）+ capabilities/seccomp（能做什么）+ rootfs（文件系统）** 四组机制的组合。这条主线在面试开场先说出来，后面所有问题都能挂上去。

### 0.2 数值约定

- **内核/sysctl 默认值**：随发行版与内核版本而变，务必现场核对；
- **建议值**：标为 *建议*，必须压测校准；
- **实测值**：本文档不提供任何未经实测的性能数字，一律给出**测量方法**。

---

# Part A · 隔离与限额原理篇（26 题）

## A1. namespace 有哪几类？各自隔离什么？

| Namespace | 隔离内容 | 容器中看得见的证据 |
| --- | --- | --- |
| **Mount** (`CLONE_NEWNS`) | 挂载点视图 | 容器里有自己的 `/`、`/proc` |
| **PID** (`CLONE_NEWPID`) | 进程号空间 | 容器里 `ps` 只看到自己的进程，PID 从 1 开始 |
| **Net** (`CLONE_NEWNET`) | 网卡/IP/路由/iptables/端口 | 每个 Pod 有自己的 netns 与 IP |
| **UTS** (`CLONE_NEWUTS`) | hostname / domainname | 容器 hostname = Pod 名 |
| **IPC** (`CLONE_NEWIPC`) | SystemV IPC / POSIX mq | 不同 Pod 的共享内存互不可见 |
| **User** (`CLONE_NEWUSER`) | uid/gid 映射 | 容器内 root ≠ 宿主 root（**rootless 的基础**） |
| **Cgroup** (`CLONE_NEWCGROUP`) | cgroup 根视图 | 容器里看 `/proc/self/cgroup` 是 `/` |
| **Time**（较新） | 部分时钟偏移 | 用于容器迁移/时钟虚拟化（版本相关） |

**必答的关键补充**：

- 命名空间通过 `/proc/<pid>/ns/*` 以文件描述符形式存在；**只要有一个 fd 或进程持有它，namespace 就不会被销毁**；
- 进入某个容器网络命名空间的排障命令：`nsenter -t <pid> -n <cmd>`（见 A3）。

## A2. 三个系统调用：clone / unshare / setns 分别做什么？

| 调用 | 作用 | 例子 |
| --- | --- | --- |
| **clone**(flags) | 创建新进程的同时创建新 namespace | runc 创建容器进程 |
| **unshare**(flags) | 为**当前进程**创建新 namespace 并加入 | `unshare -n bash`（只给自己一个 netns） |
| **setns**(fd, nstype) | 把当前进程**加入已存在的** namespace | `nsenter` 的实现基础 |

**必答**：这就是为什么"进入容器"有两条路：`kubectl exec`（走 CRI Exec，进程还在容器 cgroup/namespace 内）与 `nsenter -t <宿主PID>`（从宿主进）——后者在**容器没有 shell / 网络已断**时是唯一手段，也是面试加分点。

## A3. PID namespace 的特殊之处：为什么容器里的 PID 1 这么重要？

PID namespace 是**层级**的：子 namespace 的进程在父 namespace 里也有 PID（双重身份）。

**PID 1 的两大职责（超高频考点）**：

1. **必须 reap 僵尸进程**：容器孤儿进程会被 reparent 到 PID 1；如果 PID 1 不 `wait()`，僵尸会堆积 ⇒ `ps` 里一堆 `<defunct>`，PID 资源被占。
2. **信号处理的特殊性**：在 PID namespace 中作为 PID 1 的进程，**未处理的终止信号（SIGTERM 等）会被内核忽略**（除非它显式注册 handler）。这就是经典事故：**"容器里执行 **`kill -TERM 1`** 没反应" / "应用收不到 SIGTERM，滚动更新要等 30 秒才被 SIGKILL"**。

**解法（可直接背）**：

- 用 **exec 形式** 启动进程（`CMD ["app"]` 而不是 `CMD app "$@"` shell 形式），让业务进程直接当 PID 1；
- 或引入 `dumb-init`** / **`tini` 这样的小型 init，专门负责转发信号 + 回收僵尸。

```dockerfile
# Dockerfile 推荐写法
FROM debian
RUN apt-get update && apt-get install -y dumb-init
ENTRYPOINT ["/usr/bin/dumb-init", "--"]
CMD ["python", "app.py"]
```

## A4. user namespace 与 rootless：解决了什么？

**核心**：user namespace 允许把容器内的 uid 0 映射到宿主的**非特权 uid**（如 100000），因此：

- 容器里是 root，宿主上只是一个普通用户 ⇒ **逃逸后的破坏面大幅下降**；
- 可以在没有 root 的情况下跑容器（**rootless container/runc/podman**）。

**代价与限制（必答）**：

- 部分文件系统（overlayfs 在某些内核/发行版组合下）在 user ns 中受限 ⇒ 常用 fuse-overlayfs；
- 特权操作（如某些设备映射、部分网络能力）不可用；
- Kubernetes 侧 **User Namespace for Pods** 仍在演进阶段（**请以所用版本 Release Notes 确认**），别在面试里说成"生产默认可用"。

## A5. mount namespace 与 mount propagation：为什么容器里看不到新挂载？

- Mount namespace 隔离的是**挂载点视图**，不是数据；
- `mountPropagation` 决定挂载事件如何在宿者与容器间传播（详见存储篇 A21）；
- **忘记 **`Bidirectional` 是 CSI 节点插件、某些块设备工具失效的根因。

```bash
# 宿主侧看某个进程的挂载视图差异
cat /proc/<host-pid>/mountinfo | wc -l
cat /proc/<container-pid>/mountinfo | wc -l
nsenter -t <pid> -m mount | head
```

## A6. net namespace 与 veth pair 的底层（与网络篇联动）

```
创建 netns → 建 veth pair（两端各一个 MAC）→ 一端放进 netns（改名 eth0）→ 另一端留在 root netns
      → 配 IP + 路由 → 宿主侧按 CNI 决定挂 bridge 或走路由/tunnel
```

关键点：

- **每个 netns 有独立的路由表、iptables 规则、conntrack 条目、socket 端口空间** ⇒ 这也解释了 **hostNetwork 的 Pod 会占用宿主端口** 且**看不到 Service 的 DNAT 行为差异**；
- 排障手段：`ip netns`（仅对命名 netns 有效）、更通用的是 `nsenter -t <pid> -n ip a`。

```bash
# 找到某个容器 pause 进程在宿主上的 PID
crictl inspectp <sandbox-id> | jq .info.pid
# 进入它的网络命名空间
nsenter -t <pid> -n ip addr
nsenter -t <pid> -n ss -lntp
nsenter -t <pid> -n tcpdump -i eth0 -nn host 10.0.0.1
```

## A7. cgroup v1 与 cgroup v2 的本质区别？为什么社区在迁移？

| 维度 | cgroup v1 | **cgroup v2** |
| --- | --- | --- |
| 层级 | 每个 subsystem 一棵独立树，容易混乱 | **单一 unified 层级**，所有控制器同一棵树 |
| 内存语义 | memory 与 memsw、kmem 账目复杂 | 统一一套 `memory.*`，支持 `memory.min/low/high/max` |
| 委托（delegation） | 困难 | 原生支持（容器内再分层） |
| PSI | 无 | 有 `/proc/pressure/*`（见 C13） |
| IO 控制 | `blkio` 权重/限流语义混乱 | `io.weight` / `io.max` 语义清晰 |
| K8s 支持 | 传统 | 新版本逐步成为推荐路径（**具体支持矩阵以所用版本为准**） |

**面试答法**：迁移 v2 的收益是"**统一层级 + 更准确的资源账 + PSI 可观测**"，代价是 host OS/容器运行时/监控组件的兼容性要全部核对（**这是一次高风险的节点级变更，必须按节点池灰度**）。

## A8. CPU cgroup 的三个参数，以及最经典的 Throttling 事故

**v1**：`cpu.shares`（相对权重）、`cpu.cfs_period_us`（周期，常见 100ms）、`cpu.cfs_quota_us`（周期内可用 CPU 时间）
**v2**：`cpu.weight`、`cpu.max`（`$quota $period` 或 `max`）

```
cpu.max = 200000 100000     # 等价于 2 个核（200ms/100ms）
```

**K8s 的映射**：`resources.limits.cpu: 2` ⇒ `cpu.max = 200000 100000`；`requests.cpu: 1` ⇒ 主要转换为 `cpu.weight`（v2，范围 1\~10000）或 v1 的 `cpu.shares`（1024 基准）。

**经典事故（CPUThrottlingHigh，必考）**：

```
现象：容器 CPU 使用率并不高，但 P99 延迟很差、RT 抖动
原因：在一个 100ms 周期里，进程前 20ms 就把配额用完 ⇒ 剩下 80ms 被**限流（throttle）**，
      表现为停顿。平均使用率低 ≠ 没有被限流！
```

**证据字段**：

```bash
# v2
cat /sys/fs/cgroup/<path>/cpu.stat        # nr_throttled, throttled_usec（累计)
# v1
cat /sys/fs/cgroup/cpu/.../cpu.stat       # nr_periods, nr_throttled, throttled_time
```

```promql
# 容器级别 throttling（cAdvisor 暴露，指标名以实际环境为准）
sum by (pod, container) (rate(container_cpu_cfs_throttled_seconds_total[5m]))
# throttled 占比 = throttled_periods / total_periods
sum by (pod) (rate(container_cpu_cfs_throttled_periods_total[5m]))
  / clamp_min(sum by (pod) (rate(container_cpu_cfs_periods_total[5m])), 0.001) > 0.1
```

**治理建议（需压测校准）**：

1. **延迟敏感服务不要设 CPU limit**（只设 request），避免 quota 造成的毛刺；代价是失去"硬上限保护"⇒ 用节点配额/隔离补；
2. 若必须设：**把 limit 放大到 request 的 2\~****4 倍**作为"突发上限"，而不是等于 request；
3. 部分版本支持调整 `cpuCFSQuotaPeriod`（**feature gate 阶段，请以版本为准**）来降低抖动频率，但这是演进特性，生产要谨慎；
4. 绑核场景用 `cpuManagerPolicy: static`（见 A10）。

## A9. cpuset / CPU manager / NUMA：什么时候该绑核？

| 概念 | 作用 |
| --- | --- |
| `cpuset.cpus` (v1) / `cpuset.cpus.effective` (v2) | 允许进程在哪些 CPU 上运行 |
| Kubernetes `cpuManagerPolicy: static` | 满足条件（Guaranteed + 整数 CPU）时给 Pod **独占核**，其余 Pod 被从这些核上剔除 |
| `TopologyManager` | 让 CPU/设备/内存在 **同一 NUMA 节点**上分配 |
| NUMA 本地性 | 跨 NUMA 访问内存延迟与带宽都更差 |

**实践要点**：

- 必须配 `reservedSystemCPUs`（否则系统进程与 kubelet 会被挤到共享核上抖动）；
- 独占核**降低调度灵活性**，可能让节点出现"看着有资源但 Pod 放不下" ⇒ 需要对排不上队的事件做监控；
- HPC/低延迟网络类负载才需要，一般 Web 服务不必绑核。

```bash
numactl -H                       # 看 NUMA 拓扑
cat /sys/fs/cgroup/cpuset.cpus.effective
taskset -pc <pid>                # 看进程允许跑在哪些核
```

## A10. 内存 cgroup：为什么"进程没超 limit 却被 OOM"？（超经典）

**关键：cgroup 统计的内存包含 page cache。**

```
memory.current（v2） / memory.usage_in_bytes（v1）
    = RSS（匿名页，如堆内存）
    + page cache（文件页缓存，容器读写文件时产生）
    + slab / kernel memory（部分版本/配置下）
    + swap
```

**典型事故链**：

```
容器内大量写/读文件（日志、临时文件、备份导出）
   → page cache 暴涨
   → 计入 cgroup memory.usage
   → 触及 limit
   → cgroup OOM ⇒ 容器被杀，退出码 137
容器内 `free` 看到的 cache 虽然理论上是"可回收的"，但**内核回收不及时就会被判定为超限而触发 cgroup OOM**。
```

**排障字段**：

```bash
# v2
cat /sys/fs/cgroup/<path>/memory.stat
#   anon / file / slab / sock / shmem / file_dirty / file_writeback / inactive_file / active_file
cat /sys/fs/cgroup/<path>/memory.events        # oom_kill 计数！（最直接的证据）
cat /sys/fs/cgroup/<path>/memory.max           # 限制值
cat /sys/fs/cgroup/<path>/memory.current
# v1
cat /sys/fs/cgroup/memory/.../memory.usage_in_bytes
cat /sys/fs/cgroup/memory/.../memory.oom_control      # oom_kill 计数
```

```promql
# 容器 OOMKill（依赖 kube-state-metrics / cAdvisor）
increase(kube_pod_container_status_terminated_reason{reason="OOMKilled"}[15m])
# 工作集用量接近 limit
sum by (pod, container) (container_memory_working_set_bytes)
  / sum by (pod, container) (kube_pod_container_resource_limits{resource="memory"}) > 0.9
```

**治理**：

1. 明确"**cgroup 内存 = 应用内存 + page cache**"，给 Java/有磁盘 IO 的服务留出余量；
2. 大量 IO 的临时文件放 `emptyDir`**（或 memory medium）** 并配 `ephemeral-storage` limits，避免污染 page cache；
3. 容器侧 `limit` 要基于**含 cache 的峰值**设定（实测），不能只看 RSS；
4. 用 `memory.events` 的 `max`/`oom` 计数做告警，而不是等业务报 137。

## A11. OOM Killer 在 cgroup 场景怎么选进程？

两条通道：

| 场景 | 谁动手 | 依据 |
| --- | --- | --- |
| **容器内超过自身 limit** | **cgroup 级 OOM**（杀该 cgroup 内的进程） | 选该 cgroup 内 oom\_score 最高者 |
| **节点整体内存不足** | **全局 OOM killer** | **各进程的 **`oom_score_adj` |

`oom_score_adj` 由 kubelet 按 QoS 设置（详见 Kubelet 篇 Q14）：

| QoS | oom\_score\_adj 量级 |
| --- | --- |
| Guaranteed | −997（几乎不会被全局 OOM 选中） |
| Burstable | 依 request/allocatable 计算（通常是负数到正数区间） |
| BestEffort | 1000（最先被杀） |

```bash
# 查看某进程的 oom 倾向
cat /proc/<pid>/oom_score
cat /proc/<pid>/oom_score_adj
# 内核留下的证据
dmesg -T | grep -i "Out of memory"
journalctl -k | grep -i oom
```

> **必答**：**容器 OOM 的现场最可信证据是内核日志（memcg oom: killed + 进程列表）与 **`memory.events`** 的 oom 计数**，而不是"猜哪个进程占了内存"。

## A12. blkio / io cgroup：为什么 K8s 一直没有稳定的"IO 限速" API？

| 层 | v1 | v2 |
| --- | --- | --- |
| 权重 | `blkio.weight` / `blkio.bfq.weight` | `io.weight` |
| 上限 | `blkio.throttle.read_bps_device` / `write_iops_device` | `io.max`（`rbps/wbps/riops/wiops`） |
| 限制 | 只对某些 I/O 调度器有效；CFQ 时代已过 | 依赖支持 blkcg 的调度器（bfq/mq-deadline 等） |

**为什么难标准化（必答）**：

1. 需要内核 IO 栈支持，且**不同设备类型（HDD/NVMe）、不同调度器行为不一致**；
2. **buffer IO 的写入可能被"别人"记账**（回写时进程上下文已变）⇒ 归属不清；
3. K8s 目前稳定提供的是 `ephemeral-storage`** 的容量限制**，不是 IOPS/带宽限制；
4. 因此生产上 IO 隔离更多依赖 **专用节点池 / 存储侧 QoS / cgroup v2 手工配置**（见 C10）。

## A13. pids cgroup：fork bomb 怎么防？

- `pids.max` 限制该 cgroup 内的最大进程/线程数 ⇒ 超限的表现是 `resource temporarily unavailable`**（fork 失败）**，而不是 OOM；
- K8s 侧通过 kubelet 的 `--pod-max-pids` 给 Pod 设 pid 限制（结合 `--pod-pids-limit`，请按版本核对 flag 名）；
- Node level: `kernel.pid_max` / `threads-max` 是宿主上限；
- **排障**：`ps -eLf | grep -c <app>` 看线程数；`cat /proc/<pid>/status | grep Threads`。

## A14. devices cgroup 与设备直通：GPU 是怎么给到容器的？

- `devices` cgroup 控制 **能否 mknod / 读写某个设备节点**（黑名单式）；
- 容器运行时创建容器时把设备**挂载进容器的 **`/dev`，并把 cgroup devices 规则放行；
- GPU 场景的完整链路：device plugin 上报资源 → kubelet 申请 → **运行时 **`CreateContainer`** 里带上 devices/env/mounts** → 容器里看到 `/dev/nvidia0`。

```bash
crictl inspect <ctr> | jq '.info.runtimeSpec.linux.devices'
```

## A15. Capabilities：为什么容器不该给 CAP\_SYS\_ADMIN？

Linux 把 root 的特权拆成了几十个 capability。容器**默认只保留一小部分**（如 `CHOWN`、`SETUID`、`NET_BIND_SERVICE` 等），去掉了：`SYS_ADMIN`、`SYS_MODULE`、`SYS_TIME`、`NET_ADMIN`、`SYS_PTRACE`…

**必答要点**：

| 场景 | 说明 |
| --- | --- |
| `privileged: true` | 几乎给全所有 capability + 挂载宿主 `/dev` + 关闭部分隔离 ⇒ **等价于宿主 root**，只在极特殊场景（如 CNI DaemonSet、部分 CSI Node）使用 |
| 只想抓包 | 加 `NET_ADMIN` + `NET_RAW` 即可，不必 privileged |
| 想 ping / 绑低端口 | `NET_RAW` / `NET_BIND_SERVICE`（另注意 `net.ipv4.ping_group_range`） |
| 想 mount | `SYS_ADMIN`（**高危**，配合 mount propagation 等于能接触宿主 FS） |

```yaml
securityContext:
  capabilities:
    drop: ["ALL"]                # 强烈建议：先全丢
    add: ["NET_BIND_SERVICE"]
  privileged: false
  allowPrivilegeEscalation: false
  readOnlyRootFilesystem: true
  runAsNonRoot: true
```

## A16. seccomp：容器为什么有时会 "Operation not permitted"？

seccomp 通过 **BPF 规则**限定进程可调用的 syscall 集合。

K8s 侧三种策略：`RuntimeDefault` / `Localhost` / `Unconfined`（默认策略随版本演进，请以所用版本为准；建议显式声明而不要依赖默认）。

**经典事故（必考题）**：

```
现象：某个新镜像（或升级了 glibc/内核后）在旧节点上跑不起来：
      "unable to start container process: ... clone3: operation not permitted"
原因：较老的 seccomp 实现/容器运行时的 profile 里没有 `clone3` 这个 syscall，
      但新 glibc 的 pthread_create 会调用它。
解决：升级运行时与 seccomp 支持版本；或临时降级 profile（不推荐长期使用）。
```

**排查**：

```bash
grep -i seccomp /proc/<pid>/status        # Seccomp 字段：0/1/2
cat /proc/<pid>/status | grep -i cap      # CapEff / CapPrm
docker/crictl inspect | jq '..|.seccompProfile'
```

> 这道题的价值：它把"内核 / glibc / 运行时 / profile"四者的版本耦合讲清楚了。

## A17. SELinux 与 AppArmor 的区别？在容器里怎么起作用？

| 维度 | SELinux | AppArmor |
| --- | --- | --- |
| 实现 | 内核 LSM，基于标签（label/type enforcement） | 内核 LSM，基于路径的 profile |
| 主流发行版 | RHEL/CentOS/Rocky 系 | Ubuntu/Debian 系 |
| 容器用法 | `-Z`/config 给容器打标签；持久化卷要正确 relabel（`:Z`/`:z`） | `--security-opt apparmor=<profile>` |
| K8s 用法 | `securityContext.seLinuxOptions` | `securityContext.appArmorProfile`（较新的 API；老版本用 annotation） |

**常见坑**：**挂载卷后容器报 permission denied**，很可能是 SELinux 标签不对（而很多人第一反应是 chmod 777 —— 这是错的，会掩盖真正的问题）。

## A18. overlayfs：容器镜像是怎么"叠"起来的？

```
merged（容器看到的根目录）
   ↑
union mount
   ↑                    ↑
upperdir（可写层）     lowerdir（镜像各层，只读）
workdir（中间态）
```

**写时复制（Copy-up）规则**：

- 读文件 → 从 lower 层找；
- 改文件 → **把整个文件复制到 upper 层再改**（大文件一次改动成本高！）；
- 删文件 → 在 upper 层建 **whiteout** 标记，不真删 lower 层；
- 删目录 → **opaque whiteout**。

**由此产生的两个大规模坑（必答）**：

1. **inode 消耗翻倍/累积**：每个 lower 层的每个文件都会占 inode；镜像层多 + 文件多 ⇒ `df -i` 爆掉（表现为"No space left on device"但 `df -h` 还有空间）；
2. `df`** 释放滞后**：容器删了，宿主上被清理 soon, 但若有进程仍持有已删除文件的 fd，**空间不会释放**：

```bash
lsof -n | grep -i deleted          # 找出持有已删除文件的进程
df -i                              # inode 使用率
du -sh /var/lib/containerd /var/lib/docker
```

## A19. /dev/shm 为什么总是不够？（容器经典题）

容器运行时给每个容器默认的 `/dev/shm` **常常只有 64MiB**（默认 `shm-size` 较小，具体默认值随运行时而异）。

**典型表现**：

- Chrome/Chromium/Selenium：`Failed to create shared memory / bus error`
- PostgreSQL / Oracle / 某些 JVM：`No space left on device`（但 `df` 显示有空间！因为 `df` 看的是根，不是 `/dev/shm`）
- PyTorch DataLoader 多 worker：`unexpected EOF / bus error`

**解法**：

```yaml
volumes:
  - name: dshm
    emptyDir:
      medium: Memory         # tmpfs
      sizeLimit: 2Gi         # 建议按业务实测设定；注意它计入容器/Pod 的内存账
volumeMounts:
  - name: dshm
    mountPath: /dev/shm
```

> **注意**：tmpfs 内容计入 **memory cgroup**（anonymous page），所以要一并提高 memory limit，否则会把内存打爆。

## A20. ulimit 与文件描述符：三层限制分别是什么？

```
① 系统级：/proc/sys/fs/file-max（全部进程打开 fd 总数上限）
          /proc/sys/fs/nr_open（单个进程 fd 上限的内核侧上限）
② 用户/会话级：/etc/security/limits.conf、systemd DefaultLimitNOFILE
③ 容器/进程级：运行时创建容器时设置的 RLIMIT_NOFILE
```

**必答的 K8s 陷阱**：

- **Pod spec 里没有稳定的 ulimit 字段**（社区曾多次提案，状态请以版本为准）⇒ 实践中只能在**容器运行时层**配默认值（如 containerd CRI 配置或 `-systemd/open files`），或者依赖 systemd slice；
- "too many open files" 时要分清是 **fd 耗尽** 还是 **inotify watches 耗尽**（后者见 C5），两者报错容易被混淆。

```bash
cat /proc/<pid>/limits | grep -i "open files"
ls /proc/<pid>/fd | wc -l
cat /proc/sys/fs/file-nr
```

## A21. 为什么容器里的 `top` / `free` 看到的是宿主机的数值？（经典）

因为 `/proc/meminfo`、`/proc/stat`、`/proc/cpuinfo` **默认不受 namespace 隔离**（它们属于 procfs，不是 cgroup）。

**后果**：

- Java/Python 应用在容器里"看到的可用内存/CPU 核数"是宿主的 ⇒ 线程池、堆参数算错 ⇒ OOM/超卖；
- `kubectl top` 与容器内 `free` 数值不一致（**前者来自 cgroup 统计，是对的**）。

**治理**：

1. **应用侧显式读 cgroup**（现代 JVM/Go/Node 已原生支持 cgroup-aware）：
   ```bash
   cat /sys/fs/cgroup/memory.max          # 可用上限
   cat /sys/fs/cgroup/cpu.max             # CPU 配额
   ```
2. 或挂 **lxcfs** 提供容器化的 `/proc` 视图（适合遗留应用）；
3. 始终明确：**权威的资源用量来源应该是 cgroup（**`cpu.stat`**、**`memory.stat`**），而不是 procfs**。

## A22. cgroup driver 为什么要和 systemd 一致？（K8s 经典故障）

**问题现象**：

```
节点上存在两套 cgroup 路径：
  systemd 管理的一方：/sys/fs/cgroup/system.slice/...  (由 systemd 创建)
  cgroupfs 的一方：   /sys/fs/cgroup/kubepods.slice/... (由 kubelet/runtime 直接创建)
两者互不知情 ⇒ 资源统计错位、limit 不生效、甚至节点内存失控
```

**规则**：**kubelet 的 **`cgroupDriver`** 必须与容器运行时保持一致**，通常统一为 `systemd`。

```yaml
# kubelet
apiVersion: kubelet.config.k8s.io/v1beta1
kind: KubeletConfiguration
cgroupDriver: systemd
```

```toml
# containerd (CRI 配置片段，版本相关)
[plugins."io.containerd.grpc.v1.cri"]
  systemd_cgroup = true
```

**检查方式**：

```bash
systemctl status kubelet | grep -i cgroup
systemd-cgls                             # 看 systemd 视角的层级
cat /sys/fs/cgroup/cgroup.controllers    # v2 顶层
```

> **这是"改了不生效"类问题的头号嫌疑对象**，改完必须**重启 kubelet 并驱逐/重建节点上的 Pod** 才彻底生效（属重操作，务必按节点池灰度）。

## A23. systemd slice 层级长什么样？

```
-.slice
└── kubepods.slice                       # 所有 Pod 的总父
    ├── kubepods-burstable.slice
    │    └── kubepods-burstable-pod<uid>.slice
    │         ├── cri-containerd-<cid>.scope     # 每个容器一个 scope
    └── kubepods-besteffort.slice
    ...
system.slice                             # 系统服务（含 kubelet、containerd）
```

```bash
systemd-cgls -u kubepods.slice | head -40
systemctl status kubepods-burstable-pod<uid>.slice
systemd-cgtop -m                         # 按 cgroup 看资源占用
systemd-run --scope -p MemoryMax=1G sleep 60      # 排障时临时用 systemd 限个小任务
```

**必答**：理解这个层级，才能解释"`kubeReserved` 需要指定 cgroup 路径"的原因（见 Kubelet 篇 Q13）——**路径写错，预留就白配**。

## A24. eBPF 到底能做什么？在容器场景的典型用法？

它是一个在**内核虚拟机**里安全执行用户程序的机制，通过钩子点挂载：

| 钩子 | 用途 |
| --- | --- |
| `kprobe` / `kretprobe` | 追踪内核函数（如定位慢 IO、调度延迟） |
| `uprobe` | 追踪用户态函数 |
| `tracepoint` / raw tracepoint | 稳定内核事件点 |
| **TC ingress/egress** | 容器网络转发、策略（Cilium 的数据面） |
| **XDP** | 网卡驱动层，最高性能（DDoS 防护、负载均衡） |
| **cgroup 钩子** | 按 cgroup 做 socket 层处理（Cilium socket-LB 的基础） |
| `LSM` 钩子 | 安全审计/阻断（Tetragon 等） |

**工具链**：bcc、bpftrace、libbpf/CO-RE、Cilium/Hubble、Tetragon、Inspektor Gadget、Pixie。

```bash
# 快速定位谁在执行 execve / 谁在制造短连接
execsnoop-bpfcc
tcpconnect-bpfcc
biolatency-bpfcc -m
funclatency-bpfcc 'vfs_*'
```

> **面试加分**：eBPF 让"不重启、不改内核、不侵入应用"的可观测与安全成为可能，也是当前网络/安全方案的主流底座（与网络篇 Cilium 章节呼应）。

## A25. 容器里的时间、时区与时钟漂移

- 容器默认**共享宿主内核的时钟**（time namespace 隔离部分于 newer kernels），所以"改容器时间"通常是在改宿主时间（危险）；
- **时区**由 `/etc/localtime` 决定 ⇒ 挂载宿主文件或在镜像里 `TZ=Asia/Shanghai`；
- **时钟同步**由宿主的 chrony/ntpd 保证；**时钟漂移会导致证书校验失败、日志错序、 leases 失效** ⇒ 必须监控。

```promql
# 节点时钟偏移（node_exporter 的 timex collector 需开启）
abs(node_timex_offset_seconds) > 0.05
```

## A26. Linux 网络栈层的容器相关要点（与网络篇联动）

只在 Linux 层需要记住的三件事：

1. **必须开启 IP 转发**：`net.ipv4.ip_forward = 1`（否则跨节点/跨网卡转发全废）；
2. `net.bridge.bridge-nf-call-iptables = 1`：让**经过网桥的流量也走 iptables**（Service 的 DNAT 才能对 Pod 流量生效）——这是 CNI 报错排查里的经典项；
3. **conntrack 机制**：所有 NAT（Service nodePort/masquerade）依赖 conntrack 表（**见网络篇 A18，此处不再展开**）。

顺带两个常被问的参数：

- `net.ipv4.conf.all.rp_filter`：反向路径过滤，某些 CNI（如 Calico 的部分模式）会要求放松 ⇒ **按 CNI 官方要求设，不要求 newbies 别乱改**；
- `net.core.bpf_jit_enable`、`kernel.unprivileged_bpf_disabled`：与 eBPF 安全相关。

# Part B · 容器运行时与经典故障专项篇（14 题）

## B1. containerd / CRI-O / runc / shim 分别是什么？

```
kubelet ──CRI(gRPC)──► containerd / CRI-O       （高层：镜像、容器生命周期、CRI API）
                              │
                              └─► containerd-shim-runc-v2 ──► runc create/start
                                        │                        │
                                   持有容器 stdio 与退出状态      真正 clone namespaces + 设 cgroup + exec
                                   （所以 containerd 重启不影响运行中的容器）
```

**必答**：

- **runc** 是真正"启动容器进程"的 OCI 运行时，**一次性命令**，启动完就退出；
- **shim** 是常驻的"监护人"，负责 stdio 转发、exit code 收集，让 containerd 自身可以重启而不影响业务；
- **shim 泄漏**（成千上万个 shim 进程）会让节点 PID/内存吃紧，是高密度节点的经典问题。

```bash
ps -ef | grep containerd-shim | wc -l
crictl ps -a | wc -l
# 两者数量应该大致匹配；显著偏多 ⟹ shim 泄漏嫌疑
```

## B2. 安全容器（Kata / gVisor）什么时候用？代价是什么？

| 方案 | 隔离强度 | 性能开销 | 适用 |
| --- | --- | --- | --- |
| **runc**（默认） | 共享内核（namespace+cgroup） | 最小 | 绝大多数 |
| **gVisor (runsc)** | 用户态内核拦截 syscall | 中（syscall 密集场景明显） | 多租户 SaaS、不可信代码 |
| **Kata Containers** | **轻量虚拟机**（独立内核） | 启动慢一些、内存有额外开销 | 强隔离、合规要求 |

**必答**：K8s 侧通过 **RuntimeClass** 选择：

```yaml
apiVersion: node.k8s.io/v1
kind: RuntimeClass
metadata:
  name: kata
handler: kata
scheduling:
  nodeSelector:
    runtime: kata        # 只在支持该运行时的节点池调度
```

```yaml
spec:
  runtimeClassName: kata
```

## B3. 容器逃逸的常见面有多大？（安全必答题）

| 风险项 | 说明 | 缓解 |
| --- | --- | --- |
| `privileged: true` | 几乎等于宿主 root | **禁止**（除极少数系统 DaemonSet） |
| 挂载宿主 `/`（hostPath） | 可改宿主文件/crontab | 策略引擎禁危险 hostPath |
| 挂载 `/var/run/docker.sock` | 直接控制宿主容器 | **绝对禁止** |
| `CAP_SYS_ADMIN` + writable cgroup | 可利用 `release_agent` 提权 | drop ALL + `allowPrivilegeEscalation: false` |
| 可写 `/proc/sys`、`/sys` | 改内核参数影响全节点 | 只读挂载 + 限制 sysctl |
| 宿主内核漏洞 | 共享内核是原罪 | 及时打补丁、必要时上安全容器 |

**K8s 侧的组合防线**：

- **Pod Security Admission（baseline / restricted）**；
- `securityContext`：runAsNonRoot + readOnlyRootFilesystem + drop ALL + seccomp RuntimeDefault；
- OPA/Kyverno 做自定义策略；
- 运行时行为监控（Falco / Tetragon）。

```yaml
apiVersion: v1
kind: Pod
spec:
  securityContext:
    runAsNonRoot: true
    seccompProfile:
      type: RuntimeDefault
  containers:
    - name: app
      securityContext:
        allowPrivilegeEscalation: false
        readOnlyRootFilesystem: true
        capabilities: {drop: ["ALL"]}
```

## B4. 经典容器"玄学故障"15 例（面试杀手锏，建议直接背）

| # | 现象 | 根因 | 处置 |
| --- | --- | --- | --- |
| 1 | 容器内 `kill -TERM 1` 无反应 | PID 1 的未处理终止信号被忽略 | exec 形式入口 / dumb-init |
| 2 | Pod 里一堆 `<defunct>` | PID 1 不 reap 僵尸 | tini/dumb-init |
| 3 | `No space left on device` 但 `df -h` 有空间 | **inode 耗尽** 或 `/dev/shm` 太小 | `df -i`；扩 `/dev/shm` |
| 4 | 删了容器/大文件磁盘没释放 | 有进程持有已删除文件的 fd | `lsof ``\| grep deleted` |
| 5 | JVM 服务无事 OOM | cgroup memory 含 page cache | `memory.stat`；调 limit |
| 6 | CPU 使用率不高但 RT 抖 | **CFS throttling** | `cpu.stat nr_throttled`；放大 limit |
| 7 | 容器内 `free` 看到宿主内存 | procfs 不隔离 | 用 cgroup 数值；必要时 lxcfs |
| 8 | `ping: operation not permitted` | 缺 `NET_RAW` + `ping_group_range` | 加 cap 或调参 |
| 9 | 容器内 ulimit 不对 | Pod spec 无稳定 ulimit API | 运行时侧默认 + systemd |
| 10 | `too many open files` | fd 耗尽 **或** inotify watches 耗尽 | `/proc/<pid>/limits`、`/proc/sys/fs/inotify/*` |
| 11 | 滚动更新时应用收不到信号 | shell 形式入口 + preStop 未配 | exec 形式 + preStop |
| 12 | Java 线程数突然失败 | **pids cgroup** 超限 | `pids.max` / `--pod-max-pids` |
| 13 | 容器内时区错 | `/etc/localtime` 未挂载 | 挂 hosts path 或设 TZ |
| 14 | 挂载卷后 permission denied | SELinux 标签/fsGroup 未处理 | relabel / 配 fsGroup（**别 chmod 777 糊过去**） |
| 15 | `read-only file system` 写报错 | `readOnlyRootFilesystem: true` | 给需要写的目录挂 emptyDir |

## B5. 为什么"容器内 top/free/df 不准"会引发级联问题？

见 A21。核心危害是**应用按宿主资源做自我限流决策**（线程池大小、JVM 堆、Go GOMAXPROCS），结果是"超卖 + OOM"。所以：

- JVM 需 `-XX:+UseContainerSupport`（现代 JDK 默认）；
- Go runtime 默认 GOMAXPROCS=CPU limit 相关行为随版本演进，**高并发服务建议显式确认或设置**；
- 中间件（Nginx `worker_processes auto`、MySQL `innodb_buffer_pool_size`）同样要按 cgroup 校准。

## B6. cgroup v1 的 kmem accounting 泄漏（深度事故题）

**现象**：节点运行一段时间后出现莫名其妙的内存占用、`cgroup limit exceeded` 或 `cannot allocate memory`，且账面（`memory.usage_in_bytes`）对不上。

**根因**：某些内核版本 + cgroup v1 的 **kmem accounting**（对 slab/dentry/inode 记账）在容器删除后**没有释放**（当时的内核 kmem 计费 bug），累积到一定程度会耗尽父 cgroup 甚至占满宿主的账面 memory ⇒ 连带全局 OOM。

**处置与预防**：

- 升级到修复版本的内核；
- 或在内核启动参数关闭：`cgroup.memory=nokmem`（**需重启节点，属重大变更**）；
- 使用 containerd 时检查相关 CVE/修复版本。

> 这题是"L4 有事故经验"的强证据：**能说出"内存账对不上要看 kmem/slab"** 的候选人不多。

## B7. 共享内存 /dev/shm 事故复盘（模板）

```
现象：PyTorch DataLoader 多 worker 训练任务随机崩溃：Bus error / unexpected EOF
定位：kubectl exec 进入容器 df -h /dev/shm ⇒ 只有 64M（默认）
修复：emptyDir(medium: Memory, sizeLimit: 8Gi) 挂载到 /dev/shm，并同步抬高 memory request/limit
验证：重跑同样 batch 不再崩溃；同时监控 container_memory_usage_bytes 未超限
```

## B8. 怎么看一个容器"真正用了多少资源"？（必考实操）

```bash
# ① 找到宿主上的容器 PID
crictl inspect <ctr-id> | jq .info.pid
PID=<pid>

# ② CPU：权威数据在 cgroup（不是 /proc/stat）
cat /sys/fs/cgroup$(cat /proc/$PID/cgroup | awk -F: '$2 ~ /cpu/ {print $3}')/cpu.stat 2>/dev/null
# 或者用 systemd 视角：
systemctl status $(ps -o slice= -p $PID)

# ③ 内存：
cat /sys/fs/cgroup/.../memory.stat
cat /sys/fs/cgroup/.../memory.events        # 有 oom 事件吗？

# ④ 谁占了 fd / socket / thread
ls /proc/$PID/fd | wc -l
cat /proc/$PID/status | grep -E "Threads|VmRSS"
nsenter -t $PID -n ss -s
```

## B9. crictl 与 docker 命令对照（排障必备）

| 目的 | crictl | docker（旧） |
| --- | --- | --- |
| 列 Pod | `crictl pods` | — |
| 列容器 | `crictl ps -a` | `docker ps -a` |
| 查看镜像 | `crictl images` | `docker images` |
| 容器日志 | `crictl logs <id>` | `docker logs` |
| 进入容器 | `crictl exec -it <id> sh` | `docker exec` |
| 详情 | `crictl inspect <id>` | `docker inspect` |
| 清理无用镜像 | `crictl rmi --prune` | `docker image prune` |
| Pod 沙箱 | `crictl inspectp <id>` | — |

**必答**：生产环境**不要装 docker 只为排障**，用 `crictl` + `nsenter` + `cgroup 文件` 这套组合就够了（而且更贴近真实数据面）。

## B10. 为什么删除 Pod 后宿主机仍有残留进程？

常见原因：

- 进程处于 **D 状态**（不可中断睡眠，多在等待坏掉的 NFS/网络存储）⇒ 无法被 SIGKILL 立即回收；
- 容器内进程未退出（应用不处理信号）；
- kubelet 与 runtime 状态不一致；
- **僵尸**：父进程没 reap。

```bash
ps aux | awk '$8 ~ /D|Z/'          # D=不可中断睡眠，Z=僵尸
cat /proc/<pid>/stack              # 看内核栈（需要权限）
cat /proc/<pid>/wchan
```

**必答**：**D 状态进程是"节点可能需要重启"的强信号**（尤其是后端存储 hang 住的场景），此时应先 cordon 隔离节点，确认业务已漂走后再处理。

## B11. 容器启动到底慢在哪？（和冷启动话题联动）

```
拉取镜像 → 解包到 overlay lower 层（IO 密集）
       → runc create：clone namespaces + 建 cgroup + mount
       → runc start：execve 业务进程
       → 应用初始化（JVM/框架）
```

定位手段：

- `systemd-cgtop`、`iolatency` 看 IO；
- `funclatency`/`execsnoop` 看 exec 行为；
- `perf top -p <pid>` 看应用自身热点；
- 度量 → 对比：`kubelet_runtime_operations_duration_seconds`（见 Kubelet 篇）。

## B12. `/sys/fs/cgroup` 只读挂载导致的坑

容器内如果把 cgroup 文件系统**以可写方式挂载**，配合 `CAP_SYS_ADMIN` 就能改写自己的限额（提权手段）。所以：

- 默认只**只读**挂载 cgroupfs；
- 应用若需要读 cgroup 值，读只读挂载即可（足够）。

## B13. cgroup delegation 与"容器里再跑容器"

前提是 **cgroup v2 delegation**：把一棵子树的控制器委托给子层级 ⇒ 容器内的 systemd 才能管理自己的 cgroup。
常见场景：CI runner / DinD（Docker in Docker）/ KubeEdge 类场景。**注意它显著增加逃逸面**，一般不推荐在生产无条件开放。

## B14. 哪些系统级服务应该被限制以免拖垮节点？

- **容器运行时**本身要进 `kubeReserved`（通过 cgroup 真正限制）；
- **日志采集/Fluent Bit** 容易内存/CPU 失控 ⇒ 必须设 limits 与 backpressure；
- **监控 agent / node exporter** 同样要有 limits，避免被业务拖累反向拖累；
- **DaemonSet 全部要设 requests/limits**（它们分布在每个节点，溢出会以 N 倍放大影响）。

# Part C · 大规模节点内核 / 系统调优篇（14 题）

> 定义：**大规模 = ≥ 1000 节点 / 单节点 ≥ 50 Pod / 高并发短连接**。所有 sysctl 建议值都必须在你的机型与业务上压测校准。

## C1. 一份可直接落地的节点 sysctl 基线（示意，务必逐项验证）

```bash
cat > /etc/sysctl.d/99-k8s-node.conf <<'EOF'
# ---------- 网络转发与桥接 ----------
net.ipv4.ip_forward = 1
net.bridge.bridge-nf-call-iptables = 1
net.bridge.bridge-nf-call-ip6tables = 1

# ---------- conntrack（★ 短连接高并发必看）----------
net.netfilter.nf_conntrack_max = 1048576          # 建议初值，按实测并发调整
net.netfilter.nf_conntrack_buckets = 262144       # hash 桶数 ≈ max/4（部分环境需模块加载时设置）
net.netfilter.nf_conntrack_tcp_timeout_established = 86400
net.netfilter.nf_conntrack_tcp_timeout_time_wait = 30
net.netfilter.nf_conntrack_tcp_timeout_close_wait = 60

# ---------- 连接与端口 ----------
net.core.somaxconn = 32768                        # listen backlog，默认常过小
net.core.netdev_max_backlog = 10000               # 网卡收包队列，高 PPS 建议调大
net.ipv4.ip_local_port_range = 1024 65535
net.ipv4.tcp_tw_reuse = 1
net.ipv4.tcp_max_syn_backlog = 8192
net.ipv4.tcp_syncookies = 1

# ---------- 文件与 inotify ----------
fs.file-max = 1048576
fs.inotify.max_user_instances = 1024              # 大量 Pod 挂载 Secret/ConfigMap 时常耗尽
fs.inotify.max_user_watches = 1048576

# ---------- 虚拟内存 ----------
vm.swappiness = 10                                # K8s 通常不建议大量依赖 swap（见 C7）
vm.max_map_count = 262144                         # JVM / ES 类必看
vm.overcommit_memory = 0

# ---------- 其它 ----------
kernel.pid_max = 4194304
EOF
sysctl -p /etc/sysctl.d/99-k8s-node.conf
```

> **交付原则**：sysctl 是**节点级全局变更**，必须做到：① 配置版本化（Ansible/镜像）；② 按节点池灰度 1 台 → 10% → 50%；③ 保留回滚文件；④ 每次变更后跑一遍基准压测。

## C2. conntrack 容量怎么算？（和网络篇互补，这里给"账怎么算"）

```
nf_conntrack_max ≈ 预计并发连接数 × 1.5~2（安全系数）
buckets = nf_conntrack_max / 4    （每个桶一个链表，桶多则链表短）
每条连接约占 ~300 字节内核内存（量级估算，以实测为准）
```

```promql
node_nf_conntrack_entries / node_nf_conntrack_entries_limit > 0.75
rate(node_nf_conntrack_insert_failed[5m]) > 0
```

**必答**：**Cilium 的 socket-LB 路径不依赖 conntrack**，这是短连接极高并发场景从 Linux 层缓解此问题的关键手段（见网络篇）。

## C3. somaxconn 与 backlog：为什么默认不够？

- `net.core.somaxconn` 限制的是 **listen 队列长度**；应用如果有 backlog 参数（如 Nginx `listen ... backlog=`）取的是较小值；
- 队列满 ⇒ 新连接被丢弃（客户端看到超时/重传），且**服务端可能完全没有错误日志**；
- 高并发短连接服务要连带看 `tcp_max_syn_backlog`、`tcp_abort_on_overflow`（一般不建议开）。

## C4. epoll / fd：三层限制与"too many open files"

见 A20。补充高层手段：

```bash
# systemd 服务级默认（会影响以 systemd 方式运行的容器运行时等）
systemctl set-property containerd.service TasksMax=infinity
cat /etc/systemd/system.conf.d/99-defaults.conf
# [Manager]
# DefaultLimitNOFILE=1048576
# DefaultTasksMax=infinity
```

## C5. inotify 耗尽：为什么 K8s 节点特别容易中招？

**原因**：kubelet 对每个以 volume 方式挂载的 ConfigMap/Secret 都要 watch 文件变化，每个都要占 inotify instances/watches；Pod 密度高 + 大量 Secret ⇒ 直接耗尽。

**表现**：kubelet 报 `too many open files` / `inotify_init: failed`，节点上的 Pod 挂载开始失败。

**治理**：

- 调大 `fs.inotify.max_user_instances` 与 `max_user_watches`；
- **Secret 设 **`immutable: true`（正解，能去掉大量 watch）；
- 降低"每个 Pod 挂很多 Secret"的架构模式。

```promql
# node_exporter 需开启相应 collector，指标名以实际为准
node_filefd_allocated / node_filefd_maximum > 0.8
```

## C6. THP（透明大页）：为什么数据库类负载常常要求关掉？

THP 让内核自动把连续小页合并为 2MiB 大页，**在内存分配/回收频繁的场景会引入明显延迟尖峰**（khugepaged 整理、缺页中断变重）。Java heap、Redis、MySQL、MongoDB、各类数据库/中间件官方建议多为**关闭 THP**（或只保留 `madvise`）。

```bash
cat /sys/kernel/mm/transparent_hugepage/enabled
# 常见建议值（需实测）：[always] madvise never  → 关闭改为 never 或写入启动项 transparent_hugepage=never
echo never > /sys/kernel/mm/transparent_hugepage/enabled
```

## C7. swap：Kubernetes 到底该不该用？

| 立场 | 说明 |
| --- | --- |
| 传统 | kubelet 默认要求关闭 swap（`--fail-swap-on`），因为 swap 会让"内存用量"失去确定性，干扰 OOM 决策 |
| 演进 | 新版本对 swap 提供了受控支持（如仅限 cgroup v2 的受限 swap 行为，版本相关请核对 Release Notes） |
| 实践建议 | **在线延迟敏感服务不要依赖 swap**（一次 swap-in 可能带来数量级的延迟）；离线批处理可在受控场景下用 swap 提高装箱率 |

配套：

- `vm.swappiness` 调低（*建议 1\~**10*）；
- 若启用，**必须监控 swap 使用率与 swap-in/out 速率**，并明确有 kubelet、cgroup v2 的支持条件。

```promql
node_memory_SwapFree_bytes / node_memory_SwapTotal_bytes < 0.5
rate(node_vmstat_pswpin[5m]) > 0
```

## C8. 文件系统与磁盘：为什么推荐 imagefs 与 nodefs 分离？

**问题**：镜像层 + 容器可写层 + 容器日志共享一块磁盘时，任何一类暴涨都会导致 kubelet/运行时自身无法工作（**镜像/I/O 抖动 ⇒ PLEG 慢 ⇒ Node NotReady**）。

**治理**：

| 项 | 建议 |
| --- | --- |
| 分区 | `/var/lib/containerd`（imagefs）独占磁盘；`/var/lib/kubelet` 与根分区另设或留足空间 |
| 文件系统 | ext4 / xfs 皆可；大量小文件/超大目录场景 xfs 常更合适（**请实测**） |
| 挂载参数 | `noatime,nodiratime`（减少元数据写） |
| SSD | 开启周期性 discard（`fstrim.timer`）或在线 discard；监控 SSD 磨损 |
| 监控 | 分别对 imagefs、nodefs、inode 做容量告警 |

```bash
systemctl enable --now fstrim.timer
df -i /var/lib/containerd
```

## C9. PID / 线程：宿主与容器的双上限

```
容器级：pids.max（Pod 级由 --pod-max-pids 等设置；也与 pause loop 相关）
宿主级：kernel.pid_max、kernel.threads-max
子系统：systemd TasksMax（默认可能为 15% 之类，容易踩到）
```

**典型故障**：`resource temporarily unavailable` / `cannot allocate memory`（fork 失败常被误读成内存问题）。

```promql
# 节点 PID 使用率
node_procs_running / node_procs_threads > 0.8    # 仅为示意：实际请用 pids.current 或 exporter 指标
```

## C10. IO 隔离的现实手段（因为 K8s 没有稳定的 IO 限制 API）

优先级：

1. **专用节点池**（taint + toleration）——最有效；
2. **存储侧 QoS**（云盘 IOPS/吞吐配额、Ceph 的 mClock/qos）——最贴近；
3. **cgroup v2 io.weight / io.max**（需在运行时层或平台侧配置，**K8s 不直接提供**；务必兼容性与压测）；
4. 监控 ≥ 限制：至少先把"设备 IO 延迟/队列深度/IOPS"看清楚再谈限制。

```promql
# 磁盘 IO 利用率（node_exporter）
rate(node_disk_io_time_seconds_total[5m])
# IO 等待是对 CPU 延迟的直接伤害
rate(node_cpu_seconds_total{mode="iowait"}[5m])
```

## C11. 高 PPS 下的网络栈调优包

| 项 | 建议（需实测） |
| --- | --- |
| 网卡多队列 RSS | 队列数 ≥ CPU 核数，中断分散到多核 |
| `irqbalance` | 开启（或手动绑 IRQ，避免所有中断压在 CPU0） |
| RPS / RFS | 虚拟接口/隧道场景可考虑开启分散软中断 |
| 网卡 ring buffer | 高 PPS 适当调大（`ethtool -g/-G`） |
| `net.core.netdev_budget` / `netdev_budget_usecs` | 影响一次软中断处理包数，**调整需谨慎压测** |
| MTU | 物理网络允许时用 jumbo frame 降 PPS；**注意隧道的头开销**（见网络篇） |

```promql
# 软中断分布不均（单核打满的典型信号）
topk(5, rate(node_softirqs_total[5m]))
# 网卡丢包
rate(node_network_receive_drop_total[5m])
rate(node_network_receive_errs_total[5m])
```

## C12. NUMA 与 CPU 绑定的节点级配合

- `isolcpus` / `nohz_full` / `rcu_nocbs`（延迟敏感 + DPDK 类场景），但会减少通用调度域，慎用；
- 与 kubelet 的 `cpuManagerPolicy: static` + `topologyManagerPolicy` 一起做，且必须配 `reservedSystemCPUs`；
- 检查：`numactl -H`、`cat /proc/interrupts`、`numastat`（看跨 NUMA 内存访问）。

## C13. PSI（Pressure Stall Information）：节点压力的最现代画像

cgroup v2 时代，内核在 `/proc/pressure/{cpu,io,memory}` 提供压力指标：

```
some avg10=... avg60=... avg300=... total=...
full avg10=... （full 表示所有任务都被阻塞，最危险的信号）
```

**价值**：比"CPU 使用率"更能说明"应用到底有没有被饿到"——**CPU 使用率 100% 但 PSI cpu-some 很低，说明只是在跑，不饿；反之 PSI 高说明排队严重**。

```promql
# node_exporter 的 psi collector（需开启，指标名以 exporter 版本为准）
node_pressure_cpu_waiting_seconds_total
node_pressure_memory_waiting_seconds_total
node_pressure_io_waiting_seconds_total
# 建议先看 "some avg10" 的 gauge：node_pressure_cpu_some_avg10 之类
```

## C14. 节点级变更的标准灰度与回滚（Linux 层最容易忽视的）

| 阶段 | 范围 | 观察（≥30min，含一次高峰） | 回滚触发 |
| --- | --- | --- | --- |
| Step 1 | **1 台**（隔离/预发池） | PLEG relist、RT 抖动、软中断分布、 conntrack | 任一指标超基线 1.5× |
| Step 2 | 1% 节点 | + Pod 启动 P99、iowait | 同上 / 业务 P99 恶化 |
| Step 3 | 10% → 50% | 全 SLO Dashboard | 同上 |
| Step 4 | 100% | 稳定 ≥24h | — |

**回滚要点**：

- sysctl 回滚 = 恢复上一版配置并 `sysctl -p`（秒级）；
- **内核/cgroup 驱动/文件系统类变更不能原地回滚** ⇒ 必须 drain + 重建节点（这就是它们必须排在大版本升级窗口的原因）；
- 所有变更写入 "节点基线镜像"，保证新节点天然一致，避免配置漂移（**漂移是 Linux 层事故的最大来源**）。

# Part D · 反模式清单（可直接用于生产巡检）

| # | 反模式 | 证据（怎么发现） | 等级 | 修复动作 | 验证方式 |
| --- | --- | --- | --- | --- | --- |
| D1 | 延迟敏感在线服务设 CPU limit=requests | throttling 占比 \>10% | 🔴 高 | 去掉 limit 或放宽到 request 的 2\~4 倍 | `nr_throttled` 速率归零 + P99 改善 |
| D2 | 容器用 shell 形式入口（无 init） | 信号不生效、僵尸进程堆积 | 🔴 高 | exec 形式或 dumb-init | `kill -TERM 1` 生效、无 `<defunct>` |
| D3 | 生产用 `privileged: true` | Pod spec 审计 | 🔴 高 | 收敛 capability + PSS restricted | 策略引擎拦截生效 |
| D4 | kubelet 与运行时 cgroup driver 不一致 | 两条 cgroup 路径、limit 不生效 | 🔴 高 | 统一为 systemd | `systemd-cgls` 层级一致 |
| D5 | 只看平均 CPU 不看 throttling | RT 抖动却无头绪 | 🟠 中 | 加 throttled 指标与告警 | 压测时能复现并归因 |
| D6 | 配置节点后不做基线版本化 | 节点间参数漂移 | 🔴 高 | sysctl/运行时配置纳入节点镜像 | 巡检脚本比对一致 |
| D7 | inotify/ulimit/fd 无监控 | 突然大量挂载失败 | 🟠 中 | 调参 + Secret immutable + 告警 | 压力密度下无 limitations 报错 |
| D8 | imagefs 与 nodefs 共用根盘 | 镜像暴涨导致节点不可用 | 🔴 高 | 分离磁盘 + GC 阈值 | 写满演练不影响 kubelet |
| D9 | 忽视 inode（`df -h` 只看容量） | 空间够却报 No space | 🟠 中 | `df -i` 纳入监控 | inode 告警提前触发 |
| D10 | THP 未关（数据库节点） | RT 偶发尖峰 | 🟠 中 | 关闭 THP（按厂商建议） | 压测 P99 平稳 |
| D11 | swap 随意开启且无监控 | 偶发巨长尾延迟 | 🟠 中 | 明确策略 + 监控 swap-in/out | 在线服务无 swap 抖动 |
| D12 | DaemonSet 不设 resources | 节点被 agent 拖垮 | 🟠 中 | 全部设 requests/limits | 压测下节点资源有冗余 |
| D13 | 未在 CI 里校验 cgroup limit 与应用参数匹配 | JVM 按宿主内存算堆 | 🟠 中 | cgroup-aware + 显式参数 | 容器压测不 OOM |
| D14 | 直接 `chmod 777` 解决挂载权限 | 掩盖 SELinux/fsGroup 真因 | 🟠 中 | 正确 relabel / fsGroup + OnRootMismatch | 无 777 且功能正常 |
| D15 | 内核一次性全量升级 | 大面积驱动/运行时不兼容 | 🔴 高 | 节点池灰度 + drain/回滚窗口 | 灰度台各项指标达标 |

---

# Part E · 可观测性：指标与告警

## E1. 必看的指标清单（**先 **`curl http://<node>:9100/metrics`** 与 cgroup 文件核对**）

| 层面 | 指标 / 数据源 | 用途 |
| --- | --- | --- |
| CPU 限额 | `container_cpu_cfs_throttled_periods_total` / `_periods_total` / `_throttled_seconds_total` | throttling 检测（**核心**） |
| CPU 使用 | `container_cpu_usage_seconds_total` | 真实用量 |
| 内存 | `container_memory_working_set_bytes` / `_rss` / `_cache` | 内存账；working\_set 是 OOM 判定更接近的量 |
| OOM | `kube_pod_container_status_terminated_reason{reason="OOMKilled"}` | 容器 OOM |
| cgroup 原生 | `cpu.stat(nr_throttled/throttled_usec)`、`memory.stat`、`memory.events(oom)`、`pids.current`、`io.stat` | **权威真相**（hostCom基层党组织） |
| 进程 | `node_procs_running` / `_blocked`（D 状态） / `_zombies` | 进程健康 |
| fd / inode | `node_filefd_allocated` / `node_filesystem_files_free` | fd 与 inode |
| 内存 | `node_memory_*` + `node_vmstat_pswpin/pswpout` | 节点内存与 swap |
| 网络 | `node_network_*`、softirq、丢包 | 高 PPS 与软中断 |
| conntrack | `node_nf_conntrack_*` | NAT 表容量 |
| PSI | node\_exporter psi collector（开启后） | 真实的"被饿到"程度 |
| 时钟 | `node_timex_offset_seconds` | 时钟漂移 |

## E2. Prometheus 告警规则（可直接粘贴，阈值需实测校准）

```yaml
groups:
  - name: linux-container
    rules:
      - alert: ContainerThrottlingHigh
        expr: |
          sum by (namespace, pod, container) (rate(container_cpu_cfs_throttled_periods_total[5m]))
            / clamp_min(sum by (namespace, pod, container) (rate(container_cpu_cfs_periods_total[5m])), 0.001) > 0.1
        for: 15m
        labels: {severity: warning}
        annotations:
          summary: "容器 {{ $labels.namespace }}/{{ $labels.pod }}/{{ $labels.container }} CPU 被限流比例过高"

      - alert: ContainerOOMKilled
        expr: increase(kube_pod_container_status_terminated_reason{reason="OOMKilled"}[15m]) > 0
        for: 2m
        labels: {severity: critical}
        annotations:
          summary: "发生容器 OOMKill：{{ $labels.namespace }}/{{ $labels.pod }}"

      - alert: NodeConntrackNearFull
        expr: node_nf_conntrack_entries / node_nf_conntrack_entries_limit > 0.75
        for: 10m
        labels: {severity: warning}

      - alert: NodeFileDescriptorNearLimit
        expr: node_filefd_allocated / node_filefd_maximum > 0.8
        for: 10m
        labels: {severity: warning}

      - alert: NodeInodeAlmostExhausted
        expr: node_filesystem_files_free / node_filesystem_files < 0.1
        for: 15m
        labels: {severity: warning}
        annotations:
          summary: "{{ $labels.mountpoint }} inode 剩余不足 10%"

      - alert: NodeManyBlockedProcesses
        expr: node_procs_blocked > 10
        for: 10m
        labels: {severity: warning}
        annotations:
          summary: "节点 {{ $labels.instance }} 存在较多 D 状态进程，可能存在存储 hang"

      - alert: NodeZombiesHigh
        expr: node_procs_zombies > 50
        for: 15m
        labels: {severity: info}
        annotations:
          summary: "僵尸进程偏多，检查容器 PID 1 是否 reap"

      - alert: NodeClockDrift
        expr: abs(node_timex_offset_seconds) > 0.05
        for: 10m
        labels: {severity: warning}

      - alert: NodeSwapUsed
        expr: |
          (node_memory_SwapTotal_bytes - node_memory_SwapFree_bytes) / node_memory_SwapTotal_bytes > 0.1
        for: 15m
        labels: {severity: info}
        annotations:
          summary: "节点开始使用 swap，在线服务可能出现长尾延迟"
```

## E3. 建议的 SLO（**初始建议值，必须实测校准**）

| SLI | 建议目标 | 度量 |
| --- | --- | --- |
| 容器 CPU 限流占比 | ≤ 5%（在线） | throttled\_periods / periods |
| 容器 OOMKill 次数 | 在线 ≈ 0 | terminated\_reason=OOMKilled |
| conntrack 使用率 | ≤ 75% | node exporter |
| inode 剩余 | ≥ 10% | node exporter |
| 时钟偏移 | ≤ 50ms | node\_timex\_offset\_seconds |
| 节点 D 状态进程峰值 | ≤ 10 | node\_procs\_blocked |
| fd 使用率 | ≤ 80% | node\_filefd\_\* |

---

# Part F · 演练（GameDay）计划表

| 场景 | 注入方式 | 期望表现 | 验证的 SLO |
| --- | --- | --- | --- |
| CPU throttling 事故复现 | 给某容器设很小的 cpu limit 后跑计算 | 告警能识别出 throttling 而非"CPU 高" | 限流占比 SLO |
| 容器 OOM 复现 | 容器疯狂写文件制造 page cache | 能从 `memory.events` 的 oom 计数与内核日志确认归因，而非误判为 RSS 超限 | OOM 归因准确性 |
| /dev/shm 耗尽 | 跑多 worker 训练任务 | 有明确报错与处理手册 | 故障处理 MTTR |
| fd/inotify 耗尽 | 制造大量挂载/打开文件 | 告警提前触发；immutable Secret 能缓解 | 密度上限认知 |
| D 状态进程 | 让后端存储 hang | 能识别 cordon 决策链 | 节点隔离流程 |
| 内核参数漂移 | 手动改一台机器的 sysctl | 巡检能发现不一致 | 配置一致性 |
| conntrack 打满 | 短连接压测 | 告警早于丢包 | 连接成功率 |
| 时区/时钟漂移 | 停止 chrony 并手动偏移 | 时钟告警触发，证书/lease 行为可解释 | 时钟 SLO |
| 镜像盘写满 | 疯狂拉取镜像 | imagefs 告警 + kubelet 仍存活（磁盘分离有效） | 磁盘隔离有效性 |
| cgroup v2 迁移演练 | 独立节点池先迁 | 指标、PSI、编译差异、运行时兼容全部验证 | 迁移可行性 |

---

# Part G · 10 张面试速记卡（临阵磨枪）

1. **容器 = namespace（看得见）+ cgroup（用多少）+ capabilities/seccomp（能做什么）+ rootfs（文件系统）**。开场说这句，后面全部挂得上去。
2. **八个 namespace**：mount / pid / net / uts / ipc / user / cgroup / time；`/proc/<pid>/ns/*` 存在 ⇒ namespace 不灭。
3. **PID 1 两件事**：转发信号（**未注册的终止信号会被内核忽略**）与 reap 僵尸。解法：exec 入口或 dumb-init。
4. **cgroup memory 含 page cache** ⇒ "进程没超内存却被 OOM"的真相；`memory.events` 的 oom 计数是最硬证据。
5. **CFS throttling**：平均 CPU 不高也会被限流，看 `cpu.stat` 的 `nr_throttled`；在线服务建议少设/放宽 CPU limit。
6. **cgroupDriver 必须和 runtime 一致**（通常 systemd），否则两套层级、限制失效。
7. `privileged`** ≈ 宿主 root**；正确的姿势是 drop ALL + 按需加 + `runAsNonRoot` + `readOnlyRootFilesystem` + seccomp RuntimeDefault。
8. **overlayfs 的代价是 inode**：`No space left on device` 先看 `df -i`，再看有没有进程持有已删除文件（`lsof | grep deleted`）。
9. **容器里 **`top/free`** 看的是宿主**（procfs 不隔离）；权威数据来源是 `/sys/fs/cgroup` 与 cAdvisor。
10. **节点参数必须版本化**：sysctl/运行时配置进节点镜像，按池灰度 1 台 → 1% → 100%，内核/cgroup 驱动类变更不可原地回滚（要 drain 重建）。

---

## 附：文档自查清单

- [ ] 每条建议配了可直接套用的 Dockerfile/YAML/sysctl/PromQL/排障命令
- [ ] 所有 sysctl 与文件名（v1/v2 差异）标注了"以所在环境实际为准"
- [ ] 未提供任何未经实测的性能数字，均给出**测量方法**
- [ ] 反模式清单含：证据 / 等级 / 修复 / 验证
- [ ] SLO 标为建议初值
- [ ] 节点级变更给出灰度比例、回滚方式与"不可原地回滚"的风险提示
