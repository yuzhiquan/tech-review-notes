# k8s-apiserver-interview

# Kubernetes APIServer 面试题精讲（原理篇 + 大规模集群篇）

> 适用：中高级 SRE / 云原生平台工程师 / K8s 研发面试。
> 说明：文中数值分两类 —— **【默认值】** 为社区常见默认（随版本演进，答题前请核对目标版本），**【建议值】** 为工程经验起点，**【需实测】** 表示必须结合你的集群监控数据确定，不得照抄。
> 涉及生产变更的部分一律标注「灰度 / 回滚 / 验证」。

---

## 目录

- [Part 0：答题方法论](#part-0答题方法论)
- [Part 1：架构与组件原理（Q1–Q16）](#part-1架构与组件原理q1q16)
- [Part 2：请求链路与对象语义（Q17–Q28）](#part-2请求链路与对象语义q17q28)
- [Part 3：大规模集群高频问题（Q29–Q46）](#part-3大规模集群高频问题q29q46)
- [Part 4：故障排查实战（Q47–Q58）](#part-4故障排查实战q47q58)
- [Part 5：反模式与风险清单](#part-5反模式与风险清单)
- [Part 6：可直接套用的配置片段](#part-6可直接套用的配置片段)
- [Part 7：面试自检清单](#part-7面试自检清单)

---

## Part 0：答题方法论

面试官考察 APIServer 时，本质上在看三件事：

| 考察维度 | 表现出来的是 | 高分回答的特征 |
| --- | --- | --- |
| **原理深度** | 能不能画出请求从 `kubectl` 到 `etcd` 的完整链路 | 说出「认证 → 鉴权 → 准入 → 校验 → 存储」，并能补 APF、序列化、乐观锁 |
| **规模意识** | 知不知道 5000 节点时什么先挂 | 能说清 **写放大源**（Lease/Event/Endpoint）、**读放大源**（List 全量 / watch 风暴） |
| **排障能力** | 给一个报错能否定位 | 能给出：报错 → 指标 → 日志 → 配置 → 验证五步 |

**答题模板（强烈建议背下来）**：

1. 一句话定位（它是什么 / 解决什么问题）
2. 关键机制（2–3 个核心技术点）
3. 大规模下的失效模式（这是拉开差距的地方）
4. 可观测指标 + 修复动作 + 回滚条件

---

## Part 1：架构与组件原理（Q1–Q16）

### Q1：APIServer 在 Kubernetes 中的定位是什么？为什么它可以是无状态的？

**答案要点**

- APIServer 是整个集群的**唯一数据入口**：所有组件（kubelet、scheduler、controller-manager、operator、kubectl）都不直接读写 etcd，全部通过 APIServer 的 RESTful API。
- 它是**唯一与 etcd 通信的组件**（除 etcd 自身运维），因此承担：认证、鉴权、准入、校验、序列化/反序列化、版本转换、审计、限流。
- **无状态的原因**：状态全部落在 etcd。APIServer 本地只有 **watch cache**（每个资源类型一份内存缓存），它是 etcd 数据的增量副本，丢失后可重建，不构成持久状态。

**为什么重要**

- 决定了 APIServer 可以**水平扩容**（N 副本 + LB），这是大规模集群抗压的基础手段。
- 也决定了任何"绕过 APIServer 直接改 etcd"的操作都是高危反模式（会绕过准入、审计、校验，且可能破坏数据格式）。

**加分项**

- 提到 `--enable-priority-and-fairness`、`--goaway-chance`（HTTP/2 连接打散，避免 LB 粘滞导致负载不均）。
- 提到 APIServer 之间的**一致性**由 etcd 的线性一致读保证，而非 APIServer 自身。

---

### Q2：画一下 kube-apiserver 的内部架构分层。

**答案要点**（自顶向下五层）

```
┌──────────────────────────────────────────────────────┐
│ 1. HTTP/2 Server + Handler Chain（鉴权/审计/限流过滤器）│
├──────────────────────────────────────────────────────┤
│ 2. API 路由层（/api, /apis/<group>/<version>/...）     │
│    - Aggregator（聚合层，处理 APIService 转发）        │
│    - API Group/Version 注册、OpenAPI/Discovery         │
├──────────────────────────────────────────────────────┤
│ 3. Admission（变更准入 → 校验准入）                    │
├──────────────────────────────────────────────────────┤
│ 4. REST Storage / Registry 层（registry.Store）        │
│    - 校验、默认值、策略、版本转换、字段管理（SSA）      │
│    - Cacher（watch cache / 内存 ring buffer）          │
├──────────────────────────────────────────────────────┤
│ 5. etcd 存储后端（storage.Interface，MVCC + revision） │
└──────────────────────────────────────────────────────┘
```

**追问：Aggregator 和 CRD 的区别？**
见 Q15。

---

### Q3：`kubectl apply -f pod.yaml` 发出后，APIServer 内部发生了什么？（高频送命题）

**完整链路**

1. **客户端侧**：kubectl 读取 kubeconfig → 认证信息（客户端证书 / token）→ 计算请求签名 → 发送 `POST /api/v1/namespaces/default/pods`（Content-Type 通常是 `application/vnd.kubernetes.protobuf` 或 json）。
2. **传输层**：LB → 任一 APIServer 实例（HTTP/2 长连接可能粘滞）。
3. **Handler Chain（过滤器链，从外到内主干顺序）**：
   - Panic Recovery / RequestInfo 解析（verb、resource、namespace、name）
   - **Audit**（审计记录，含 audit-id）
   - **Authentication**（认证，得到 user/groups）
   - **Impersonation**（模拟身份，`--as` 场景）
   - **Priority & Fairness**（APF 限流/排队，按 user/groups/verb/resource 分流）
   - **Authorization**（RBAC 鉴权决策）
   - **Admission**：先 Mutating（改对象）→ 再 Validating（只读校验）
4. **反序列化 + 版本转换**：把请求体转成内部版本（`internal`），填默认值。
5. **校验**：字段校验、不可变字段校验（immutable field）。
6. **存储**：`registry.Store.Create` → 乐观锁检查（create 时 uid/resourceVersion 为空）→ 写入 etcd（`etcd3.store`，带 TTL/lease 视资源而定）→ 返回带 `resourceVersion` 的对象。
7. **响应**：序列化回请求的版本 → 返回客户端。
8. **异步**：写入成功触发 etcd watch 事件 → Cacher 更新 watch cache → 所有 watch 该资源的 client（kubelet/scheduler/controller 的 informer）收到事件。

> 版本提示：Handler Chain 的**细粒度顺序在不同版本有微调**（例如 APF 与审计的相对位置），答题抓主干「认证 → APF → 鉴权 → 准入 → 校验 → 存储」即可；APF 位于认证之后，因为它需要 user/groups 才能做 FlowSchema 匹配。

**加分项**

- 提到 `kubectl apply` 是 **Server-Side Apply**（1.22 GA），走 `PATCH + application/apply-patch+yaml`，携带 `fieldManager`。
- 提到 APIServer 会打 **Warning header**（如已废弃 API）。

---

### Q4：APIServer 支持哪些认证方式？生产上怎么选？

| 方式 | 原理 | 适用 | 生产建议 |
| --- | --- | --- | --- |
| **X509 客户端证书** | CA 签发的客户端证书，CN=用户名，O=组 | 管理员、kubelet、控制面组件 | kubeadm 默认；注意证书轮转 |
| **ServiceAccount Token** | 签发的 JWT（1.24+ 需通过 TokenRequest API 动态申请，不再自动生成 Secret） | 集群内 Pod | 用 `projected volume` + 短时效 token |
| **OIDC / OpenID Connect** | 外部 IdP 签发 ID token | 人类用户、SSO | 企业首选，配合 `--oidc-*` 参数 |
| **Webhook Token Review** | 转发 token 给外部服务校验 | 对接自有账号体系 | 注意 webhook 是 APIServer 的强依赖，需高可用 |
| **Bootstrap Token** | 短期 token，用于节点引导加入 | 节点扩容 | 用完即删 |
| **Anonymous** | 匿名请求，映射到 `system:anonymous` | 默认开启 | **生产建议：**`--anonymous-auth=false` |

**关键追问**

- **ServiceAccount Token 从 1.24 起不再自动生成 Secret**（LegacyServiceAccountTokenNoAutoGeneration），必须调用 `TokenRequest` API。旧代码依赖 `sa.Secrets` 会失效。
- **证书里的 CN/O** → user/groups，这是 RBAC 绑定的依据（`system:masters` 组等价于 root，慎用）。

---

### Q5：RBAC 的鉴权流程？`Role` 和 `ClusterRole` 的关系？

**答案要点**

- RBAC 由 `rules`（apiGroups / resources / verbs / resourceNames）构成，通过 `RoleBinding`（命名空间级）或 `ClusterRoleBinding`（集群级）把主体（User/Group/ServiceAccount）绑定到角色。
- **鉴权顺序**：APIServer 按 `--authorization-mode` 配置的顺序依次询问各 authorizer（如 `Node,RBAC`），**任一 Allow 即放行，全部 NoOpinion 则拒绝**；没有 Deny 优先（RBAC 是白名单模型）。
- 决策被缓存（`--authorization-webhook-cache-*`，对 webhook 模式有效），RBAC 本身基于 informer 内存数据，规则变更有秒级传播延迟。

**反模式（高频考点）**

- 把 `cluster-admin` 绑给 ServiceAccount（等价于给每个 Pod root）。
- 用 `system:masters` 组作为日常操作账号 —— **绕过所有 RBAC 检查**，审计里也难以区分操作者。
- wildcard `resources: ["*"], verbs: ["*"]`。

**Node Authorizer 考点**

- `system:nodes` 组的 kubelet 只能读写**自己节点**相关的资源（Node、本节点的 Pod、本节点 Pod 的 Secret/ConfigMap、Lease、PV 附着等）。
- **NodeRestriction** 准入插件配合，防止 kubelet 越权改别人的 Pod / 给自身加 label（`node-restriction.kubernetes.io/` 前缀限制）。

---

### Q6：Admission Control 是什么？Mutating 和 Validating 的执行顺序？

**答案要点**

- Admission 是**对象持久化前的最后一道关卡**，能看到完整的请求对象和（变更阶段的）修改结果。
- **执行顺序**：
  1. **MutatingAdmission**（可以修改对象）：内置插件（NamespaceLifecycle、LimitRanger、ServiceAccount、DefaultStorageClass、PodSecurity…）→ **MutatingAdmissionWebhook**（内置 mutate 全部跑完后才调用外部 webhook，保证 webhook 看到最终形态）
  2. **ValidatingAdmission**（只读，不改对象）：内置校验插件 → **ValidatingAdmissionWebhook**（基本在最后）
- 原因：webhook 必须看到内置插件处理后的完整对象，否则注入的 sidecar 会被后续插件忽略。

**配置要点（webhook 必背参数）**

```yaml
failurePolicy: Fail        # Fail=webhook 挂了就拒绝请求（强依赖）；Ignore=放行（弱依赖）
sideEffects: None          # 声明无外部副作用，配合 dry-run
timeoutSeconds: 3          # 【建议值】≤5s；越长大面积超时越容易拖垮 APIServer
reinvocationPolicy: Never  # Mutating webhook 修改后是否再次调用
matchPolicy: Equivalent    # 是否匹配资源的等价版本
objectSelector: {}         # 缩小命中范围 —— 大规模集群最重要的性能开关
```

**大规模风险**

- webhook 是**同步阻塞**的：一个慢 webhook 会同步抬高所有请求的 P99，并占用 APIServer 的并发额度（inflight 请求持有内存）。
- 反模式：`failurePolicy: Fail` + 单点部署 + 无 `objectSelector`（命中全部 Pod 创建）。

---

### Q7：resourceVersion 是什么？乐观锁怎么实现？

**答案要点**

- `resourceVersion` 本质是 **etcd 的 MVCC revision**（全局单调递增的整数），并不透明 —— 但**语义上不保证可比较**，客户端只能做「相等/不等」判断，不能做大小比较。
- **乐观锁流程**：
  1. 客户端 GET 拿到 `metadata.resourceVersion`
  2. UPDATE 时带上该值
  3. APIServer 转换为 etcd 的 `txn`：`if revision == X then put else fail`
  4. 不匹配 → 返回 **409 Conflict**，客户端必须**重新 Get 再重试**（不能直接重试旧对象）
- **意义**：无锁并发，避免分布式锁开销；代价是写冲突时需要重试（热对象上写冲突明显）。

**实战建议**

- 客户端必须实现 **指数退避 + 冲突重试**（client-go 的 `retry.RetryOnConflict`）。
- 控制器里常见反模式：先 Get 再改再 Update，且缓存陈旧 → 频繁 409。

---

### Q8：List-Watch 机制是怎么工作的？Informer 为什么用它？

**答案要点**

1. **List**：informer 首次启动时做一次全量 List（可分页），拿到所有对象 + 当前 `resourceVersion`。
2. **Watch**：用该 `resourceVersion` 建立长连接，APIServer 持续推送增量事件（ADDED/MODIFIED/DELETED/BOOKMARK）。
3. **本地 Store + DeltaFIFO**：informer 把对象放本地 indexer 缓存，业务代码从缓存读，**不直接打 APIServer**。
4. **Resync**：周期性（数小时量级，【需实测】按集群规模配置）把本地对象重新入队触发 handler，**不产生 APIServer 请求**（这是常见误解 —— resync 不重新 List）。
5. **Relist**：当 watch 断连且资源版本过期（收到 410 Gone / ErrExpired）时，才重新 List。

**为什么不用轮询**：推模式把"读放大"从 O(客户端数 × 频率) 降到 O(变更数)；且 APIServer 侧有 watch cache，事件分发不落 etcd。

**大规模风险**

- informer **重建风暴**：APIServer 重启/滚动升级 → 所有客户端同时 relist → 瞬时 List 请求打爆 APIServer 和 etcd。
- 缓解：客户端 relist 加**抖动（jitter）**；升级 APIServer 时错峰；1.27+ 的 **WatchList** 特性（用 watch 代替 list 做初始化，走 watch cache，避免 etcd 全量读）值得关注。

---

### Q9：watch cache 是什么？会带来什么问题？

**答案要点**

- 每个（资源类型 × 版本）在 APIServer 内存中维护一个 **Cacher**：内部是 etcd 数据的全量快照 + 按 revision 索引的 **ring buffer**（事件历史）。
- **List 请求**：默认（不指定 resourceVersion 或 `resourceVersion=0`）从 watch cache 读，**不读 etcd**（性能好，但可能**读到几毫秒前的旧数据**）。
- **强一致读**：显式指定 `resourceVersion`（非 0）会穿透到 etcd 做 quorum read，保证不回退，但代价高。
- **Watch 请求**：直接消费 watch cache，不压 etcd。

**关键参数**

- `--watch-cache=true`（默认开）
- `--default-watch-cache-size`【默认值 100】—— ring buffer 槽位数
- `--watch-cache-sizes=pods#1000,nodes#500,namespaces#10` —— 按资源单独调

**失效模式**

- 客户端请求的 `resourceVersion` 太旧（已被 ring buffer 淘汰或被 etcd compact）→ 返回 **410 Gone / "too old resource version"** → 客户端必须 relist。
- ring buffer 太小 + 高频变更资源 → 大量 410 → relist 风暴 → etcd 读放大。
- 大对象（如 Pod 列表大）会吃掉大量 APIServer 内存。

**调优建议（需实测）**

- 观察指标：`apiserver_watch_cache_*`、`apiserver_cache_list_fetched_objects_total` vs `apiserver_cache_list_returned_objects_total`（两者接近说明缓存命中差/过滤发生在缓存之后）。
- 高频变更资源（pods、events、leases、endpointslice）适当放大 `--watch-cache-sizes`，但要核算 APIServer 内存：`槽位数 × 单对象平均大小 × 副本数`。

---

### Q10：Bookmark 事件 / Progress Notify 解决什么问题？

**答案要点**

- **问题**：客户端 watch 长时间无事件时，无法区分「确实没变更」还是「连接已假死但 TCP 未断」，也不知道自己落后到什么程度。
- **Bookmark**：APIServer 周期性发送一个**只有 resourceVersion、无对象体**的事件（`type=BOOKMARK`）。客户端更新本地"已知最新 RV"。
- **价值**：客户端重连时可以从 bookmark 记录的 RV 续传，而**不必从头 relist**；同时能检测到连接存活。
- **开启**：客户端 watch 时设置 `allowWatchBookmarks=true`。
- **Progress Notify**：APIServer 侧的周期性推进（可通过 `--watch-progress-notify-interval` 调整，**默认值随版本变化，请以目标版本 **`--help`** 或 kubeadm 配置为准**）。

**面试加分**：能说清「无变更资源（如 ConfigMap）的 watch 也必须有心跳，否则无法保活和推进 RV」。

---

### Q11：APIServer 与 etcd 的交互细节？etcd 里存的是什么格式？

**答案要点**

- 存储路径：`/registry/<resource-plural>/<namespace>/<name>`，例如 `/registry/pods/default/nginx`。
- 存储编码：**protobuf**（默认，二进制、体积小）而非 JSON；历史上是 JSON（老集群可用 `etcdctl get` 直接看，1.6+ 起默认 protobuf）。
- 每次写操作产生新的 **etcd revision**，旧版本保留（MVCC），靠 **compaction** 回收。
- APIServer 的写路径：`storage.Interface.GuaranteedUpdate` → etcd `txn`（compare-and-swap）。

**关键运维参数**

- `--etcd-compaction-interval`【默认值 5m】：APIServer 定期压缩 etcd 历史 revision（保留最近 5 分钟）。
- etcd `--quota-backend-bytes`【默认 2GB，生产常见 4–8GB】：超限后 etcd 触发 **NOSPACE alarm**，集群**只读**。
- etcd `--max-request-bytes`【默认约 1.5MB】：单个对象超限报 "request is too large"（Endpoint 大对象经典问题）。

---

### Q12：APIServer 如何做限流？从 `--max-requests-inflight` 到 APF 的演进？

**答案要点**

**旧方案（已过时）**：`--max-requests-inflight`（默认 400）+ `--max-mutating-requests-inflight`（默认 200）全局计数器 —— **问题**：无法区分请求来源，一个写 CRD 的批处理任务就能把 kubelet 心跳挤掉。

**新方案：API Priority and Fairness（APF，KEP-1040）**

- 自 **1.23 起默认开启**（Beta），后续版本 GA；启用后旧的两个 flag 失效/移除。
- **两个核心对象**：
  - `FlowSchema`：匹配规则（subjects 用户/组/SA + resourceRules/nonResourceRules）+ `matchingPrecedence` + `distinguisherMethod`（按 user 或 namespace 分流）。
  - `PriorityLevelConfiguration`：`assuredConcurrencyShares`（保证并发份额）+ `queuing`（queues / queueLengthLimit / handSize，实现**洗牌分片 shuffle-sharding**，避免一个 noisy neighbor 占满全部队列）。
- **exempt 优先级**：`system:masters` 等系统高优请求**不排队、不限流**，保证"再怎么炸，管理员还能登录排查"。
- **默认内置 FlowSchema**：`exempt`、`probes`、`system-leader-election`、`workload-leader-election`、`system-nodes`、`kube-controller-manager`、`kube-scheduler`、`kube-system-service-accounts`、`service-accounts`、`global-default`。
（较新版本引入 `node-high` 优先级，专门保障 kubelet 心跳。）

**诊断指标**

```
apiserver_flowcontrol_rejected_requests_total{priority_level,flow_schema,reason}
   # reason: queue-full / concurrency-limit / time-out / cancelled
apiserver_flowcontrol_current_inqueue_requests{priority_level,flow_schema}
apiserver_flowcontrol_request_queue_length_after_enqueue{priority_level,flow_schema}
apiserver_flowcontrol_dispatched_requests_total
apiserver_flowcontrol_request_exec_seconds / request_wait_seconds
```

**调优动作（灰度 + 回滚）**

- 给关键控制面组件单独建 FlowSchema、提 `assuredConcurrencyShares`。
- 给批处理/CRD 同步类客户端建**低优先级 + 有限队列**（或直接要求他们降速）。
- 回滚：删除自定义 FlowSchema/PriorityLevelConfiguration，系统会回落到 `global-default`（`kubectl delete` 即回滚，无状态，风险低）。

---

### Q13：APIServer 的 API 版本与 Discovery 机制？kubectl 为什么第一次执行慢？

**答案要点**

- API 组织为 **Group / Version / Resource (GVR)**；同一资源可有多个版本（`apps/v1`、`apps/v1beta1`），APIServer 内部用 **internal version** 做 trung gian，通过生成的 conversion 函数或 **conversion webhook** 转换。
- **Discovery**：`/api`、`/apis`、`/apis/<group>/<version>`、`/openapi/v2`、`/openapi/v3`。kubectl 首次运行会拉取 discovery 并缓存到 `~/.kube/cache/discovery`（TTL 约 10 分钟）。
- **为什么第一次慢**：需要先拉 discovery + OpenAPI 文档（几 MB），后续走本地缓存。
- **大规模风险**：discovery 文档随 CRD 数量膨胀；几百个 CRD 会让 `/openapi/v2` 达到数十 MB，kubectl 冷启动和 APIServer 内存都受影响。
  - 缓解：控制 CRD 数量、用 `kubectl --cache-dir`、让 CI 复用缓存、避免频繁创建无用的 CRD。
- **版本弃用（Deprecation）**：alpha（可能随时变）/ beta（兼容 3 个 minor 或 9 个月）/ GA（长期兼容）。面试常问：`extensions/v1beta1` Ingress、Policy API 的迁移。

---

### Q14：CRD 和 Aggregated API（APIService）的区别？什么时候该用聚合？

| 维度 | CRD | Aggregated API（APIService） |
| --- | --- | --- |
| 存储 | **共享集群 etcd** | **自带存储**（可独立 etcd / 数据库） |
| 实现 | 只写 YAML schema，无需代码 | 需要实现并部署一个扩展 APIServer |
| 能力 | 受限于内置策略（subresource、字段校验有限） | 任意 API 语义、自定义 subresource、任意存储后端 |
| 性能影响 | **直接吃主 etcd 的容量和 QPS** | 与主集群**故障隔离** |
| 典型例子 | cert-manager、Prometheus Operator CR | **metrics-server**、kube-aggregator 下的 service-catalog |

**选择原则（大规模集群重点）**

- 对象数量大 / 写 QPS 高 / 需要与主集群故障隔离 → **Aggregated API + 独立 etcd**（metrics-server 是典型：它的数据量极大，绝不进主 etcd）。
- 对象少、生命周期简单 → CRD。
- **反模式**：把百万级 CR（如自定义指标、审计记录）塞进 CRD → 主 etcd 被拖垮。

**聚合层转发链路**

```
kubectl get --raw /apis/metrics.k8s.io/v1beta1/nodes
  → kube-aggregator 匹配 APIService → 转发到 service/namespace
  → 扩展 APIServer 返回 → 聚合层原样返回
```

- APIService 不可用时，请求返回 **503 Service Unavailable**（`kubectl top` 的经典报错）。

---

### Q15：Server-Side Apply（SSA）与 `kubectl apply` 的字段管理？

**答案要点**

- 传统 `kubectl apply` 依赖 `metadata.annotations["kubectl.kubernetes.io/last-applied-configuration"]`（三路合并），多控制器同时改同一对象会**互相覆盖**。
- **SSA**：客户端发送 `PATCH` + `Content-Type: application/apply-patch+yaml`，携带 `?fieldManager=xxx&force=...`；APIServer 在 `metadata.managedFields` 中记录**每个字段由哪个 manager 拥有**。
- **冲突处理**：若目标字段已被其他 manager 拥有 → 返回 **409 Conflict（Apply failed）**；`force=true` 则强制抢占并转移所有权。
- **大规模注意**：`managedFields` 会**增大每个对象的体积**（etcd 存储 + 网络传输 + watch 事件），大规模集群可考虑关掉（`--feature-gates=ServerSideApply=false` 会失去能力，更常见做法是保留但精简字段管理器数量）。

---

### Q16：Secret 的静态加密（Encryption at Rest）怎么做？

**答案要点**

- 配置 `EncryptionConfiguration`，通过 `--encryption-provider-config` 挂载。

```yaml
apiVersion: apiserver.config.k8s.io/v1
kind: EncryptionConfiguration
resources:
  - resources: ["secrets", "configmaps"]     # 可扩展到 CRD 的 secret 类资源
    providers:
      - aescbc:                               # 或 aesgcm / kms v2
          keys:
            - name: key1
              secret: <base64 32字节随机密钥>
      - identity: {}                          # 兜底：读取未加密的旧数据
```

- **provider 顺序**：写入用**列表第一个**；读取时**逐个尝试**直到解出 → 支持密钥轮转（新 key 放最前，旧 key 保留可读）。
- **KMS v2（1.29 GA）**：密钥托管在云 KMS/HashiCorp Vault，APIServer 通过 unix socket 与 KMS plugin 通信。
- **运维要点**
  - 只能加密**新的写入**；存量需执行 `kubectl get secrets -A -o json | kubectl replace -f -` 强制重写。
  - 修改配置后必须**重启所有 APIServer**（滚动），且期间不要写入正在加密的资源类型。
  - **回滚预案**：保留 `identity` 在列表中，出问题可回退到明文读取；但已加密数据不可"降级"（除非重写）。

---

## Part 2：请求链路与对象语义（Q17–Q28）

### Q17：一个 Pod 从创建到被调度、启动，APIServer 参与哪些环节？

```
kubectl create pod
  ① APIServer: 认证/鉴权/准入 → etcd（Pod，spec.nodeName 为空）
  ② Scheduler: watch 到未绑定 Pod → 预选/打分 → 通过 APIServer 写回 binding（Pod.spec.nodeName）
  ③ Kubelet: watch 到调度到本节点的 Pod → 拉镜像、创建容器
  ④ Kubelet: 通过 APIServer 更新 Pod.status（容器状态、IP）
  ⑤ Kubelet: 更新 Node.status + Lease（心跳）
  ⑥ 各 controller: watch 并 reconcile（ReplicaSet/Endpoint/Service…）
```

**关键点**

- **APIServer 从不主动调用任何人**，一切靠 watch 驱动（声明式 API 的核心：控制循环 = 观察当前状态 → 计算期望差 → 写回）。
- 每个环节的写入都是一次独立的 etcd 写，写放大由此产生。

---

### Q18：为什么 APIServer 的 API 是"声明式"的？声明式和命令式的区别？

- **命令式**：`kubectl run`、`kubectl scale` —— 客户端告诉系统"做什么"，失败需要客户端重试，无法处理并发。
- **声明式**：`kubectl apply -f` —— 客户端声明"期望终态"，由 controller 的**控制循环**持续逼近，天然支持重试、幂等、自我修复。
- **代价**：
  - 终态收敛需要时间，观测上表现为"最终一致"，需要容忍短暂不一致。
  - 控制器 bug 会导致"抖动"（flapping）—— 两个控制器对同一字段期望不同，反复互相覆盖。

---

### Q19：`Finalizer`、`OwnerReference`、垃圾回收（GC）的原理？

- **Finalizer**：`metadata.finalizers` 非空时，DELETE 只会设置 `deletionTimestamp`（进入 Terminating），**对象不真正删除**，直到 finalizer 被移除。用于外部资源清理（PV、LoadBalancer、云盘）。
  - **故障模式**：finalizer 的 controller 挂了 → 对象卡在 Terminating → 需人工 `kubectl patch` 移除 finalizer（有风险，需确认外部资源已释放）。
- **OwnerReference + GC**：`garbage collector` controller 通过 watch 维护**依赖图**，删除 owner 时级联删除 dependent（`--cascade=orphan/background/foreground`）。
  - **大规模风险**：CRD 数量多、图庞大 → GC controller 内存和 CPU 吃紧；级联删除时产生**瞬时写风暴**。

---

### Q20：APIServer 的审计（Audit）机制？生产怎么配才不拖垮性能？

- **阶段**：`RequestReceived` → `ResponseStarted`（长连接 watch）→ `ResponseComplete` → `Panic`。
- **级别**：`None` / `Metadata`（只记元数据）/`Request`（含请求体）/`RequestResponse`（**含响应体，最贵**）。
- **后端**：`log`（文件）、`webhook`（外部收集）、`dynamic`（AuditSink，已废弃）。

**生产建议（性能优先）**

```yaml
apiVersion: audit.k8s.io/v1
kind: Policy
rules:
  # 1) 不记录只读低价值请求
  - level: None
    users: ["system:kube-proxy"]
    verbs: ["watch"]
    resources: [{group: "", resources: ["endpoints","services","endpointslices"]}]
  # 2) 不记录高频心跳
  - level: None
    users: ["system:serviceaccount:kube-system:generic-garbage-collector"]
    verbs: ["list","watch"]
  # 3) 敏感资源请求记元数据（不记 body，避免 Secret 明文落盘）
  - level: Metadata
    resources: [{group: "", resources: ["secrets","configmaps"]}, {group: "rbac.authorization.k8s.io"}]
  # 4) 写操作全记
  - level: Request
    verbs: ["create","update","patch","delete"]
  # 5) 兜底
  - level: Metadata
```

- **性能开关**：`--audit-log-maxsize/--audit-log-maxbackup/--audit-log-maxage`（日志轮转）、`--audit-webhook-batch-buffer-size` + `--audit-webhook-batch-max-size`（异步批量，避免 webhook 阻塞请求）。
- **反模式**：全量 `RequestResponse` 且写本地盘 → APIServer IO 打满、P99 飙升、磁盘被打爆。

---

### Q21：什么是 API 的幂等性？APIServer 怎么保证？

- **Create**：靠 `metadata.uid` + name 唯一性保证（同名重复创建返回 409 AlreadyExists）。
- **Update/Patch**：靠 `resourceVersion` 的乐观锁（409 冲突）。
- **Apply（SSA）**：靠 `managedFields` 幂等（相同输入重复 apply 不产生变更）。
- **Delete**：幂等（删除不存在的对象返回 404，客户端视为已完成）。
- **面试重点**：控制器代码必须**幂等**，因为 reconcile 会被重复触发（informer resync、事件重放、重启）。

---

### Q22：`--dry-run=server` 的作用和限制？

- 服务端执行完整的 **admission + validation + 默认值填充**，但**不持久化**（`dryRun=true` 传给 storage 层，etcd 写入被跳过）。
- **限制**：
  - 依赖 admission webhook 支持 dry-run（必须声明 `sideEffects: None` 或 `NoneOnDryRun`）。
  - 有副作用的 webhook（如自动创建云资源）必须声明 `sideEffects: Some`，否则 dry-run 时会被拒绝。
  - **不校验资源配额是否真的够**（配额是并发相关的）。
- **用途**：CI 里做 YAML 合规校验、策略预检（Kyverno/Gatekeeper 的 dry-run 模式）。

---

### Q23：APIServer 优雅关闭（Graceful Shutdown）与滚动升级会发生什么？

- `--shutdown-delay-duration`：收到 SIGTERM 后延迟关闭，让 LB 摘流、客户端感知。
- **关闭顺序**：先停止接收新请求 → 关闭 watch 连接 → 等待 inflight 请求完成（受 `--shutdown-send-timeout` 限制）。
- **客户端影响（大规模考点）**：
  - APIServer 实例下线 → 所有连到它的 watch 断连 → 客户端**同时**重连 + relist → **惊群（thundering herd）**。
  - 缓解：APIServer 多副本、错峰滚动、客户端带 jitter 的退避重连、适当调大 watch cache 使 relist 命中缓存。
- 相关特性：`--goaway-chance`（APIServer 随机发送 HTTP/2 GOAWAY，主动打散长连接，避免 LB 粘滞导致负载不均）—— **大规模集群建议开启并【需实测】调整**。

---

### Q24：kubelet 与 APIServer 的交互有哪些？为什么 Node 心跳要用 Lease？

- kubelet 会 **watch**：分配给本节点的 Pod、相关 Secret/ConfigMap、Service/EndpointSlice。
- kubelet 会 **写**：Pod status、Node status、Node Lease、CSI volume attach 状态、Event。
- **Node 状态更新频率**：`--node-status-update-frequency`【默认值 10s】（有变化时上报），`--node-status-report-frequency`【默认值 5m】（无变化时也强制上报）。
- **为什么引入 Lease**（1.14 GA）：
  - 每次更新完整 Node 对象 = 一次大对象写（Node.status 含 images、conditions、allocatable…），5000 节点 × 每 10s = **约 500 次写/s 的大对象写入**，etcd 直接被压垮。
  - Lease 是极小的对象（约 250B），心跳只更新 Lease；Node 对象的 status 只在**真正变化时**才写。
  - Node controller 通过 Lease 判定 NotReady（`--node-monitor-grace-period`【默认 50s】）。
- **大规模收益**：把节点心跳的 etcd 写压力降低一到两个数量级（**具体倍数需在你的集群上实测统计 **`apiserver_request_total{verb=PUT,resource=leases}`** 与 **`resource=nodes`** 的比例**）。

---

### Q25：Event 为什么是大规模集群的头号杀手？怎么处理？

- Event 由各组件高频产生，默认 `--event-ttl`【默认值 1h】。
- **问题**：Event 是 CR（events.k8s.io/v1），每个都是 etcd 写；异常时（如镜像拉取失败反复重试、CrashLoopBackOff）会**指数级放大**。
- **处理方案（按优先级）**
  1. **独立 etcd**：`--etcd-servers-overrides=/events#https://etcd-events-1:2379;...`，把 Event 从主 etcd 剥离（最有效）。
  2. 缩短 `--event-ttl`（如 30m，【建议值】需结合排障需求）。
  3. 审计策略里对 `system:serviceaccount:kube-system:*` 的 event 写入设为 `None`。
  4. 用 `--event-qps` / `--event-burst`（kubelet 侧）限流。
  5. 治理根因：修掉反复重启的 Pod，而不是只清 Event。

---

### Q26：Endpoint vs EndpointSlice？为什么要切分？

- 旧 Endpoint 对象：一个 Service 一个 Endpoint 对象，包含**所有**后端 IP。
- **问题**：etcd 单对象上限约 1.5MB（`--max-request-bytes`），大型 Service（上千 Pod）会导致：
  - 写入失败（"request is too large"）
  - 单次变更触发**全对象重写**（一个 Pod 变更 → 整个大对象重写 → 写放大）
  - 所有 kube-proxy watch 到巨大事件
- **EndpointSlice**（1.21 GA）：默认每个 Slice **最多 100 个 endpoint**（`--max-endpoints-per-slice`，【默认值 100】，可调 1–1000），变更粒度小、增量传播。
- **大集群收益**：kube-proxy 的 watch 事件体积和 etcd 写放大显著下降（**需实测对比**）。

---

### Q27：client-go 的限速（QPS/Burst）为什么会影响 APIServer？

- client-go 内置 **令牌桶 + 指数退避**：默认 QPS=5、Burst=10（**各组件版本间有差异，以实际启动参数为准**）。
- **大规模问题**：
  - 默认值太低 → 控制器启动慢、reconcile 积压（5000 节点下 kube-controller-manager 常年打满 QPS）。
  - 客户端被限速 → 大量 429 → client-go 退避重试 → 进一步堆积。
  - 客户端被 APF 拒绝（429 + `Retry-After`）→ 若客户端不遵守 `Retry-After`，重试风暴会雪上加霜。
- **配置示例（kubelet）**

```yaml
apiVersion: kubelet.config.k8s.io/v1beta1
kind: KubeletConfiguration
kubeAPIQPS: 50        # 【建议值】需结合节点数与 Pod 密度实测
kubeAPIBurst: 100
```

- **controller-manager / scheduler**：`--kube-api-qps` / `--kube-api-burst`。

---

### Q28：APIServer 的证书体系？证书过期会发生什么？

- **证书分类**：
  - `ca.crt`（集群 CA）
  - `apiserver.crt`（服务端证书，含 `--apiserver-cert-extra-sans` 中的所有 LB VIP/域名）
  - `apiserver-kubelet-client`（APIServer 访问 kubelet）
  - `front-proxy-ca/front-proxy-client`（聚合层 → 扩展 APIServer）
  - `apiserver-etcd-client` / `etcd-server` / `etcd-peer`
  - ServiceAccount 签名密钥对（`--service-account-key-file` / `--service-account-signing-key-file`）
- **过期后果**：整集群不可用（`x509: certificate has expired`），kubectl 全部失败。
- **运维**
  - kubeadm：`kubeadm certs check-expiration`、`kubeadm certs renew all`（**注意：renew 后需重启 APIServer/CM/Scheduler 的静态 Pod**）。
  - 监控：用 `apiserver_client_certificate_expiration_seconds` 设告警（**剩余 \< 30 天告警**）。
  - **反模式**：只在证书过期当天才发现；SA 签名密钥轮转未留旧 key 导致存量 token 全部失效。

---

## Part 3：大规模集群高频问题（Q29–Q46）

> 先记住官方"大规模集群"参考上限（**社区文档口径，随版本演进**）：
> **节点 ≤ 5000，Pod ≤ 150000，容器 ≤ 300000，单节点 Pod ≤ 110**。
> 超过这些量级需要专门调优，且必须自行压测验证。

### Q29：大规模集群下，APIServer 最先出现的瓶颈是什么？

按经验排序（**需在你的集群用监控验证顺序**）：

1. **etcd 写入放大**（Lease、Event、EndpointSlice、Pod status、CR 状态更新）
2. **etcd 读放大**（全量 List、无分页、watch cache miss、informer relist 风暴）
3. **APIServer 内存 / CPU**（watch cache 太大、大 List 请求、审计日志、protobuf 编解码）
4. **限流与排队**（APF 队列满、客户端限速）
5. **webhook 时延**（同步阻塞，直接计入请求 P99）
6. **网络与 LB**（长连接粘滞、单 LB 带宽、跨 AZ 时延）

---

### Q30：如何给 APIServer 做容量规划与压测？

**必测项**

- 稳态：`apiserver_request_total` 按 verb/resource 拆分的总 QPS。
- 时延：`histogram_quantile(0.99, sum(rate(apiserver_request_duration_seconds_bucket[5m])) by (le, verb, resource))`。
- 并发：`apiserver_current_inflight_requests`、`apiserver_flowcontrol_current_inqueue_requests`。
- 对象数：`etcd_debugging_mvcc_keys_total`、`apiserver_storage_objects`（按资源）。
- etcd 健康：`etcd_disk_wal_fsync_duration_seconds`（**P99 应 \< 10ms，这是硬指标**）、`etcd_server_slow_apply_total`、`etcd_mvcc_db_total_size_in_bytes`。

**压测方法**

- 用 `kubemark`（hollow-node）模拟大规模节点，比真买机器便宜。
- 用真实业务流量的**影子回放**（录制 `apiserver_request_*` 分布，用脚本重放）。
- **压力测试必须在非生产集群做**；生产只做**灰度观察**。

---

### Q31：etcd 出现 `mvcc: database space exceeded`（NOSPACE）怎么救？

**原因**：db 达到 `--quota-backend-bytes`，etcd 触发 alarm 并进入**只读**。

**修复步骤（生产操作需审批 + 变更窗口）**

```bash
# 1) 查看状态（先确认 db size 与 revision）
ETCDCTL_API=3 etcdctl --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  endpoint status --write-out=table

# 2) 先看 alarm
etcdctl alarm list

# 3) 压缩到当前 revision（会短暂影响读）
rev=$(etcdctl endpoint status --write-out="json" | jq -r '.[0].Status.header.revision')
etcdctl compact "$rev" --physical

# 4) 每个成员逐个 defrag（defrag 会阻塞该成员，务必逐个做、且先做 follower）
etcdctl defrag --endpoints=... --command-timeout=60s

# 5) 解除 alarm（不解除则仍只读）
etcdctl alarm disarm
```

**根因治理**

- 确认 APIServer 的 `--etcd-compaction-interval` 正常生效（默认 5m）。
- 排查写放大源头（Event/Lease/CR 高频写）。
- 提高 `--quota-backend-bytes`（**提高配额只是延后爆炸时间，不是解法**）。
- 定期 defrag（**低峰期、逐个成员、有 leader 切换预案**）。

**回滚/兜底**：操作前必须有 etcd **快照备份**（`etcdctl snapshot save`），灾难恢复演练至少每季度一次。

---

### Q32：如何减少 etcd 的写放大？（综合题，必考）

| 手段 | 做法 | 收益等级 |
| --- | --- | --- |
| Node 心跳用 Lease | 默认开启，确认 kubelet 未禁用 | 高 |
| Event 独立 etcd | `--etcd-servers-overrides=/events#...` | 高 |
| Event TTL 缩短 | `--event-ttl=30m` | 中 |
| EndpointSlice | 默认开启，调 `--max-endpoints-per-slice` | 中 |
| 控制 CR 状态更新频率 | 控制器里只在**状态真正变化**时 Update；用 `apiequality.Semantic.DeepEqual` 比较 | 高 |
| 避免 `status` 与 `spec` 打同一对象高频写 | 用独立 CR 存状态 | 中 |
| 关闭不必要的 controller / addon | 每个 controller 都有 watch+写 | 中 |
| 大 CRD 迁移到聚合 API | 独立 etcd | 高 |

**代码层反模式（面试加分）**

```go
// 反模式：无条件 Update，即使 status 没变也会产生一次 etcd 写 + 一次事件广播
s.Status.ObservedGeneration = s.Generation
r.Status().Update(ctx, s)   // ← 每次 reconcile 都写

// 正确：先比较
if !apiequality.Semantic.DeepEqual(old.Status, new.Status) {
    r.Status().Update(ctx, s)
}
```

---

### Q33：如何减少 etcd 的读放大 / List 压力？

1. **禁止全量 List**：所有客户端必须分页（`limit` + `continue`）或用 informer。
2. `resourceVersion=0`：List 请求显式带 `resourceVersion=0` → 走 watch cache，不压 etcd（默认行为已是缓存读，但显式声明更清晰）。
3. **字段/标签选择器下推**：`--field-selector`、`label selector` 在 APIServer 侧过滤（注意：watch cache 只对 **namespace + name + 部分 field** 建索引，其他选择器需要在缓存后遍历 → 观察 `apiserver_cache_list_fetched_objects_total` 判断浪费）。
4. **Metadata-only 请求**：只要元数据时用 `Accept: application/json;as=PartialObjectMetadataList`（client-go 的 `metadataOnly` option），体积和 CPU 大幅下降。
5. **共享 informer**：同一进程内多个控制器复用同一个 SharedInformer，避免重复 watch。
6. **减少客户端数量**：DaemonSet 类 agent 尽量每个节点只跑一份、用同一个 informer。

---

### Q34：为什么 APIServer 会 OOM？怎么排查与预防？

**常见原因**

- **大 List 请求**：一次性 List 全部 Pod/Event（不带 limit）→ 服务端序列化巨型对象，内存峰值 = 对象数 × 单对象大小 × 并发度。
- **watch cache 过大**：`--watch-cache-sizes` 调得过大，尤其是 pods/events。
- **审计日志 **`RequestResponse`** 全量**：请求体在内存堆积 + 磁盘爆。
- **大量并发 watch**：每个 watch 连接在服务端持有 buffer。
- **巨型 CRD 对象**（单个 CR 几 MB）。

**排查**

```
# 1) 找出大请求
topk(10, sum by (resource,verb,user) (rate(apiserver_request_duration_seconds_sum[5m])))
# 2) 找出返回对象数最多的 list
topk(10, apiserver_storage_list_fetched_objects_total)
# 3) 进程内存与 GC
container_memory_working_set_bytes{container="kube-apiserver"}
go_goroutines{job="apiserver"}     # goroutine 泄漏
```

- `kubectl get --raw /debug/pprof/heap` 抓 heap profile（需开启 profiling，注意安全）。

**预防（可直接套用）**

- 强制分页：自研平台在 API 网关层强制注入 `limit`。
- 合理设置 `requests/limits`（**内存 limit 需实测后设置，且 APIServer 内存与对象数强相关，不要照抄别人家的值**）。
- 审计策略降级为 `Metadata`。
- 用 `--max-request-bytes` 限制单个请求体大小。

---

### Q35：APF 队列被打满（大量 429 / `queue-full`）怎么定位？

**定位三步**

```promql
# 1) 谁在被拒绝
sum by (flow_schema, priority_level, reason) (
  rate(apiserver_flowcontrol_rejected_requests_total[5m])
)
# 2) 队列积压情况
sum by (priority_level, flow_schema) (apiserver_flowcontrol_current_inqueue_requests)
# 3) 等待时长（判断是否只是排队慢而非拒绝）
histogram_quantile(0.99,
  sum by (le, flow_schema) (rate(apiserver_flowcontrol_request_wait_seconds_bucket[5m]))
)
```

**根因与对策**

| 现象 | 根因 | 对策 |
| --- | --- | --- |
| `global-default` 大量 `queue-full` | 无分类的客户端太多 | 给主要客户端建专属 FlowSchema |
| 某个业务 SA 占满 `workload-low` | noisy neighbor | 降低其 `assuredConcurrencyShares` 或用 `distinguisherMethod: ByUser` 隔离 |
| `concurrency-limit` | 总并发超配 | 提升对应 PL 的 shares（注意总盘子有限，此消彼长） |
| `time-out` | 请求堆积太久 | 缩短 `queues`/`queueLengthLimit` 让失败更快，客户端退避重试 |

**变更纪律**：APF 对象变更是**热生效、无状态**，`kubectl delete` 即回滚；但仍建议先在灰度集群验证。

---

### Q36：5000 节点集群，kubelet 的心跳与 Pod status 会给 APIServer 多大压力？

**量级估算方法（不给死数字，给算式，面试时这样答最稳）**

```
Lease 写 QPS      ≈ 节点数 / node-status-update-frequency(10s)        = 5000/10 = 500 QPS
Pod status 写 QPS  ≈ 稳态很小；但滚动发布/驱逐时会瞬时放大（需实测）
Event 写 QPS       ≈ 异常期可达数千（需实测），稳态接近 0
EndpointSlice      ≈ 与 Service 变更频率相关
```

把每一项都变成 PromQL 实测：

```
sum(rate(apiserver_request_total{verb=~"PUT|PATCH",resource="leases"}[5m]))
sum(rate(apiserver_request_total{verb=~"PUT|PATCH",resource="pods"}[5m]))
sum(rate(apiserver_request_total{verb="create",resource="events"}[5m]))
```

**降压手段**

- 心跳间隔（**改这个会影响故障检测时延，必须评估 SLO**）：`--node-status-update-frequency` 从 10s 调到 20s 可降一半写，但 Node NotReady 检测时延从 50s 起算会增加 → **必须与业务 SLO 一起决策**。
- 独立 etcd 放 Lease（`--etcd-servers-overrides=/registry/leases#...`）。
- 只在必要时上报 Node status（默认机制已做）。

---

### Q37：APIServer 多副本 + LB 会遇到什么坑？

1. **HTTP/2 连接粘滞**：客户端复用长连接 → 大量连接落在少数 APIServer 实例上 → 负载不均。
   - 对策：`--goaway-chance`（随机断开让客户端重连，重新打散），LB 层用 least-connection / 连接数上限。
2. **watch 长连接**：watch 是长连接，LB 的 idle timeout 会切断 → 客户端需正确处理重连（`410`/EOF → relist）。
3. **LB 健康检查**：用 `/readyz`（不要用 `/healthz`，后者含更多检查可能误摘流）；用 `/livez` 做存活。
4. **跨 AZ 流量**：etcd 有 leader，跨 AZ 的写会增加时延 → APIServer 与 etcd **同 AZ 部署优先**。
5. **滚动升级惊群**：见 Q23。

---

### Q38：`/healthz`、`/readyz`、`/livez` 有什么区别？

- `/healthz`：传统健康检查（1.16 起被拆分），现在默认等同于 ping + log + 若干检查。
- `/readyz`：**是否可接流量**（含 etcd 连通性、poststarthook 完成状态）。LB 必须用这个。
  - `kubectl get --raw='/readyz?verbose'` 可以看每一项。
- `/livez`：**是否存活**（不含会导致重启的依赖检查，避免级联重启）。

**排障技巧**：`/readyz?verbose` 能直接告诉你哪个 post-start hook（如 `aggregator-openapi`、`crd-informer-synced`）没就绪 —— 聚合 APIService 挂了导致 APIServer 不 Ready 是经典故障。

---

### Q39：大集群下 CRD 的管理有哪些坑？

1. **CRD 数量膨胀 discovery/OpenAPI**（见 Q13）。
2. **CR 数量巨大** → 主 etcd 容量与写压力 → 迁移到聚合 API（Q14）。
3. **CRD 的 schema 校验开销**：复杂 `openAPIV3Schema` 会显著增加 APIServer CPU（每次请求都要校验）。
   - 对策：避免超深嵌套 schema；必要时用 x-kubernetes-validations（CEL）但注意性能。
4. **CRD 版本转换**：conversion webhook 是同步调用，与 admission webhook 一样有 P99 风险。
5. **finalizer 卡死** → 大量 Terminating CR 堆积。
6. **CRD 删除会连带删除所有 CR**（危险操作，需审批）。

---

### Q40：APIServer 的性能调优参数清单（可以当 checklist 背）

| 参数 | 作用 | 备注 |
| --- | --- | --- |
| `--enable-priority-and-fairness=true` | APF 限流 | 1.23+ 默认 |
| `--watch-cache-sizes=` | 按资源调 watch cache | 【需实测】 |
| `--default-watch-cache-size` | 默认槽位（默认 100） | 【默认值】 |
| `--etcd-compaction-interval=5m` | 历史 revision 压缩 | 【默认值】 |
| `--event-ttl=1h` | Event 保留时长 | 【默认值】 |
| `--etcd-servers-overrides=` | 按资源前缀拆分 etcd | 高收益 |
| `--max-requests-inflight` | 旧限流（APF 启用后无效） | 注意版本 |
| `--goaway-chance` | 打散 HTTP/2 长连接 | 【需实测】 |
| `--shutdown-delay-duration` | 优雅关闭延迟 | 配合 LB 摘流 |
| `--audit-*` | 审计策略与异步批量 | 见 Q20 |
| `--encryption-provider-config` | 静态加密 | 见 Q16 |
| `--anonymous-auth=false` | 关闭匿名 | 安全基线 |
| `--tls-cipher-suites` | 限定 TLS 套件 | 安全基线 |
| `--request-timeout` | 非长连接请求超时 | 【默认值】1m（1.30+ 有变更） |

---

### Q41：为什么"每节点 Pod 数 ≤ 110"这个限制存在？

- **本质不是 110 个 Pod 放不下**，而是：
  1. 每节点 Pod 太多 → 单个 Node 对象上的 Pod 列表变大 → kubelet 的 watch 事件变多
  2. IPv4 地址耗尽（每个 Pod 一个 IP）
  3. 一个节点故障影响面过大（爆炸半径）
  4. 单节点上的容器密度导致内核资源（cgroup、inode、进程数）压力
- 突破手段：更大的 Pod CIDR、IPv6/dual-stack、节点池拆分、提高密度前先压测 kubelet + containerd。

---

### Q42：多集群 / 联邦 vs 单集群扩容，怎么选？

| 方案 | 优点 | 代价 | 适用 |
| --- | --- | --- | --- |
| **单集群纵向调优** | 运维简单、资源池统一、调度全局最优 | 有物理上限、爆炸半径大 | 绝大多数场景（\<3000 节点优选） |
| **拆分多个集群** | 故障隔离、可按业务/租户定制 | 多套控制面、跨集群服务发现/调度复杂 | 多租户强隔离、跨地域 |
| **联邦（KubeFed / 自研）** | 统一分发 | 成熟度低、语义复杂、最终一致难保证 | 谨慎评估 |

**面试观点**：**优先把单集群调优到极致，再考虑拆分**。联邦的复杂度远高于收益，除非有硬性隔离/地域要求。

---

### Q43：如何为 APIServer 设定 SLO？

**示例 SLO（【建议值】，需与业务共同确定）**

| SLO | 目标 | 测量方式 |
| --- | --- | --- |
| 可用性 | 99.95%（月度） | `1 - (5xx + 超时) / 总请求`，数据源 `apiserver_request_total{code=~"5.."}` |
| 读请求时延 | P99 \< 1s（LIST/GET 内置资源） | `apiserver_request_duration_seconds` |
| 写请求时延 | P99 \< 1.5s | 同上 |
| 限流拒绝率 | \< 0.1% | `apiserver_flowcontrol_rejected_requests_total` |
| etcd WAL fsync | P99 \< 10ms | `etcd_disk_wal_fsync_duration_seconds` |

**错误预算**：99.95% → 月度错误预算约 **21.9 分钟**。预算耗尽即冻结变更（这是 SRE 的核心纪律）。
**注意**：APIServer 的 SLO ≠ 业务 SLO。要单独定义"控制面可用"与"业务可用"，前者不可用会导致**无法自愈**（不能调度、不能扩容），影响是长尾的。

---

### Q44：APIServer 挂了，正在运行的业务会怎样？

- **已运行的 Pod 不会立即挂**：kubelet 本地有期望状态，容器继续跑；网络（CNI 数据面）不依赖 APIServer。
- **会立刻失效的**：
  - 无法调度新 Pod、无法扩缩容（HPA 失效）
  - Pod 重建失败（重启后 kubelet 需要 watch 新 Pod 才能创建 —— 实际上 kubelet 本地有 checkpoint，能短暂维持，但无 APIServer 时无法感知变更）
  - Service DNS / EndpointSlice 不再更新（新扩的副本不会进负载均衡）
  - `kubectl` 全部失败
  - 节点 NotReady 后无法驱逐（**雪崩风险**）
- **结论**：APIServer 的 MTTR 直接决定恢复能力 → 必须做**慢启动演练**（定期演练"APIServer 全挂 → 恢复"）。

---

### Q45：如何做 APIServer 的可观测性建设？（三位一体）

**Metrics（关键指标）**

```
apiserver_request_total{code,verb,resource,subresource}
apiserver_request_duration_seconds_bucket{verb,resource,scope}
apiserver_current_inflight_requests
apiserver_flowcontrol_*            （限流全家族）
apiserver_cache_list_fetched_objects_total / _returned_objects_total
apiserver_watch_events_sizes_bucket
etcd_request_duration_seconds{operation,type}
etcd_mvcc_db_total_size_in_bytes / etcd_debugging_mvcc_keys_total
etcd_disk_wal_fsync_duration_seconds_bucket
etcd_server_slow_apply_total / etcd_server_leader_changes_seen_total
apiserver_client_certificate_expiration_seconds
aggregator_unavailable_apiservice_total
```

**Logs**

- APIServer 日志级别 `--v=2`（生产），排障临时 `--v=4`（**注意：高 verbosity 性能开销明显，只能短时间开**）。
- 审计日志 → 集中采集（审计是"谁改了什么"的唯一真相来源）。

**Tracing**

- 关键：**APF 排队的等待时间**要单独看（`apiserver_flowcontrol_request_wait_seconds`），因为它常常被误认为是"APIServer 慢"，实际是"被限流了"。

**告警降噪原则**

- 只对 **SLO 相关信号**告警（`5xx burn rate`、P99 时延、etcd fsync、证书过期、leader 切换频繁）。
- 不要对"单个请求 409"告警（这是正常的并发冲突）。

---

### Q46：举一个你做过的大规模集群 APIServer 优化案例（开放式题的答题框架）

用 **STAR + 数据** 结构回答：

1. **Situation**：规模（节点/Pod/CR 数量）、业务特征、部署方式。
2. **Problem/痛点**：用**指标**描述（例如"APIServer P99 从 200ms 劣化到 3s，etcd WAL fsync P99 达 40ms，DB 接近 8GB 配额"）。
3. **Action**：分层治理（写放大 / 读放大 / 限流 / webhook / 客户端），每一步给出配置片段与灰度比例。
4. **Result**：**量化前后对比**（P99、QPS、DB size、告警数），并说明回滚条件与验证方式。
5. **Learn**：沉淀为规范/自动化检查（如 CI 里禁止无 limit 的 List、CRD 上线评审 checklist）。

> 面试官真正想听的是：**你有没有用数据说话，有没有灰度与回滚意识。**

---

## Part 4：故障排查实战（Q47–Q58）

### Q47：`409 Conflict` 是什么？怎么处理？

- **含义**：乐观并发冲突（resourceVersion 不匹配），或 SSA 的字段所有权冲突。
- **处理**：
  - Update 类：重新 GET → 应用变更 → 重试（client-go：`retry.RetryOnConflict`）。
  - Apply 类：查看 `managedFields` 找出冲突的 manager，决定 `force=true` 还是让对方先让出字段。
- **不是故障**：分布式系统里 409 是正常现象，只要比例不高（**建议监控 409 占比，突增说明有控制器在打架**）。

---

### Q48：`429 Too many requests` 有哪几种来源？

| 来源 | 特征 | 处理 |
| --- | --- | --- |
| **APF 拒绝** | 响应体含 "Too many requests"，apiserver 侧 `apiserver_flowcontrol_rejected_requests_total` 上升 | 调 FlowSchema/PL，或让客户端降速 |
| **client-go 令牌桶** | 客户端本地限速，**不会**在 APIServer 侧留下 429 | 调 `--kube-api-qps/burst` |
| **EventRateLimit 准入插件** | 针对 Event 资源限流 | 调 EventRateLimit 配置或治理 Event 源 |
| **云厂商 API 网关/限流** | 托管集群特有 | 提配额 |

**关键**：先看响应是否有 `Retry-After`，客户端**必须遵守**它，否则重试风暴。

---

### Q49：`etcdserver: request timed out` / `context deadline exceeded` 怎么排查？

**定位顺序**

1. **磁盘**：`etcd_disk_wal_fsync_duration_seconds` P99（**\>10ms 就是磁盘不达标**）。
   - 物理机：确认 SSD/NVMe，不是网络盘（云盘需注意 IOPS 与突发配额）。
   - 压测：`fio --rw=write --ioengine=sync --fdatasync=1 --size=22m --bs=2300 --name=mytest`（社区推荐姿势）。
2. **网络**：`etcd_network_peer_round_trip_time_seconds`（**同 AZ 应 \< 数 ms；跨 AZ 部署是常见坑**）。
3. **CPU/内存**：etcd 进程是否被打满（与大 APIServer 抢资源）。
4. **db 大小**：接近 quota → 写放大 → 慢。
5. **leader 抖动**：`etcd_server_leader_changes_seen_total` 频繁上升 → 检查网络/磁盘/心跳参数（`--heartbeat-interval`【默认 100ms】、`--election-timeout`【默认 1000ms】，跨地域部署需要调大）。

---

### Q50：`the server could not find the requested resource` 怎么排查？

按顺序检查：

1. **API 版本/路径写错**（GVR 拼错）。`kubectl api-resources` 确认。
2. **CRD 未创建或未 Established**：`kubectl get crd <name> -o yaml` 看 `status.conditions[Type=Established]`。
3. **聚合 APIService 不可用**：`kubectl get apiservices` 看 `Available=False` → 常见原因：扩展 APIServer Pod 挂了、Service/Endpoint 不通、证书校验失败（需 `--enable-aggregator-routing` 或 CA 配置正确）。
4. **RBAC 拒绝时**是 `403 Forbidden`（不是 404）—— 注意区分：现代 APIServer 对无权限且资源不存在的情况可能返回 404 以避免资源枚举。
5. **多集群/上下文错误**：`kubectl config current-context`。

---

### Q51：`Internal error occurred: failed calling webhook "xxx"` 怎么排查？

**排查清单**

1. webhook Pod 是否 Running / 副本数是否为 1（**单点** → 就绪即断流）。
2. **证书**：webhook 的 `caBundle` 是否匹配服务端证书、是否过期（TLS 错误会明确提示 x509）。
3. **网络**：APIServer → webhook Service 的连通性（CNI 策略、NetworkPolicy）。
4. **超时**：`timeoutSeconds` 是否太小（默认 10s，建议 3–5s）。
5. **selector**：是否命中了不该命中的请求（导致放大）。
6. **failurePolicy**：`Fail` 时 webhook 不可用 = 业务不可用。关键路径 webhook 应有多副本 + PDB + 独立节点池。

**应急**（需审批）：把 `failurePolicy` 临时改为 `Ignore`，或删除 webhook 配置。
**风险控制**：改 Ignore 意味着策略失效（安全/合规风险），必须同步通知安全团队并在事后补齐。

---

### Q52：`kubectl exec/logs` 失败与 APIServer 的关系？

- `exec/logs/attach/port-forward` 走的是 **SPDY/WebSocket 升级**（`/api/v1/ns/x/pods/y/exec`），再经 APIServer **反向代理到 kubelet 的 10250 端口**。
- **依赖**：
  - `apiserver-kubelet-client` 证书被 kubelet 信任（`--client-ca-file`）
  - kubelet 的 `--anonymous-auth=false` + webhook 授权（APIServer 需要 `nodes/proxy`、`nodes/log` 权限 —— RBAC `system:kubelet-api-admin` 或内置 `system:kube-apiserver-to-kubelet` ClusterRole）
  - 节点地址类型（`--kubelet-preferred-address-types=InternalIP,Hostname,...`）
- **常见报错**：`error: unable to upgrade connection: Forbidden`、`x509: certificate signed by unknown authority`、`pod does not exist`。
- **大规模注意**：exec/logs 是**长连接**，计入 inflight 但走 long-running 通道，不被 APF 排队 —— 大量 exec（如 CI）会占用连接资源。

---

### Q53：APIServer 日志里 `too old resource version` 是什么意思？

- 客户端请求的 `resourceVersion` 已被 watch cache 淘汰（超出 ring buffer）或已被 etcd compact。
- 返回 **410 Gone**，客户端必须 **relist**。
- **大规模危害**：如果大批量客户端同时收到 410 → 同时 relist → **惊群**。
- **缓解**：
  - 调大对应资源的 `--watch-cache-sizes`
  - 客户端开启 bookmark（`allowWatchBookmarks=true`）保持 RV 推进
  - 客户端 relist 加 jitter
  - 排查为什么客户端会持有过旧的 RV（例如 informer 长时间卡住、GC 停顿）

---

### Q54：如何判断是"APIServer 慢"还是"etcd 慢"还是"被限流了"？

**决策树**

```
请求 P99 高
 ├─ apiserver_flowcontrol_request_wait_seconds 高
 │    → 被 APF 限流/排队（不是慢，是被限）
 ├─ etcd_request_duration_seconds{operation=write} 高
 │    → etcd 慢 → 看 etcd_disk_wal_fsync_duration_seconds
 │         ├─ fsync 慢 → 磁盘问题（换 NVMe / 调 IOPS）
 │         └─ fsync 快但 request 慢 → 网络/leader/大对象
 └─ 两者都正常但 request_duration 高
      → APIServer 自身瓶颈：CPU（编解码/校验）、GC、watch cache miss、webhook
         看 apiserver_flowcontrol_request_exec_seconds 与 webhook 耗时
```

---

### Q55：APIServer 无法启动 / CrashLoopBackOff 常见原因？

1. **etcd 不可达或证书错误**（最常见）：`--etcd-servers`、`--etcd-cafile/certfile/keyfile`。
2. **端口/地址冲突**：`--advertise-address`、`--bind-address`、`--secure-port`。
3. **证书 SAN 不含 LB 地址**：客户端报 x509。
4. `--service-cluster-ip-range`** 与已有 Service CIDR 冲突**。
5. **静态 Pod manifest 语法错误**（kubeadm 场景看 `/etc/kubernetes/manifests/kube-apiserver.yaml`）。
6. **内存 limit 太小** → OOMKilled（`dmesg`/`kubectl describe` 看 Last State）。
7. **feature gate / flag 在升级后被移除** → 启动即失败（**升级前必须校验 flag 兼容性**）。

---

### Q56：APIServer 升级需要注意什么？（变更管理题）

**标准流程**

1. **兼容性校验**：目标版本的废弃 API、移除的 flag、feature gate 状态变化（`kubeadm upgrade plan`、官方 changelog）。
2. **备份**：etcd snapshot + 静态 Pod manifest 备份。
3. **灰度**：先升级 1 个 APIServer 副本并观察（`--shutdown-delay-duration` 配合 LB 摘流）。
   - **灰度比例建议**：控制面 1/N → 观察 30 分钟（**【建议值】需按业务 SLO 调整**）→ 全量。
4. **观察指标**：5xx 率、P99、APF 拒绝数、etcd 慢请求、`/readyz`。
5. **回滚触发条件（明确写死）**：
   - 5xx 率 \> 1% 持续 5 分钟
   - P99 较基线劣化 \> 2 倍持续 10 分钟
   - APF 拒绝率 \> 1%
   - 关键 webhook 大面积失败
6. **回滚方式**：还原静态 Pod manifest 到旧镜像 tag；**注意 API 版本兼容性 —— 一旦新版本写入了新字段/新存储版本，回退可能无法读取**（所以必须先备份 etcd 并确认 storageVersion 迁移策略）。

---

### Q57：如何保护 APIServer 免于"一个坏客户端拖垮全集群"？

分层防御（**面试高分答案**）：

1. **客户端侧**：client-go 令牌桶限速、共享 informer、分页、退避重试遵守 `Retry-After`。
2. **APF 侧**：为不同客户端分配 FlowSchema + 有限队列 + `distinguisherMethod: ByUser` 隔离噪声源；关键系统流量走高优先级。
3. **准入侧**：`EventRateLimit`、`ResourceQuota`、webhook 的 `objectSelector` 缩小范围。
4. **存储侧**：etcd 拆分（`--etcd-servers-overrides`）、独立 etcd 实例。
5. **网关侧**：自研平台在 APIServer 前加一层 API 网关做**按租户限流**（托管集群常见做法）。
6. **治理侧**：CRD 上线评审（对象量级预估）、禁止无 limit 的 List、定期巡检 `apiserver_request_total` TopN 用户。

---

### Q58：说出 5 个 APIServer 的反模式（收尾题，最容易翻车）

1. 业务代码/运维脚本**直接读写 etcd**。
2. 给 ServiceAccount 绑 `cluster-admin`。
3. 用 `system:masters` 做日常操作。
4. 审计策略全量 `RequestResponse`（性能 + Secret 明文落盘双重风险）。
5. webhook `failurePolicy: Fail` + 单副本 + 无限超时 + 无 selector。
6. 客户端全量 List 不分页、不复用 informer。
7. 单 etcd 承载所有资源（含 Event/metrics），且不设监控告警。
8. 把百万级对象做成 CRD 塞进主 etcd。

---

## Part 5：反模式与风险清单

> 每条含：反模式 / 证据 / 风险等级 / 修复动作 / 验证方式。

| # | 反模式 | 证据（现象/配置） | 等级 | 修复动作 | 验证方式 |
| --- | --- | --- | --- | --- | --- |
| 1 | 全量 List 不分页 | `apiserver_storage_list_fetched_objects_total` 单请求数万；APIServer 内存尖峰 | 高 | 客户端强制 `limit` + `continue`；改用 informer | 观察单次 List 返回对象数与 APIServer RSS |
| 2 | Event 与主资源共 etcd | `etcd_mvcc_db_total_size_in_bytes` 快速逼近 quota | 高 | `--etcd-servers-overrides=/events#<独立etcd>`；`--event-ttl=30m` | DB size 增长率下降；quota 使用率 \< 60% |
| 3 | 无差别 `failurePolicy: Fail` webhook | webhook Pod 单点；APIServer 报 failed calling webhook | 高 | 多副本 + PDB + `objectSelector` + `timeoutSeconds≤5`；关键路径评估改 `Ignore` | 混沌演练：kill 一个 webhook Pod，Pod 创建成功率仍 ≥ SLO |
| 4 | 控制器无条件 Update status | etcd 写 QPS 与 reconcile 次数 1:1 | 中高 | `DeepEqual` 比较后再写 | `rate(apiserver_request_total{verb=PUT,resource=<cr>})` 下降 |
| 5 | 百万级 CR 走 CRD | 主 etcd keys 数千万、DB 超限 | 高 | 迁移到 Aggregated API + 独立存储 | 主 etcd DB size 与 keys 数下降 |
| 6 | 审计全量 RequestResponse | 审计日志 TB/天；APIServer CPU/IO 高 | 中高 | 降级 `Metadata`；异步批量后端 | APIServer P99 回落；日志量下降 |
| 7 | 无 APF 分类，全走 global-default | `apiserver_flowcontrol_rejected_requests_total{flow_schema="global-default"}` 高 | 中高 | 为核心组件建 FlowSchema | 拒绝率 \< 0.1%；等待时长 P99 达标 |
| 8 | HTTP/2 长连接粘滞 | 各 APIServer 副本 CPU/QPS 严重不均 | 中 | `--goaway-chance`；LB least-conn | 副本间 CPU 差值 \< 20% |
| 9 | 使用 `system:masters` 日常运维 | kubeconfig 中 `O=system:masters` | 中高 | 改用最小权限 RBAC 账号 | 审计日志可追溯到具体人；RBAC 生效 |
| 10 | 升级前不校验 flag/API 兼容性 | APIServer CrashLoopBackOff | 中高 | `kubeadm upgrade plan` + 变更评审 | 灰度副本 `/readyz` 正常、5xx 无上升 |
| 11 | 证书过期无告警 | 集群整体不可用 | 高 | 监控 `apiserver_client_certificate_expiration_seconds`，\<30 天告警 | 演练：`kubeadm certs renew all` + 重启静态 Pod |
| 12 | 无 etcd 备份与恢复演练 | 故障后无法恢复 | 高 | 定期 snapshot + 每季度演练 | 演练 RTO/RPO 达标并留档 |

---

## Part 6：可直接套用的配置片段

### 6.1 APF：为关键组件提优先级、为批处理降优先级

```yaml
# 1) 给核心控制器一个专属高优先级
apiVersion: flowcontrol.apiserver.k8s.io/v1
kind: PriorityLevelConfiguration
metadata:
  name: platform-critical
spec:
  type: Limited
  limited:
    assuredConcurrencyShares: 100          # 【建议值】按实际并发实测调整
    limitResponse:
      type: Queue
      queuing:
        queues: 64
        queueLengthLimit: 200
        handSize: 6                        # shuffle sharding，隔离噪声源
---
apiVersion: flowcontrol.apiserver.k8s.io/v1
kind: FlowSchema
metadata:
  name: platform-critical-controllers
spec:
  priorityLevelConfiguration:
    name: platform-critical
  matchingPrecedence: 100                  # 越小越优先（1 最高，1000 最低）
  distinguisherMethod:
    type: ByUser
  rules:
    - subjects:
        - kind: ServiceAccount
          serviceAccount: {name: "platform-controller", namespace: "kube-system"}
        - kind: ServiceAccount
          serviceAccount: {name: "csi-provisioner", namespace: "kube-system"}
      resourceRules:
        - apiGroups: ["*"]
          resources: ["*"]
          verbs: ["*"]
---
# 2) 给批处理/数据同步类客户端一个"可牺牲"优先级
apiVersion: flowcontrol.apiserver.k8s.io/v1
kind: PriorityLevelConfiguration
metadata:
  name: batch-low
spec:
  type: Limited
  limited:
    assuredConcurrencyShares: 5
    limitResponse:
      type: Queue
      queuing:
        queues: 16
        queueLengthLimit: 20
        handSize: 4
---
apiVersion: flowcontrol.apiserver.k8s.io/v1
kind: FlowSchema
metadata:
  name: batch-sync-clients
spec:
  priorityLevelConfiguration:
    name: batch-low
  matchingPrecedence: 900
  distinguisherMethod:
    type: ByUser
  rules:
    - subjects:
        - kind: ServiceAccount
          serviceAccount: {name: "data-sync", namespace: "ops"}
      resourceRules:
        - apiGroups: ["*"]
          resources: ["*"]
          verbs: ["*"]
```

> **回滚**：`kubectl delete flowschema/<name>` + `kubectl delete prioritylevelconfiguration/<name>`，立即回落默认行为，无状态残留。

### 6.2 一个"生产级"的 Admission Webhook 配置

```yaml
apiVersion: admissionregistration.k8s.io/v1
kind: MutatingWebhookConfiguration
metadata:
  name: sidecar-injector
webhooks:
  - name: injector.example.com
    admissionReviewVersions: ["v1"]
    sideEffects: None                  # 声明无副作用，允许 dry-run
    timeoutSeconds: 3                  # 【建议值】≤5，越短越安全
    failurePolicy: Fail                # 关键注入用 Fail；非关键用 Ignore
    reinvocationPolicy: IfNeeded       # 其他 webhook 改了对象后是否再调用
    matchPolicy: Equivalent
    clientConfig:
      service:
        name: injector
        namespace: injector-system
        path: /mutate
        port: 443
      caBundle: <BASE64_CA>
    namespaceSelector:
      matchExpressions:
        - key: kubernetes.io/metadata.name
          operator: NotIn
          values: ["kube-system"]      # 排除系统命名空间，减小爆炸半径
    objectSelector:                    # 关键：只命中带特定标签的 Pod
      matchLabels:
        inject.example.com/enabled: "true"
    rules:
      - apiGroups: [""]
        apiVersions: ["v1"]
        operations: ["CREATE"]
        resources: ["pods"]
        scope: "Namespaced"
```

### 6.3 kubeadm 场景下的 APIServer 额外参数

```yaml
# /etc/kubernetes/manifests/kube-apiserver.yaml  → spec.containers[0].command
- --enable-priority-and-fairness=true
- --etcd-compaction-interval=5m            # 【默认值】
- --event-ttl=1h                           # 高 Event 量可降到 30m
- --etcd-servers-overrides=/events#https://etcd-events-0:2379;https://etcd-events-1:2379;https://etcd-events-2:2379
- --watch-cache-sizes=pods#1000,nodes#500,namespaces#10,endpointslices#500   # 【需实测】
- --anonymous-auth=false
- --audit-policy-file=/etc/kubernetes/audit/policy.yaml
- --audit-log-path=/var/log/kubernetes/audit.log
- --audit-log-maxage=7
- --audit-log-maxbackup=5
- --audit-log-maxsize=200
- --shutdown-delay-duration=20s            # 配合 LB 摘流【建议值】
```

> **变更纪律**：静态 Pod manifest 修改后 kubelet 会自动重启 APIServer；**必须逐个节点修改并观察 **`/readyz`，禁止同时改所有 master。

### 6.4 关键告警规则（Prometheus）

```yaml
groups:
  - name: apiserver-slo
    rules:
      - alert: APIServerHigh5xxRate
        expr: |
          sum(rate(apiserver_request_total{code=~"5.."}[5m]))
          / sum(rate(apiserver_request_total[5m])) > 0.01
        for: 5m
        labels: {severity: critical}
        annotations: {summary: "APIServer 5xx 率 > 1%（SLO 烧穿）"}

      - alert: APIServerLatencyP99
        expr: |
          histogram_quantile(0.99,
            sum by (le) (rate(apiserver_request_duration_seconds_bucket{verb!~"WATCH"}[5m]))
          ) > 1
        for: 10m
        labels: {severity: warning}

      - alert: APIServerFlowControlRejecting
        expr: |
          sum(rate(apiserver_flowcontrol_rejected_requests_total[5m]))
          / sum(rate(apiserver_request_total[5m])) > 0.001
        for: 5m
        labels: {severity: warning}

      - alert: EtcdWALFsyncSlow
        expr: |
          histogram_quantile(0.99,
            sum by (le) (rate(etcd_disk_wal_fsync_duration_seconds_bucket[5m]))
          ) > 0.01
        for: 10m
        labels: {severity: critical}
        annotations: {summary: "etcd WAL fsync P99 > 10ms，磁盘不达标"}

      - alert: EtcdDBNearQuota
        expr: etcd_mvcc_db_total_size_in_bytes / etcd_server_quota_backend_bytes > 0.8
        for: 15m
        labels: {severity: critical}

      - alert: EtcdLeaderChanges
        expr: increase(etcd_server_leader_changes_seen_total[30m]) > 3
        labels: {severity: critical}

      - alert: APIServerCertExpiringSoon
        expr: |
          min by (job) (apiserver_client_certificate_expiration_seconds) < 30*24*3600
        labels: {severity: warning}
```

---

## Part 7：面试自检清单

能不看资料回答以下问题，说明 APIServer 这块过关了：

- [ ] 能画出 `kubectl apply` → etcd 的完整链路，并说出每个过滤器在做什么
- [ ] 说清 Mutating / Validating 的先后与原因
- [ ] 解释 `resourceVersion` 与乐观锁，以及 409 的正确处理方式
- [ ] 说清 List-Watch + Informer 的三段：List → Watch → Resync/Relist，且知道 resync **不**打 APIServer
- [ ] 解释 watch cache 的作用与 410 Gone 的成因
- [ ] 说出 Bookmark 的作用（保活 + RV 推进 + 避免 relist）
- [ ] 讲清 APF 的 FlowSchema / PriorityLevel / shuffle sharding，能写出 YAML
- [ ] 说出大规模集群的三大压力源：Lease/Event/CR 写放大、List 读放大、webhook 时延
- [ ] 知道官方规模上限与"每节点 Pod ≤ 110"的原因
- [ ] 能给出 etcd NOSPACE 的处置步骤（含 alarm disarm 与 defrag 顺序）
- [ ] 能给出 APIServer P99 高时的决策树（限流 / etcd / 自身）
- [ ] 能给出 SLO + 错误预算的具体数字与测量 PromQL
- [ ] 能把任意优化动作配上"灰度比例 + 回滚触发条件 + 验证指标"

---

**免责与边界**：本文为面试复习与架构评审材料，不针对任何具体生产集群直接执行操作。所有生产变更请走：变更评审 → 备份 → 灰度 → 观察 → 回滚预案。文中【默认值】请以目标版本的官方参数文档为准，【需实测】项请以你的监控数据为准。
