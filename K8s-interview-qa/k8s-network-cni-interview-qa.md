# k8s-network-cni-interview-qa

# Kubernetes 网络 / CNI 面试题全解析：原理篇 × 方案模式篇 × 大规模优化篇

> 适用角色：SRE / 云原生运维 / 平台工程师面试（社招 3 年起 + 偏高阶）
> 姊妹篇：《Kubernetes Scheduler 面试题全解析》《Kubernetes Kubelet 面试题全解析》
> 版本约定：以 **1.28 \~**** 1.31 主线**为准；涉及默认值、feature gate、GA/Beta 状态、内核版本要求的地方，一律标注"以所用版本官方 Release Notes / 插件文档为准"。

---

## 0. 使用说明

### 0.1 网络方向的面试考察层次

| 层级 | 考点 | 表现 |
| --- | --- | --- |
| L1 会用 | 会写 Service/Ingress/NetworkPolicy YAML | 只会 apply |
| L2 懂原理 | 说得清 Pod 出网封包路径、CNI 干了什么、ClusterIP 为什么不存在实体网卡、DNS 怎么走 | 能画封包路径图 |
| L3 懂选型 | 能比较 Overlay / 路由 / ENI 直路由三类模式的代价，能报出对应 CNI 的组件与配置项 | 能说 trade-off |
| L4 有事故经验 | 能讲 DNS 5 秒抖动、conntrack 满、iptables 规则膨胀、IP 耗尽、MTU 黑洞的根因与治理 | 有 PromQL、有灰度、有回滚 |

> **答题模板**：封包路径（从 Pod 网卡一路讲到目标网卡）→ 谁在这条路上做了什么（CNI/kube-proxy/CoreDNS) → 给证据（配置项 / metric / 内核参数）→ 补坑。

### 0.2 数值约定

- **源码/官方默认值**：标注字段，提示现场核对；
- **建议值**：标为 *建议*，必须压测校准；
- **实测值**：本文档不提供未经实测的性能数字（吞吐、PPS、延迟提升百分比），一律给出**测量与对比方法**。

---

# Part A · 原理篇（26 题）

## A1. Kubernetes 的网络模型是什么？为什么这样设计？

**四大假设（必须背）**：

1. **所有 Pod 之间可以不经过 NAT 直接通信**（同一 IP 平面）；
2. **Node 与所有 Pod 之间可以直接通信**（方便健康检查、日志采集、管控）；
3. **Pod 看到的自己的 IP 与别人看到的它的 IP 是同一个 IP**（身份自洽）；
4. 网络的实现由 **CNI 插件**负责，K8s 本身只定义要求、不实现。

**为什么这样设计**：把网络（网络地址分配与互通）与编排（容器生命周期）解耦，任何满足 CNI 规范的实现都能接入；同时"扁平 IP 网络"让服务发现、网络策略、可观测性都能统一在一套 IP 语义上。

**必答补充**：K8s **不要求**：

- 不做负载均衡（那是 Service/kube-proxy/LB 的事）；
- 不做 DNS（CoreDNS 是可选组件）；
- 不做网络策略（NetworkPolicy 是 API，需要 CNI 实现，**不实现就完全不生效且默认放通**）。

---

## A2. Pod 到底是怎么"有网"的？把封包路径讲一遍

```
                   ┌─────────────────── Node ───────────────────┐
  Pod netns        │                                             │
  ┌─────────┐      │  root netns                                 │
  │ eth0    │◄─────┼──► vethXXXX@ifNN  ←→ 宿主 bridge / 路由表     │
  │  路由   │      │      │                                       │
  │ default │      │      ├── CNI 下发：veth pair + IP + 路由       │
  │ via eth0│      │      ├── 跨节点：走 overlay 隧道 或 宿主路由表  │
  └─────────┘      │      │            （下一跳是对端节点 IP）      │
       ▲           │      └── 出公网/Service：iptables/IPVS + NAT   │
       │           │                                               │
   pause sandbox   │  kube-proxy 维护 Service VIP 规则              │
   持有 netns/IP   └───────────────────────────────────────────────┘
```

**创建流程（kubelet → CNI）**：

```
kubelet → CRI RunPodSandbox → CNI ADD
    Plugin 完成：
      1. 创建 netns（由 sandbox pause 容器持有）
      2. 建 veth pair：一端进 Pod netns 改名 eth0，另一端留在 root netns
      3. 把宿主侧 veth 挂到 bridge / 或直接由 CNI 写路由
      4. 调用 IPAM 申请 IP
      5. 配置 Pod 内 IP + 默认路由 + ARP 代理/neigh
      6. 返回结果（含 IP）→ kubelet 写入 Pod status
删除时反向 CNI DEL
```

**必答要点**：

- **Pod IP 实际在 pause / sandbox 容器上**，业务容器共享这个 netns（所以同一 Pod 内各容器 `localhost` 互通）；
- **CNI ADD 失败 ⇒ **`FailedCreatePodSandBox`，Pod 永远进不了 Running；
- CNI 配置文件在 `/etc/cni/net.d/*.conf*`（按字典序取第一个有效配置），二进制在 `/opt/cni/bin/`。

---

## A3. CNI 规范本身规定了什么？为什么 kubelet 不自己实现 IPAM？

**CNI 规定的三件事**：

1. 配置文件格式（JSON，含 `cniVersion`、`type`、IPAM 段）；
2. 二进制插件通过 **stdin + 环境变量 + 配置文件路径**被调用；
3. 必须实现 `ADD` / `DEL` / `CHECK` / `VERSION`（kubelet 主要只用 ADD/DEL）。

**插件分类**：

| 类型 | 职责 | 例子 |
| --- | --- | --- |
| **main / interface 插件** | 真正建网卡和数据面 | bridge、ptp、macvlan、ipvlan、host-local route、calico、cilium-cni |
| **IPAM 插件** | 只管地址分配 | host-local、static、whereabouts、Calico/Cilium 自带的 IPAM |
| **meta 插件** | 组合与增强 | portmap（hostPort）、bandwidth（限速）、tuning、firewall |

**链式调用**：配置里是 list 时，插件按顺序执行，**前一个的结果传给后一个**。

**为什么不由 kubelet 实现**：这本身就是为了解耦——不同环境（云网络、IDC、DPDK、SR-IOV）需要完全不同的数据面实现，kubelet 只负责"按配置调用二进制、拿到 IP、写进 Pod status"，别的一概不管：

```json
{
  "cniVersion": "1.0.0",
  "name": "k8s-pod-network",
  "type": "calico",
  "ipam": { "type": "calico-ipam" }
}
```

**必答**：

- CNI v1.0+（v0.4 已弃用），新插件应支持 CHECK/GC（`garbage-collect`）；
- \*\*CNI-in-CNI（chained 插件）\*\*用于 Multus 多网卡场景（主线网卡 + 业务网卡）；
- kubelet 侧只看 CNI 返回结果的 IP，其它一概不管。

---

## A4. ClusterIP 到底是什么？为什么说它"没有实体网卡"？

**ClusterIP 是一个纯数据面规则意义上的 VIP**：

- 它没有对应的网络设备，`ip addr` 里**在本机看不到 ClusterIP**（有些 CNI/kube-proxy 实现会把它放到 dummy 接口 `kube-ipvs0` 上，便于外部发布 CLUSTER IP 到 BGP；这是实现细节）；
- 它的"存在"靠的是 **kube-proxy 下发的 iptables/IPVS/eBPF 规则**：把目的地址换成后端 Pod IP（DNAT）。

```
Pod: curl http://svc:80
  ① Pod DNS 解析 svc → ClusterIP
  ② 封包 dst=ClusterIP:80 → 走 Pod 默认路由出 netns
  ③ 宿主 netns 的 iptables/IPVS/eBPF 规则命中 → DNAT dst=PodIP:8080
  ④ 若跨节点 → 由 CNI 数据面送到对端节点
  ⑤ 到达后端 Pod
```

**三种 Service VIP 类型**：

| 类型 | 特点 | 注意 |
| --- | --- | --- |
| **ClusterIP** | 集群内 VIP | 最常用 |
| **NodePort** | 每个 Node 上开端口转发 | 端口范围默认 30000-32767 |
| **LoadBalancer** | 云厂商 LB → NodePort/直连 Pod | 云 LB 与云 CNI 的交互因厂商而异 |
| **Headless**（`clusterIP: None`） | **不做 VIP**，DNS 直接返回 Pod IP 列表 | 有状态服务的关键 |

**必答**：

- Headless Service 才有 `podname.svc.ns.svc.cluster.local` 的稳定 DNS 记录与 SRV 记录，是 StatefulSet 稳定网络身份的基础；
- `externalName` 是纯 CNAME，没有 selector 也没有转发规则。

---

## A5. kube-proxy 三种模式怎么演进的？

| 模式 | 机制 | 复杂度 | 现状 |
| --- | --- | --- | --- |
| **userspace**（最初） | kube-proxy 进程自己做代理转发 | 差 | 已弃用 |
| **iptables** | 内核 netfilter 规则做 DNAT | 规则数 O(规则数)，匹配是**顺序遍历** | 长期默认 |
| **IPVS** | 内核 LVS，哈希表 O(1) | 规则再多也是常数查找 | 大规模可选 |
| **nftables**（新 gewgaws） | nftables 后端 | 较新 | **版本相关（alpha/beta 阶段），需按 Release Notes 确认** |
| **Cilium socket-LB**（替代 kube-proxy） | eBPF 在 socket 层完成转换 | 最好 | 方案侧选择 |

**iptables 模式的核心缺点（必答）**：

1. **规则数量与 Service/Endpoint 规模线性相关**，且更新要 `iptables-restore` 全量下发；
2. 匹配是**链表顺序匹配**，规则越多单包延迟越高；
3. 负载均衡用 statistic random 模式，**无真正的调度算法**（IPVS 才有 rr/lc/sh/dh/sed/nq）；
4. 大量 Endpoints 更新会引发周期性 CPU 尖峰。

**切换 IPVS 的配置（可直接套用）**：

