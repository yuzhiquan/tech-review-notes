# k8s-controlplane-interview-qa

# Kubernetes 控制面面试题全解析：apiserver × etcd × controller-manager × kube-proxy

> 适用角色：SRE / 云原生运维 / 平台工程师面试（社招 3 年起 + 偏高阶）
> 姊妹篇：《Scheduler 篇》《Kubelet 篇》《网络 / CNI 篇》
> 版本约定：以 **1.28 \~**** 1.31 主线**为准；凡涉及默认值、feature gate、GA/Beta 状态、flag 名称的地方，一律标注"以所用版本 `--help` 与官方 Release Notes 为准"。

---

## 0. 使用说明

### 0.1 控制面四件套的职责速览

| 组件 | 一句话职责 | 是否有状态 | 是否选主 |
| --- | --- | --- | --- |
| **kube-apiserver** | 集群唯一的 API 入口：认证/授权/准入/校验/审计 + 读写 etcd | **无状态**（可水平扩容） | 否 |
| **etcd** | 唯一的持久化存储（所有对象的唯一真相源） | **有状态** | 是（Raft leader） |
| **kube-controller-manager** | 跑各种控制器，让"实际状态"收敛到"期望状态" | 基本无状态（依赖 apiserver） | **是**（Lease） |
| **kube-scheduler** | 给未绑定 Pod 选节点 | 无状态 | 是（Lease） |
| **kube-proxy** | 节点数据面：把 Service/EndpointSlice 翻译成转发规则 | 无状态 | 否 |
| **cloud-controller-manager** | 云厂商相关逻辑（节点生命周期/LB/路由） | 无状态 | 是（通常） |

### 0.2 常用端口清单（面试高频，容易混）

| 组件 | 端口 | 用途 |
| --- | --- | --- |
| kube-apiserver | **6443**（默认 secure port） | API + `/healthz` `/readyz` `/livez` `/metrics` |
| kubelet | **10250** API、**10248** healthz、10255 read-only（应关闭） | 见 Kubelet 篇 |
| kube-scheduler | 10259 | healthz + metrics（HTTPS） |
| kube-controller-manager | 10257 | healthz + metrics（HTTPS） |
| kube-proxy | **10249** metrics、**10256** healthz | 端口可能因版本/参数而变，请核对 |
| etcd | 2379 client、**2380 peer** | 客户端 / 集群内部通信 |
| kube-controller-manager | 10252 / kube-scheduler 10251 | 已废弃的 insecure 端口 |

> **提示**：端口表最适合在面试开始时主动交代，能立刻体现"干过"。

### 0.3 数值约定

- **源码/官方默认值**：标注字段，提示现场核对；
- **建议值**：标为 *建议*，必须压测校准；
- **实测值**：本文档不提供任何未经实测的性能数字，一律给出**测量方法**。

---

# Part A · kube-apiserver 原理篇（15 题）

## A1. 一次 kubectl 请求，在 apiserver 里到底走过哪些关卡？

> kubectl apply → 走的是：**认证 → 审计 → 模拟 → 授权 → 准入 → 校验 → 存储(etcd)**。

```
kubectl apply pod.yaml
  │
  ▼ ① HTTP/TLS 建立，APF（优先级与公平性）先决定是否接纳/排队
  ▼ ② RequestInfo 解析（verb/resource/subresource/namespace/group/version）
  ▼ ③ MaxInFlight 限流 + 非长连接请求超时
  ▼ ④ CORS
  ▼ ⑤ Authentication（认证：你是谁）
        X509 客户端证书 / ServiceAccount Token / OIDC / Webhook / TokenRequest / RequestHeader(aggregation)
  ▼ ⑥ Audit（记录审计日志；未通过认证也会记录失败）
  ▼ ⑦ Impersonation（模拟：有 impersonate 权限时可伪装成别人）
  ▼ ⑧ Authorization（授权：你能不能做）
        RBAC / Node / Webhook / AlwaysAllow(默认禁止用于生产开放)
  ▼ ⑨ Admission（准入：改内容 + 放不放行）
        MutatingAdmissionWebhook → 校验 → ValidatingAdmissionWebhook
        内置：NamespaceLifecycle, LimitRanger, ServiceAccount, ResourceQuota,
              NodeRestriction, PodSecurity, Priority, DefaultStorageClass,
              DefaultTolerationSeconds, TaintNodesByCondition, StorageObjectInUseProtection,
              CertificateApproval/Signing/SubjectRestriction, EventRateLimit, OwnerReferencesPermissionEnforcement...
  ▼ ⑩ Validation（字段级校验 + OpenAPI schema；CRD 走 structural schema + pruning）
  ▼ ⑪ REST Storage → etcd（write 走 MVCC + resourceVersion 乐观锁）
  ▼ ⑫ 返回响应；审计记录结束
```

> **必答**：这条 handler chain 的构造在 apiserver 源码的 `DefaultBuildHandlerChain` 里，**具体包裹顺序请对照所用版本源码**（这块每年都微调）。面试时说出"认证→审计→授权→准入→存储"的主线 + 提到"审计在授权之前，所以被拒绝的请求也会被记录"就很专业。

## A2. 认证有哪些方式？生产上怎么配？

| 方式 | 适用 | 备注 |
| --- | --- | --- |
| **X509 客户端证书** | 管控组件（scheduler/controller-manager/kubelet）、管理员 | CN=用户名，O=组 |
| **ServiceAccount Token**（推荐 **Bound/Projected Token**） | Pod 内访问 API | 老版 Secret token **永不过期且有泄露风险**，优先用 TokenRequest 短期 token |
| **OIDC** | 企业身份打通 | 配合 `--oidc-*` 参数 |
| **Webhook Token** | 外部认证系统 | 注意超时与失败即拒绝 |
| **Bootstrap Token** | 节点加入集群 | 仅用于 bootstrap 阶段 |
| **RequestHeader（认证代理）** | 聚合层/统一网关 | 需要正确配置 `--requestheader-*`，否则有身份伪造风险 |

**必答要点**：

- 多种 authenticator 是"**或**"关系，任一通过即成功，**都不通过则看是否允许匿名**（生产应关闭 anonymous 对敏感资源的访问）；
- SA token 的 audience 绑定与短期化，能显著降低凭据泄露的危害半径。

## A3. RBAC 的四件套和常见坑？

Role / ClusterRole / RoleBinding / ClusterRoleBinding。

```yaml
# 最小权限的只读举例（不要动不动 cluster-admin）
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: pod-reader
rules:
  - apiGroups: [""]
    resources: ["pods", "pods/log"]
    verbs: ["get", "list", "watch"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: read-pods
  namespace: prod
subjects:
  - kind: ServiceAccount
    name: app
    namespace: prod
roleRef:
  kind: ClusterRole
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io
```

**坑清单（必答）**：

1. `system:masters`** 组等于超级用户**，不要给普通账号塞进这个组；
2. `verbs: ["*"]` / `resources: ["*"]` 用多了等于没做 RBAC；
3. **secrets 的 get/list** 要格外小心（list secrets ≈ 拿到该 ns 全部凭据）；
4. RoleBinding 可以引用 ClusterRole 但**权限被限制在本 namespace**（这是常见误解）；
5. 内置 `admin/edit/view` ClusterRole 用 `aggregationRule` 自动聚合 CRD 权限。

## A4. 准入控制（Admission）和普通校验的区别？

| 维度 | Admission | Validation（字段校验） |
| --- | --- | --- |
| 时机 | 校验**之前**（mutating 会改对象） | 准入之后、写库之前 |
| 能否修改对象 | Mutating 可以（如注入 sidecar、给默认值） | 不可以 |
| 扩展性 | Admission Webhook / 内置插件 | OpenAPI schema / CEL 校验（CRD） |
| 典型用途 | PodSecurity、ResourceQuota、LimitRanger、默认 StorageClass、自动 sidecar | 字段合法性、枚举、结构裁剪 |

**Webhook 的生产风险（必答）**：

- `failurePolicy: Fail` ⇒ webhook 挂了会**阻断全集群变更操作**（严重时连 Pod 都创建不了）。缓解手段：给 webhook 加 `namespaceSelector` 缩小作用域 + 排除 kube-system + 多副本跨故障域 + 设置短超时。
- `sideEffects` / `reinvocationPolicy` / `matchPolicy` 都要理解清楚。

