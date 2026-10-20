# istio-interview-qa

# Istio 面试题全解析：原理篇 × 方案模式篇 × 大规模优化篇

> 适用角色：SRE / 云原生平台 / 服务网格负责人面试（社招 3 年起 + 偏高阶）
> 姊妹篇：《Scheduler》《Kubelet》《网络 / CNI》《控制面》《存储 / CSI》《Linux / 容器底层》《GPU / 大模型》《排障实战 60 例》
> 版本约定：以 **Istio 1.20 \~**** 1.24 时代**为背景（istiod 单控制面 + Envoy 数据面 + 可选 Ambient）。**版本号、feature 阶段（alpha/beta/GA）、默认值变化很快**，凡涉及版本的地方均标注"以所用版本的 Release Notes / 官方文档为准"。

---

## 0. 使用说明

### 0.1 Istio 面试的考察层次

| 层级 | 考点 | 表现 |
| --- | --- | --- |
| L1 会用 | 会写 VirtualService / DestinationRule / Gateway | 只会抄 YAML |
| L2 懂原理 | 说得清 istiod、xDS、Envoy 拦截、mTLS 身份、证书下发 | 能画架构与流量路径 |
| L3 懂落地 | 会做金丝雀、mTLS 迁移、多集群、限流熔断、可观测 | 有迁移步骤与灰度方案 |
| L4 有规模与事故经验 | 能讲"控制面推送风暴""指标基数爆炸""注入后启动变慢/探针失败""503 排查" | 有指标、有裁剪手段、有回滚 |

> **答题模板**：先说**流量怎么走**（谁拦截 → 谁决策 → 谁转发），再说**控制面怎么下发**（xDS 与身份），最后给**证据**（CR 字段 / Envoy 配置 / 指标名 / 诊断命令）。

### 0.2 术语速查

| 缩写/名词 | 含义 |
| --- | --- |
| **istiod** | 合并后的控制面：Pilot（配置转换与 xDS 推送）+ CA（证书签发） |
| **Envoy / istio-proxy** | 数据面代理（sidecar 或 gateway 形态） |
| **pilot-agent** | sidecar 容器里管理 Envoy 生命周期与 bootstrap 的小进程 |
| **xDS** | Envoy 动态配置协议：LDS/RDS/CDS/EDS/SDS + ADS 聚合 |
| **HBONE** | Ambient 模式下基于 HTTP/2 的隧道协议（默认端口 15008 相关） |
| **ztunnel / waypoint** | Ambient 模式的节点级 L4 代理 / 按需 L7 代理 |
| **PeerAuthentication** | mTLS 模式（STRICT / PERMISSIVE / DISABLE） |
| **AuthorizationPolicy** | L7 授权（ALLOW / DENY / CUSTOM） |
| **SPIFFE** | 工作负载身份格式：`spiffe://<trust-domain>/ns/<ns>/sa/<sa>` |
| **ServiceEntry / WorkloadEntry** | 把网格外服务或非 K8s 负载纳入网格 |
| **Telemetry API** | 声明式配置指标/日志/追踪（取代大量 EnvoyFilter 用法） |

### 0.3 一张"包路径"总览（先建立画面）

```
① 入站：Client → LB → Ingress Gateway(Envoy) → [HBONE/mTLS] → Sidecar(Envoy inbound) → App
② 出站：App → Sidecar 拦截(iptables/TPROXY) → 策略/路由 → [mTLS] → 对端 Sidecar → 对端 App
③ 控制面：istiod 监听 K8s/CRD → 生成 Envoy 配置 → 通过 xDS(ADS, 15012) 推送给所有 proxy
④ 身份：istiod CA 签发证书 → 通过 SDS 下发 → 双向 mTLS 建立，身份 = SPIFFE ID
```

---

# Part A · 原理篇（26 题）

## A1. 服务网格到底解决什么问题？Istio 不做哪些事？

**解决**（把"服务治理"从业务代码里下沉到基础设施）：

| 能力 | 说明 |
| --- | --- |
| **流量治理** | 路由、分流、灰度、镜像、故障注入、重试/超时、熔断 |
| **安全** | 工作负载身份（SPIFFE）、**mTLS 透明加密**、L7 授权（JWT/属性） |
| **可观测** | 统一的 RED 指标、访问日志、链路追踪（含 mTLS/响应标志等上下文） |
| **出口/边界治理** | Egress 统一出口、审计与策略 |

**不做（高频送分点）**：

- 不负责 **Pod 之间的连通性**（那是 CNI 的事，见《网络篇》）；
- 不负责 **服务发现本身**（它消费 K8s Service/EndpointSlice，或用 ServiceEntry 扩展）；
- 不实现 **NetworkPolicy**（L3/4 网络策略仍由 CNI 执行，两者互补）；
- 不是 **API 网关的全部**（Gateway API / Ingress 仍负责南北向，网格管东西向 + 部分南北向）；
- 也不替代 **业务层的幂等/事务**（重试只能治理瞬时故障）。

**一句话定位**：**Istio = 以身份为中心的东西向流量与策略平面**。

## A2. Istio 架构怎么演进到现在这样？

| 阶段 | 形态 |
| --- | --- |
| 早期 1.0\~1.4 | **微服务化控制面**：Pilot（流量）、Mixer（策略与遥测、**性能瓶颈**）、Citadel（CA）、Galley（配置校验）多进程 |
| 1.5 起 | **合并为单体 istiod**：Mixer 逻辑外置（遥测交给 Envoy 原生统计 + Telemetry API），控制面大幅简化 |
| 1.6\~1.19 | 持续强化：VM 支持、多集群、Telemetry API、Gateway API 支持等 |
| 近版本 | 引入 **Ambient Mesh（sidecarless）**：ztunnel + waypoint；同时 **Kubernetes Gateway API** 成为推荐的入口配置方式（**具体 GA 状态以版本为准**） |

**必答的"为什么"**：Mixer 被移除是因为它是**每次调用都要打点上报的中心化组件**，造成延迟与可用性耦合——现在遥测主要由 **Envoy 侧统计 + Telemetry API 配置**完成。

## A3. Envoy 数据面：关键对象与进程模型

| Envoy 概念 | 作用 |
| --- | --- |
| **Listener** | 监听端口；**filter chain** 决定怎么处理（HTTP 连接管理器、TCP 代理等） |
| **Route（RDS）** | 域名/路径匹配 → 目标 cluster |
| **Cluster（CDS）** | 逻辑上游（对应 K8s Service 或 subset） |
| **Endpoint（EDS）** | cluster 的真实后端地址（来自 EndpointSlice 或 ServiceEntry） |
| **Secret（SDS）** | 证书与密钥（**按需下发，不落盘**） |
| **Filter** | 处理管道（HTTP filter：路由、认证、限流、遥测） |

**进程模型**：Envoy 是**多线程 + 事件驱动**，主线程负责配置与管理，worker 线程处理连接；通过 **热重启（hot restart）** 与 **--drain-time** 实现无损重启。

**sidecar 容器里其实有两个进程**：

- `pilot-agent`：引导与管理 Envoy（生成 bootstrap、健康探测 15021、metrics 聚合 15020）；
- `envoy`：真正的数据面。

```bash
# 排障常用
kubectl exec <pod> -c istio-proxy -- curl -s localhost:15000/config_dump | less
kubectl exec <pod> -c istio-proxy -- curl -s localhost:15000/clusters
kubectl exec <pod> -c istio-proxy -- curl -s localhost:15000/listeners
kubectl exec <pod> -c istio-proxy -- pilot-agent request GET /debug/config_dump
```

**常用端口（请与所用版本核对）**：

| 端口 | 用途 |
| --- | --- |
| 15001 | Envoy **出站**重定向监听 |
| 15006 | Envoy **入站**重定向监听 |
| 15000 | Envoy admin 接口（config\_dump、stats、日志级别） |
| 15020 | pilot-agent 的聚合监控端口（merged metrics） |
| 15021 | sidecar 就绪健康 `/healthz/ready` |
| 15010 / 15012 | istiod 的 xDS（明文 / mTLS） |
| 15014 | istiod 自身的监控端口 |
| 15008 | Ambient（HBONE 隧道）相关端口 |