```yaml
apiVersion: kubeproxy.config.k8s.io/v1alpha1
kind: KubeProxyConfiguration
mode: "ipvs"
ipvs:
  scheduler: "rr"          # rr | lc | dh | sh | sed | nq
  strictARP: true          # 配合 MetalLB 等 ARP 型 LB 时需要
  syncPeriod: 30s
  minSyncPeriod: 5s
conntrack:
  maxPerCore: 131072       # 建议初值，需按节点连接数实测校准
  tcpEstablishedTimeout: 24h
  tcpCloseWaitTimeout: 1h
```

> **注意**：切到 IPVS **不等于不用 iptables**——IPVS 模式下 kube-proxy 仍会用 iptables/ipset 处理 MASQUERADE(SNAT)、包过滤与健康检查相关流程。

---

## A6. EndpointSlice 为什么取代了 Endpoints？

**背景**：一个 Service 下若有 5000 个 Endpoint，Endpoints 是一个巨大的单对象，任何 endpoint 变更都要**全对象更新**，引发：etcd 写放大 + 所有 watch 者收到大 payload。

**EndpointSlice 的改进**：

- 将 Endpoints **切片**，默认每个 slice 最多 100 个 endpoint（由 kube-controller-manager 的 `--max-endpoints-per-slice` 控制，默认值请核对）；
- 变更时只更新受影响的那一小块 ⇒ 显著减少 etcd 与网络开销；
- 支持 `TopologyAwareHints`** / **`trafficDistribution: PreferClose`（亲和到同 zone），降低跨可用区流量与延迟（该功能随版本演进，请以 Release Notes 为准）。

```promql
# EndpointSlice 数量与变更频率（判断有没有因 endpoint 变动导致的写放大）
kube_endpointslice_info
```

---

## A7. 集群 DNS 是怎么工作的？`ndots:5` 是什么坑？（超高频）

```
Pod /etc/resolv.conf:
  nameserver 10.96.0.10                        # ClusterIP of kube-dns Service
  search default.svc.cluster.local svc.cluster.local cluster.local
  options ndots:5                              # ← 关键点
```

**规则**：名字里 **点的个数 \< ndots（默认 5）** 时，会先依次按 search 域拼上再做 A/AAAA 查询，**没采中后再查原名**。

```
Pod 内部访问 www.baidu.com（1 个点 < 5）：
  1) www.baidu.com.default.svc.cluster.local   → NXDOMAIN
  2) www.baidu.com.svc.cluster.local           → NXDOMAIN
  3) www.baidu.com.cluster.local               → NXDOMAIN
  4) www.baidu.com                             → 命中
⇒ QPS 被放大 4~5 倍（IPv4+IPv6 双查询时更夸张，可达 8~10 倍，需实测）
```

**治理手段**：

| 手段 | 做法 |
| --- | --- |
| **写法优化** | 外部域名写 FQDN，末尾加 `.`（`www.baidu.com.`）⇒ 直接查原名 |
| **降 ndots** | Pod 侧 `dnsConfig.options: [{name: ndots, value: "2"}]`（**要验证不影响内部短域名**） |
| **NodeLocal DNSCache** | 节点本地缓存（DaemonSet），查询走本地 TCP/UDP loopback，绕开 conntrack 竞态、降低 CoreDNS QPS |
| CoreDNS 扩容 | `cluster-proportional-autoscaler` 按节点/CPU 数自动扩容 |

```yaml
# Pod 侧降 ndots（注意：改了可能导致依赖 search 域的短域名失效，务必压测）
spec:
  dnsConfig:
    options:
      - name: ndots
        value: "2"
```

**NodeLocal DNSCache 部署（关键片段）**：

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: node-local-dns
  namespace: kube-system
spec:
  template:
    spec:
      hostNetwork: true
      dnsPolicy: ClusterFirstWithHostNet     # 关键：hostNetwork 下仍需集群 DNS
      priorityClassName: system-node-critical
      tolerations:
        - key: CriticalAddonsOnly
          operator: Exists
      containers:
        - name: node-cache
          image: registry.k8s.io/dns/k8s-dns-node-cache:1.22.28   # 版本请按自身镜像仓库核对
          args:
            - "-localip"
            - "169.254.20.10"                # 每个节点上一个 link-local 地址
            - "-conf"
            - "/etc/coredns/Corefile"
            - "-upstreamsvc"
            - "kube-dns-upstream"            # 上游指向原 CoreDNS Service
          ports:
            - containerPort: 53
              name: dns
              protocol: UDP
```

```yaml
# CoreDNS 的 CM：给 nodelocaldns 提供 ClusterIP 做 upstream
kubectl -n kube-system get svc kube-dns-upstream -o yaml   # 无 selector 的 Service，手动维护 endpoint
```

> **部署要点**：切换 nodelocaldns 需要把 Pod 的 `nameserver` 指到 `169.254.20.10`——通常通过 kubelet 的 `--cluster-dns` 配置 + 节点替换灰度实现，**属于有损变更，必须灰度**。

---

## A8. 经典的"DNS 间歇性 5 秒超时"是怎么回事？（事故级必考题）

**现象**：Pod 内请求偶发卡 5 秒后报错，概率极低但持续存在，和业务/网络负载无关。

**根因**：**内核 conntrack + UDP NAT 的竞态**（社区长期已知问题）。同一时刻并发的 DNS 查询复用了同一个源端口，DNAT 时 conntrack 表项冲突 ⇒ 其中一个包被丢弃。UDP 无重传机制得当 ⟹ glibc resolver 等 5 秒超时（resolver 默认 `timeout:5 attempts:2` 相关）。

**证据**：

- CoreDNS 侧看到请求没到 / 重复查询；
- 节点 `nf_conntrack_insert_failed` / `conntrack -S` 的 `insert_failed` 计数增长；
- Pod 侧 `resolv.conf` 里有两个以上 search 域 + ndots=5。

**解法（按优先级）**：

1. **NodeLocal DNSCache**（最推荐）：本机链路-local 通信，绕开 conntrack NAT 竞态，且降低 CoreDNS QPS；
2. 内核侧：升级到包含相关修复的内核 / 开启 conntrack 相关优化（**具体 patch/参数因内核版本而异，务必在测试环境验证**）；
3. 应用侧使用 TCP / 或 `options single-request-reopen`（串行化，减少冲突概率）；
4. 降低查询次数（FQDN + 降 ndots）。

```promql
# CoreDNS 侧看超时与错误分布（指标含义请与目标版本核对）
histogram_quantile(0.99, sum by (le) (rate(coredns_dns_request_duration_seconds_bucket[5m])))
sum(rate(coredns_dns_requests_total{rcode="SERVFAIL"}[5m]))
# 缓存命中率（NodeLocal 上线后应显著上升）
sum(rate(coredns_cache_hits_total[5m])) / clamp_min(sum(rate(coredns_cache_hits_total[5m]) + rate(coredns_cache_misses_total[5m])), 0.001)
```

---

## A9. NetworkPolicy 是怎么生效的？为什么配了没效果？

**关键点**：NetworkPolicy 只是**声明式 API**，K8s 自身（kubelet/kube-proxy）**不实现它**，必须由 **CNI 数据面**落地：

| CNI | 实现载体 |
| --- | --- |
| Calico | iptables/ipsets 或 eBPF |
| Cilium | eBPF（支持 L7、DNS-aware） |
| Antrea | OVS |
| Flannel | **不支持**（需搭配其它方案，如 Canal/额外方案） |

**三个必答的语义**：

1. **默认全通**：没有任何 NetworkPolicy 选中某 Pod 时，它可被任意来源访问；
2. **一旦被任意 ingress 策略选中，未被显式允许的方向即拒绝**（egress 同理，互不影响）；
3. namespace/podSelector/ipBlock 三要素组合，标签驱动的。

**最佳实践（可直接套用）**：先 default-deny，再逐步放行。

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
  namespace: prod
spec:
  podSelector: {}                 # 选中本 ns 所有 Pod
  policyTypes: ["Ingress", "Egress"]
---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-app-to-db
  namespace: prod
spec:
  podSelector:
    matchLabels: {app: db}
  policyTypes: ["Ingress"]
  ingress:
    - from:
        - podSelector:
            matchLabels: {app: web}
      ports:
        - protocol: TCP
          port: 5432
```

**排坑清单**：

- 忘了包含 DNS 出口 ⇒ 加了 default-deny 后**所有域名解析失败**（务必放行 kube-system/CoreDNS 的 53/UDP、TCP）；
- 健康检查来源（云 LB / ingress / kubelet probe）被拦 ⇒ Probe 失败；
- NetworkPolicy **不影响 Service VIP 层面的负载均衡**，只在 Pod↔Pod 入口/出口处生效；
- 大规模下 iptables 实现的策略会有**规则数线性膨胀**问题（见 C8）。

---

## A10. Overlay vs 路由 vs 云网卡直连：三种数据面模式的本质区别？

| 模式 | 代表 | 封包开销 | 对底层网络的要求 | 适用 |
| --- | --- | --- | --- | --- |
| **Overlay（隧道）** | Flannel VXLAN、Calico VXLAN/IPIP、Cilium VXLAN/Geneve | 有封装头（VXLAN 约 50 字节 IPv4），MTU 变小 | 只要求节点 IP 互通（可跨子网/跨 VPC） | 通用、云上跨可用区、网络不可控场景 |
| **路由（纯三层）** | Calico BGP / host-gw / Cilium native routing | 无封装，直接路由转发 | **要求 Pod CIDR 在底层网络可路由**（同二层/路由可达） | IDC、可控网络、追求性能 |
| **CrossSubnet 混合** | Calico `IPIPMode: CrossSubnet` | 同子网直路由、跨子网才封装 | — | **IDC/云上最常见折中方案** |
| **云网卡（ENI 直连）** | AWS VPC CNI、aliyun Terway、Azure CNI（部分模式） | Pod 直接拿 VPC IP，无隧道 | 依赖云厂商 API 分配网卡/IP | 云托管集群、需要 Pod 具备 VPC IP 的场景 |

**必答**：选择的关键不是"哪个更快"，而是 **底层网络是否允许你把 Pod CIDR 通告出去**，以及 **是否需要 Pod IP 在集群外可见（安全组/审计/直连）**。

---

## A11. VXLAN / IPIP / Geneve 的封装代价怎么算？MTU 为什么老出问题？