```yaml
apiVersion: admissionregistration.k8s.io/v1
kind: ValidatingWebhookConfiguration
metadata:
  name: my-policy
webhooks:
  - name: policy.example.com
    admissionReviewVersions: ["v1"]
    sideEffects: None
    failurePolicy: Fail            # 想清楚：Fail 有全局风险
    timeoutSeconds: 3              # 建议：短超时，别拖垮 apiserver
    namespaceSelector:
      matchExpressions:
        - key: kubernetes.io/metadata.name
          operator: NotIn
          values: ["kube-system"]  # 关键：给系统命名空间留后门
    rules:
      - operations: ["CREATE", "UPDATE"]
        apiGroups: ["apps"]
        apiVersions: ["v1"]
        resources: ["deployments"]
    clientConfig:
      service:
        name: policy-webhook
        namespace: policy-system
        path: /validate
```

## A5. 审计日志怎么配置？有什么副作用？

四个 level：

| Level | 记录内容 | 成本 |
| --- | --- | --- |
| `None` | 不记录 | 无 |
| `Metadata` | 请求元数据（谁、何时、什么资源、什么动作） | 低 |
| `Request` | 元数据 + 请求体 | 中 |
| `RequestResponse` | 元数据 + 请求体 + 响应体 | **高**（切勿对 Secrets 全面开启） |

Stage：`RequestReceived` / `ResponseStarted`（长连接 watch 才有）/ `ResponseComplete` / `Panic`。

```yaml
# audit-policy.yaml（示意）
apiVersion: audit.k8s.io/v1
kind: Policy
rules:
  - level: Metadata
    verbs: ["get", "list", "watch"]
    resources:
      - group: ""
        resources: ["*"]
  - level: RequestResponse
    verbs: ["create", "update", "patch", "delete"]
    resources:
      - group: ""
        resources: ["pods", "services"]
  - level: Request
    verbs: ["create", "update", "delete", "patch"]
    resources:
      - group: "rbac.authorization.k8s.io"
  - level: None
    resources:
      - group: ""
        resources: ["events"]        # Event 量极大，通常直接关
  - level: Request
    verbs: ["get", "list", "watch"]
    resources:
      - group: ""
        resources: ["secrets"]       # 特别注意合规与体量
omitStages:
  - RequestReceived
```

**结论（必答）**：审计是"合规刚需 + 性能杀手"，必须**按需裁剪 + 落盘/离线投递**，并监控审计后端的写入失败（后端打满时会阻塞请求）。

## A6. resourceVersion、watch 与乐观锁是怎么回事？（控制器基础）

- **etcd 是 MVCC 存储**，每次写都会推进全局 revision；对象的 `metadata.resourceVersion` 就是当时的 revision。
- **乐观并发**：更新对象时必须带上 resourceVersion，若期间被别人改过 ⇒ **409 Conflict**，客户端要**重新 Get 再重试**（这就是 client-go/控制器里的重试逻辑源泉）。
- **watch 语义**：从某个 resourceVersion 开始接收增量事件；如果请求的版本已被压缩 ⇒ 返回 **410 Gone / "too old resource version"** ⇒ 客户端必须 **relist**。

**list 的一致性差异（高频坑）**：

| 请求 | 行为 |
| --- | --- |
| 不指定 resourceVersion | **quorum read**：直读 etcd，一致性最强，**代价最高** |
| `resourceVersion=0` | 从 apiserver 的 **watch cache** 读，**可能略微过期**，但非常快、对 etcd 压力小 |

```promql
# watch cache 是否命中 / 是否频繁从头同步（按各自集群核对指标名）
histogram_quantile(0.99, sum by (resource, le) (rate(apiserver_cache_list_duration_seconds_bucket[5m])))
histogram_quantile(0.99, sum by (resource, le) (rate(apiserver_cache_watch_duration_seconds_bucket[5m])))
```

## A7. APF（API Priority and Fairness）解决什么问题？

**背景**：早期只有 `--max-requests-inflight` 这种"总量/变重量"限流，一旦某个客户端（比如写一个死循环 list 的 controller）把限额占满，**其它所有组件（含 scheduler/kubelet）全部被饿死**。

**APF 的做法**：把请求按 **FlowSchema**（谁/什么请求）分到 **PriorityLevel**（优先级），每个优先级有自己的**并发份额（seats）与队列**，用类似公平排队的算法分配资源 ⇒ **关键组件不会被吵闹邻居饿死**。

```yaml
# 给"控制面关键组件"单独保障（示意，字段请按版本核对）
apiVersion: flowcontrol.apiserver.k8s.io/v1
kind: PriorityLevelConfiguration
metadata:
  name: leader-election
spec:
  type: Limited
  limited:
    assuredConcurrencyShares: 10
    limitResponse:
      type: Queue
      queuing:
        queueLengthLimit: 50
        handSize: 5
        queues: 16
---
apiVersion: flowcontrol.apiserver.k8s.io/v1
kind: FlowSchema
metadata:
  name: system-leader-election
spec:
  priorityLevelConfiguration:
    name: leader-election
  matchingPrecedence: 100
  distinguisherMethod:
    type: ByUser
  rules:
    - resourceRules:
        - apiGroups: ["coordination.k8s.io"]
          resources: ["leases"]
          verbs: ["get", "update"]
      subjects:
        - kind: Group
          group:
            name: system:masters
```

**生产实践**：至少给 **kubelet / scheduler / controller-manager / leader election / 系统探针** 保留专用优先级；并对"全集群 list 类请求"单独降优先级限流。

```promql
# APF 排队与拒绝情况
sum by (priority_level) (apiserver_flowcontrol_current_inqueue_requests)
sum by (priority_level) (rate(apiserver_flowcontrol_rejected_requests_total[5m]))
sum(rate(apiserver_flowcontrol_dispatched_requests_total[5m]))
```

## A8. apiserver 是无状态的吗？那它怎么扩容和高可用？

**是的，apiserver 无状态**（状态全在 etcd）：

- 可以跑 N 个副本，前面挂负载均衡；
- 不需要 leader election；
- **滚动升级/重启一个副本**不会影响其它副本（但要保证 LB 健康检查把它摘出去）。

**高可用要点**：

| 要点 | 说明 |
| --- | --- |
| LB 健康检查 | 用 `/readyz`（而不是 `/healthz`），并且要理解 `--shutdown-delay-duration` 之类的优雅退出机制 |
| 副本跨故障域 | 至少跨 3 个可用区/机架 |
| **watch fan-out** | 副本越多，watch 连接分散越均匀；但 etcd 侧 watch 数是 "副本数 × watch 数" |
| etcd 是真正的瓶颈 | apiserver 扩容解决不了 etcd 慢 |

## A9. watch cache 是什么？为什么要按资源调？

apiserver 在每个实例内维护了 etcd 的 **watch cache**（资源的内存视图），服务 List/Watch 请求，从而**避免每个 list 都打到 etcd**。

**调优点**：不同类型资源的热点差别很大：

```bash
# 给热点资源更大的缓存（示意；默认值与 hot wallet 以版本为准）
--watch-cache-sizes=pods#1000,endpointslices#1000,configmaps#500,secrets#500,nodes#500
--default-watch-cache-size=100
```

**风险**：watch cache 太大会吃内存；太小会频繁回退到 etcd 造成延迟抖动。**一定要结合 **`apiserver_cache_*`** 指标与内存实测来调**（各资源的默认值请按版本核对）。

## A10. Event 洪水怎么办？

Event 是"高频创建 + 高频覆盖"的对象，历史上常把 etcd 打爆。

**治理**：

1. **缩短 Event 保留**：`--event-ttl`（默认通常 1h，**请核对你的版本**）；
2. **EventRateLimit 准入插件**限速（按 namespace/user/server 维度）；
3. 把 events 拆到**独立 etcd 集群**：`--etcd-servers-overrides=/events#https://etcd-events:2379`（大集群常用手段）；
4. 治理根因：别让控制器疯狂写 Event（如 Pending Pod 洪水）。

```promql
sum(rate(apiserver_request_total{resource="events",verb=~"create|patch|update"}[5m]))
```

## A11. Secret 静态加密怎么做？

```yaml
# encryption-config.yaml（示意，请按版本核对 apiVersion 与 provider 语法）
apiVersion: apiserver.config.k8s.io/v1
kind: EncryptionConfiguration
resources:
  - resources: ["secrets"]
    providers:
      - kms:
          name: my-kms-plugin
          endpoint: unix:///var/run/kmsplugin/socket.sock
          cachesize: 1000
          timeout: 3s
      - aescbc:
          keys:
            - name: key1
              secret: <base64-32bytes>
      - identity: {}      # 兜底：没有 key 时明文
```

```bash
kube-apiserver --encryption-provider-config=/etc/kubernetes/encryption-config.yaml
```

**必答**：