## A4. xDS 与控制面推送模型（必考）

```
istiod 监听：Service / EndpointSlice / VirtualService / DestinationRule / Gateway / AuthZ ...
   → 组装成 Envoy 配置的"模型"
   → 通过 ADS（gRPC 双向流，通常 15012 端口）推送给每个 proxy
   → proxy 应用配置并 ACK/NACK
```

**要点**：

- **ADS**：把各类 xDS 聚合在一条流上，保证**更新顺序一致**（避免"路由指向了还没就绪的 cluster"这种中间态）；
- **增量推送**：配置变化只推送受影响的部分，但**影响面分析做得不好就会变成"推送风暴"**（见 C1）；
- **NACK**：proxy 拒绝错误配置 ⇒ 会保留旧配置，但**istiod 侧要能看到 NACK 并告警**；
- **debounce / 节流**：Endpoint 抖动会被合并，避免频繁推送（相关环境变量/参数以版本为准）。

```promql
# istiod 侧推送与连接（指标名以版本为准）
pilot_xds_pushes
pilot_xds_push_errors
pilot_xds_eds_instances    # EDS 规模
pilot_proxy_convergence_time_bucket
# NACK 往往是"配置错误"的第一信号
pilot_xds_cds_reject / pilot_xds_lds_reject 之类（以实际指标名为准）
```

## A5. sidecar 是怎么被注入的？

```
① 给 namespace 打 label：istio-injection=enabled  （或为 Pod 打 sidecar.istio.io/inject=true）
② 创建 Pod 时，API Server 回调 istiod 的 mutating webhook（sidecar-injector）
③ webhook 把 istio-init（或 CNI 模式）与 istio-proxy 两个容器注入 Pod spec
④ istio-init：写 iptables 规则把流量重定向到 15001/15006（**需要 NET_ADMIN**）
   或用 istio-cni 插件：由节点 CNI 链路完成重定向（**免 NET_ADMIN**，更安全）
⑤ istio-proxy 启动、连 istiod、拉配置、就绪（15021）后应用容器才被认为可服务
```

**关键注解（高频）**：

```yaml
metadata:
  annotations:
    sidecar.istio.io/inject: "true"
    traffic.sidecar.istio.io/excludeOutboundPorts: "3306"     # 出站不走代理
    traffic.sidecar.istio.io/excludeInboundPorts: "22"
    traffic.sidecar.istio.io/includeInboundPorts: "8080"
    proxy.istio.io/config: '{"holdApplicationUntilProxyStarts": true}'
    sidecar.istio.io/proxyCPU: "500m"
    sidecar.istio.io/proxyMemoryLimit: "512Mi"
```

**必答的两条**：

1. `holdApplicationUntilProxyStarts` 解决"应用比 sidecar 先起来 ⇒ 早期请求丢失/探针失败"；
2. **istio-cni** 的价值是**去掉 NET\_****ADMIN 特权**，符合安全基线（代价是节点多一个 CNI 插件依赖）。

## A6. 流量拦截：iptables 是怎么"偷走"流量的？

- 出站：Pod 内所有到其它服务的 TCP 被 **REDIRECT/TPROXY** 到 **15001**；Envoy 根据原始目的（通过 SO\_ORIGINAL\_DST / TPROXY）决定路由；
- 入站：到业务端口的流量被重定向到 **15006**；Envoy 处理后转发给本机应用端口；
- 排除：自身 Envoy 的流量、健康检查、以及 excludePorts 配置。

**常见坑（必答）**：

- 应用**需要拿到客户端真实 IP** ⇒ 需要正确传递（Envoy 会写入 `X-Forwarded-For`、`X-Envoy-External-Address`）；
- **非 TCP 协议/特殊端口**（如某些数据库协议）拦截后可能出问题 ⇒ 用 excludePorts；
- **hostNetwork: true 的 Pod** 注入通常不适用（流量不进 Pod netns）；
- 与 CNI 插件的兼容性（链式 CNI 顺序、istio-cni 与 Calico/Cilium 的协作要验证）。

## A7. 为什么默认"全量下发"在大规模集群会炸？怎么收敛？

istiod 默认把**集群内所有服务与配置**下发给每个 sidecar（这样任何服务都能被解析与路由）。结果是：**配置体积随服务数线性增长**，服务上千后 Envoy 内存与 istiod 推送成本都会失控。

**收敛三件套（必答，且是大规模第一优化）**：

```yaml
# ① Sidecar CR：限制该命名空间内 sidecar 的可见范围
apiVersion: networking.istio.io/v1
kind: Sidecar
metadata:
  name: default
  namespace: prod
spec:
  egress:
    - hosts:
        - "./*"                 # 本命名空间
        - "istio-system/*"      # 控制面
        - "infra/*"             # 依赖的基础设施命名空间
```

```yaml
# ② istiod 的 discoverySelectors：只发现指定命名空间（MeshConfig 侧，版本相关）
meshConfig:
  discoverySelectors:
    - matchLabels:
        istio-discovery: enabled
```

```yaml
# ③ 给 CR 用 exportTo 限制可见范围
spec:
  exportTo: ["."]     # 只在本命名空间可见
```

> 面试时说出"**默认全量下发是 Istio 在大规模下的头号成本来源，收敛靠 Sidecar CR + discoverySelectors + exportTo**"就是标准答案。

## A8. 流量管理 API 各自管什么？（最容易混的一组）

| CRD | 作用 | 常见坑 |
| --- | --- | --- |
| **Gateway** | 定义**网格边缘**（Ingress/Egress）的监听：端口、协议、TLS、hosts | 它不是路由；只负责"开门" |
| **VirtualService** | **路由规则**：把请求匹配后指向某个 destination（可加权/匹配/重写/重试） | 匹配有**优先级顺序**；多 VS 对同一 host 会合并，行为易混乱 |
| **DestinationRule** | **目标策略**：subset 定义、负载均衡、连接池、异常点检测、TLS 设置 | 没有 DR 就没有 subset；TLS 设置与 PeerAuthentication 冲突会 503 |
| **ServiceEntry** | 把**网格外**服务加入服务发现（外部 API、VM 服务） | 没注册则访问被拦或走 Passthrough，策略不生效 |
| **WorkloadEntry / WorkloadGroup** | 非 K8s 工作负载（VM）注册与分组 | 需要配合 ServiceEntry 使用 |
| **Sidecar**（资源） | 收敛 sidecar 可见性（见 A7） | 名字与其它概念同名，容易混 |
| **EnvoyFilter** | 直接改 Envoy 配置 | **最后手段**，升级易碎，优先用 Telemetry API / WasmPlugin |

**必答的"host 匹配"坑**：

- VS 的 `hosts` 要写**服务在网格内的名字**（K8s Service 名或 FQDN），写错不会报错但路由不生效；
- gateway 场景要写外部域名，且 `gateways` 字段要指定 gateway 名；
- 短名与 FQDN 的解析差异，是"为什么本地能通、从网关进不来"的常见原因。

## A9. 路由匹配的优先级规则（必考细节）

```
按以下顺序，先匹配者胜：
  1. 更具体的 URI（精确 > 前缀 > 正则）
  2. 更具体的 method / headers / queryParams 等匹配条件
  3. 最后落到默认的 route（无 match 的规则）
```

**坑**：把"兜底路由"写在最前面 ⇒ 后面所有规则失效。**建议总是把最具体的规则放前面，兜底放最后**，并用 `delegate` 拆分大型路由表。

```yaml
apiVersion: networking.istio.io/v1
kind: VirtualService
metadata:
  name: reviews
spec:
  hosts: [reviews]
  http:
    - match:
        - headers:
            end-user: {exact: jason}     # ① 更具体
      route:
        - destination: {host: reviews, subset: v2}
    - route:                              # ② 兜底放最后
        - destination: {host: reviews, subset: v1}
```

## A10. DestinationRule：subset 从哪来？负载均衡与熔断怎么配？