| 封装 | IPv4 头部开销（估算） | 说明 |
| --- | --- | --- |
| VXLAN | \~50 字节 | 外层 IP(20) + UDP(8) + VXLAN(8) + 以太网(14) 量级 |
| IPIP | \~20 字节 | 只加一层 IP 头 |
| Geneve | 可变（通常 ≥50） | 支持可变长选项 |

**MTU 计算公式（建议值）**：`Pod MTU ≈ 宿主网卡 MTU - 封装开销`，1500 网卡上：

- VXLAN ⇒ 常被配置为 **1440/1450**（不同 CNI 默认值不同，**请以所用插件文档为准**）；
- IPIP ⇒ **1480**。

**MTU 不一致的典型故障**：

- 大包被丢弃但小包正常（ping 通但 curl 大文件卡死）；
- TLS 握手卡住、gRPC 偶发 reset；
- ICMP 被禁 ⇒ PMTUD 失效 ⇒ **黑洞连接**（只能靠 TCP MSS clamp 缓解）。

**治理**：

- 统一 MTU 配置（CNI 侧显式指定，不依赖自动探测结果漂移）；
- 必要时开启 **MSS clamping**（Calico/Cilium 都有相关配置）；
- 网络侧放通必要 ICMP type 3 code 4。

---

## A12. 跨节点 Pod 通信：Calico BGP 模式下封包怎么走？

```
Pod A (10.244.1.10) → Node1 路由表：10.244.2.0/24 via Node2-IP dev eth0
    → Node2 收到 → 路由表：10.244.2.10 dev calixxxx scope link → Pod B
这条路被 **BIRD/BGP** 在节点之间互相通告：每个 Node 把自己持有的 Pod CIDR 通告给其它 Node
```

**组件分工（必答）**：

| 组件 | 作用 |
| --- | --- |
| **Felix** | 数据面执行者：下发路由、iptables/ipsets/eBPF 规则、写 ACL |
| **BIRD**（或 GoBGP / FRR） | BGP 客户端，与其他节点/ToR 交换路由 |
| **confd** | 监听 datastore 变化，生成 BIRD 配置 |
| **Typha** | **大规模关键**：替所有 Felix 统一 watch datastore，减少 datastore 连接数与压力 |
| **calico-kube-controllers** | 同步 node/pod 状态、GC IPAM 泄漏等 |

**规模要点**：BGP **full-mesh 的连接数是 O(n²)**，节点数上百后必须上 **Route Reflector（RR）**（*建议 \>100 节点启用 RR，具体阈值以官方文档与实测为准*），或由 ToR 交换机做 RR（推荐生产做法，把 BGP 交给网络设备）。

---

## A13. Cilium 的 eBPF 数据面有什么不同？

**核心思路**：把转发、负载均衡、策略执行放进内核 eBPF 程序（挂在网络设备 TC/XDP、socket 层、cgroup），**尽量绕开 netfilter（iptables）这条链表**。

```
传统：nic → netfilter PREROUTING → iptables 链顺序匹配 → conntrack → 转发
Cilium：nic(TC ingress eBPF) → eBPF map 查表 → 直接转发 / 尾调用，策略在 map 里
```

**关键能力**：

| 能力 | 说明 |
| --- | --- |
| **kube-proxy 替代（socket LB）** | 在 `connect()` 阶段就把 ClusterIP 转成 PodIP，**不经过 NAT/iptables**；连接建立更快，且天然保真客户端 IP |
| **策略规模可扩展** | 基于 identity（label 派生）+ eBPF map，**避免每 endpoint 一套规则链** |
| **Hubble 可观测** | flow/policy/drop 层面的可观测，含 **drop reason** |
| **XDP / Bandwidth Manager** | XDP 加速；EDT + FQ 做带宽整形（替代 tc tbf） |
| **ClusterMesh** | 多集群网络与服务互联 |
| **加密** | WireGuard / IPsec 透明加密（有性能成本，需实测） |

**必答的限制/坑**：

- 对 **内核版本有要求**（不同功能要求不同版本，请以 Cilium 官方矩阵核对）；
- socket LB 与某些场景需要特别注意：**hostPort**、某些策略/NAT 交互、与 hostNetwork Pod 的行为差异等，升级前请逐个验证；
- eBPF maps 占用内存，大规模要关注 map 大小限制；
- 排障工具从 `iptables-save` 变成 `bpftool` / `cilium monitor` / Hubble —— **团队技能栈要换**。

## A14. 云厂商 CNI（ENI 类）和自建 CNI 的本质差异？

| 维度 | 云 CNI（如 VPC CNI / Terway） | 自建 CNI（Calico/Cilium 通用模式） |
| --- | --- | --- |
| Pod IP 来源 | **VPC 地址空间**（集群外也可见、可路由） | 集群私有 CIDR |
| 数据面 | 通常无需隧道（直路由 ENI） | 视模式而定 |
| 节点 Pod 数上限 | **受实例网卡规格限制**（ENI 数 × 每网卡 IP 数） | 主要受 `--max-pods` 和 CIDR 规划限制 |
| 安全能力 | 可用 VPC 安全组精细管控 Pod | 依赖 NetworkPolicy |
| 运维依赖 | 强依赖云 API（限流/配额风险） | 依赖集群自身组件 |
| 迁移性 | 差（绑定厂商） | 好 |

**必答的成本/风险点**：

- 云 API 限流会导致 **Pod 创建变慢甚至失败**（IP 分配失败 → `FailedCreatePodSandBox`）；
- **IP 规划必须和 VPC 子网协调**，Pod 数暴涨容易撞到子网耗尽；
- 故障排查要跨越"云网络 + K8s"两层边界。

---

## A15. AWS VPC CNI 的几种模式？（进阶高频）

| 模式 | 机制 | 优势 | 注意 |
| --- | --- | --- | --- |
| **Secondary IP**（默认） | 每个 ENI 分配多个辅助私网 IP，Pod 直接用 | 原生 VPC IP | 单节点 Pod 数 = ENI 数 × 每 ENI IP 数（受实例规格限制） |
| **Prefix Delegation** | 每个 ENI 分配 /28 前缀而非单个 IP | **大幅提升单节点 Pod 密度**，减少 ENI 与 API 调用 | 需要子网有连续的空闲地址块；版本/参数相关（`ENABLE_PREFIX_DELEGATION`） |
| **Custom Networking** | Pod 使用独立子网/安全组 | IP 隔离、缓解主子网耗尽 | 子网规划复杂 |
| **IPv6 模式** | Pod 拿 IPv6 地址 | 彻底缓解 IPv4 枯竭 | 要求配套组件支持双栈 |

**关键调优参数（环境变量，取值以官方版本为准）**：

```yaml
# aws-node DaemonSet 常用参数（示意，务必按官方文档核对默认值）
env:
  - name: WARM_ENI_TARGET        # 期望保持的空闲 ENI 数
    value: "1"
  - name: WARM_IP_TARGET         # 期望保持的空闲 IP 数（推荐用它替代 WARM_ENI_TARGET）
    value: "5"
  - name: MINIMUM_IP_TARGET      # 至少保留的 IP 数
    value: "10"
  - name: ENABLE_PREFIX_DELEGATION
    value: "true"
  - name: AWS_VPC_K8S_CNI_CUSTOM_NETWORK_CFG
    value: "false"
```

> **实践建议（需实测校准）**：`WARM_IP_TARGET` 太小 ⇒ 突发扩容时等 IP 导致 Pod 起得慢；太大 ⇒ 长期占用 VPC 地址造成浪费。应结合"Pod 创建速率"与"子网剩余 IP"一起调，并配专属告警。

---

## A16. 常见 CNI 方案横向对比（选型总表，必须能背）

| 方案 | 数据面 | 主要模式 | NetworkPolicy | L7 策略 | 加密 | 观测/IPAM | 适用 |
| --- | --- | --- | --- | --- | --- | --- | --- |
| **Flannel** | 简单 | VXLAN / host-gw | ❌ 不支持 | ❌ | ❌ | 弱 / host-local | 小集群、学习、无策略诉求 |
| **Calico** | iptables/ipsets 或 eBPF | BGP / VXLAN / IPIP / CrossSubnet / NoEncap | ✅ | 部分（需企业版/组合方案） | ✅ WireGuard | Felix 指标+Typha 规模扩展 | **IDC/通用主力方案** |
| **Cilium** | eBPF | native routing / VXLAN / Geneve / ENI | ✅✅ | ✅ (Envoy/L7, DNS-aware) | WireGuard / IPsec | **Hubble**（最强） | 大规模、高性能、可观測需求 |
| **Antrea** | OVS/OVS-based | Geneve / noEncap / hybrid | ✅ | 部分 | IPsec | Octant/Flow 相关能力 | VMware 生态、复杂 L2 场景 |
| **OVN-Kubernetes** | OVN/OVS | Geneve / localnet | ✅ | — | — | 较弱 | Openshift / OpenStack 生态 |
| **Multus + SR-IOV/DPDK** | 硬件（VF） | 多网卡直通 | 依赖主 CNI | — | — | — | NFV / 电信 / 超高性能诉求 |
| **云厂商 CNI** | 云网络直通 | ENI | 视实现 | — | — | — | 托管集群、需 VPC IP |

**选型决策树（可直接背）**：

```
1. 是否必须使用 VPC IP（合规/安全组/外部直连）？
     是 → 云厂商 CNI（并评估 Pod 密度是否满足）
     否 ↓
2. 底层网络能通告 Pod CIDR（IDC/同二层/可控路由）？
     能 → Calico BGP / Cilium native routing（性能最优）
     不能 ↓
3. 同可用区/同二层居多、少量跨子网？
     是 → CrossSubnet（IPIP/VXLAN 混合）
     否 → 全 Overlay（VXLAN/Geneve）
4. 是否需要 L7/DNS策略、强可观测、是否规模 ≥ 1000 节点或 Service 很多？
     是 → Cilium（eBPF）
5. 纯 NFV/电信 或需要第二块网卡 → Multus + SR-IOV/DPDK/macvlan（接受运维复杂度）
```

---

## A17. Calico 的 IPPool 各项参数怎么配？