- Secret 落盘加密不等于运行时安全，**RBAC 才是第一道防线**；
- 密钥轮换要走"双 key 并存 → 重新写一遍（rewrite）—— `kubectl get secrets --all-namespaces -o json | kubectl replace -f -` → 移除旧 key"的流程，且必须在维护窗口执行并先备份；
- KMS provider 挂了会直接导致 apiserver 无法读取 secrets（**强依赖风险**）。

## A12. 聚合层（Aggregated API / APIService）是干什么的？

让第三方提供的 API（如 metrics-server 的 `metrics.k8s.io`、自定义 AA）\*\* kubectl 起来像原生资源\*\*。

```
kubectl get --raw /apis/metrics.k8s.io/v1beta1/nodes
   → apiserver 的 kube-aggregator 依据 APIService 对象代理到后端 Service
```

**必答**：

- 聚合层依赖 **RequestHeader 认证**，配置不当会有**身份伪造风险**；
- AA 后端挂了会导致该 API 不可用甚至影响 discovery/聚合 OpenAPI，要配 `--enable-aggregator-routing` 与后端高可用。

## A13. CRD vs Aggregated API server 怎么选？

| 维度 | CRD | Aggregated API（AA） |
| --- | --- | --- |
| 实现成本 | 低（YAML 即可） | 高（要写 Go server） |
| 存储 | 走主 etcd | 可自定义存储（甚至外部系统） |
| 校验表达力 | OpenAPI + **CEL**（新版本） | 任意复杂逻辑 |
| 支持 subresource/协议 | 有限（status/scale 等有支持） | 完整 |
| 适合 | 大多数场景 | metrics-server 这类、需外部存储的场合 |

## A14. apiserver 的常见慢请求根因？

| 现象 | 常见根因 | 证据 |
| --- | --- | --- |
| write 慢 | etcd 提交延迟（WAL fsync） | `etcd_disk_wal_fsync_duration_seconds` |
| list 慢 | 没用 watch cache / 全量 list / 无 selector | `apiserver_cache_*`、request size、对象数量 |
| watch 断/重试 | RV 被压缩 ⇒ 410 | 客户端日志 "too old resource version" |
| 偶发 429/排队 | APF 队列满 | `apiserver_flowcontrol_*` |
| 特定资源慢 | Webhook 慢或 Fail | `apiserver_admission_webhook_admission_duration_seconds` |
| 全部变慢 | 某个 controller 疯狂 list/watch | `apiserver_request_duration_seconds` 按 client/user 分解 |

```promql
# 按资源、verb 看 P99
histogram_quantile(0.99, sum by (resource, verb, le) (rate(apiserver_request_duration_seconds_bucket[5m])))
# 按客户端身份找"吵闹邻居"
sum by (user, resource, verb) (rate(apiserver_request_total[5m]))
# 当前在途请求
sum(apiserver_current_inflight_requests)
```

## A15. apiserver 的关键启动参数有哪些？（能报出来很加分）

```bash
--max-requests-inflight=800             # 只读类并发上限（建议值，需实测校准）
--max-mutating-requests-inflight=200    # 变更类并发上限
--enable-priority-and-fairness=true     # APF
--request-timeout=60s                   # 非长连接请求超时
--watch-cache-sizes=pods#1000
--default-watch-cache-size=100
--event-ttl=1h
--audit-policy-file=/etc/kubernetes/audit.yaml
--encryption-provider-config=/etc/kubernetes/encryption.yaml
--profiling=false                       # 生产建议关闭
--tls-cipher-suites=...                 # 关闭弱套件
--feature-gates=...
```

# Part B · etcd 篇（12 题）

> etcd 是 K8s 唯一的持久化真相源，**它的健康状况直接等于集群的健康状况**。

## B1. etcd 的数据模型长什么样？

- **MVCC + bbolt**：每次写生成一个新 revision（单调递增），读可以在某个 revision 上做快照；
- key 的组织形式大致是 `/registry/<resource>/<namespace>/<name>`（v3）；
- `resourceVersion`** 本质上就是 etcd 的 revision**；
- **compaction（压缩）** 负责删掉历史 revision；**defrag（碎片整理）** 负责把磁盘空间真正归还。

**必答**：etcd **不是**"对象数据库"，它既不理解 Kubernetes 语义，也不提供二级索引——所有过滤都在 apiserver 侧完成，这解释了为什么"不加 selector 的 list 全量请求"那么贵。

## B2. Raft 与 quorum：为什么必须是奇数节点？

| 成员数 | quorum（可写所需多数） | 可容忍故障节点数 |
| --- | --- | --- |
| 3 | 2 | 1 |
| 5 | 3 | 2 |
| 4 | 3 | 1（**不划算**） |

**结论**：偶数成员除了多花钱没有任何容错收益 ⇒ 用 **3（常规）/ 5（大集群或跨 AZ 要求更高）**。

**部署要点**：

- **leader 选举与写都要经过 leader**，写延迟 ≈ 到 leader 的 RTT + WAL fsync；
- **跨地域不要做同一 etcd 集群**（RTT 直接决定写性能），要跨地域就做多集群；
- 同城多可用区是常见做法，但要接受"跨 AZ RTT"带来的写延迟代价，并压测；
- `learner` 成员可以让你安全地添加/预热新成员（不参与 quorum）。

## B3. 最经典的 etcd 事故：`mvcc: database space exceeded`（NOSPACE）

**现象**：集群突然不能写（Pod 无法创建、删除对象也失败），但读一般还能进行。

**根因链**：

```
写入量大（Event/ConfigMap/频繁更新）→ 历史 revision 未及时压缩 → 达到 quota-backend-bytes → 触发 NOSPACE alarm → etcd 进入只读
```

**处置步骤（必须记住）**：

```bash
# 1) 确认状态
etcdctl endpoint status --write-out=table --cluster
etcdctl alarm list
# 2) 压缩到当前 revision
rev=$(etcdctl endpoint status --write-out="json" | python3 -c "import sys,json;print(json.load(sys.stdin)[0]['status']['header']['revision'])")
etcdctl compact "$rev" --physical
# 3) 每个成员做碎片整理（会引起瞬时阻塞，逐个执行）
etcdctl defrag --cluster
# 4) 清除报警
etcdctl alarm disarm
# 5) 确认恢复
etcdctl endpoint status --write-out=table --cluster
```

**预防**：

- 配自动压缩：`--auto-compaction-mode=revision --auto-compaction-retention=1000`（示意值，**请与你所用发行版核对**）；
- 适当调大 `--quota-backend-bytes`（etcd 默认 2GiB；K8s 集群常见设为更大值，**务必先压测且注意恢复时间**）；
- Event 独立集群 + `--event-ttl`；
- 监控 `etcd_db_total_size_in_bytes / etcd_server_quota_backend_bytes`；
- **定期做 defrag 演练**（这是最容易"不敢在生产执行"的操作）。

```promql
etcd_db_total_size_in_bytes / etcd_server_quota_backend_bytes > 0.8
etcd_server_quota_backend_bytes
```

## B4. etcd 的头号性能指标是什么？

**磁盘。** etcd 官方硬件建议的量级参考（**请以 etcd 官方文档为准**）：

- **WAL fsync P99 应 \< 10ms**；
- **backend commit P99 应 \< 25ms**；
- 典型症状：`etcd to commit took too long`、`apply request took too long`、`leader heartbeat took too long`。

```promql
histogram_quantile(0.99, rate(etcd_disk_wal_fsync_duration_seconds_bucket[5m]))
histogram_quantile(0.99, rate(etcd_disk_backend_commit_duration_seconds_bucket[5m]))
# 网络 RTT（Peer 之间）
histogram_quantile(0.99, sum by (To, le) (rate(etcd_network_peer_round_trip_time_seconds_bucket[5m])))
# leader 变更频率（频繁切主是大忌）
increase(etcd_server_leader_changes_seen_total[30m])
# 慢请求/失败提议
increase(etcd_server_slow_applies_total[10m])
increase(etcd_server_proposals_failed_total[10m])
rate(etcd_server_proposals_pending[1m]) > 5
```

**必答的治理动作**：NVMe SSD 独占磁盘（不要和其它 IO 密集进程共享）、网络延迟达标、不要跑在"共享/节流型"云盘上、使用独立节点不要与 workload 混部。

## B5. K8s 里 etcd 到底存了什么？什么不存？

| 存 | 不存 |
| --- | --- |
| 所有 Kubernetes 对象的最新状态（Pod/Deployment/Secret/ConfigMap/Service/EndpointSlice/Node/Lease/PVC/VPC-ish CRD 等） | **容器文件系统内容**（在节点上） |
| 事件的当前对象（在有 TTL 的情况下会过期删除） | 容器日志（节点磁盘；etcd 里只有 Pod 对象） |
| CRD 自定义对象 | metrics / 监控时序数据 |
| Lease、TokenRequest 相关状态等 | 实际网络转发规则（节点上） |