```yaml
apiVersion: networking.istio.io/v1
kind: DestinationRule
metadata:
  name: reviews
spec:
  host: reviews
  trafficPolicy:
    loadBalancer:
      simple: LEAST_REQUEST              # ROUND_ROBIN/LEAST_CONN/LEAST_REQUEST/RANDOM
      # 一致性哈希（会话保持）：
      # consistentHash: {httpHeaderName: x-user}
    connectionPool:
      tcp: {maxConnections: 100, connectTimeout: 5s}
      http: {h2UpgradePolicy: UPGRADE, http2MaxRequests: 1000, maxRequestsPerConnection: 100}
    outlierDetection:
      consecutive5xxErrors: 5            # 连续 5 次 5xx ⇒ 摘除
      interval: 10s
      baseEjectionTime: 30s
      maxEjectionPercent: 50
  subsets:
    - name: v1
      labels: {version: v1}
    - name: v2
      labels: {version: v2}
```

**必答**：

- **subset 的 labels 匹配的是 endpoint 的标签**（来自 Pod labels），不是 Service selector；
- `outlierDetection` 是**被动健康检查**（Envoy 侧，基于实际请求结果），与 K8s 的 readiness 互补；
- 熔断是**每个上游（cluster）维度**的，不是全局的；
- `trafficPolicy.tls` 与 `PeerAuthentication` 不一致是 **503 的高频原因**（见 B8）。

## A11. 弹性四件套：超时、重试、熔断、故障注入

```yaml
http:
  - route:
      - destination: {host: ratings}
    timeout: 3s
    retries:
      attempts: 3
      perTryTimeout: 1s
      retryOn: 5xx,connect-failure,refused-stream,retriable-4xx
      retryBudget:                 # 限制重试占比，防重试风暴
        percent: 20
        minRetryConcurrency: 10
```

| 能力 | 注意 |
| --- | --- |
| **超时** | Envoy 默认无超时，必须显式配（**这是很多人不知道的默认值**） |
| **重试** | 只对幂等操作使用；**必须配 retryBudget**，否则雪崩放大；POST 慎用 |
| **熔断/异常点检测** | 见 A10 |
| **故障注入** | `fault.delay/abort`，只用于测试环境 |
| **流量镜像** | `mirror` + `mirrorPercentage`，请求发给影子服务且**忽略响应**，用于上线前验证 |

## A12. mTLS 与身份：Istio 安全的三块基石

```
① 身份：每个工作负载有 SPIFFE ID = spiffe://<trust-domain>/ns/<ns>/sa/<sa>
        （来自 Pod 的 ServiceAccount，由 istiod CA 签发证书）
② 传输：mTLS（PeerAuthentication 控制模式）
③ 授权：AuthorizationPolicy 基于身份/属性做 ALLOW/DENY
```

```yaml
# 网格级：宽松模式（迁移期）→ 严格模式
apiVersion: security.istio.io/v1
kind: PeerAuthentication
metadata:
  name: default
  namespace: istio-system          # 网格级
spec:
  mtls:
    mode: STRICT                   # STRICT | PERMISSIVE | DISABLE
```

```yaml
# 授权：只允许特定 SA 访问
apiVersion: security.istio.io/v1
kind: AuthorizationPolicy
metadata:
  name: allow-web-to-api
  namespace: prod
spec:
  selector:
    matchLabels: {app: api}
  action: ALLOW
  rules:
    - from:
        - source: {principals: ["cluster.local/ns/prod/sa/web"]}
      to:
        - operation: {methods: ["GET", "POST"], ports: ["8080"]}
```

```yaml
# JWT 认证（RequestAuthentication + 授权要求）
apiVersion: security.istio.io/v1
kind: RequestAuthentication
metadata:
  name: jwt
  namespace: prod
spec:
  selector: {matchLabels: {app: api}}
  jwtRules:
    - issuer: "https://issuer.example.com"
      jwksUri: "https://issuer.example.com/.well-known/jwks.json"
      audiences: ["my-api"]
```

**必答坑**：

- `RequestAuthentication` **只验证 JWT，不强制要求**；要拒绝无 token 请求必须再配 `AuthorizationPolicy` 要求 `requestPrincipals`；
- `AuthorizationPolicy` 的 `DENY` 优先于 `ALLOW`；没有匹配任何 ALLOW 时**默认是拒绝还是允许**取决于是否存在该 selector 的 ALLOW 策略（**这是最容易答错的地方：一旦有 ALLOW 策略选中该工作负载，未匹配的请求即被拒绝**）；
- mTLS 模式冲突（服务端 STRICT、客户端 DISABLE）⇒ **503 upstream connect error**。

## A13. 证书是怎么签发和轮换的？

```
proxy 启动 → 通过 SDS 向 istiod 请求证书（带 SA token 做身份认证）
   → istiod CA（或插件 CA：cert-manager/Vault/SPIRE/自定义 root）签发
   → 证书短期有效（默认 TTL 以版本为准，常见为 24 小时量级）
   → 到期前自动轮换（无需重启 Pod）
```

**必答**：

- 短期证书 + 自动轮换是设计核心（**泄露风险窗口小**）；
- 生产上通常用**自定义根 CA / 外部 CA 插件**以满足合规（**插拔方式以版本文档为准**）；
- **控制面不可用时**，已有 sidecar 仍能用现有证书通信，但**新 Pod 拿不到证书 ⇒ 起不来** ⇒ istiod 必须 HA（见 C11）。

## A14. 入口与出口：Gateway / Ingress / Egress 的关系

| 组件 | 角色 |
| --- | --- |
| **Ingress Gateway** | 南北向入口（Deployment + Envoy，通常配 LB Service），由 `Gateway` + `VirtualService` 配置 |
| **Kubernetes Ingress** | 也能被 Istio 支持（通过 IngressClass），但**社区更推荐 Gateway API / Istio Gateway** |
| **Egress Gateway** | 统一出口：**固定出口 IP + 集中审计/策略 + TLS 发起**，用于合规场景 |
| **East-West Gateway** | 多集群之间的跨集群流量入口（见 A18） |

```yaml
apiVersion: networking.istio.io/v1
kind: Gateway
metadata:
  name: web-gw
  namespace: istio-system
spec:
  selector:
    istio: ingressgateway
  servers:
    - port: {number: 80, name: http, protocol: HTTP}
      hosts: ["*.example.com"]
    - port: {number: 443, name: https, protocol: HTTPS}
      hosts: ["*.example.com"]
      tls: {mode: SIMPLE, credentialName: example-tls}   # 引用 K8s Secret（SDS）
```

**必答**：`credentialName` 让 gateway 通过 SDS 动态取证书，**无需挂载 Secret 卷、无需重启**（这是常见加分点）。

## A15. Ambient Mesh 是什么？为什么它被视为下一代形态？

| 组成 | 位置 | 职责 |
| --- | --- | --- |
| **ztunnel** | 每个节点一个（DaemonSet） | **L4**：mTLS、认证、L4 授权、遥测（不含 L7 解析） |
| **waypoint** | 按需，通常按服务账号部署 | **L7**：HTTP 路由、鉴权、L7 遥测与策略 |
| **HBONE** | 隧道 | 基于 HTTP/2 的封装（默认端口 15008 相关），承载 mTLS 与元数据 |

**价值（必答）**：

1. **不再给每个 Pod 注入 sidecar** ⇒ 省内存/CPU、Pod 启动更快、**应用与代理解耦**（升级网格不需要重启业务 Pod）；
2. **渐进采用**：无 sidecar 的 Pod 与有 sidecar 的 Pod 可以互通，可按需给服务打开 waypoint 拿 L7 能力；
3. 更适合"只想拿 mTLS 与可观测、不需要复杂 L7 策略"的场景。

**限制与风险（必须说）**：

- 功能覆盖与 sidecar 模式不完全等价（**具体差异以所用版本官方矩阵为准**）；
- hostNetwork、某些 CNI/内核依赖、以及部分高级特性需要逐项验证；
- 排障工具链从"进 sidecar 看 config\_dump"变成"看 ztunnel/waypoint"，团队技能要更新。

## A16. Sidecar vs Ambient vs "不用网格"怎么选？