```yaml
apiVersion: projectcalico.org/v3
kind: IPPool
metadata:
  name: default-ipv4-ippool
spec:
  cidr: 10.244.0.0/16
  blockSize: 26                   # 每个 block 大小，默认 26（64 个地址/块）
  ipipMode: CrossSubnet           # Always | CrossSubnet | Never
  vxlanMode: Never                # Always | CrossSubnet | Never
  natOutgoing: true               # Pod 出集群时对 Pod IP 做 SNAT
  nodeSelector: all()
  allowedUse: ["Workload", "Tunnel"]
```

**必答**：

| 参数 | 说明 | 坑 |
| --- | --- | --- |
| `blockSize` | IPAM 分配粒度，**创建后不可改**（必须新建 IPPool 迁移，属重操作） | **上线前必须规划** |
| `ipipMode: CrossSubnet` | 同二层直路由、跨子网才封装 | 云上最常见配置 |
| `NATOutgoing` | 出公网 SNAT | 关闭后 Pod IP 需在外部可路由 |
| `allowedUse` | 区分 workload 地址与隧道地址（较新能力，版本相关） | — |

**IPAM 泄漏排查**：节点删除后仍占用 block ⇒ 用 `calicoctl ipam check` 识别泄漏并 `calicoctl ipam release --force`（**破坏性操作，须走工单**）。

---

## A18. conntrack 是什么？为什么会成为瓶颈？

**conntrack（连接跟踪）**：netfilter 维护每条连接的元组表，支撑 NAT 与状态防火墙。

**瓶颈表现**：

- 日志 `nf_conntrack: table full, dropping packet`；
- 新连接建立失败/延迟上升，尤其短连接高并发场景。

**关键参数**：

```bash
# 查看容量与当前使用
sysctl net.netfilter.nf_conntrack_max
cat /proc/sys/net/netfilter/nf_conntrack_count

# 调整（建议值需实测校准；写入 /etc/sysctl.d/ 持久化）
net.netfilter.nf_conntrack_max = 1048576
net.netfilter.nf_conntrack_buckets = 262144      # hashsize，通常为 max/4
net.netfilter.nf_conntrack_tcp_timeout_established = 86400
net.netfilter.nf_conntrack_tcp_timeout_time_wait = 30
```

**关系公式**：`nf_conntrack_max = nf_conntrack_buckets * 4`（hashsize 是底层哈希桶数）。

**治理**：

- 按"节点预计并发连接数 × 安全系数"规划 `nf_conntrack_max`（**必须实测**）；
- 缩短 TIME\_WAIT 等 timeout；
- **IPVS 模式仍依赖 conntrack**（NAT 模式），而 **Cilium socket-LB 路径不依赖 conntrack**，这是后者在高并发短连接场景的显著优势之一。

```promql
# 依赖 node_exporter，指标名按 exporter 版本核对
node_nf_conntrack_entries / node_nf_conntrack_entries_limit > 0.8
increase(node_nf_conntrack_insert_failed[5m]) > 0
```

---

## A19. `externalTrafficPolicy` 的两个取值有什么区别？（高频）

| 取值 | 行为 | 优点 | 缺点 |
| --- | --- | --- | --- |
| **Cluster**（默认） | NodePort/LB 流量可在**集群内再次转发**到任意节点的 Pod | 负载均衡均匀 | **丢失客户端真实 IP**（做了额外一跳 SNAT），多一跳延迟 |
| **Local** | 只转发给**本节点上存在的** Pod | **保留源 IP**、少一跳 | 若本节点无该服务 Pod ⇒ 该节点健康检查失败；可能**负载不均** |

```yaml
apiVersion: v1
kind: Service
metadata:
  name: web
spec:
  type: LoadBalancer
  externalTrafficPolicy: Local     # 保留源 IP
  # healthCheckNodePort 会由系统自动分配，或显式指定
```

**必答的组合要点**：

- `Local` 模式下要确保 LB 的**健康检查能感知"本节点有无可用 Pod"**（K8s 会通过 healthz/healthCheckNodePort 返回），否则流量打到无后端节点会失败；
- `Local` + 单副本服务 ⇒ LB 只会把流量导到唯一节点，**请结合多副本 + 拓扑打散**使用；
- 想要源 IP 又想要均衡，可用代理协议（PROXY protocol）或上层网关注入 header。

---

## A20. Pod 访问自身的 Service（hairpin）为什么有时不通？

**hairpin/NAT loopback**：Pod 通过 Service VIP 访问"自己作为后端"的服务时，封包从 Pod 出去、又被转发回同一个 Pod，需要内核/网桥支持 hairpin（`hairpin_mode`）。

- 排查：kubelet/hostPort/`docker-compose` 类的 bridge hairpin；也可在 Service 侧要求 CNI/kube-proxy 正确处理（不同 CNI 行为不同）；
- 经典解法：应用层避免"自己访问自己的 Service"，改用 localhost 或 Pod IP 直连；
- 也可检查 CNI 是否开启 `hairpinMode` / `promiscuous` 相关配置。

---

## A21. hostNetwork / hostPort 与 DNS 策略？

```yaml
spec:
  hostNetwork: true
  dnsPolicy: ClusterFirstWithHostNet   # 关键：hostNetwork 下默认 dnsPolicy 会退化为 Default
```

| `dnsPolicy` | 含义 |
| --- | --- |
| `ClusterFirst`（默认） | 先问集群 DNS（CoreDNS），不中的再转发上游 |
| `ClusterFirstWithHostNet` | **hostNetwork Pod 想用集群 DNS 必须显式配** |
| `Default` | 用节点 `/etc/resolv.conf` |
| `None` | 完全自定义，配合 `dnsConfig` |

**hostPort 的坑**：

- 端口占用是**节点级**的 ⇒ 同节点多副本必然冲突（调度器看不到这种冲突，会照常调度，然后 Pod 起来失败）；
- 优先用 NodePort / HostPort 替代方案，或用 DaemonSet 固定单实例。

---

## A22. IPv4/IPv6 双栈要注意什么？

- 双栈要求集群/Pod/Service CIDR 均成对配置（`--service-cluster-ip-range`、`Pod CIDR` 双段）；
- Service 的 `ipFamilyPolicy`（`SingleStack`/`PreferDualStack`/`RequiredDualStack`）与 `ipFamilies` 控制行为；
- **注意**：修改已有 Service 的 ipFamilies 有严格约束（通常不可从单栈直接改双栈，需重建）；
- 应用、监控、外部 LB 都要支持 IPv6，端到端才成立。

---

## A23. Service、Ingress、Gateway API 到底什么关系？

| 层次 | 对象 | 职责 | 谁实现 |
| --- | --- | --- | --- |
| **L3/L4（东西向）** | Service（ClusterIP/NodePort/LB） | Pod 集合的稳定入口与负载均衡 | kube-proxy / CNI |
| **L7 南北向** | Ingress | HTTP/HTTPS 路由、TLS 终止 | Ingress Controller（NGINX/APISIX/Traefik…） |
| **新一代南北向/东西向 API** | **Gateway API**（GatewayClass/Gateway/HTTPRoute…） | 面向角色的 API（基础设施/集群运维/应用开发者），支持更丰富的路由与流量能力 | 各类 Gateway 实现 |

**为什么有 Gateway API**：Ingress 表达能力弱（靠 annotation 扩展导致实现之间不可移植），Gateway API 通过**角色划分 + 细粒度路由 + 明确的一致性测试**来解决。它已是社区推荐的演进方向（**具体 API 版本与 GA 范围请以 Release Notes 为准**）。

---

## A24. Service Mesh 和 CNI 是竞争关系吗？

**不是**，职责分层不同：

| 层 | 提供什么 | 代表 |
| --- | --- | --- |
| **CNI（L3/L4）** | IP 分配、Pod 互通、网络策略执行 | Calico / Cilium |
| **Mesh（L7）** | 服务治理：重试/超时/熔断/mTLS/灰度/可观测 | Istio / Linkerd |
| **CNI 的上层扩展** | Cilium 提供部分 L7 策略、可替代部分 mesh 功能（如 mTLS/L7 策略） | Cilium Service Mesh |
| **Sidecar vs Ambient** | Sidecar：每 Pod 注入代理；Ambient：节点级 ztunnel + 按需 waypoint | Istio ambient |

**必答**：Mesh 是"应用层治理"，CNI 是"网络层连通与策略"。两者可以同时存在（Cilium 提供 CNI + mesh 部分能力），但要注意**双重策略叠加导致的排障复杂度**。

---

## A25. 网络加密（WireGuard / IPsec / mTLS）怎么选？

| 方案 | 层 | 成本 | 适用 |
| --- | --- | --- | --- |
| **WireGuard**（CNI 提供） | L3 Pod 流量 | 相对较低、现代算法，**有性能成本需实测** | 节点间 Pod 流量加密 |
| **IPsec 透明加密** | L3 | 相对更高 | 合规强要求 |
| **mTLS（Mesh）** | L7 | CPU 成本主要在 sidecar | 服务身份 + 应用层加密 |
| **TLS Ingress/gateway** | L7 边界 | 边缘处 | 南北向加密 |

**必答**：加密会引入 CPU 与 MTU/分片开销（隧道类尤其），且会让**抓包排障变难**，要先按业务合规要求确定范围（全量 vs 特定 Namespace），并做压测。

---

## A26. 带宽限速怎么做？

两条路线：

```yaml
# ① CNI bandwidth 插件：靠 Pod annotation + tc tbf
apiVersion: v1
kind: Pod
metadata:
  annotations:
    kubernetes.io/ingress-bandwidth: 10M
    kubernetes.io/egress-bandwidth: 20M
spec:
  containers:
    - name: app
      image: app:1.0
```

```bash
# ② Cilium Bandwidth Manager：EDT + FQ（更现代的实现，需对应版本支持）
# helm values 示意
bandwidthManager:
  enabled: true
  bbr: true        # 开启后影响拥塞控制策略，务必压测
```

**必答**：tc tbf 的实现在高 PPS 场景自身有开销；限速策略通常只用于防"吵闹邻居"，不要当核心流量治理手段。

# Part B · 方案 / 模式专项篇（18 题）

> 这一篇按"具体方案和它的各种模式"出牌，是大厂面试网络方向最爱追问的部分。

## B1. Flannel 的三种后端有什么区别？