**必答**：**etcd 里没有任何"业务数据"和"运行时数据"** —— 所以"把 etcd 备份好 + 有正确的恢复流程"理论上能重建整个集群状态，但**节点上的容器与本地卷数据不在其中**（有状态服务要另行备份）。

## B6. etcd 备份与恢复的标准流程？（SRE 必考）

**备份**：

```bash
ETCDCTL_API=3 etcdctl \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  snapshot save /backup/etcd-$(date +%Y%m%d-%H%M).db

# 校验快照可用性（不要等到真出事才发现备份是坏的）
etcdctl snapshot status /backup/etcd-xxx.db --write-out=table
```

**恢复（示意，务必先在演练环境验证）**：

```bash
# 1) 停止所有 apiserver（停止写入是关键）
# 2) 停掉 etcd 成员
# 3) 逐成员 restore（每个成员用自己的 --name/--initial-advertise-peer-urls）
etcdctl snapshot restore /backup/etcd-xxx.db \
  --name etcd-1 \
  --initial-cluster "etcd-1=https://10.0.0.1:2380,etcd-2=https://10.0.0.2:2380,etcd-3=https://10.0.0.3:2380" \
  --initial-cluster-token etcd-cluster-token \
  --initial-advertise-peer-urls https://10.0.0.1:2380 \
  --data-dir /var/lib/etcd-restored
# 4) 用新的 data-dir 启动 etcd
# 5) 启动 apiserver，验证资源一致性
# 6) 强制重启 kubelet/重建 Pod 让实际状态对账（必要时按服务逐个滚动）
```

**演练要求**：恢复演练必须**定期执行**并计时（MTTR），并在演练中验证"恢复后的数据时间点是否符合 RPO 目标"。

## B7. etcd 要不要和 apiserver 分开部署？

| 方案 | 优点 | 缺点 |
| --- | --- | --- |
| **同机部署**（kubeadm 常见） | 简单、少机器 | 磁盘/网络资源竞争；大规模（\>\~1000 节点）不推荐 |
| **独立 etcd 集群**（推荐中大规模） | 资源隔离、可独立扩容/调优/备份 | 多一组机器与证书管理 |

**建议**：集群规模上来或稳定性要求高时，**etcd 独立部署**几乎是必选项，且管理面要能做到"先停 apiserver 再动 etcd"。

## B8. defrag 有什么风险？怎么安全地做？

- defrag 会**重建 bbolt 数据库**，期间该成员**阻塞/拒绝请求**（视版本），因此对 leader 执行会触发重新选主 ⇒ 抖动；
- 做法：**逐个成员执行**，先在 follower 上做；某些版本支持 `--read-only-timeout` / 延迟 defrag 的规则（`--defrag-rule-method`，请核对）；
- 磁盘空间要足够容纳临时文件；
- 建议在低峰 + 有回滚窗口 + 已备份的前提下执行，且**先在演练集群熟练**。

## B9. 证书与 TLS：etcd 的一整套证书体系

etcd 需要：

- **server cert**（给客户端用）、**peer cert**（成员间）、**CA**；
- apiserver 连接 etcd 需要 client cert（`--etcd-certfile` 等）。

**典型事故**：证书过期 ⇒ apiserver 无法连 etcd ⇒ **整个集群控制面不可用**。

**治理**：

- 监控证书有效期并提前告警；
- 证书轮换必须走受控流程并在变更后验证 `etcdctl endpoint health`；
- kubeadm/托管服务会自动处理，**不要手动改一半**

```promql
# 依赖 kubelet 或自采集方式暴露的证书过期指标（指标名以实际环境为准）
# 建议：对 /etc/kubernetes/pki/** 做定时检查并输出 gauge
```

## B10. etcd 的 key 个数/对象数太多会怎样？

- 单 key 过大、单 key 个数过多都会增加 bbolt 树深度与 commit 成本；
- **Pod/Event/EndpointSlice/Lease 是数量增长最快的对象**；
- **不要往 ConfigMap/Secret 里塞大二进制**（会影响 watch cache、list 成本，甚至超过对象大小限制）；
- 清理策略：删除无用 CRD 对象、控制 Event TTL、审视 Lease 数量。

```promql
# 对象数量视图（需要 kube-state-metrics 或 apiserver 侧计数）
apiserver_storage_objects{resource="pods"}        # 指标名与可用性请按版本核对
apiserver_storage_objects{resource="endpointslices"}
```

## B11. etcd 的成员替换/扩容怎么做？

```
加入新成员（推荐先以 learner 身份加入，等数据追平）
   → etcdctl member add <name> --peer-urls=... --learner
   → 启动新成员进程
   → 追平后 etcdctl member promote <id>
替换旧成员：加入新 → 删除旧（一次只改一个成员，改完必须确认健康）
```

**禁止**：**同时改多个成员**、在没有健康 quorum 的情况下操作、在没有快照的情况下做成员变更。

## B12. etcd 常见日志报错速查？

| 日志 | 含义 | 处置 |
| --- | --- | --- |
| `database space exceeded` | 配额满 | B3 处置流程 |
| `apply request took too long` | commit 慢（通常是磁盘） | 查 fsync/commit 延迟 |
| `leader heartbeat took too long` | 网络或磁盘慢 | 查 peer RTT + fsync |
| `request timed out` | 客户端超时、集群忙 | 查 in-flight 与慢请求 |
| `etcdserver: mvcc: required revision has been compacted` | 客户端 watch 太老 | 客户端 relist；缩短 list 周期 |
| `authentication failed, invalid peer certificate` | 证书错/过期 | 校验证书 |
| `read-only range request ... took too long` | 读慢（大 list / 磁盘慢） | 优化 list + 磁盘 |

---

# Part C · kube-controller-manager 篇（12 题）

## C1. 控制器的通用编程范式是什么？

```
Informer(ListAndWatch)  →  本地 Store(缓存)  →  EventHandler 把 key 放进 WorkQueue
                                             ↓
                                       worker 取 key
                                             ↓
                                   Reconcile(key)：
                                       读期望状态(API) vs 读实际状态(API/外部系统)
                                       → 有差异则写 API 让它收敛
                                             ↓
                                   失败 → 按限流器(指数退避)重新入队
```

**必答的四个要点**：

1. **Level-triggered，不是 edge-triggered** —— Reconcile 回答的是"现在应该是什么样"，而不是"发生了什么事件"；
2. **幂等**：同一个 key 反复 reconcile 结果一致；
3. **resyncPeriod** 兜底：即使事件丢失也会周期性重放；
4. **不持有外部真相源**：状态要么从 API 读，要么从外部系统读，绝不假设"我上次记的那份是对的"。

## C2. controller-manager 里到底有哪些控制器？（要能报出一大串）

| 分类 | 控制器 | 作用 |
| --- | --- | --- |
| 工作负载 | Deployment、ReplicaSet、StatefulSet、DaemonSet、Job、**CronJob** | 副本维持、滚动更新、定时任务 |
| 网络 | EndpointSlice controller、Endpoints controller、ServiceAccount token controller | 服务发现后端集合、SA token |
| 节点 | Node lifecycle controller、**taint manager**、clusterrole aggregation | NotReady 标记/驱逐、聚合内置 ClusterRole |
| 存储 | PV controller、Attach/Detach controller、volume expand controller | 卷绑定/挂载/扩容 |
| 资源治理 | **ResourceQuota controller**、Namespace controller | 配额用量计算、命名空间生命周期 |
| 对象关系 | **Garbage collector controller**（级联删除） | ownerReference 级联删除 |
| 证书 | CSR approver/signer 相关、Certificate controller | 证书签发（**注意：node serving CSR 的批准策略依赖发行版**） |
| 扩缩容 | HPA controller（常挂在 controller-manager 或独立部署） | 基于指标扩副本 |
| 其它 | Job TTL controller（finished 后清理）、ownership 相关 | TTL 清理等 |

```bash
# 按需启停（示意）
kube-controller-manager --controllers=*,bootstrapsigner,tokencleaner
```

## C3. controller-manager 怎么保证只有一个在工作？

- 基于 `coordination.k8s.io/v1 Lease` 做 leader election（kube-system 命名空间）
- 只有 leader 执行 reconcile；其它实例待命
- 相关默认lease/renew/retry时长请以所用版本为准

```promql
# 观察是否有多主或频繁切主（指标名以实际暴露情况为准）
increase(leader_election_master_status[10m])
```

## C4. Node lifecycle controller 与 taint manager 的完整时间账