| 维度 | Sidecar | **Ambient** | 无网格（只用 CNI + 库） |
| --- | --- | --- | --- |
| 每 Pod 开销 | 一个 Envoy（内存/CPU 随配置增长） | 无（节点级共享） | 无 |
| 升级网格 | 需滚动重启业务 Pod | **通常不需要重启业务** | — |
| L7 能力 | 全 | 按需（waypoint） | 靠 SDK |
| 隔离性 | 强（每 Pod 独立代理） | 节点级共享（多租户需评估） | 应用内 |
| 成熟度 | 最成熟 | 较新（**版本相关**） | — |
| 适用 | 需要完整 L7 策略/可观测的核心服务 | 想先拿 mTLS 与基础遥测、追求低开销 | 简单场景 |

**标准答法**：

> "不要为了上网格而上网格。如果诉求只是 mTLS + 基础可观测，Ambient 的性价比明显更高；如果需要复杂的 L7 路由/鉴权/限流，sidecar（或带 waypoint）更稳。**两者可以在同一网格共存**，这就是渐进迁移的路径。"

## A17. DNS 与无 VIP 服务：为什么 Istio 需要 DNS 代理？

**问题**：K8s 的 **Headless Service**（ClusterIP=None）没有 VIP，返回的是 Pod IP 列表。Envoy 若只按 VIP 转发，就处理不了这类服务；此外**外部服务（ServiceEntry）也没有 VIP**。

**解法**：Envoy 开启 **DNS 代理/捕获**，sidecar 自己解析 DNS 并把解析结果纳入路由与负载均衡。

**相关坑（必答）**：

- StatefulSet + Headless 的服务必须确认 DNS 捕获生效，否则访问失败或负载不均；
- 与 `ndots`/search 域的组合会放大 DNS 查询（与《网络篇》A7 同源）；
- 解析失败的排障：看 Envoy 的 `dns` 相关统计与 `upstream` 状态。

## A18. 多集群的三种拓扑（与"为什么复杂"）

| 拓扑 | 说明 | 适用 |
| --- | --- | --- |
| **Multi-primary（多主）** | 每个集群有自己的 istiod，都连接所有集群的 API Server | 高可用最好，运维最复杂 |
| **Primary-remote（主-远）** | 一个主集群的 istiod 管理远端集群的数据面 | 常见、简化控制面 |
| **External control plane** | 控制面在专用集群（甚至托管） | 厂商/托管场景 |

**关键点**：

- **东西向网关**用于跨集群流量（网络不扁平时必须）；
- **共享/独立信任域**：跨集群 mTLS 需要统一或联邦的信任配置；
- **网络要求**：扁平网络（Pod IP 可直连）最简单；非扁平则依赖网关与隧道；
- **服务发现**：远端集群的 Service 如何被发现（远端 secret/配置，**机制以版本为准**）。

## A19. VM / 非 K8s 负载怎么接入网格？

- **WorkloadGroup**：描述一组非 K8s 工作负载（模板）；
- **WorkloadEntry**：注册具体实例（IP/端口/labels），常与 **ServiceEntry** 配合，让它们像网格内服务一样被发现与路由；
- 也有 **Workload Onboarding** 类方案用于自动化接入（**成熟度以版本为准**）。

```yaml
apiVersion: networking.istio.io/v1
kind: WorkloadEntry
metadata:
  name: vm-db-1
  namespace: prod
spec:
  address: 10.10.1.5
  labels: {app: db, version: v1}
  serviceAccount: db
```

## A20. 可观测性：Istio 提供什么？成本在哪？

| 类型 | 内容 |
| --- | --- |
| **指标** | `istio_requests_total`、`istio_request_duration_milliseconds`、`istio_request_bytes`、`istio_tcp_*` 等（由 Envoy 统计并通过 15020 暴露） |
| **访问日志** | 默认可开启（文本或 JSON），含身份、响应标志、上游等上下文 |
| **追踪** | 传播 W3C `traceparent` / B3，采样率可配，导出到 OTel/Jaeger 等 |
| **拓扑/可视化** | Kiali 等 |

**成本（必答）**：

- **指标基数爆炸**：`istio_requests_total` 带 source/destination 的工作负载、命名空间、principal、响应码、响应标志等多个标签 ⇒ 服务数平方级增长的组合；
- 访问日志全量写盘/采集很贵；
- **解法**：用 **Telemetry API 裁剪维度**、调整采样、Prometheus 侧 relabel 丢弃高基数标签。

```yaml
# Telemetry API：去掉高基数维度（示意，字段以版本为准）
apiVersion: telemetry.istio.io/v1
kind: Telemetry
metadata:
  name: mesh-default
  namespace: istio-system
spec:
  metrics:
    - providers: [{name: prometheus}]
      overrides:
        - match: {metric: REQUEST_COUNT, mode: CLIENT_AND_SERVER}
          tagOverrides:
            request_host: {operation: REMOVE}
            source_principal: {operation: REMOVE}
```

## A21. 与 K8s 原生能力如何分工？

| 能力 | K8s 原生 | Istio |
| --- | --- | --- |
| 服务发现 | Service + EndpointSlice | 消费它，并可用 ServiceEntry 扩展 |
| 负载均衡 | kube-proxy（L4，per-connection） | Envoy（L7，per-request，带重试/超时/异常点） |
| 入口 | Ingress / Gateway API | Ingress Gateway（也支持 Gateway API） |
| 网络策略 | **NetworkPolicy（L3/4，CNI 实现）** | AuthorizationPolicy（L7，身份与属性） |
| mTLS | 无 | 有 |
| 可观测 | 需自行埋点 | 自动 RED 指标 + 追踪上下文 |

**必答**：**NetworkPolicy 与 AuthorizationPolicy 不冲突、应叠加**——前者管"能不能连到端口"，后者管"以什么身份、操作什么资源"。

## A22. 协议与 SNI：TLS 直通 vs 终止

| 模式 | 行为 | 适用 |
| --- | --- | --- |
| **PASSTHROUGH** | Envoy 不解密，按 SNI 转发 | 外部 HTTPS、客户端证书认证 |
| **SIMPLE / MUTUAL** | 在网关终止 TLS（可选再 mTLS 到后端） | 常规 Web |
| **AUTO\_****PASSTHROUGH** | 多集群/跨网格场景 | 东西向 |

**坑**：

- 不配置 SNI 的 TLS 流量在 PASSTHROUGH 下无法路由；
- 端口协议识别错（HTTP 被当成 TCP）⇒ L7 能力全部失效（见 A23）。

## A23. 端口命名/协议识别：为什么"服务看起来通但 L7 规则不生效"？

Envoy 需要知道端口上的协议才能应用 HTTP 路由与策略。识别顺序大致是：

```
① 显式声明（Service port 的 name = http-<suffix> / grpc-... 或 appProtocol）
② 否则做协议嗅探（有代价且不总是准）
③ 仍无法识别 ⇒ 按 TCP 处理（只有 L4 能力）
```

**必答**：

- 现代写法推荐用 `appProtocol`（Service port 字段）显式声明；
- 名字写错（如 `http2` 写成 `http2-xxx` 之外的形式）会导致协议识别失败；
- TCP 服务（MySQL、Redis）不要强行按 HTTP 处理。

## A24. istiod 自身的高可用与依赖

- istiod 是**无状态化依赖**（配置来自 K8s API）⇒ 可多副本；
- 但**证书签发/部分协调逻辑有选主**（以版本实现为准）⇒ 通常 2\~3 副本；
- **istiod 挂了的影响**：已有流量不受影响（数据面已持有配置与证书），但**新 Pod 起不来、配置变更不生效** ⇒ 必须 HA + PDB + 资源保障。

## A25. 版本兼容与升级模型

- **控制面与数据面版本差**有官方支持范围（**常见是相差不超过一个 minor 版本，请以所用版本官方支持矩阵核对**）；
- **金丝雀升级**：安装新版本控制面时带 **revision**，通过 label/tag 让部分命名空间逐步切换数据面：

```bash
istioctl install --revision=1-24-0
istioctl tag set prod-canary --revision 1-24-0
kubectl label ns prod istio.io/rev=prod-canary --overwrite
kubectl rollout restart deploy -n prod        # 让 sidecar 换版本
# 验证通过后，把默认 tag 指向新 revision，再逐命名空间滚动
```

**必答**：升级后要验证：① istiod 无 NACK；② 关键服务流量与 mTLS 正常；③ 指标与追踪不丢。