| 后端 | 机制 | 要求 | 性能 | 备注 |
| --- | --- | --- | --- | --- |
| **VXLAN**（默认） | 封装 VXLAN，MAC-in-UDP | 只要节点 IP 互通 | 有封装开销 | 通用 |
| **host-gw** | 在节点上写静态路由，宿主充当网关 | **要求所有节点在同一个二层网络** | 接近裸网络（无封装） | 二层场景最优，注意 ARP 规模 |
| **UDP**（老） | 用户态转发 | — | 差 | 基本不再使用 |

**必答结论**：Flannel **不支持 NetworkPolicy**，所以生产上要么换成 Calico/Cilium，要么叠加策略方案。Flannel 只适合"先跑起来"的场景。

## B2. Flannel host-gw 为什么不能用跨子网？会出什么问题？

host-gw 依赖"**下一跳就是对端节点 IP，且二层可达**"（靠 ARP 解析 MAC）。跨三层子网后网关不做这个路由 ⇒ ARP 无法解析 ⇒ 封包出不去 ⇒ Pod 之间不通（**且常常是部分通、部分不通**，很容易误判）。

## B3. Calico：`ipipMode` 与 `vxlanMode` 怎么选？

| 场景 | 建议模式 |
| --- | --- |
| IDC 全网可被通告 Pod CIDR | `Never`（纯 BGP 路由），性能最好 |
| IDC 同二层 + 少量跨三层 | `IPIPMode: CrossSubnet` |
| 云上 VPC（通常不允许通告 Pod 路由/不支持 BGP） | `VXLANMode: Always` 或 `CrossSubnet` |
| 需要 Pod IP 在外部可见 | `Never` + 通告路由给 ToR + `natOutgoing: false` |

> **注意**：通常二者互斥（不应同时 Always），具体互斥规则以 Calico 版本文档为准。

## B4. BGP 规模问题：什么时候必须上 Route Reflector？

**问题**：full mesh 的 BGP 会话数是 `n(n-1)/2`，500 节点 ⇒ 约 12.5 万条会话，节点 CPU 与收敛都吃不消。

**解法**：

1. **Route Reflector（RR）**：节点只与少数 RR 建会话 ⇒ 会话数降到 O(n)；
2. **ToR 做 RR**（生产推荐）：把 BGP 交给网络设备，节点只与本机 ToR 建会话；
3. 启用 **BFD / graceful restart** 加快收敛、避免短暂中断（参数按网络团队规范）。

> **建议（需实测校准）**：节点数超过约 100 就该引入 RR；具体阈值按下例回填。

```bash
# Calico 侧大致流程（示意，请以所用版本文档为准）
# 1) 停用默认 node-to-node mesh
calicoctl patch bgpconfiguration default -p '{"spec":{"nodeToNodeMeshEnabled":false}}'
# 2) 部署 CR 或用 ToR 承担 RR 角色，配置 node 的 peerSelector
```

## B5. Calico Typha 是干什么用的？不装会怎样？

**作用**：所有节点的 Felix 都要 watch 集群状态（datastore）。节点数很多时，**每个 Felix 一条 watch ⇒ 对 kube-apiserver/etcd 产生 O(n) 倍读压力**。

**Typha 就是这层聚合/代理**：Felix 连 Typha，Typha 统一 watch 再广播 ⇒ 大幅降低 datastore 连接数与 watch 量。

**告警与实践**：

- Typha 挂了/数量不足 ⇒ Felix 连不上、策略更新延迟（表现为策略"不生效/延迟生效"）；
- Typha 应跑多副本（一般 3 个以上），并跨故障域分布，**是必须监控的组件**。

## B6. Calico eBPF 模式和 iptables 模式怎么选？

| 维度 | iptables/ipsets | eBPF |
| --- | --- | --- |
| 规则随规模增长 | 线性、单包延迟随规则数上升 | map 查表，规模扩展性更好 |
| 源 IP 保留（kube-proxy 场景） | 会做 SNAT | 可原生保留 |
| 内核要求 | 低 | **有版本要求**（按 Cilium/Calico 官方矩阵核对） |
| 排障工具链 | iptables/ipset | bpftool / calico-bpf / 日志 |
| 成熟度/心智负担 | 低 | 中高 |

**建议**：要"跟着社区未来走 + 性能敏感 + 规模大"就上 eBPF；要"稳定第一、团队 skill 有限"先用 iptables 模式，并在**独立的灰度集群/节点池**验证 eBPF。

## B7. Cilium：native routing vs tunnel vs ENI 模式？

| 模式 | 机制 | 适用 |
| --- | --- | --- |
| **native routing** | 无封装，依赖 Pod CIDR 可路由（或 ToR 路由） | IDC/可控网络，性能最优 |
| **tunnel**（vxlan/geneve） | 封装 | 底层不可控/跨子网 |
| **ENI / cloud provider IPAM** | Pod 用云网卡 IP（cluster-pool vs ENI IPAM 差别） | 云托管、需 VPC IP |
| **ClusterMesh** | 多集群 Pod IP 互通 + 全局服务 | 多集群/容灾 |

```yaml
# Helm values 示意 —— 具体取值严格按所用版本 Chart 文档核对
tunnel: vxlan
kubeProxyReplacement: strict     # 替代 kube-proxy（or probe/partial/false）
ipv4:
  enabled: true
ipam:
  mode: cluster-pool             # or eni / kubernetes
autoDirectNodeRoutes: true
bpf:
  masquerade: true
  hostLegacyRouting: false
bandwidthManager:
  enabled: true
hubble:
  enabled: true
  metrics:
    enabled:
      - dns:query
      - flow:source=namespace
```

## B8. Cilium 替换 kube-proxy 到底值不值？（进阶高频）

**收益**：

- ClusterIP 转换发生在 socket 层，**不经过 iptables/nat/nf\_****conntrack** ⇒ 规则数不再随规模线性增长，短连接高并发场景收益明显；
- **天然保留客户端 IP**（不需要 `externalTrafficPolicy: Local` 的副作用）；
- 支持 DSR（Direct Server Return）、Maglev 一致性哈希等能力（特性可用性以版本为准）。

**风险/限制**：

- 依赖较新内核（请查矩阵）；
- 与 hostPort、部分 NAT/策略组合存在已知限制，**必须在灰度环境逐项验证**；
- 排障方式改变（要会 `cilium monitor` / Hubble / bpftool）；
- 一些依赖 kube-proxy iptables 规则的遗留组件可能失效。

**给出决策口径**：

> "值不值取决于 Service 数量与短连接 QPS。Service 上百/上千、短连接密集、已经出现 iptables 规则更新导致的 CPU 尖峰时，替换收益很明显；否则就把它当成下一个演进阶段，先在独立节点池验证。"

## B9. Cilium Hubble 能解决哪些传统手段解决不了的问题？

| 能力 | 传统方式 | Hubble |
| --- | --- | --- |
| Pod 间流量拓扑 | 靠 tcpdump + 人肉 | 直接出 flow 拓扑 |
| **丢包原因（drop reason）** | 几乎无法定位 | 输出 policy/l4/ct 等原因 |
| L7（HTTP/DNS/Kafka）可见性 | 需 sidecar/抓包 | 部分场景原生可见 |
| 策略命中/拒绝 | 只能靠试 | 可见 denied/forwarded 判断 |

排障示例：

```bash
hubble observe --pod app --verdict DROPPED -f
cilium monitor --type drop
# 或直接看 drop reason 分布指标（指标名按版本核对）
```

## B10. ClusterMesh（多集群）要注意什么？

- 要求各集群 **Pod CIDR 不冲突**（这是最容易翻车的点，必须提前规划）；
- 控制面通过共享 KV/ClusterMesh API server 互通，**这是强依赖组件**，要做 HA；
- 全局服务（global service）跨集群发现带来的**跨集群故障域**要评估；
- 跨集群流量会走隧道 ⇒ 关注 MTU 与延迟；
- SRE 视角：**先明确 SLO（是否需要跨集群容灾）再上，别为炫技上**。

## B11. Antrea（OVS 系）适合什么场景？

- 基于 OVS/OVS 流水线做数据面，适合**复杂 L2/L3 场景**（如 VM 与 Pod 混部、OpenStack/VMware 生态）；
- 支持 NetworkPolicy 与 Antrea 自定义策略、IPsec 加密、flow 导出等；
- **成本**：OVS 的用户态组件 + datapath 复杂度，资源占用与排障门槛相对高。

## B12. Multus + SR-IOV/DPDK 什么时候才该用？

**适用场景**：NFV/电信、超高 PPS/低延迟要求、需要 Pod 多张网卡（业务网卡与管理网卡分离）。

```
Multus（meta 插件，负责调用多个 CNI）
   ├── 主 CNI（如 Calico）→ 提供集群网络 eth0
   └── 附加网卡（SR-IOV VF / macvlan / ipvlan）→ net1
          ↑ 资源由 device plugin 上报：intel.com/intel_sriov_netdevice
```

```yaml
# NetworkAttachmentDefinition（示意）
apiVersion: k8s.cni.cncf.io/v1
kind: NetworkAttachmentDefinition
metadata:
  name: sriov-net
spec:
  config: |
    { "cniVersion":"1.0.0","type":"sriov","ipam":{"type":"host-local","subnet":"192.168.100.0/24"} }
---
apiVersion: v1
kind: Pod
metadata:
  annotations:
    k8s.v1.cni.cncf.io/networks: sriov-net
spec:
  containers:
    - name: app
      resources:
        limits:
          intel.com/intel_sriov_netdevice: "1"
```

**代价（必答）**：失去"Pod 可迁移/可用度高"的部分优势——SR-IOV VF 与**具体节点网卡绑定**，迁移后 IP/MAC 可能变化，且依赖节点池硬件一致性。

## B13. macvlan / ipvlan 的特点与坑？

| 模式 | 特点 | 坑 |
| --- | --- | --- |
| **macvlan** | 每个子接口有独立 MAC，性能好 | **Pod 无法直接访问宿主 IP**（大多数实现下不能用它访问宿主）；交换机需允许大量 MAC |
| **ipvlan L2** | 共享宿主 MAC | 同二层 |
| **ipvlan L3** | 三层转发 | 广播/ARP 受限，需路由配合 |

> 二者都不适合作为默认集群网络，常见用途是"给特定 Pod 一个二层/三层直通网卡"。