```
① kubelet 每 ~10s 更新 Node Lease
② controller-manager 超过 --node-monitor-grace-period（默认 50s）判定节点不健康
③ 给节点打 condition：Ready=Unknown/False
④ taint manager 打 taint：node.kubernetes.io/not-ready 或 node.kubernetes.io/unreachable（NoExecute）
⑤ Pod 默认注入了 tolerationSeconds=300 的容忍
⑥ 5 分钟后 taint manager 才真正驱逐 Pod → 重新 Pod 调度
```

**结论**：节点故障约 **5\~****6 分钟**后工作负载才会被重新拉起到其它节点（与 Kubelet 篇 Q16 完全对应，面试时可联动作答）。

## C5. 滚动更新（Deployment）到底是怎么算的？

```
maxSurge      = 25% / n：允许超出期望副本数的比例
maxUnavailable= 25% / n：允许不可用的比例
```

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web
spec:
  replicas: 10
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 2            # 建议：按绝对值给，比百分比更可预测
      maxUnavailable: 0      # 零中断场景建议配 0（需要集群有冗余资源）
  minReadySeconds: 15        # 建议：给"新 Pod 已就绪但还没稳定"留观察窗
  progressDeadlineSeconds: 600
  revisionHistoryLimit: 10
```

**必答**：

- Deployment 只是"创造了新的 ReplicaSet"并按策略缩旧扩新，实际扩缩数量由 **ReplicaSet controller** 完成；
- `maxUnavailable: 0` + PDB 才能保证真正零中断；
- `ProgressDeadlineExceeded` 说明更新在期限内没完成（常见原因：镜像拉不下、配额不够、探针不过）。

## C6. Garbage Collector 的三种删除策略？

| `propagationPolicy` | 行为 |
| --- | --- |
| `Foreground` | 先删除依赖对象，再删 owner（会进入 deleting 状态等待） |
| `Background`（常见默认） | 立即删 owner，GC 后台清理依赖对象 |
| `Orphan` | 删 owner 但**保留**子对象（断开 ownerReference） |

**必答**：`Finalizer` 会**阻塞删除**（对象保留 `deletionTimestamp` 直到 finalizer 被拿掉）——这是"Terminating 卡死"的头号原因。

## C7. ResourceQuota controller 与 admission ResourceQuota 的分工？

- **Admission（ResourceQuota 准入插件）**：在创建/更新请求时**检查是否会超配额**，超了直接拒绝（这是"为什么 Pod 创建被 403 forbidden"的原因之一）；
- **Controller（ResourceQuota controller）**：统计各命名空间的已用量并写回 `status.used`（用量修正是异步的）。

**坑**：异步统计意味着短暂窗口内可能超额；所以要结合配额 + 上层审批 + LimitRange 默认值。

## C8. Attach/Detach vs Volume manager 怎么分工？

| 层 | 谁 | 干什么 |
| --- | --- | --- |
| **Attach/Detach**（controller-manager） | 与控制面/云 API 交互 | 把云盘"挂到虚拟机" |
| **Volume manager**（kubelet） | 节点侧 | 挂载文件系统到 Pod 目录 stage/publish |
| **PV controller** | 控制面 | PVC↔PV 绑定、延迟绑定（WaitForFirstConsumer） |

**排障分界线**：`FailedAttachVolume` 属控制面/云 API 问题；`FailedMount` 多半是 kubelet/CSI 节点侧问题。

## C9. HPA 的指标管线与常见参数？

```
cAdvisor → kubelet summary/metrics → metrics-server →
      apiserver aggregated API(metrics.k8s.io) → HPA controller → Deployment 副本数
自定义/外部指标：
      Prometheus Adapter → custom.metrics.k8s.io / external.metrics.k8s.io
```

**常见参数**（请核对版本默认值）：`--horizontal-pod-autoscaler-sync-period`、`--horizontal-pod-autoscaler-downscale-stabilization`（防抖动）、`--horizontal-pod-autoscaler-tolerance`。

**实践**：给**扩容快、缩容慢**（different stabilization windows，`behavior.scaleDown.stabilizationWindowSeconds`），避免"抖动导致副本数反复横跳"。

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
spec:
  behavior:
    scaleUp:
      stabilizationWindowSeconds: 0
      policies: [{type: Percent, value: 100, periodSeconds: 60}]
    scaleDown:
      stabilizationWindowSeconds: 300       # 建议初值，需要按业务调
```

## C10. 大规模下 controller-manager 的调优参数？

```bash
--kube-api-qps=100 --kube-api-burst=150     # 建议初值，需实测
--concurrent-replicaset-syncs=20
--concurrent-deployment-syncs=10
--concurrent-endpoint-syncs=10
--concurrent-service-endpoint-syncs=10
--concurrent-gc-syncs=20
--concurrent-resource-quota-syncs=10
--node-monitor-grace-period=50s
--node-monitor-period=5s
```

**风险**：并发调高会放大对 apiserver/etcd 的压力（尤其 endpoint sync），**必须与 apiserver/etcd 容量一起压测**。

## C11. Deployment 卡住（rollout 卡住）怎么排查？

| 症状/证据 | 原因 |
| --- | --- |
| `ImagePullBackOff` | 镜像/仓库/凭据问题 |
| Pod Pending 且 FailedScheduling | 资源/亲和/污点/配额（见 Scheduler 篇） |
| `forbidden: exceeded quota` | ResourceQuota |
| 更新没法进展 | PDB 限制导致旧 Pod 无法删除 |
| 新 Pod 一直 NotReady | 探针/应用/网络策略 |
| `ProgressDeadlineExceeded` | 上述任一原因超时 |
| RS 数量异常多 | `revisionHistoryLimit` 过大/频繁发布 |

```bash
kubectl rollout status deploy/web
kubectl describe deploy/web | tail -30        # Conditions 有原因
kubectl get rs --sort-by=.metadata.creationTimestamp | tail
kubectl describe rs <new-rs>                  # ReplicaSet 的 Events 最有信息量
```

## C12. 自研控制器要注意什么？

- 用 **controller-runtime（CR 生态）** 而不是手写 informer；
- Reconcile 快速返回 + 用 `RequeueAfter`/事件驱动；
- **写操作要有 check-then-act 的乐观锁重试**（409 处理）；
- 合理限制并发 worker 数与 QPS/burst；
- **不要高频写 Event**（会放大 etcd 压力）；
- 提供清晰 status/conditions，方便排障；
- 健康检查 + metrics + leader election（单副本语义时）必备。

# Part D · kube-proxy 专项篇（8 题）

## D1. kube-proxy 到底"转发"了吗？（最大误区）

**没有。** kube-proxy **只写规则**，真正转发数据包的是内核（netfilter/NAT 或 IPVS），或者完全不经过 netfilter（Cilium socket-LB 在 socket 层完成）。

```
Service/EndpointSlice 变化
    → kube-proxy watch 到
    → 计算规则集合
    → 通过 iptables-restore / ipvsadm-equivalent netlink / nft 下发
    → 内核按规则转发包（kube-proxy 不参与单包处理）
```

**结论**：kube-proxy 挂掉 ⇒ **已下发的规则仍然生效**（已有连接与 Service 访问不会立刻断），但**后续 Service/Endpoint 变更不再被同步**（这是"为什么节点重启前后不一致、某些后端没更新"的根因）。

## D2. kube-proxy 有哪几种模式？各自的关键差异？

| 模式 | 数据面 | 查找复杂度 | 主要优点 | 主要缺点 |
| --- | --- | --- | --- | --- |
| **userspace** | 进程代理 | — | 历史实现 | 已弃用 |
| **iptables** | netfilter 链 | O(规则数) | 默认、依赖少 | 规模大时规则膨胀、更新全量下发、无真调度算法 |
| **ipvs** | IPVS + ipset | O(1) | 规模友好、有调度算法 | 仍用 iptables 处理 SNAT/包过滤；内核模块依赖 |
| **nftables** | nftables | 较好 | 较新的替代方案 | **版本相关（处于演进阶段），请以 Release Notes 确认成熟度** |
| **无 kube-proxy（CNI 替代）** | eBPF socket-LB | 好 | 绕开 netfilter/conntrack、保留源 IP | 需 Cilium 等支持，依赖较新内核 |

## D3. iptables 模式下的一条链路长什么样？

```
外部访问 ClusterIP:80
  PREROUTING → KUBE-SERVICES 链（匹配 ClusterIP）
       → per-service 链 KUBE-SVC-XXX（statistic random → 挑一个后端）
       → per-endpoint 链 KUBE-SEP-YYY（DNAT 到 PodIP:Port）
  NodePort 场景：先经 KUBE-NODEPORTS
  出集群/源 SNAT：POSTROUTING → KUBE-POSTROUTING → MASQUERADE/RETURN（由 ipset KUBE-LOOP-BACK 等控制）
```