## A26. 网格里最常见的"服务不通"原因排序

| 排序 | 原因 | 快速验证 |
| --- | --- | --- |
| 1 | **端口协议识别错**（当 TCP 处理） | `config_dump` 看 listener 类型 |
| 2 | **mTLS 模式冲突**（STRICT 侧收到明文） | 日志 `upstream connect error / 503 UF,URX` |
| 3 | **VirtualService host 写错** / 未绑定 Gateway | 看 Envoy route 是否包含目标 |
| 4 | **外部服务未注册 ServiceEntry** | 出站被拒或走 Passthrough |
| 5 | **Sidecar 未注入 / 注入失败** | `kubectl get pod -o jsonpath` 看容器数 |
| 6 | **AuthorizationPolicy 未放行**（尤其新加 ALLOW 后默认拒绝） | 临时改 action 或删策略验证 |
| 7 | **DNS/Headless 问题** | 看 DNS 统计与解析结果 |
| 8 | **熔断/异常点把上游摘了** | 看 outlier 统计与 `outlier_detection` 状态 |

```bash
# 万能三连：看 proxy 状态、看配置同步、看统计
istioctl proxy-status
istioctl analyze -n <ns>
istioctl proxy-config route <pod> -o json
kubectl exec <pod> -c istio-proxy -- curl -s localhost:15000/stats | grep -i "upstream_rq\|outlier"
```

# Part B · 方案 / 模式专项篇（14 题）

## B1. 注入方式对比：什么时候用哪种？

| 方式 | 说明 | 适用 |
| --- | --- | --- |
| **namespace label 自动注入** | `istio-injection=enabled` | 常规 |
| **Pod annotation 覆盖** | `sidecar.istio.io/inject: "true"` 覆盖 ns 设置 | 例外场景/灰度 |
| **istio-cni** | 由 CNI 链路做重定向（**免 NET\_****ADMIN**） | 有安全基线要求（禁特权/禁 NET\_ADMIN） |
| **Ambient（无 sidecar）** | namespace 打 ambient 标签 + 按需 waypoint | 想要低开销与免重启升级 |
| **手动注入** | `istioctl kube-inject` | 调试/特殊控制器 |

```bash
# 常用开关（顺序：ns label 是基础，pod annotation 优先）
kubectl label ns prod istio-injection=enabled
kubectl label ns legacy istio-injection=disabled
# Ambient（标签名以版本为准）
kubectl label ns prod istio.io/dataplane-mode=ambient
```

## B2. 金丝雀发布的三种做法？

```yaml
# ① 原生 VirtualService 权重
apiVersion: networking.istio.io/v1
kind: VirtualService
metadata: {name: reviews, namespace: prod}
spec:
  hosts: [reviews]
  http:
    - route:
        - destination: {host: reviews, subset: v1}
          weight: 90
        - destination: {host: reviews, subset: v2}
          weight: 10
```

| 方式 | 特点 |
| --- | --- |
| 手动改权重 | 简单，但没有自动化判断 |
| **Flagger / Argo Rollouts** | 基于指标自动推进/回滚（**生产推荐**） |
| 基于请求头/用户 | 用于"内部先试"（见 A9 示例） |

**必答**：金丝雀要配**一致哈希或会话保持**（避免同一用户在 v1/v2 之间跳），并且要有**自动回滚阈值**。

## B3. 流量镜像（Shadow）怎么用？

```yaml
http:
  - route:
      - destination: {host: reviews, subset: v1}
    mirror:
      host: reviews
      subset: v2
    mirrorPercentage: {value: 10.0}
```

**必答**：镜像请求的**响应被丢弃**，所以它适合验证新版本是否能处理真实流量；但**有写操作的服务要谨慎**（可能造成重复写入）⇒ 通常要求影子服务"只接受不落库"或使用独立存储。

## B4. 熔断与限流：为什么"本地限流"不精确？

| 类型 | 机制 | 精度 |
| --- | --- | --- |
| **本地限流（Envoy local rate limit / token bucket）** | 每个 proxy 各自计数 | 实际配额 ≈ 单实例配额 × 副本数，**不精确** |
| **全局限流（Rate Limit Service）** | 外部 RLS + Redis 计数 | 精确，但多一跳且是**新的依赖与故障点** |
| **熔断（outlierDetection / 连接池）** | 按上游健康度摘除 | 保护调用方 |

**实践建议**：本地限流用于"防打爆自己"的兜底；全局限流用于租户配额；**两者都要有"限流生效"的可观测**（否则出问题不知道是被限流还是服务挂了）。

## B5. JWT + 授权的完整闭环（常见漏配）

```yaml
# ① 认证：验证 JWT（不强制要求）
apiVersion: security.istio.io/v1
kind: RequestAuthentication
metadata: {name: jwt, namespace: prod}
spec:
  selector: {matchLabels: {app: api}}
  jwtRules:
    - issuer: "https://issuer.example.com"
      jwksUri: "https://issuer.example.com/.well-known/jwks.json"
---
# ② 授权：强制必须带有效 JWT（关键！只配 ① 不会拒绝无 token 请求）
apiVersion: security.istio.io/v1
kind: AuthorizationPolicy
metadata: {name: require-jwt, namespace: prod}
spec:
  selector: {matchLabels: {app: api}}
  action: ALLOW
  rules:
    - from:
        - source: {requestPrincipals: ["https://issuer.example.com/*"]}
```

## B6. mTLS 从 PERMISSIVE 迁移到 STRICT 的标准步骤（落地大题）

```
① 全网格设 PERMISSIVE（同时接受明文与 mTLS）
② 用指标确认"明文流量已经消失"：
      istio_requests_total 的 connection_security_policy="mutual_tls" 占比 → 100%
③ 按命名空间逐个切 STRICT（先非关键、再核心）
④ 每次切换后观察 503/失败率
⑤ 全网格 STRICT + 保留"例外"（如需要明文健康检查的来源）
```

```promql
# ② 的证据：非 mTLS 请求占比
sum(rate(istio_requests_total{connection_security_policy!="mutual_tls"}[5m]))
  / clamp_min(sum(rate(istio_requests_total[5m])), 0.001)
```

**必答的坑**：

- 某些**外部健康检查/监控**走明文 ⇒ 切 STRICT 后失败 ⇒ 需要 `PeerAuthentication` 的 `portLevelMtls` 例外，或让探测端也进网格；
- **非网格客户端**（集群外直连 Pod IP）在 STRICT 下会被拒；
- DR 的 `trafficPolicy.tls.mode` 与 PeerAuthentication 冲突时会 503。

## B7. Egress 治理：三种策略怎么选？

| 方案 | 说明 |
| --- | --- |
| **PASSTHROUGH（默认）** | 允许所有出站，但**无法审计与策略** |
| **ServiceEntry + 策略** | 把外部服务纳入网格：可命名、可策略、可遥测 |
| **Egress Gateway** | 统一出口 IP、集中 TLS/审计/限流（合规场景必需） |

```yaml
apiVersion: networking.istio.io/v1
kind: ServiceEntry
metadata: {name: external-api, namespace: prod}
spec:
  hosts: ["api.external.com"]
  location: MESH_EXTERNAL
  ports: [{number: 443, name: https, protocol: HTTPS}]
  resolution: DNS
```

**配套**：`outboundTrafficPolicy.mode = REGISTRY_ONLY`（只允许已注册服务出站）⇒ 防止"偷偷访问外部"，但需要先把所有合法外部依赖注册全（**这是最常踩的坑**）。

## B8. Gateway API vs Istio API：未来怎么选？

| 维度 | Istio API（Gateway + VirtualService） | **Kubernetes Gateway API**（Gateway + HTTPRoute） |
| --- | --- | --- |
| 定位 | Istio 专有 | **K8s 标准**，跨实现可移植 |
| 表达能力 | 很强（含 subset、DestinationRule） | 标准化，部分高级能力靠实现扩展 |
| 团队分工 | 单一团队 | **角色划分**（基础设施/集群运维/应用开发） |
| 演进 | 仍支持 | **社区推荐方向**（支持程度以版本为准） |

**建议**：新集群入口优先 **Gateway API**，东西向与高级流量治理仍用 VirtualService/DestinationRule；迁移要按服务逐个验证（**不要一次性切换**）。