## B14. 怎么验证网络策略真的生效了？（面试常见实操题）

```bash
# 1) 确认 CNI 支持并开启了策略 enforcement（部分 CNI 有 "policy mode"）
# 2) 起一个测试 Pod，从被拒绝侧 curl
kubectl run curler --rm -it --image=curlimages/curl -- sh
curl -m 3 http://target.svc:80     # 期望 timeout / connection refused
# 3) CNI 侧证据
cilium policy get                  # Cilium
iptables-save | grep cali          # Calico（iptables 模式）
hubble observe --verdict DROPPED   # Cilium 观测
```

**必须验证的两个反向场景**：① 允许列表是否漏了 DNS（kube-system 53/UDP+TCP）与健康探测来源；② 加策略前后业务是否有误伤（配 metrics 告警同开）。

## B15. Ingress Controller 选型要考虑什么？

| 维度 | 关注点 |
| --- | --- |
| 性能与并发模型 | NGINX 类 worker 模型、reload 开销（大规模路由的热更新抖动） |
| 动态配置能力 | APISIX/Traefik 等支持热更新/插件生态 |
| 一致性/Gateway API 支持 | 未来演进方向 |
| 观测 | 是否方便导出 per-route 延迟/错误率 |
| 灰度能力 | 是否支持基于 header/权重的金丝雀 |
| 高可用 | 多副本 + 拓扑打散 + PDB |

**生产要点**：Ingress 是南北向单点故障源，必须：多副本跨可用区 + PDB + 优雅终止 + 单独的 LB + 容量压测 + 变更灰度。

## B16. Kubernetes Service CIDR 和 Pod CIDR 怎么规划？

```
原则：
  Pod CIDR：    按"单节点最大 Pod 数 × 最大节点数"估算，并预留倍数（建议 ≥2 倍）
  Service CIDR：按最大 Service 数估算，同样预留；注意它**不可在线扩容**（多数实现下扩容属于重建级操作）
  Cloud CNI：   Pod CIDR 需落在 VPC 子网内 ⇒ 必须与云网络团队协同预留
```

**禁止的事**：

- Service CIDR 与 Pod CIDR 重叠；
- 集群 CIDR 与办公网/IDC 现有网段冲突（会导致路由黑洞）；
- 规划过小又无法在线调整。

## B17. 怎么排查"Pod 起来了但服务访问不通"？（综合排障套路）

```
① DNS：nslookup svc / 看 /etc/resolv.conf 是否指向预期 DNS
② Service：kubectl get endpointslices -l kubernetes.io/service-name=svc   （有无后端）
③ kube-proxy/CNI 本地规则：iptables-save | grep <clusterIP> 或 ipvsadm -Ln / cilium service list
④ Pod 直连：curl <PodIP>:<port>              （排除 Service 层面问题）
⑤ 跨节点：从另一节点 curl PodIP              （排除 CNI 跨节点问题）
⑥ 策略：即使没配 NetworkPolicy，也可能有集群默认策略/主机防火墙
⑦ CNI 组件健康：cilium status / calico-node 日志 / felix Ready
⑧ 抓包：tcpdump 在 veth + 宿主网卡同时抓，对比丢在哪一跳
```

## B18. 哪些网络问题是"看似随机"的偶发故障？（经验题）

| 现象 | 可能根因 |
| --- | --- |
| DNS 偶发 5 秒超时 | conntrack 竞态（A8） |
| TLS 握手偶发失败、大文件传输卡死 | MTU 不一致 / PMTUD 黑洞（A11） |
| 短连接高峰期新连接失败 | conntrack 满 / TIME\_WAIT / 源端口耗尽 |
| 访问自己 Service 偶发失败 | hairpin 未开启（A20） |
| 跨可用区偶发延迟高 | LB 跨区转发到无后端节点（`externalTrafficPolicy`）+ 重试 |
| 滚动更新期间报错 | Endpoint 传播延迟 + 优雅终止没配（kubelet 篇 Q11） |
| 特定节点上的 Pod 全不通 | CNI DaemonSet 在该节点异常 / IPAM 泄漏 / 路由缺失 |

# Part C · 大规模集群网络优化篇（14 题）

> 定义：**大规模 = ≥ 1000 节点 / Service 数上千 / 单集群 Pod 数万以上 / 峰值 QPS 高**。

## C1. 先量化：网络侧的瓶颈一般落在哪 5 个位置？

| # | 瓶颈 | 量化指标 | 解法方向 |
| --- | --- | --- | --- |
| 1 | **iptables/IPVS 规则规模** | `kubeproxy_sync_proxy_rules_duration_seconds`（同步耗时）、节点 \`iptables-save | wc -l\` |
| 2 | **DNS 查询放大与延迟** | `coredns_dns_requests_total`、`coredns_dns_request_duration_seconds` | NodeLocal DNSCache、降 ndots、CoreDNS 自动扩容 |
| 3 | **conntrack 容量** | `node_nf_conntrack_entries / limit` | 调表大小、缩短 timeout、socket-LB 路径 |
| 4 | **IPAM / IP 地址** | 节点 Pod CIDR 使用率、CNI IPAM 分配错误 | CIDR 规划、前缀委派、回收泄漏 |
| 5 | **跨可用区/跨区流量** | 流量计费与延迟 | 拓扑感知路由（TopologyAwareHints / `trafficDistribution: PreferClose`） |

**标准答法**：

> "我会先用三组指标判断瓶颈位置：① kube-proxy 规则同步耗时与时延分布；② CoreDNS QPS/延迟/缓存命中率；③ 节点 conntrack 使用率。这三个基本能决定我是换数据面、加 DNS 缓存，还是调内核参数。"

## C2. iptables 规则膨胀怎么治理？

**症状**：

- `iptables-save | wc -l` 达到数万甚至十万级；
- Service/Endpoint 每次变更都触发全量 `iptables-restore`，CPU 出现尖峰；
- 网络编程延迟（Service 变更到真正生效）显著变长。

**证据与量化**：

```promql
# kube-proxy 规则同步耗时（最关键的领先指标）
histogram_quantile(0.99, sum by (le) (rate(kubeproxy_sync_proxy_rules_duration_seconds_bucket[5m])))

# 网络编程延迟：从 EndpointSlice 变更到规则生效的时延（指标存在与否依版本而定）
histogram_quantile(0.99, sum by (le) (rate(kubeproxy_network_programming_duration_seconds_bucket[5m])))
```

```bash
# 节点侧规则量（用于基线对比）
iptables-save | wc -l
ipvsadm -Ln | wc -l
```

**解法（按力度递增）**：

1. **切 IPVS**：规则查找 O(1)，且 ipset 聚合后端集合；
2. **上 Cilium kube-proxy replacement**：彻底绕开 netfilter；
3. 关注**新的 nftables 后端**（版本相关，属演进选项，需先在小集群验证）；
4. 架构侧：减少不必要的 Service（如批处理内部直连 PodIP/Headless/SRV），控制 Service 数量增长。

## C3. DNS 大规模优化的完整方案？

```
① NodeLocal DNSCache DaemonSet（hostNetwork + link-local IP）
      ⇒ 大幅降低 CoreDNS QPS，并规避 conntrack 竞态导致的 5 秒超时
② CoreDNS 自动扩容：cluster-proportional-autoscaler
③ CoreDNS 自身调优：缓存、forward 策略、健康检查、autopath（谨慎使用）
④ Pod 侧：FQDN 写法 + 合理 ndots
⑤ 监控：QPS、延迟 P99、SERVFAIL 率、缓存命中率
```

```yaml
# CoreDNS 自动扩容（示意 ConfigMap/linear 模式）
apiVersion: apps/v1
kind: Deployment
metadata:
  name: coredns-autoscaler
  namespace: kube-system
spec:
  template:
    spec:
      containers:
        - image: registry.k8s.io/cpa/cluster-proportional-autoscaler:1.8.9   # 版本自行核对
          name: autoscaler
          command:
            - /cluster-proportional-autoscaler
            - --namespace=kube-system
            - --configmap=coredns-autoscaler
            - --target=deployment/coredns
            - --logtostderr=true
            - --nodelabels=node-role.kubernetes.io/control-plane
            - --poll-period-seconds=15