**排障命令**：

```bash
iptables-save -t nat | grep -A5 <clusterIP>
iptables-save -t nat | wc -l              # 规则规模
ipvsadm -Ln --stats                       # IPVS 模式
nft list ruleset                          # nftables 模式
```

## D4. kube-proxy 的关键参数有哪些？

```yaml
apiVersion: kubeproxy.config.k8s.io/v1alpha1
kind: KubeProxyConfiguration
mode: ipvs
# 通用
bindAddress: 0.0.0.0
clusterCIDR: 10.244.0.0/16
healthzBindAddress: 0.0.0.0:10256         # 端口请按版本核对
metricsBindAddress: 0.0.0.0:10249
detectLocalMode: ClusterCIDR
# 抖动控制：避免 Endpoint 频繁变化引起规则疯狂重写
iptables:
  syncPeriod: 30s
  minSyncPeriod: 5s
ipvs:
  syncPeriod: 30s
  minSyncPeriod: 5s
  scheduler: rr
  excludeCIDRs: []
  tcpTimeout: 0s
conntrack:
  maxPerCore: 131072                      # 建议初值，需实测
  tcpEstablishedTimeout: 24h
  tcpCloseWaitTimeout: 1h
```

**要点**：`minSyncPeriod` 太小会让 kube-proxy 在 Endpoint 抖动时频繁全量刷新规则（CPU 尖峰）；太大则收敛变慢。

## D5. kube-proxy 挂了会怎样？怎么发现？

| 表现 | 说明 |
| --- | --- |
| 已有 Service 访问正常 | 规则还在内核里 |
| 新增/删除 Service、Endpoint 变更不生效 | 规则不同步 |
| Pod 访问新上线的后端失败 | 旧规则残留 |

```promql
# 规则同步是否还活着（关键指标，务必先核对目标版本是否暴露）
histogram_quantile(0.99, sum by (le) (rate(kubeproxy_sync_proxy_rules_duration_seconds_bucket[5m])))
# 最后一次成功同步时间：长时间不更新即意味着 kube-proxy 卡了
time() - kubeproxy_sync_proxy_rules_last_timestamp_seconds > 300
# 进程存活
up{job="kube-proxy"} == 0
```

## D6. `healthz` 与 LB 健康检查的关系？

- kube-proxy 提供 `/healthz`（默认端口 10256，请核对），可用于 DaemonSet 的 liveness/readiness；
- **它不代表数据面一定正常**，只说明进程活着；
- 真正要监控的是"规则同步是否及时" + "业务合成探测是否成功"。

## D7. 大规模集群 kube-proxy 的优化清单？

1. **切 IPVS**（或 nftables/Cilium）：摆脱 iptables 规则线性增长；
2. 合理设置 `minSyncPeriod`，避免抖动放大；
3. conntrack 扩容 + timeout 调优（见网络篇 C5）；
4. 控制 Service/Endpoint 数量增长（架构侧）；
5. EndpointSlice 默认切分更细可以减少单次更新 payload（`-max-endpoints-per-slice`）；
6. kube-proxy 的 API 客户端限流（`--kube-api-burst/qps`）别太小，否则大规模下追不上。

## D8. 什么时候该"去掉" kube-proxy？

满足以下多数条件时可考虑 **Cilium kube-proxy replacement**：

- Service/Endpoint 数量大且持续增长；
- 短连接高并发，conntrack 成为瓶颈；
- 需要保留客户端真实 IP 而不想用 `externalTrafficPolicy: Local`；
- 已经具备 eBPF 的排障能力（团队 skill 到位）。

**反之**（保守为宜）：内核版本不满足、有强依赖 netfilter 的遗留组件、团队缺乏 eBPF 排障经验。

---

# Part E · 大规模控制面优化篇（12 题）

## E1. 控制面的容量到底由什么决定？

```
apiserver QPS 上限 ≈ min(
    CPU：序列化/准入/审计的 CPU 成本,
    etcd 写延迟：每次写都要 WAL fsync + Raft quorum,
    watch fan-out：副本数 × 各组件 watch 数,
    list 成本：大对象/全量 list 打穿缓存
)
```

**结论（必答）**：**apiserver 可以横向扩容，etcd 不可以轻易扩容**。控制面性能治理的终点几乎总是落在 etcd 磁盘延迟与"少写少 list"。

## E2. 最常见的两类"控制面杀手"：大 list 与 watch 洪水

| 反模式 | 后果 | 治理 |
| --- | --- | --- |
| 控制器全集群 list 且不带 selector | 巨大响应体 + 序列化 CPU | 强制走 informer + field/label selector + 分页 |
| 每个组件都自己 list/watch 同一资源 | N 倍流量 | 使用 **SharedInformer**；或走 informer 缓存代理 |
| watch 不带 bookmarks / 反复断开重放 | 消息洪峰 | 开启 `allowWatchBookmarks`、指数退避、合理 relist 间隔 |
| 客户端忘记限流 | 打满 apiserver | client-go `--kube-api-qps/burst` 或 client 侧 rate limiter |

```promql
# 谁在大量请求？（找吵闹邻居的第一条语句）
sum by (user, verb, resource) (rate(apiserver_request_total[5m]))
# 响应体大小（大 list 的直接证据）
histogram_quantile(0.99, sum by (resource, verb, le) (rate(apiserver_response_sizes_bucket[5m])))
```

## E3. 对象数量与体积治理

- **避免在 ConfigMap/Secret 里放大对象**（有大小限制；大对象会拖慢 list/watch/etcd）；
- **Event 治理**（TTL、准入限速、独立 etcd）；
- **减少 Lease/CRD 的无效增长**；
- 定期审计：`kubectl get crd`、按对象类型统计 Top N。

```bash
# 按对象类型快速估算数量（示意思路）
kubectl get --raw /metrics | grep apiserver_storage_objects
```

## E4. APF 的生产配置思路

```
① 先区分"关键流量"：leader-election、kubelet 心跳/status、scheduler 绑定、控制面探针
② 为它们配独占 PriorityLevel + 队列（避免被吵闹邻居挤掉）
③ 对"全集群 list / 非关键 controller"配低优先级 + 短队列（打不赢就让它们失败重试）
④ 监控 rejected/queue depth，逐步调 shares
```

> **注意**：错误的 FlowSchema 会**误伤关键组件**，任何 APF 变更都应在测试集群验证，并保留可快速删除（回滚）的变更记录。

## E5. watch cache 与 per-resource 调优

- 热点资源（pods、endpointslices、nodes、configmaps、secrets）优先加大 cache；
- 观察 `apiserver_cache_*` 指标与 **apiserver 内存 RSS**；
- 内存与一致性之间取值：**建议先给少量关键资源加大，观察 7 天再决定是否继续调**。

## E6. etcd 侧的大规模最佳实践（汇总）

| 项 | 建议 |
| --- | --- |
| 磁盘 | 独立 NVMe SSD，不共享 |
| 配额 | 按数据量规划 `--quota-backend-bytes`，并留出恢复时间余量 |
| 压缩 | 开启自动压缩；保留策略按审计/回溯需求设定 |
| defrag | 定期低峰执行 + 演练过 |
| Event | 独立 etcd 集群（大集群） |
| 机密性 | Secret 静态加密；注意 KMS 可用性风险 |
| 拓扑 | 3 或 5 成员，**同地域低延迟网络**，跨可用区需压测 |
| 备份 | 定时快照 + 定期恢复演练（RPO/RTO 写进 SLO） |

## E7. 控制面组件升级的标准顺序与风险

```
① 先备份 etcd 快照（这一步永远排第一）
② 升 etcd（一次一个成员，确认 health 再下一个）
③ 升/滚动 apiserver（先灰度一个副本，观察后再全量）
④ 升 kube-controller-manager / kube-scheduler（依赖 leader election 自动切换）
⑤ 升 cloud-controller-manager、kube-proxy、CNI、CoreDNS
⑥ 最后升 kubelet（按节点池灰度，遵循 skew ≤ 3 且不得高于 apiserver）
```

**风险与回滚**：

- **etcd 版本回退几乎不可行**（数据格式/dbVersion 一般只能向前），所以 etcd 升级必须"先快照、慢一点、一次一个"；
- apiserver 回滚 = 换回旧版本镜像/二进制，**要注意 API 是否已经对新字段写入**（可能不可逆）；
- kube-proxy/CNI 回滚请见网络篇 C13 的双池灰度法。

## E8. 多控制面副本的流量与优雅下线