## B9. 网格内的"入口"选型：Ingress Gateway vs Ingress Controller？

| 方案 | 适用 |
| --- | --- |
| **Istio Ingress Gateway** | 已经用网格、希望入口与网格策略/遥测一致 |
| **独立 Ingress/Nginx + 网格** | 入口团队与网格团队分离；或入口需要大量 Nginx 特性 |
| **Gateway API 实现** | 想统一南北向标准 |

**必答**：入口是**单点故障源**，不管选哪个都要：多副本跨区 + PDB + 优雅终止 + 容量压测 + 独立告警。

## B10. Sidecar 资源与启动顺序调优

```yaml
metadata:
  annotations:
    proxy.istio.io/config: |
      holdApplicationUntilProxyStarts: true
      concurrency: 2                      # Envoy worker 数（按 CPU 限额实测调整）
      proxyStatsMatcher:                  # 裁剪统计，降低 CPU/内存
        inclusionRegexps: [".*upstream_rq_total.*", ".*circuit_breakers.*"]
    sidecar.istio.io/proxyCPU: "500m"
    sidecar.istio.io/proxyCPULimit: "2"
    sidecar.istio.io/proxyMemory: "256Mi"
    sidecar.istio.io/proxyMemoryLimit: "1Gi"
```

**必答**：

- `holdApplicationUntilProxyStarts` 让**应用容器等 sidecar 就绪**，避免早期请求丢失；
- Envoy 的 **CPU/内存随"可见服务数"增长** ⇒ 资源调优必须配合 A7 的配置收敛，否则加资源只是治标；
- `proxyStatsMatcher` 裁剪统计能显著降低 Envoy 内存（**大规模必备**）。

## B11. 与 spring/SDK 治理共存时的注意点

- **重试叠加**：SDK 重试 × 网格重试 ⇒ 放大倍数相乘 ⇒ 必须统一约定（通常保留网格重试，SDK 关掉或只保留少量）；
- **超时叠加**：外层超时 \< 内层重试总时长 ⇒ 白重试；
- **服务发现冲突**：SDK 直连 IP + 网格路由 ⇒ 绕过策略与遥测 ⇒ 应统一走 Service 名。

## B12. 网格与 CNI 的配合（与网络篇联动）

- Istio 依赖 CNI 提供 **Pod IP 可达**；Cilium 的 eBPF/kube-proxy replacement 与 sidecar 的 iptables 拦截可以共存，但**必须逐项验证**；
- **Ambient/ztunnel 与 CNI 的耦合更紧**（依赖节点级转发），选型时要看官方兼容矩阵；
- NetworkPolicy 需要放行：**istiod（15010/15012/15014/15020）、DNS、以及应用端口**——**忘了放行 istiod 是新 namespace 启用网格后 Pod 起不来的头号原因**。

## B13. 常见"网格引入后变慢/失败"的现象归因

| 现象 | 原因 |
| --- | --- |
| Pod 启动变慢 | sidecar 拉配置 + `holdApplicationUntilProxyStarts`；或 istiod 压力大 |
| 探针失败 | 探针流量被拦截/未放行；或 sidecar 未就绪 |
| P99 增加 | 每跳多两次用户态代理；连接池不足；HTTP/1.1 未升级 HTTP/2 |
| 偶发 503 | mTLS 冲突、熔断摘除、Endpoint 变更瞬时空窗 |
| 内存暴涨 | 配置全量下发 + 统计未裁剪 |
| 控制面 NACK | 配置错误（DR/VS 冲突） |

## B14. 网格的"渐进落地路线"（架构题）

```
阶段 1：只做可观测（注入 + 遥测，不开 mTLS 严格模式）
阶段 2：mTLS PERMISSIVE → 逐 ns STRICT（拿到身份）
阶段 3：超时/重试/熔断统一到网格（去掉 SDK 重复实现）
阶段 4：金丝雀与流量镜像常态化（配合 Flagger/Argo）
阶段 5：授权（AuthorizationPolicy）按服务逐步开启
阶段 6：按需引入 Ambient 降低开销 / 或按需 waypoint
```

> **关键原则**：**每一步都要有可回滚开关**（label 控制注入、CR 控制策略），不要一次性全开。

---

# Part C · 大规模集群网格优化篇（12 题）

## C1. 控制面推送风暴：为什么会"一个变更打满 istiod"

**机制**：Service/Endpoint/VS/DR 变更 ⇒ istiod 重新计算 ⇒ 向**所有相关 proxy** 推送。服务数与 sidecar 数都很大时，一次"热门服务"变更可能触发成千上万次推送 ⇒ CPU 打满、推送延迟、配置滞后。

**治理**：

1. **Sidecar CR 收敛可见性**（最关键，见 A7）；
2. `discoverySelectors` 缩小发现范围；
3. `exportTo` 限制 CR 可见；
4. 控制 Endpoint 抖动频率（Pod 频繁重建会让 EDS 不断变更）；
5. 关注推送相关指标与 NACK（见 C10 告警）。

```bash
# 直观判断配置规模
istioctl proxy-config cluster <pod> -o json | jq 'length'
istioctl proxy-status                # 看有多少 proxy 处于 Stale/Synced
```

## C2. 数据面内存与 CPU 的主要来源

| 来源 | 治理 |
| --- | --- |
| **可见服务/配置数量** | Sidecar CR + discoverySelectors |
| **统计指标基数** | `proxyStatsMatcher` 裁剪 |
| **连接数**（每个上游连接池） | 调整连接池上限、开启 HTTP/2 复用 |
| **访问日志** | 按需开启 + 采样/裁剪字段 |
| **Envoy worker 数** | 按 CPU 限额设置 `concurrency` |

## C3. 指标基数爆炸：怎么把 Prometheus 救回来

```promql
# 先看基数（最直接的证据）
topk(10, count by (__name__)({__name__=~"istio_.*"}))
# 看时间序列总量增长
sum(scrape_samples_scraped{job=~".*istio.*"})
```

**治理**：

- **Telemetry API 删除高基数维度**（`source_principal`、`request_host`、`response_flags` 等按需保留）；
- Prometheus 侧 `metric_relabel_configs` drop 高基数标签；
- 关闭不必要的 CLIENT 侧指标（保留 SERVER 侧即可覆盖 RED）；
- 按命名空间分级（核心服务保留明细，其它聚合）。

## C4. Endpoint 规模与 EDS 更新

- 大服务（数千 endpoint）的 EDS 更新体积大且频繁；
- EndpointSlice 已在 K8s 侧降低变更成本（见《控制面篇》A6），网格侧要避免"每次 Pod 重建都引发大面积推送"；
- 治理：稳定 Pod（减少不必要的重建）、合理分批、观察 `pilot_xds_*` 推送量。

## C5. 网格对延迟的影响怎么量化？

```
单跳成本 ≈ 出站 sidecar 处理 + 入站 sidecar 处理（两次用户态转发 + 可能的 mTLS 加解密）
量化方法（务必实测，不要用别人的数字）：
  A/B 对比：同一服务在"网格内 vs 直连 PodIP"的 P50/P99
  指标：istio_request_duration_milliseconds（SERVER 侧）+ 业务侧 P99
  控制变量：同样 QPS、同样副本、同样连接池
```

**结论口径**：网格带来的是"**可控的常数级延迟成本 + 可观测与治理收益**"，是否划算由业务 SLO 决定；**延迟极敏感的核心链路要单独压测**。

## C6. Ambient 在大规模下的收益与限制

**收益**：省掉每 Pod sidecar ⇒ 内存/CPU 总量下降明显；升级网格**不需要滚动重启业务 Pod**（这是运维上最大的收益）。

**限制**：L7 能力依赖 waypoint（要按需规划 waypoint 数量与容量）；功能等价性与版本成熟度需逐项验证；**多租户共享 ztunnel 的隔离性要评估**。

## C7. istiod 的 HA 与容量

| 项 | 做法 |
| --- | --- |
| 副本 | 通常 2\~3，跨可用区 |
| PDB | 保证至少 1 可用 |
| 资源 | 按"Sidecar 数 × 配置规模"实测给 requests/limits |
| 依赖 | apiserver 的 watch 压力（可用 discoverySelectors 降低） |
| 监控 | `up{job="istiod"}`、推送错误、NACK、proxy Stale 数量、内存增长 |