```

```promql
# CoreDNS QPS 与延迟（判断是否需要扩容 / 是否已到瓶颈）
sum(rate(coredns_dns_requests_total[5m]))
histogram_quantile(0.99, sum by (le) (rate(coredns_dns_request_duration_seconds_bucket[5m])))
```

> **注意**：CoreDNS 副本数增加会提高 kube-dns Service 后端的均衡性，但过度扩容会放大 EndpointSlice 变更频率，**建议按节点数比例线性扩容（coresPerReplica/nodesPerReplica 需实测校准）**。

## C4. NodeLocal DNSCache 上线怎么灰度？（有损变更，必须给方案）

| 阶段 | 范围 | 观察指标 | 回滚触发条件 |
| --- | --- | --- | --- |
| Step 1 | **1 个节点**（隔离/OFF 搅动池） | Pod DNS 成功率、CoreDNS QPS、业务 P99 | DNS SERVFAIL 率上升或成功率跌破基线 |
| Step 2 | 1% 节点 | + Pod 重启后解析正确性 | 同上 / 出现解析错误或解析到错误地址 |
| Step 3 | 10% → 50% | 全 SLO Dashboard | 同上 |
| Step 4 | 100% | 稳定 24h | — |

**回滚要点**：NodeLocal 的启用通常改的是 kubelet 的 DNS 配置（`-cluster-dns`）⇒ **回滚需要重启 kubelet 并让现有 Pod 重建**才能完全恢复；务必：

1. 保证 `kube-dns-upstream` Service 与 endpoint 配置正确（否则 Cache 找不到上游 ⇒ 全局解析失败）；
2. 灰度期间保留原 CoreDNS Service 不被替换；
3. 明确回滚步骤（改回 kubelet 配置 → 重启 → 滚动替换 Pod）。

> **这是典型的"看似局部实则全节点"的变更**，面试里能讲清它的回滚代价很加分。

## C5. conntrack 满的预防与扩容？

```bash
# sysctl 持久化（建议值需按业务并发实测校准）
cat > /etc/sysctl.d/90-conntrack.conf <<'EOF'
net.netfilter.nf_conntrack_max = 1048576
net.netfilter.nf_conntrack_buckets = 262144
net.netfilter.nf_conntrack_tcp_timeout_time_wait = 30
net.netfilter.nf_conntrack_tcp_timeout_close_wait = 60
EOF
sysctl -p /etc/sysctl.d/90-conntrack.conf
```

**注意**：`nf_conntrack_buckets` 是**哈希桶数**，变更会重建表（大数据量下有瞬时开销），且部分环境只能在模块加载时设置。**必须在变更窗口执行并观察**。

**监控（必配）**：

```promql
node_nf_conntrack_entries / node_nf_conntrack_entries_limit > 0.75
increase(node_nf_conntrack_insert_failed[5m]) > 0
rate(node_nf_conntrack_drops[5m]) > 0
```

## C6. IP 地址耗尽怎么提前发现和避免？

**容量模型（务必自己算一遍）**：

```
集群 Pod 总 IP 需求 = 预期最大节点数 × 单节点最大 Pod 数(maxPods)
（还要考虑滚动更新期间新旧版本并存 ≈ ×2）
```

**规划建议（建议初值，需按业务校准）**：

| 场景 | 建议 |
| --- | --- |
| 通用集群 | Pod CIDR 预留 ≥ 2× 实际需求；单节点 maxPods 留 20% 余量 |
| 云 CNI（每节点 Pod 数受网卡限制） | 确认实例规格 → 计算 `ENI 数 × 每 ENI IP 数`，并开启前缀委派 |
| Calico IPAM | 上线前确定 `blockSize`（**创建后不可改**） |

**监控**：

```promql
# 节点已用 IP vs 上限（CNI 侧指标名依插件而定，以下为思路）
# Calico: felix_ipam_* 系列；Cilium: cilium_ipam_*；需要先 _ 核对
# 兜底：直接用 kubelet 已运行 Pod 数 / maxPods
sum by (node) (kubelet_running_pods) / 110 > 0.8
```

**回收泄漏**：节点删除后 IPAM block 未释放、早已终止的 Pod 残留 endpoint ⇒ 用 CNI 自带工具核查（`calicoctl ipam check`、`cilium status`）。

## C7. Service 数量爆炸怎么办？

- **架构治理**：不必要的 Service 收敛；内部调用用 Headless Service / SRV / 服务网格的直连机制；
- **数据面**：IPVS/eBPF 降低单次转发规则的匹配成本；
- **避免 ClusterIP 耗尽**：Service CIDR 要早规划（多数实现不可平滑扩容）；
- 监控 Service/EndpointSlice 总数增长趋势：

```promql
kube_service_info
count(kube_endpointslice_info)
```

## C8. NetworkPolicy 大规模落地的成本？

| 实现 | 规模成本 | 建议 |
| --- | --- | --- |
| iptables/ipsets 实现 | 每个 endpoint 一套链，**规则数随 Pod×策略数增长** | 控制策略数量与 selector 复杂度；定期审查僵尸策略 |
| eBPF map 实现 | 基于 identity + map 查表，增长更平缓 | 优先（同时满足内核要求） |

**实践建议**：

- 用**分层策略**：default-deny + 少量粗粒度放行，而不是每个服务一张精细表；
- 把策略对象数量、CNI 侧的策略同步延迟纳入监控（如 Cilium 的 `cilium_policy_*`、Calico 的 `felix_*`，**指标名请与目标版本核对**）；
- 每次策略变更都要在**灰度 Namespace**先验证再推全量。

## C9. 跨可用区流量怎么降？

```yaml
# ① Service 侧开启拓扑感知（新老两套写法，随版本演进；请以所用版本为准）
apiVersion: v1
kind: Service
metadata:
  name: web
  annotations:
    service.kubernetes.io/topology-aware-hints: auto      # 老写法
spec:
  # 新写法（版本相关）：trafficDistribution: PreferClose
  selector:
    app: web
  ports:
    - port: 80
      targetPort: 8080