- LB 健康检查建议用 `/readyz`；
- 副本滚动时要保证 **watch 连接平滑迁移**（客户端要有重连退避）；
- 注意 `--shutdown-delay-duration`（或对应机制）配合 LB 摘除延迟，避免"已经停止服务但 LB 还在转发"。

```promql
# apiserver 就绪度
sum(up{job="apiserver"}) / count(up{job="apiserver"}) < 0.66
# 请求 5xx 比例
sum(rate(apiserver_request_total{code=~"5.."}[5m])) / sum(rate(apiserver_request_total[5m])) > 0.01
```

## E9. 灾备：etcd 快照 vs 对象级备份

| 方案 | 覆盖 | 恢复粒度 | 适用 |
| --- | --- | --- | --- |
| **etcd 快照** | 全部对象状态 | 整集群（很难只恢复单个 ns） | 集群级灾难恢复 |
| **Velero/对象导出** | 指定资源/命名空间 | **细粒度** | 误删恢复、迁移、审计留存 |

**建议**：**两者都要** —— velero 负责日常误操作恢复（高频、低风险），etcd 快照负责集群级重建（低频、高风险、必须演练）。

## E10. 控制面 SLO 怎么定？（SIG Scalability 口径思路）

社区对 API 延迟有明确的 SLI 定义（**口径请以 SIG Scalability 文档为准**）：

- mutating / read-only 请求的 P99 延迟（按 request scope 区分：cluster / namespace / resource-level）；
- 通常 apiserver 会暴露专门的 SLI histogram（如 `apiserver_request_sli_duration_seconds_bucket`，**指标是否存在请先在目标节点 **`/metrics`** 核对**）。

```promql
# 通用版本（可直接用）
histogram_quantile(0.99, sum by (le) (rate(apiserver_request_duration_seconds_bucket{verb!="WATCH"}[5m])))
# 分 scope 观察
histogram_quantile(0.99, sum by (scope, le) (rate(apiserver_request_duration_seconds_bucket{verb!="WATCH"}[5m])))
# 写操作单独看
histogram_quantile(0.99, sum by (le) (rate(apiserver_request_duration_seconds_bucket{verb=~"POST|PUT|PATCH|DELETE"}[5m])))
```

## E11. 控制面故障的三类典型剧本

| 剧本 | 现象 | 首要动作 |
| --- | --- | --- |
| **etcd NOSPACE** | 全部写失败 | B3：compact → defrag → alarm disarm |
| **etcd 慢/频繁切主** | 全集群 API 延迟高、大量重试 | 查 fsync/commit/Peer RTT；查磁盘和网络 |
| **证书过期** | 组件互相 TLS 失败、无人/auth 失败 | 轮换证书（**平时就该有提前告警**） |
| **Webhook Fail 卡全集群** | 所有变更被拒 | 立即删除/禁用该 ValidatingWebhookConfiguration |
| **APF 配错误伤** | 关键组件 timeout | 删除/修正 FlowSchema-PriorityLevel |
| **LB 导到坏 apiserver** | 部分请求失败 | 健康检查用 `/readyz`，确认 LB 已摘除坏实例 |

## E12. 给你一个 2000 节点的新集群，控制面怎么设计基线？（综合题）

按顺序答：

1. **问 SLO**：可用性目标？RPO/RTO？业务对控制面抖动的容忍度？
2. **拓扑**：apiserver ≥ 3 副本跨可用区 + LB 用 `/readyz`；etcd 3/5 成员、**同城低延迟**、独立节点与独占磁盘（NVMe）。
3. **容量规划**：对象数、Event 量、list/watch 主来源；是否 Event 独立 etcd。
4. **参数基线**：`--max-requests-inflight/-mutating`、`--watch-cache-sizes`、`--event-ttl`、`--auto-compaction-*`、`--quota-backend-bytes`。
5. **APF**：为 kubelet/scheduler/controller/leader-election/探针保留优先级。
6. **安全**：关闭匿名、最小 RBAC、NodeRestriction、Secret 加密、审计策略、webhook 作用域限制。
7. **可观测**：apiserver/etcd/controller-manager/kube-proxy 指标 + 8 条核心告警（见 G 篇）。
8. **备份**：etcd 定时快照 + 定期恢复演练 + Velero 细粒度备份。
9. **演练**：GameDay（见 H 篇）。
10. **变更治理**：E7 的升级顺序 + 灰度 + 回滚预案，全部落到工单与脚本。

# Part F · 反模式清单（可直接用于生产巡检）

| # | 反模式 | 证据（怎么发现） | 等级 | 修复动作 | 验证方式 |
| --- | --- | --- | --- | --- | --- |
| F1 | 图省事给 `cluster-admin` | RBAC 审计里大量通配绑定 | 🔴 高 | 收敛到最小权限 ClusterRole/Role | 权限审计脚本通过 |
| F2 | Admission Webhook `failurePolicy: Fail` 且作用于所有 ns（含 kube-system） | webhook 挂 ⇒ 全集群变更失败 | 🔴 高 | 加 namespaceSelector 排除系统 ns + 多副本 + 短超时 | 停掉 webhook 副本，系统 ns 仍可变更 |
| F3 | 控制器全集群大 list（无 selector/无分页） | `apiserver_response_sizes` 巨大、CPU 高 | 🔴 高 | informer + selector + 分页 | `apiserver_request_duration_seconds` 下降 |
| F4 | 未配置自动压缩/定期 defrag | `etcd_db_total_size_in_bytes` 持续爬升 | 🔴 高 | 配 auto-compaction + 定期 defrag 演练 | 空间占用回到稳定水位 |
| F5 | etcd 与 workload 混部 / 共享机械/节流盘 | fsync P99 高、慢日志 | 🔴 高 | 独立节点 + NVMe | fsync P99 达标（实测） |
| F6 | 没有 etcd 备份或从未演练恢复 | 无快照文件/无恢复记录 | 🔴 高 | 定时快照 + 定期 restore 演练 | 演练记录中恢复成功且计时达标 |
| F7 | 证书无过期告警 | 证书到期突然全集群不可用 | 🔴 高 | 证书有效期监控 + 提前轮换 | 告警在到期前 30 天触发 |
| F8 | APF 未配置（全靠总量限流） | 某客户端打满限额饿死 scheduler/kubelet | 🟠 中 | 配 FlowSchema 保障关键组件 | 压测吵闹邻居时关键组件不受影响 |
| F9 | Event 无 TTL / 无限速 | Event QPS 巨大 | 🟠 中 | `--event-ttl` + EventRateLimit + 独立 etcd | Event 写入速率回归合理区间 |
| F10 | 大规模仍用 iptables kube-proxy | 规则同步慢、CPU 尖峰 | 🟠 中 | 切 IPVS / eBPF | `kubeproxy_sync_proxy_rules_duration_seconds` 下降 |
| F11 | Secret 未静态加密 | 合规不达标 | 🟠 中 | EncryptionConfiguration（注意 KMS 可用性） | etcd 快照中 secret 非明文 |
| F12 | `--profiling=true` 或暴露不必要端口 | 安全扫描告警 | 🟠 中 | 关闭 profiling、收紧安全组 | 端口探测与基线一致 |
| F13 | 控制器并发参数盲目调大 | apiserver/etcd 被打爆 | 🟠 中 | 与容量一起压测后再定 | 压测无 5xx 与 etcd 慢日志 |
| F14 | HPA 缩容无稳定窗口 | 副本数反复横跳 | 🟡 低 | 配 `stabilizationWindowSeconds` | 观察副本数曲线平稳 |
| F15 | 直接升级/维护 etcd 未先快照 | 出问题无法回退 | 🔴 高 | 每次变更先备份 + 一次改一个成员 | 变更工单含备份与回滚步骤 |

---

# Part G · 可观测性：指标与告警（可直接落地）

## G1. 指标清单（**请先 **`curl -k https://<apiserver>:6443/metrics`**、**`curl http://<node>:2382/metrics`**（etcd）核对后再上线**）