## C8. Gateway 的容量与高可用

- **入口/出口 Gateway 是"集中式 Envoy"** ⇒ 必须独立压测与容量规划；
- 多副本跨区 + PDB + 优雅终止（`terminationGracePeriodSeconds` 要够长，Envoy 需要 drain）；
- HPA 基于 CPU/QPS 扩容，并预留"突发"余量；
- **优雅终止是网关事故高发点**：drain 时间不足会导致连接重置（与《Kubelet 篇》Q11 同源）。

## C9. 多集群的运维成本与是否值得

| 收益 | 成本 |
| --- | --- |
| 容灾、就近访问、隔离 | 控制面部署复杂度、跨集群服务发现、信任域、网关与网络要求 |

**建议**：**先明确是否需要跨集群**。很多场景用"多集群 + 上层 DNS/全局负载均衡切换"即可，不必做全互通的单一网格。

## C10. 网格的最小告警集

```promql
# ① istiod 存活与推送
up{job="istiod"} == 0
rate(pilot_xds_push_errors[5m]) > 0
# ② proxy 不同步（Stale）比例（istioctl proxy-status 的指标化版本，指标名以版本为准）
# ③ 5xx 与 mTLS 混合状态
sum(rate(istio_requests_total{response_code=~"5.."}[5m]))
  / clamp_min(sum(rate(istio_requests_total[5m])), 0.001) > 0.01
# ④ 明文流量（迁移 STRICT 前必须归零）
sum(rate(istio_requests_total{connection_security_policy!="mutual_tls"}[5m])) > 0
# ⑤ 熔断/异常点生效（被摘除的上游）
sum(rate(envoy_cluster_outlier_detection_ejections_active[5m])) > 0   # 指标名以版本为准
# ⑥ 网关容量
sum(rate(istio_requests_total{reporter="source"}[5m])) by (destination_service_name)
```

## C11. 网格变更的灰度与回滚（生产治理）

| 变更 | 灰度 | 回滚触发 | 回滚方式 |
| --- | --- | --- | --- |
| **控制面升级** | 新 revision → 单 ns 切换 → 逐步全量 | NACK 上升、503 上升、istiod 内存异常 | 把 ns label 切回旧 revision + 滚动重启 |
| **mTLS 收紧** | 单 ns → 多 ns | 非 mTLS 流量未归零 / 失败率上升 | 改回 PERMISSIVE（**改 CR 即生效，最快**） |
| **授权策略** | 先 DENY 少量、再 ALLOW 细化 | 健康检查失败、业务报错 | 删除/放宽策略 |
| **Telemetry 裁剪** | 单 ns | 指标缺失 | 恢复覆盖配置 |
| **Ambient 启用** | 单 ns/单服务 | 连通性或功能缺失 | 去掉 ns 标签 + 重建 Pod |

> **黄金规则**：**所有网格策略都应该"可一键回退"**——优先用 CR/label 表达，而不是改镜像或改注入模板。

## C12. 给你一个 1000 服务的新集群，怎么设计网格？（综合题）

1. **先问 SLO 与诉求**：要 mTLS？要 L7 授权？要统一金丝雀？能接受多少延迟与资源开销？
2. **形态选择**：先 Ambient 还是直接 sidecar（按 L7 需求与版本成熟度定）；
3. **边界**：入口用 Gateway API + Ingress Gateway；出口按合规决定是否 Egress Gateway；
4. **规模治理**：**Sidecar CR + discoverySelectors + exportTo** 必须在首批服务接入时就落地（事后补很痛）；
5. **可观测**：Telemetry 裁剪 + 采样 + 指标 relabel + 访问日志按需；
6. **安全**：mTLS 迁移路线（PERMISSIVE→STRICT）、授权按服务推进、外部 CA 接入；
7. **HA**：istiod 多副本跨区 + PDB；Gateway 多副本 + 优雅终止 + 压测；
8. **变更治理**：revision 升级、label 灰度、一键回退；
9. **演练**：istiod 挂、Gateway 挂、误配策略、证书过期、推送风暴。

# Part D · 反模式清单（可直接用于生产巡检）

| # | 反模式 | 证据 | 等级 | 修复 | 验证 |
| --- | --- | --- | --- | --- | --- |
| D1 | 大规模下不做配置收敛（全量下发） | Envoy 内存高、istiod 推送量大 | 🔴 高 | Sidecar CR + discoverySelectors + exportTo | cluster 数量与推送量下降（实测） |
| D2 | 直接全网格 STRICT | 明文探测/外部客户端 503 | 🔴 高 | 按 B6 分阶段迁移，先证明明文流量归零 | `connection_security_policy` 全为 mutual\_tls |
| D3 | 只配 RequestAuthentication 以为"加了鉴权" | 无 token 请求仍被放行 | 🔴 高 | 必须配 AuthorizationPolicy 要求 requestPrincipals | 无 token 请求返回 403 |
| D4 | 大量使用 EnvoyFilter | 升级后被破坏 | 🔴 高 | 优先 Telemetry API / 原生 CR；EnvoyFilter 作为最后手段并做升级验证 | 升级演练通过 |
| D5 | 未放行 istiod 端口的 NetworkPolicy | 新 ns 启用网格后 Pod 全挂 | 🔴 高 | 放通 15010/15012/15014/15020 与 DNS | 新 ns 注入后 Pod 正常启动 |
| D6 | 指标全量开启不做裁剪 | Prometheus 基数爆炸、抓取超时 | 🔴 高 | Telemetry 裁剪 + relabel | 时间序列数量与抓取耗时回归 |
| D7 | 重试不设 budget、POST 也重试 | 雪崩放大 | 🔴 高 | retryBudget + 仅幂等接口重试 | 故障注入演练不放大流量 |
| D8 | 未配超时（依赖 Envoy 默认） | 请求无限悬挂 | 🟠 中 | 显式配 timeout/perTryTimeout | 超时行为符合 SLO |
| D9 | 启用 REGISTRY\_ONLY 但未注册全部外部依赖 | 外部调用突然全失败 | 🔴 高 | 先梳理依赖并建 ServiceEntry，再收紧 | 依赖清单对账完成 |
| D10 | sidecar 资源一刀切（默认不改） | 大 ns 的 sidecar OOM/CPU 打满 | 🟠 中 | 按可见规模分级设置资源 + 裁剪统计 | 内存回到基线 |
| D11 | 网关 drain 时间不足 | 发布/缩容时连接重置 | 🟠 中 | 放大 grace + 配置 drain 相关参数（版本相关） | 滚动网关期间无连接重置 |
| D12 | 网格与 SDK 重试/超时叠加 | 放大倍数相乘 | 🟠 中 | 统一约定，关闭一侧 | 压测确认放大倍数 |
| D13 | 端口协议未显式声明 | L7 规则"配了不生效" | 🟠 中 | 用 `appProtocol`/端口命名 | config\_dump 中 listener 为 HTTP 类型 |
| D14 | istiod 单副本/无 PDB | istiod 维护期新 Pod 起不来 | 🔴 高 | 多副本跨区 + PDB + 资源保障 | 杀一个副本不影响新 Pod 启动 |
| D15 | 一次性把全部服务接入网格 | 问题集中爆发且难回退 | 🟠 中 | 按 ns/服务分批，label 可回退 | 每批次有验证窗口与回滚记录 |

---

# Part E · 可观测性：指标、SLO 与告警

## E1. 核心指标清单（**请先 **`kubectl exec ... -c istio-proxy -- curl :15020/stats/prometheus`** 核对**）

| 指标 | 含义 | 用途 |
| --- | --- | --- |
| `istio_requests_total` | 请求数（含 reporter/source/destination/响应码/响应标志/mTLS 状态等标签） | RED 的 Rate/Errors |
| `istio_request_duration_milliseconds` | 请求延迟直方图 | RED 的 Duration |
| `istio_request_bytes` / `istio_response_bytes` | 流量字节 | 容量 |
| `istio_tcp_*` | TCP 层指标 | 非 HTTP 服务 |
| `pilot_xds_pushes` / `_push_errors` / NACK 系列 | 控制面推送 | 控制面健康 |
| `istio_agent_...` / `envoy_*` | sidecar/Envoy 自身状态 | 数据面健康 |
| 控制面/数据面进程指标 | `process_*`、`container_memory_*` | 容量 |