```

**配套动作**：

- Deployment 用 **TopologySpreadConstraints** 让副本均匀分布到各可用区（否则"优先本地"会导致某区永远没有后端）；
- 注意：拓扑感知会牺牲**全局均衡**，热点/流量倾斜场景要压测；
- Ingress/网关侧也要做同区优先，否则只治标。

## C10. MTU / 分片问题的大规模治理？

- 全集群统一 MTU，并纳入"节点池基线"（配置漂移是最常见的事故源）；
- CNI 侧显式写死 MTU，避免依赖自动探测；
- 网络链路变更（如更换隧道、VXLAN offload）后**必须重新核算 MTU**；
- 开启 MSS clamp 作为兜底；
- 把"大包丢包率"作为常态化巡检项。

```bash
# 快速验证：从 Pod 内发大包（需要宿主机/对端配合 tcpdump 看是否分片被丢）
kubectl exec -it po -- ping -M do -s 8972 <peer-ip>     # 逐步降 size 找到临界值
```

## C11. 东西向流量可观测怎么落地？

| 手段 | 覆盖 | 成本 |
| --- | --- | --- |
| Hubble / CNI flow | L3/L4 + 部分 L7，含 drop reason | 较低（CNI 自带） |
| Service Mesh 遥测 | L7 全量指标/trace | sidecar 资源 + 复杂度 |
| 主机侧 eBPF 采集 | 跨容器宿主机视角 | 需额外组件 |
| 传统 netflow | 网络设备侧 | 与 CNI 语义割裂 |

**建议**：先由 CNI 提供 L3/L4 与丢包原因（性价比最高），L7 再按业务重要性选择性引入 mesh。

## C12. 大规模集群要不要拆"多集群"？

网络侧的真实动机只有三个：**① 故障域隔离；② IP 地址空间不足；③ 地域/延迟**。

- ClusterMesh/Submariner 类方案能打通 Pod IP，但会**把故障域重新连起来**——这与"隔离"目标矛盾；
- 更常见做法：**多集群 + 独立网络 + 上层 DNS/网关做流量调度**（容灾切换），而不是全互通。

## C13. 一次 CNI 变更的标准灰度与回滚（全网级变更）

**CNI 是集群里最不能随便动的东西之一**。规范流程：

| 阶段 | 做法 | 回滚触发 |
| --- | --- | --- |
| 准备 | 新 CNI 只在**新增节点池**启用（老节点池不动），并确保双方可互通（临时隧道/路由） | — |
| Step 1 | 把**无状态、可重建**的测试服务调度到新池（用 nodeSelector/taint 控制） | 连通性失败、DNS 失败、策略不生效 |
| Step 2 | 10% 业务 Pod 迁过去 | P99 恶化 / 错误率上升 |
| Step 3 | 50% → 100%，并逐步引导新节点加入新池 | SLO 指标异常 |
| 收尾 | 老节点下线（**注意**：已有 Pod 通常必须重建才能获得新 CNI 的 IP） | — |

**回滚要点**：

- 回滚 = 把 Pod 调度回旧节点池（改 nodeSelector/Pod 调度），**旧节点池必须完整保留到验收通过之后**；
- CNI 迁移通常**无法原地回滚**（已分配的 Pod IP 不会自动变化），所以"双池并存"是唯一安全的做法；
- 必须在窗口内完成，并准备"业务全量重建"的应急脚本。

## C14. 新集群网络基线设计（综合题，按顺序答）

1. **问 SLO**：是否需要 Pod IP 集群外可见？是否需要 L7 策略？是否需要跨集群？合规是否要求加密？
2. **CIDR 规划**：Pod/Service CIDR 与现网无冲突、留 2× 余量；云上要与 VPC 团队对齐。
3. **方案选型**：按 A16 决策树。
4. **模式配置**：是否需要 Overlay/CrossSubnet/原生路由；MTU 统一。
5. **规模组件**：Calico Typha/RR 或 Cilium（含 kube-proxy replacement 决策）。
6. **DNS**：NodeLocal DNSCache + CoreDNS 自动扩容 + ndots 治理。
7. **内核参数基线**：conntrack、somaxconn、somax backlog、`ip_forward` 等。
8. **策略**：default-deny 分层策略 + DNS 出口豁免。
9. **观测**：CNI 指标、CoreDNS、kube-proxy、conntrack、Hubble/flow。
10. **变更治理 + GameDay**：按 C13 灰度，定期做"CNI 组件挂掉/节点失联/IP 耗尽"演练。

# Part D · 反模式清单（可直接用于生产巡检）

| # | 反模式 | 证据（怎么发现） | 等级 | 修复动作 | 验证方式 |
| --- | --- | --- | --- | --- | --- |
| D1 | 用 Flannel 却以为 NetworkPolicy 生效 | 配了策略仍能访问 | 🔴 高 | 换支持策略的 CNI | B14 的验证脚本通过 |
| D2 | Service/Ingress 全集群无拓扑感知，跨可用区流量巨大 | 跨区流量计费、延迟高于同区 | 🟠 中 | 开 `PrefastClose`/hints + 拓扑打散 | 跨区流量占比下降（★实测） |
| D3 | 未部署 NodeLocal DNSCache 却有海量 DNS 查询 | `coredns_dns_requests_total` 极高、偶发 5s | 🔴 高 | 灰度上线 NodeLocal | SERVFAIL/5s 超时归零 |
| D4 | 外部域名未写 FQDN + ndots=5 | DNS QPS 放大 4\~10 倍 | 🟠 中 | 写 FQDN / 调 ndots | CoreDNS QPS 下降到放大部分消失 |
| D5 | `nf_conntrack_max` 保持默认 | `insert_failed` 增长、`table full` 日志 | 🔴 高 | 按并发连接实测调参 | 高压场景无丢计数 |
| D6 | Service 数上千仍用 iptables kube-proxy | 规则同步 P99 高、CPU 尖峰 | 🟠 中 | 切 IPVS 或 eBPF 替代 | `kubeproxy_sync_proxy_rules_duration_seconds` 回归基线 |
| D7 | MTU 未统一/未写死 | 大包丢、TLS 偶发失败 | 🔴 高 | 统一 MTU + MSS clamp | 大包 ping/压测通过 |
| D8 | Calico IPPool `blockSize` 未规划就上线 | 后期发现不可改 | 🟠 中 | 新建 IPPool 迁移（重操作） | `calicoctl ipam` 检查通过 |
| D9 | BGP full mesh 跑到几百节点 | BGP 会话数 O(n²)、收敛慢 | 🔴 高 | 上 Route Reflector / ToR RR | 会话数降到 O(n) |
| D10 | Typha 副本数不足/未监控 | 策略更新延迟、Felix 报错 | 🟠 中 | 多副本 + 跨故障域 + 告警 | 模拟杀 Typha 不影响策略下发 |
| D11 | default-deny 忘记放行 DNS | 加策略后全部解析失败 | 🔴 高 | 显式放行 kube-system 53/UDP+TCP | 加策略后 DNS 探测成功 |
| D12 | `externalTrafficPolicy: Local` 但副本未按区打散 | 部分区无后端，LB 健康检查失败 | 🔴 高 | 副本数 ≥ 可用区数 + TopologySpread | 各区均有 Ready 后端 |
| D13 | 生产直接原地换 CNI | 全集群网络中断 | 🔴 高 | 双节点池灰度迁移（C13） | 回滚到旧池成功 |
| D14 | 没有 Networks 丢包原因可观测 | 故障只能猜 | 🟠 中 | Hubble/flow + drop reason 告警 | 注入策略拒绝能立刻看到 DROPPED 原因 |
| D15 | Service CIDR 规划过小 | Service 数量接近上限 | 🟠 中 | 早期扩容/重建规划（不可平滑改） | 容量台账有充足余量 |

---

# Part E · 可观测性：SLO、指标与告警

## E1. 建议的网络侧 SLO（**初始建议值，必须实测校准**）

| SLI | 建议目标（起点） | 度量方式 |
| --- | --- | --- |
| Pod 网络连通成功率（同集群 Pod→Service） | ≥ 99.99% | 合成探测（blackbox-exporter 常驻 Pod） |
| DNS 解析成功率（内部域名） | ≥ 99.99% | blackbox DNS probe + CoreDNS SERVFAIL 率 |
| DNS P99 延迟 | ≤ 50ms | `coredns_dns_request_duration_seconds` |
| Service 变更后网络编程延迟 P99 | ≤ 5s | `kubeproxy_network_programming_duration_seconds`（版本相关） |
| kube-proxy 规则同步 P99 | ≤ 2s | `kubeproxy_sync_proxy_rules_duration_seconds` |
| conntrack 使用率 | ≤ 75% | node\_exporter |
| 丢包（drop）率 | ≈ 0（非策略拒绝类） | CNI flow |

> 错误预算：以 30 天滚动窗口，burn rate \> 14.4×（短期）/ 6×（长期）分别升级为紧急告警与工单。

## E2. 指标清单（**上线前必须在目标节点/组件核对该版本的指标名**）

| 组件 | 指标 | 用途 |
| --- | --- | --- |
| kube-proxy | `kubeproxy_sync_proxy_rules_duration_seconds` | 规则同步耗时（核心领先指标） |
| kube-proxy | `kubeproxy_network_programming_duration_seconds` | 变更到生效的网络编程延迟 |
| CoreDNS | `coredns_dns_requests_total` / `_request_duration_seconds` / `_responses_total` | DNS QPS/延迟/响应码 |
| CoreDNS | `coredns_cache_hits_total` / `coredns_cache_misses_total` | 缓存命中率 |
| node | `node_nf_conntrack_entries` / `_limit` / `node_nf_conntrack_drops` | conntrack 容量 |
| cilium-agent | `cilium_*`（如 `cilium_policy_*`、`cilium_datapath_*`、`cilium_ipam_*`、`cilium_controllers_failing`） | Cilium 健康/策略/IPAM |
| hubble | `hubble_...` / export black (dependson) | flow 与 drop reason |
| calico-node/felix | `felix_*`（如 `felix_int_dataplane_failures`、`felix_active_local_endpoints`、`felix_ipsets_calico`、`felix_resyncs_total`） | Felix 数据面健康 |
| cAdvisor / node | 网络接口 PPS、丢包、错包 | 网卡层问题 |

## E3. Prometheus 告警规则（可直接粘贴，阈值需实测校准）

```yaml
groups:
  - name: k8s-network
    rules:
      - alert: KubeProxySyncSlow
        expr: |
          histogram_quantile(0.99,
            sum by (le) (rate(kubeproxy_sync_proxy_rules_duration_seconds_bucket[5m]))) > 2
        for: 10m
        labels: {severity: warning}
        annotations:
          summary: "kube-proxy 规则同步 P99 达 {{ $value }}s，考虑切换 IPVS/eBPF 或收敛 Service"

      - alert: CoreDNSHighLatency
        expr: |
          histogram_quantile(0.99,
            sum by (le) (rate(coredns_dns_request_duration_seconds_bucket[5m]))) > 0.05
        for: 10m
        labels: {severity: warning}
        annotations:
          summary: "CoreDNS P99 解析延迟 {{ $value }}s"

      - alert: CoreDNSServfail
        expr: sum(rate(coredns_dns_responses_total{rcode="SERVFAIL"}[5m])) > 1
        for: 5m
        labels: {severity: critical}
        annotations:
          summary: "CoreDNS 出现 SERVFAIL，可能出现 5 秒超时问题"

      - alert: ConntrackNearFull
        expr: node_nf_conntrack_entries / node_nf_conntrack_entries_limit > 0.75
        for: 10m
        labels: {severity: warning}
        annotations:
          summary: "节点 {{ $labels.instance }} conntrack 使用率 {{ $value | humanizePercentage }}"

      - alert: ConntrackInsertFailed
        expr: rate(node_nf_conntrack_insert_failed[5m]) > 0
        for: 5m
        labels: {severity: critical}
        annotations:
          summary: "conntrack 插入失败，正在丢连接"

      - alert: CNINotReady
        expr: kube_node_status_condition{condition="NetworkUnavailable",status="true"} == 1
        for: 5m
        labels: {severity: critical}
        annotations:
          summary: "节点 {{ $labels.node }} 网络不可用，检查 CNI DaemonSet"

      - alert: NodePodIPPressure
        expr: sum by (node) (kubelet_running_pods) / 110 > 0.85
        for: 30m
        labels: {severity: info}
        annotations:
          summary: "节点 {{ $labels.node }} Pod 数接近 maxPods 上限"

      # 合成检测：集群内部 DNS/HTTP 成功率，建议用 blackbox-exporter 常驻 Pod
      - alert: PodHTTPProbeFailing
        expr: probe_success{job="kube-network-probe"} == 0
        for: 3m
        labels: {severity: critical}
        annotations:
          summary: "合成探测失败：{{ $labels.target }}"
```

---

# Part F · 演练（GameDay）计划表

| 场景 | 注入方式 | 期望表现 | 验证的 SLO |
| --- | --- | --- | --- |
| CoreDNS 全部挂掉 | scale NodeLocal/ CoreDNS to 0 | NodeLocal 缓存兜住一段时间；告警及时 | DNS 可用性 |
| conntrack 打满 | 短连接压测 | 告警先于丢包触发，业务降级可控 | 连接成功率 |
| kube-proxy 挂掉 | 停 DaemonSet Pod | 已建规则不失效，变更不再生效；能快速恢复 | 变更可用性 |
| CNI agent 挂掉（单节点） | 停 CNI Pod | 已有 Pod 通信通常不中断、新 Pod 起不来；节点被标记 NetworkUnavailable | MTTR |
| 策略误配 | 下发错误 NetworkPolicy | 立刻能看到 DROPPED + 业务告警，且能秒级删除策略恢复 | 策略变更安全 |
| IP 耗尽演练 | 在测试集群塞满 CIDR | 新 Pod 起不来且有明确告警 | IPAM 容量管理 |
| MTU 不一致 | 故意调整某节点 MTU | 大包失败被巡检发现 | 数据面一致性 |
| 跨可用区故障 | 隔离一个可用区 | 拓扑感知把流量导走 | 容灾 RTO |

---

# Part G · 10 张面试速记卡

1. **K8s 网络四句话**：Pod 互通无 NAT；Node 与 Pod 互通；Pod IP 自洽；实现交给 CNI。
2. **封包路径**：Pod eth0 → veth → 宿主（bridge/路由）→（隧道/路由/ENI）→ 对端宿主 → veth → Pod；Service IP 只是 DNAT 规则，没有网卡。
3. **CNI 三类插件**：main（建网卡）/ IPAM（分 IP）/ meta（portmap、bandwidth 等增强），ADD/DEL/CHECK 三接口 + stdin/env 调用。
4. **DNS 两大坑**：ndots=5 让外部域名查询放大 4\~10 倍；conntrack 竞态导致 5 秒超时 ⇒ NodeLocal DNSCache 是标准解。
5. **模式选择看的是"底层能否通告 Pod CIDR"**：能 → BGP/原生路由；不能 → Overlay；折中 → CrossSubnet；要 VPC IP → 云 CNI。
6. **MTU 是隐形炸弹**：VXLAN/IPIP 会让可用 MTU 变小；1514 变 1440/1450；配错就是"小包通大包死"。
7. **conntrack 是隐形的表**：规则我调了一百万次却忘了给 conntrack 扩容，短连接高峰就丢连接。
8. **NetworkPolicy 是 CNI 实现的**：Flannel 不支持；default-deny 一定记得放行 DNS。
9. **规模三件套**：Service 多 → IPVS/eBPF；DNS 压力大 → NodeLocal + 自动扩容；BGP 节点多 → Route Reflector（Calico 还有 Typha 扛 watch 压力）。
10. **CNI 迁移只能"双池并存灰度"**，不能原地换；回滚 = 把 Pod 调度回旧节点池，老池要保留到验收通过。

---

## 附：文档自查清单

- [ ] 每条建议配了可直接套用的 YAML / Helm values / sysctl / PromQL
- [ ] 所有默认值（ndots=5、maxEndpointsPerSlice=100、conntrack、VXLAN MTU 等）标注了"以版本为准"
- [ ] 未提供任何未经实测的性能数字，全部给出**对比测量方法**
- [ ] 反模式清单含：证据 / 等级 / 修复 / 验证
- [ ] SLO 标为建议初值 + burn rate 口径
- [ ] 高危变更（CNI 迁移、NodeLocal 上线）给出灰度比例、回滚触发条件与回滚代价