| 组件 | 指标 | 用途 |
| --- | --- | --- |
| apiserver | `apiserver_request_duration_seconds` / `_ request_total` | 延迟与 QPS（按 verb/resource/code/scope） |
| apiserver | `apiserver_current_inflight_requests` / `_mutating` | 并发与限流 |
| apiserver | `apiserver_flowcontrol_dispatched_requests_total` / `_rejected_requests_total` / `_current_inqueue_requests` | APF 健康 |
| apiserver | `apiserver_cache_list_duration_seconds` / `_watch_duration_seconds` | watch cache 命中与重放 |
| apiserver | `apiserver_admission_webhook_admission_duration_seconds` / `_fail_open_count` | 准入 webhook 是否拖慢/失效 |
| apiserver | `apiserver_audit_event_total` / `_request_total` / `_level_total` | 审计健康与成本 |
| apiserver | `apiserver_storage_objects`（可用性依版本） | 对象数量趋势 |
| etcd | `etcd_disk_wal_fsync_duration_seconds` / `_backend_commit_duration_seconds` | **最重要的磁盘指标** |
| etcd | `etcd_server_leader_changes_seen_total` / `etcd_server_has_leader` | 选主稳定性 |
| etcd | `etcd_network_peer_round_trip_time_seconds` | Peer 网络 RTT |
| etcd | `etcd_db_total_size_in_bytes` / `etcd_server_quota_backend_bytes` | 容量 |
| etcd | `etcd_server_proposals_failed_total` / `_pending` / `etcd_server_slow_applies_total` | 写入健康 |
| controller-manager | `rest_client_*`、`queue depth/workqueue_*`（按实现） | 客户端行为与队列积压 |
| kube-proxy | `kubeproxy_sync_proxy_rules_duration_seconds` / `_last_timestamp_seconds` | 规则同步健康 |

## G2. Prometheus 告警规则（可直接粘贴，阈值需实测校准）

```yaml
groups:
  - name: k8s-controlplane
    rules:
      - alert: ApiserverDown
        expr: up{job="apiserver"} == 0
        for: 2m
        labels: {severity: critical}

      - alert: ApiserverHighLatency
        expr: |
          histogram_quantile(0.99,
            sum by (resource, verb, le) (rate(apiserver_request_duration_seconds_bucket{verb!="WATCH"}[5m]))) > 4
        for: 10m
        labels: {severity: warning}
        annotations:
          summary: "{{ $labels.verb }} {{ $labels.resource }} P99 达 {{ $value }}s"

      - alert: ApiserverHighErrorRate
        expr: |
          sum(rate(apiserver_request_total{code=~"5.."}[5m]))
            / clamp_min(sum(rate(apiserver_request_total[5m])), 0.001) > 0.01
        for: 10m
        labels: {severity: critical}

      - alert: EtcdNoLeader
        expr: etcd_server_has_leader == 0
        for: 2m
        labels: {severity: critical}

      - alert: EtcdLeaderFrequentChange
        expr: increase(etcd_server_leader_changes_seen_total[30m]) > 3
        for: 5m
        labels: {severity: critical}

      - alert: EtcdDiskWalFsyncSlow
        expr: histogram_quantile(0.99, rate(etcd_disk_wal_fsync_duration_seconds_bucket[5m])) > 0.01
        for: 10m
        labels: {severity: warning}
        annotations:
          summary: "etcd WAL fsync P99 {{ $value }}s（建议 < 0.01s）"

      - alert: EtcdBackendCommitSlow
        expr: histogram_quantile(0.99, rate(etcd_disk_backend_commit_duration_seconds_bucket[5m])) > 0.025
        for: 10m
        labels: {severity: warning}

      - alert: EtcdQuotaNearlyFull
        expr: etcd_db_total_size_in_bytes / etcd_server_quota_backend_bytes > 0.8
        for: 30m
        labels: {severity: critical}
        annotations:
          summary: "etcd 数据库用量已达配额 {{ $value | humanizePercentage }}，尽快 compact/defrag"

      - alert: AdmissionWebhookSlow
        expr: |
          histogram_quantile(0.99,
            sum by (name, le) (rate(apiserver_admission_webhook_admission_duration_seconds_bucket[5m]))) > 0.5
        for: 10m
        labels: {severity: warning}
        annotations:
          summary: "准入 webhook {{ $labels.name }} P99 耗时 {{ $value }}s，可能拖慢所有变更"

      - alert: ApfRequestsRejected
        expr: sum by (priority_level) (rate(apiserver_flowcontrol_rejected_requests_total[5m])) > 0.1
        for: 10m
        labels: {severity: warning}
        annotations:
          summary: "APF 优先级 {{ $labels.priority_level }} 开始拒绝请求"

      - alert: KubeProxySyncStalled
        expr: time() - kubeproxy_sync_proxy_rules_last_timestamp_seconds > 300
        for: 5m
        labels: {severity: critical}
        annotations:
          summary: "实例 {{ $labels.instance }} 的 kube-proxy 已 5 分钟未同步规则"
```

## G3. 建议的控制面 SLO（**初始建议值，必须实测校准**）

| SLI | 建议目标（起点） |
| --- | --- |
| API 可用性（非 5xx 比例，按 5 分钟窗口） | ≥ 99.9% |
| 读请求 P99（cluster-scoped 除外） | ≤ 1s |
| 写请求 P99 | ≤ 2s |
| 集群限定范围读 P99 | ≤ 5s |
| etcd WAL fsync P99 | ≤ 10ms |
| etcd leader 变更次数（30 天） | ≤ 3 |
| 控制面 RTO（灾难恢复） | ≤ 1h（按演练实测） |
| 控制面 RPO | ≤ 15min（按备份频率） |

> 错误预算：30 天滚动窗口，burn rate 14.4×（短期紧急）/ 6×（长期工单）两级升级。

---

# Part H · 演练（GameDay）计划表

| 场景 | 注入方式 | 期望表现 | 验证的 SLO |
| --- | --- | --- | --- |
| apiserver 单副本挂 | 停一个 Pod/进程 | LB 摘除，业务无感；剩余副本够承载 | 控制面可用性 |
| etcd 单成员挂 | 停一个成员 | quorum 仍在，写不中断；成员能重新加入 | 容错 1 成员 |
| etcd WAL 变慢 | 人为加 IO 压力 | 告警先触发，业务降级可控 | 磁盘 SLI |
| etcd NOSPACE | 写满/触发 alarm | 运维按 B3 流程在 RTO 内恢复 | 恢复流程有效性 |
| etcd 恢复演练 | 用快照 restore 到隔离环境 | 数据一致，时间点符合 RPO | RPO/RTO |
| 证书到期 | 临时替换短有效期证书 | 告警提前触发且轮换流程可用 | 证书治理 |
| Webhook Fail | 故意停用 webhook 服务 | 因为 scope 限制，核心 ns 不受影响 | 变更可用性 |
| 吵闹邻居 | 起一个疯狂 list 的 Pod | APF 保住 kubelet/scheduler 配额 | APF 有效性 |
| kube-proxy 挂 | 停 DaemonSet Pod | 已有通信不断，变更不同步；能被发现 | 数据面可观测 |
| 控制面网络分区 | 模拟 apiserver 与 etcd 不通 | 快速定位，有明确告警 | MTTR |

---

# Part I · 10 张面试速记卡（临阵磨枪）

1. **唯一入口**：apiserver 是**唯一**与 etcd 交互的组件，本身**无状态**，可水平扩容；真正的瓶颈永远在 **etcd 磁盘**。
2. **请求主线**：认证 → 审计 →（模拟）→ 授权 → 准入 → 校验 → 存储；**审计在授权之前**，所以被拒绝的请求也会被记录。
3. **APF**：用 FlowSchema + PriorityLevel 给关键组件留额度，替代"总量限流饿死邻居"的老做法。
4. **resourceVersion 就是 etcd revision**：更新必须带 RV，冲突 ⇒ **409**；watch RV 太老 ⇒ **410 Gone**，必须 relist。
5. **list 一致性**：不带 RV = quorum read（强一致、贵）；`resourceVersion=0` = 读 apiserver watch cache（快、可能略旧）。
6. **etcd 三条命**：WAL fsync P99（\<10ms）、quota 用量（\<80%）、leader 变更（要极少）。NOSPACE = 全集群只读。
7. **etcd 操作铁律**：一次改一个成员；每次变更前先快照；**版本回退基本不可行**。
8. **控制器范式**：informer + workqueue + **幂等 reconcile（level-triggered）** + 退避重排 + resync 兜底。
9. **节点故障时间账**：lease 10s → grace 50s → NotReady taint → Pod 默认容忍 300s ⇒ **约 5\~****6 分钟**才重调度。
10. **kube-proxy 不转发包**，它只下规则（iptables/IPVS/nftables/eBPF）；挂了**已有规则还在**，只是不再同步变更。

---

## 附：文档自查清单

- [ ] 每个组件都给了可直接套用的 YAML / flags / PromQL / etcdctl 命令
- [ ] 所有默认值（50s grace、300s toleration、10249/10256、quota 2GiB 等）标注"以版本核对"
- [ ] 未提供任何未经实测的性能数字，均给出**测量方法**
- [ ] 反模式清单含：证据 / 等级 / 修复 / 验证
- [ ] SLO 标为建议初值 + burn rate 口径 + RPO/RTO
- [ ] 变更（尤其是 etcd）给出执行顺序、一次一个成员、先备份、不可回退的风险提示