## E2. 建议的 SLO（**初始建议值，需实测校准**）

| SLI | 建议目标 |
| --- | --- |
| 网格内服务成功率（非 5xx） | ≥ 99.9% |
| 网格引入的延迟增量 P99 | 明确预算（如 ≤ 10ms/跳，**必须 A/B 实测**） |
| mTLS 覆盖率（迁移完成后） | 100% `mutual_tls` |
| istiod 可用性 | ≥ 99.95%（多副本 + PDB） |
| 配置同步延迟（变更 → proxy 生效）P99 | ≤ 30s（需实测） |
| 控制面 NACK | = 0（\>0 即告警） |

## E3. Prometheus 告警规则（可直接粘贴，阈值需实测校准）

```yaml
groups:
  - name: istio
    rules:
      - alert: IstiodDown
        expr: up{job="istiod"} == 0
        for: 2m
        labels: {severity: critical}
        annotations: {summary: "istiod 不可用：新 Pod 无法获取配置与证书"}

      - alert: IstioXdsPushErrors
        expr: rate(pilot_xds_push_errors[5m]) > 0
        for: 10m
        labels: {severity: warning}
        annotations: {summary: "istiod 推送出现错误，检查配置合法性"}

      - alert: IstioMeshErrorRateHigh
        expr: |
          sum(rate(istio_requests_total{response_code=~"5.."}[5m]))
            / clamp_min(sum(rate(istio_requests_total[5m])), 0.001) > 0.01
        for: 10m
        labels: {severity: critical}
        annotations: {summary: "网格内 5xx 比例 {{ $value | humanizePercentage }}"}

      - alert: IstioPlaintextTraffic
        expr: sum(rate(istio_requests_total{connection_security_policy!="mutual_tls"}[5m])) > 0
        for: 30m
        labels: {severity: info}
        annotations: {summary: "仍存在明文流量，STRICT 迁移前置检查"}

      - alert: IstioResponseFlagAbnormal
        expr: |
          sum(rate(istio_requests_total{response_flags!="-"}[5m])) > 0
        for: 10m
        labels: {severity: warning}
        annotations: {summary: "出现响应标志异常（UF/URX/DC 等），检查上游连接与熔断"}

      - alert: IstioScrapeSlow
        expr: |
          histogram_quantile(0.99,
            sum by (le) (rate(scrape_duration_seconds_bucket{job=~".*istio.*"}[5m]))) > 5
        for: 15m
        labels: {severity: warning}
        annotations: {summary: "istio 相关抓取耗时过高，通常意味着指标基数需要裁剪"}

      - alert: IstioGatewayHighLatency
        expr: |
          histogram_quantile(0.99,
            sum by (le) (rate(istio_request_duration_milliseconds_bucket{reporter="destination"}[5m]))) > 1000
        for: 10m
        labels: {severity: warning}
        annotations: {summary: "网格请求 P99 超过 1s"}
```

> 抓取耗时是最实用的"基数爆炸"领先指标：抓取变慢通常先于 Prometheus 内存问题出现。

## E4. 排障工具箱（命令速查）

```bash
istioctl analyze -n <ns>                       # 配置静态检查（上线前必跑）
istioctl proxy-status                          # 各 proxy 是否同步（Stale 要查）
istioctl proxy-config cluster <pod> -o json    # 上游
istioctl proxy-config route <pod> -o json      # 路由是否包含目标
istioctl proxy-config listener <pod>           # 端口与协议
istioctl proxy-config endpoint <pod> --port 8080
kubectl exec <pod> -c istio-proxy -- curl -s localhost:15000/config_dump
kubectl exec <pod> -c istio-proxy -- curl -s localhost:15000/stats | grep -E "upstream_rq|outlier|ssl"
kubectl logs <pod> -c istio-proxy --tail=200
kubectl logs -n istio-system deploy/istiod --tail=200
```

---

# Part F · 演练（GameDay）计划表

| 场景 | 注入方式 | 期望表现 | 验证的 SLO |
| --- | --- | --- | --- |
| istiod 全部挂掉 | 缩到 0 副本 | 存量通信不受影响；**新 Pod 起不来**且告警及时 | 控制面可用性 + MTTR |
| istiod 单副本挂 | 杀一个 Pod | 无感知；新 Pod 仍可获取配置 | HA 有效性 |
| 网关副本滚动 | 滚动更新 gateway | 无连接重置（drain 生效） | 优雅终止 |
| 误配策略 | 下发错误 AuthorizationPolicy | 立刻看到失败 + 可秒级删除恢复 | 变更安全 |
| mTLS 收紧 | 某 ns 切 STRICT | 明文流量先归零；切换后无 503 | 迁移流程 |
| 配置推送风暴 | 批量重建大量 Pod | 推送指标可控，无 NACK | 控制面容量 |
| 指标基数爆炸 | 放开全部维度 | 告警/抓取耗时可发现 | 可观测治理 |
| 证书异常 | 阻断证书轮换 | 新 Pod 失败可定位到 SDS | 证书链路 |
| Envoy OOM | 压测大配置 ns | 内存增长有告警与兜底 | 数据面容量 |
| 网格升级 | 新 revision + 单 ns 切换 | 逐 ns 验证通过后再放量 | 升级安全性 |

---

# Part G · 10 张面试速记卡

1. **Istio = 身份为中心的东西向流量与策略平面**；不负责 Pod 连通性（CNI）、不负责服务发现本身、不替代 NetworkPolicy。
2. **架构**：istiod（Pilot + CA）+ Envoy 数据面；Mixer 已移除，遥测靠 Envoy 统计 + Telemetry API。
3. **端口**：15001 出站 / 15006 入站 / 15000 admin / 15020 聚合监控 / 15021 就绪 / 15012 xDS(mTLS) / 15008 HBONE(Ambient)。
4. **注入**：mutating webhook 注入 `istio-init`（iptables，需 NET\_ADMIN）或 **istio-cni**（免特权）+ `istio-proxy`；`holdApplicationUntilProxyStarts` 保证顺序。
5. **大规模头号成本 = 全量配置下发** ⇒ **Sidecar CR + discoverySelectors + exportTo** 三件套收敛。
6. **CR 分工**：Gateway 开门、VirtualService 路由、DestinationRule 目标策略与 subset、ServiceEntry 网格外服务、Sidecar 收敛可见性、**EnvoyFilter 是最后手段**。
7. **安全三件套**：SPIFFE 身份 + PeerAuthentication(mTLS) + AuthorizationPolicy；**RequestAuthentication 只验证不强制**，必须配授权才拒绝无 token。
8. **mTLS 迁移**：PERMISSIVE → 证明明文归零 → 逐 ns STRICT；保留 portLevel 例外应对明文探测。
9. **Ambient**：ztunnel(L4 节点级) + waypoint(L7 按需) + HBONE；最大收益是**升级网格不用重启业务 Pod**，限制是功能等价性与成熟度要逐项验证。
10. **可观测成本**：`istio_requests_total` 基数易爆炸 ⇒ Telemetry 裁剪 + 采样 + Prometheus relabel；**所有网格策略都要能一键回退**（CR/label 表达优先）。

---

## 附：文档自查清单

- [ ] 每条建议配了可直接套用的 YAML / 注解 / 命令 / PromQL
- [ ] 所有版本号、alpha/beta/GA 状态、默认值（证书 TTL、端口、资源）均标注"以所用版本为准"
- [ ] 未提供任何未经实测的性能数字（延迟增量等）⇒ 改为**A/B 测量方法**
- [ ] 反模式清单含：证据 / 等级 / 修复 / 验证
- [ ] SLO 标为建议初值
- [ ] 变更（升级 / 策略收紧 / Ambient 启用）给出灰度步骤、回滚触发与"一键回退"方式
- [ ] 与《网络 / CNI》《控制面》《Kubelet》《排障 60 例》做了显式交叉引用
