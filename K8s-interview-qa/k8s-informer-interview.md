# k8s-informer-interview

# Kubernetes Informer / client-go 面试题精讲（原理篇 + 大规模集群篇）

> 适用：中高级 SRE / 云原生平台工程师 / Operator 开发面试。
> 说明：文中数值分三类 —— **【默认值】** 为 client-go 社区常见默认（**随版本演进，答题前请核对目标版本源码**）；**【建议值】** 为工程经验起点；**【需实测】** 表示必须结合你的集群监控数据确定，不得照抄。
> 涉及生产变更的部分一律标注「灰度 / 回滚 / 验证」。

---

## 目录

- [Part 0：答题方法论](#part-0答题方法论)
- [Part 1：架构与核心组件（Q1–Q16）](#part-1架构与核心组件q1q16)
- [Part 2：事件与生命周期语义（Q17–Q30）](#part-2事件与生命周期语义q17q30)
- [Part 3：大规模集群高频问题（Q31–Q46）](#part-3大规模集群高频问题q31q46)
- [Part 4：故障排查实战（Q47–Q56）](#part-4故障排查实战q47q56)
- [Part 5：反模式与风险清单](#part-5反模式与风险清单)
- [Part 6：可直接套用的代码与配置](#part-6可直接套用的代码与配置)
- [Part 7：面试自检清单](#part-7面试自检清单)

---

## Part 0：答题方法论

面试官考察 Informer，本质上看三件事：

| 考察维度 | 表现出来的是 | 高分回答的特征 |
| --- | --- | --- |
| **组件拆解** | 能不能说出 Reflector / DeltaFIFO / Indexer / Controller / Processor 各自的职责 | 能画出数据流向，并说清"为什么需要 DeltaFIFO 这一层" |
| **语义精确** | resync vs relist 是否混淆 | 能明确指出 **resync 不产生任何 APIServer 请求** |
| **规模意识** | 是否知道 15 万 Pod 时 informer 会怎样 | 能量化内存模型：`进程数 × 对象数 × 单对象大小`，并给出 Transform / selector / 分片方案 |

**答题模板**

1. 一句话定位（解决什么问题）
2. 关键机制（2–3 个核心技术点，最好带组件名）
3. 大规模下的失效模式
4. 可观测指标 + 修复动作 + 回滚条件

---

## Part 1：架构与核心组件（Q1–Q16）

### Q1：Informer 解决什么问题？为什么不能直接轮询 APIServer？

**答案要点**

- **问题**：控制器需要持续知道"集群里现在有什么"。朴素做法是每个控制循环都 `GET/LIST` 一次 APIServer。
- **代价**：N 个控制器 × 每秒 M 次轮询 = 请求量随客户端数线性放大；且绝大多数轮询是**无效读**（没变更）。
- **Informer 的解法**：
  1. **List 一次**拿全量 + `resourceVersion`
  2. **Watch 长连接**接收增量事件（推模式）
  3. **本地缓存（Indexer）**：业务代码从内存读，**零 API 请求**
- **收益**：把读压力从 `O(客户端数 × 轮询频率)` 降到 `O(实际变更数)`；同时本地查询是**微秒级**，控制器吞吐大幅提升。

**加分项**

- 这正是 Kubernetes "声明式 API + 控制循环" 能成立的性能基础：**APIServer 从不主动调用任何人，一切由 list-watch 驱动**。

---

### Q2：画出 Informer 的完整架构与数据流向。（必考）

```
                       ┌─────────────────────────────────────────┐
                       │           kube-apiserver                │
                       │   (watch cache → 事件流 / etcd 兜底)     │
                       └───────────┬─────────────────────────────┘
                             List  ↑  │ Watch(resourceVersion)
                                   │  ↓
        ┌──────────────────────────────────────────────────────────┐
        │ 1. Reflector（反射器）                                     │
        │    - List 全量（分页，pager）→ 得到 resourceVersion        │
        │    - Watch 增量 → 事件流                                   │
        │    - 断连/410 → 退避重试 → relist                          │
        │    - 只负责"取数据"，不理解业务                             │
        └───────────────┬──────────────────────────────────────────┘
                        │ Add / Update / Delete / Replace / Sync
                        ↓
        ┌──────────────────────────────────────────────────────────┐
        │ 2. DeltaFIFO（增量队列）                                   │
        │    - key = namespace/name                                 │
        │    - 同一 key 的事件合并去重                               │
        │    - KnownObjects 指向 Indexer（用于判断"真删除"）          │
        └───────────────┬──────────────────────────────────────────┘
                        │ Pop()（阻塞式，无数据时等待）
                        ↓
        ┌──────────────────────────────────────────────────────────┐
        │ 3. Controller（informer 内部控制器）                        │
        │    - processLoop: Pop → 更新 Indexer → 分发给 listener     │
        └───────┬──────────────────────────────┬───────────────────┘
                ↓                              ↓
   ┌────────────────────────┐   ┌──────────────────────────────────┐
   │ 4. Indexer（本地缓存）  │   │ 5. SharedProcessor               │
   │    - threadSafeStore   │   │    - 每个 handler 一个 listener   │
   │    - 支持自定义索引     │   │    - 环形 buffer（默认 1000）     │
   │    - 业务代码直接读     │   │    - 串行分发，慢 handler 会堆积  │
   └────────────────────────┘   └───────────────┬──────────────────┘
                                                ↓
                                     ResourceEventHandler
                                     (OnAdd / OnUpdate / OnDelete)
                                                ↓
                                        workqueue（去重 + 限速）
                                                ↓
                                        Worker 并发 reconcile
```

**一句话记忆**：**Reflector 取、DeltaFIFO 攒、Controller 转、Indexer 存、Processor 发**。

---

### Q3：Reflector 具体做什么？watch 断了会怎样？

**核心职责**

- 调用 `ListerWatcher.List()` 拿全量对象 → `DeltaFIFO.Replace()`
- 调用 `ListerWatcher.Watch(resourceVersion)` 拿增量 → `DeltaFIFO.Add/Update/Delete`
- 维护 `lastSyncResourceVersion`，保证断连后能从断点续传

**断连处理（关键）**

| 情况 | 行为 |
| --- | --- |
| 网络抖动、连接被 LB 切断、APIServer 重启 | watch 返回 error → **指数退避重试 Watch**（用同一个 RV 续传，不 relist） |
| 返回 `410 Gone` / `ErrExpired`（RV 太旧） | **触发 relist**（全量 List → Replace） |
| APIServer 返回 429 | 客户端退避（client-go 令牌桶 + APF 的 `Retry-After`） |
| 请求超时 | 重试 |

**退避参数**（client-go Reflector 的 backoffManager，【默认值】，**请以目标版本源码核对**）

- 初始延迟约 **800ms**，最大约 **30s**，倍增系数 **2.0**，重置周期约 **2min**。

**watch 超时设计（面试加分）**

- client-go 给每个 watch 请求设置 **随机超时**（基础值约 5 分钟，实际取值在 5–10 分钟区间随机，【默认值】）。
- **目的**：避免所有 Reflector 在同一时刻重连造成惊群（thundering herd）。源码注释明确说明这是"故意不与 APIServer 用同一个超时"。

---

### Q4：DeltaFIFO 是什么？为什么需要它，而不是直接把事件塞给 handler？

**答案要点**

DeltaFIFO 是一个 **按 key 去重的增量队列**，元素为 `Deltas{ Type, Object }`，`Type` 有五种：

| 类型 | 产生时机 | 语义 |
| --- | --- | --- |
| `Added` | watch 收到 ADDED | 新对象 |
| `Updated` | watch 收到 MODIFIED | 对象变更 |
| `Deleted` | watch 收到 DELETED / relist 时缓存中存在但新列表中没有 | 对象消失 |
| `Replaced` | List 全量结果（relist） | 全量同步（较新版本引入，旧版本用 `Sync`） |
| `Sync` | resync 触发 | 周期性重新投递，**不代表对象真的变了** |

**为什么需要它（三个核心价值）**

1. **去重**：高频变更的对象（如 Pod status 每秒变）在队列里只保留累积的 delta，避免 handler 被淹没 —— 但注意：**同 key 的多个 delta 会合并为一个 Deltas 列表，handler 会按顺序全部处理，不会真的丢弃中间状态**（这点常被误解）。
2. **保证顺序**：同一 key 的事件严格按序处理（FIFO），避免旧状态覆盖新状态。
3. **兜底删除检测**：`Replace()` 时会对比 `KnownObjects`（Indexer）与新列表，**新列表中缺失的对象会补发 Deleted 事件** —— 这是解决"watch 断连期间删除事件丢失"的核心机制。

---

### Q5：Indexer / ThreadSafeStore 是什么？怎么用自定义索引？

- **Indexer** = 本地缓存 + 索引。底层是 `threadSafeStore`（`map[key]object` + `map[indexName]map[indexValue]set[key]`），读写锁保护。
- 默认 key 函数：`MetaNamespaceKeyFunc` → `namespace/name`（无 namespace 则只有 name）。
- 默认索引：`namespace`（按 namespace 反查）。

**自定义索引示例**（见 Part 6.4）
常见用途：按 `spec.nodeName` 索引 Pod、按 `spec.serviceAccountName` 索引。

**面试追问**：Indexer 里的对象是**深拷贝**还是引用？

- 存入时 client-go 会做浅拷贝/深拷贝处理（`DeepCopy`），但**业务代码从 Indexer 取出的对象不应直接修改**（会污染缓存），必须先 `DeepCopy()` 再改。这是高频踩分点。

---

### Q6：Informer 里的 "Controller" 和 Kubernetes 的 "Controller" 是一个东西吗？

不是，这是命名上的经典混淆：

| 名称 | 位置 | 职责 |
| --- | --- | --- |
| `cache.Controller`（informer 内部） | `client-go/tools/cache/controller.go` | 驱动 Reflector + DeltaFIFO，把事件落到 Indexer 并分发给 handler。**不含业务逻辑** |
| 业务 Controller / Reconciler | 用户代码 | 注册 handler → 入队 → 执行调谐逻辑 |

`NewInformer` / `NewIndexerInformer` 内部都会创建这个 `Controller`。

---

### Q7：`SharedInformer` 为什么叫 "Shared"？共享了什么？（高频）

**共享的是**：同一个资源类型的 **一个 Reflector + 一个 DeltaFIFO + 一个 Indexer**（即一条 watch 连接和一份缓存），多个 handler 通过 `SharedProcessor` 共享同一份事件流。

**价值**

- 一个进程内 10 个控制器都关心 Pod → **只有 1 次 List、1 条 Watch、1 份缓存**，而不是 10 份。
- 大规模集群下这是**最重要的内存/连接优化手段**。

**用法**

```go
factory := informers.NewSharedInformerFactory(clientset, 0)   // 0 = 不自动 resync
podInformer := factory.Core().V1().Pods().Informer()
podInformer.AddEventHandler(handlerA)
podInformer.AddEventHandler(handlerB)   // 同一个 informer，共享缓存
factory.Start(ctx.Done())
factory.WaitForCacheSync(ctx.Done())
```

**注意**：`SharedInformerFactory` 有 `Shutdown()`；调用后需重新创建 factory 才能再次 start（informer 不支持 restart）。

---

### Q8：`AddEventHandlerWithResyncPeriod` 的 resync period 是什么语义？

- 可为**单个 handler** 设置不同的 resync 周期；`SharedInformerFactory` 的整体 resyncPeriod 与所有注册 handler 的周期取**最小值**驱动。
- resync 到达时，把 Indexer 中**所有对象**以 `Sync` 类型重新入队 → 触发对应 handler 的 `OnUpdate`。
- **关键**：resync **不会**触发 APIServer 请求（见 Q18）。

---

### Q9：`WaitForCacheSync` / `HasSynced` 的语义？为什么必须在 Start 之后调用？

- `HasSynced()` 返回该 informer 的 DeltaFIFO 是否已完成**首次全量同步**（即第一次 List 的结果已全部 Pop 并写入 Indexer）。
- `WaitForCacheSync(stopCh)` 阻塞等待所有已注册 informer 同步完成；返回 `map[type]bool`。
- **必须先 **`Start()`** 再 **`WaitForCacheSync()`，否则永远等不到（informer 没启动）。
- **语义边界**：只代表"**首次**同步完成"。之后 relist / 断连重连**不会**让 HasSynced 变回 false，业务代码也不应依赖它判断"数据是否最新"。
- **反模式**：`WaitForCacheSync` 返回 false 却继续执行 → 控制器基于空缓存工作 → 会误判"对象不存在"而**疯狂创建/删除**。必须检查返回值并处理。

---

### Q10：ResourceEventHandler 的三个回调语义？UpdateFunc 的 old/new 从哪来？

```go
type ResourceEventHandler interface {
    OnAdd(obj interface{}, isInInitialList bool)     // 较新版本增加 isInInitialList
    OnUpdate(oldObj, newObj interface{})
    OnDelete(obj interface{})
}
```

- **OnAdd**：首次 List 期间的每个对象都会回调一次；之后 watch 到新对象也回调。
  - `isInInitialList=true` 用于区分"启动时的存量对象"与"运行期新增"（**重要**：避免启动时对 15 万个存量对象做全量业务动作）。
- **OnUpdate**：`oldObj` 来自 Indexer 中的旧值，`newObj` 是新值。resync 时也会回调（old == new）。
- **OnDelete**：对象从缓存移除时回调。
  - **陷阱**：收到的是 `DeletedFinalStateUnknown`（包含 `Key` + `Obj`）的情况 —— 当 DeltaFIFO 无法拿到最终状态（如 watch 断连期间的删除）时，只能拿到 key。业务代码**必须处理这个类型**，否则会漏删外部资源（如云盘、DNS 记录）。

---

### Q11：workqueue 的作用？去重、限速、延迟分别怎么实现？

**核心能力**

1. **去重**：同一 key 在队列中只保留一份（正在处理时再次 Add 只会标记 dirty，处理完 `Done()` 后重新入队）。天然避免"同一对象被并发 reconcile"。
2. **限速（RateLimiter）**：控制 key 重新入队的节奏。
   - 默认组合（`DefaultTypedControllerRateLimiter`，【默认值】，以源码为准）：
     - `ItemExponentialFailureRateLimiter(5ms, 1000s)`：失败次数越多，延迟指数增长
     - `ItemBucketRateLimiter(10 qps, burst 100)`：整体令牌桶兜底
3. **延迟入队**：`AddAfter(key, duration)`（延迟队列）。
4. **忘记失败计数**：`Forget(key)`（成功后清除退避计数）。
5. **优雅关闭**：`ShutDownWithDrain()`（先排空再关闭，用于原地升级/重启不丢任务）。

**关键纪律**

- `Get()` 后**必须** `Done(key)`，否则该 key 永远被认为"正在处理"，不再入队 → **静默卡死**（这是最经典的 Operator bug）。
- 处理失败要 `AddRateLimited(key)` 而非 `Add(key)`。
- 处理成功要 `Forget(key)`。

---

### Q12：为什么 handler 里只应该做 `enqueue`，不能做业务逻辑？（高频）

**原因（三个层次）**

1. **阻塞分发**：`SharedProcessor` 对每个 listener 用**单个 goroutine 串行分发**。handler 慢 → 该 listener 的 buffer 堆积 → 事件延迟。
2. **内存增长**：`processorListener` 有一个**环形 buffer（【默认值】1000）**，满了之后新通知会进入**无界的 **`pendingNotifications`** 切片** → handler 持续慢会导致内存线性上涨直至 OOM。
3. **阻塞整个 informer**：分发是同步调用链的一部分，慢 handler 会拖慢 DeltaFIFO 的消费速度，进而影响**所有** listener（因为 processor 是同步遍历的）。

**正确姿势**

```go
func (c *Ctrl) OnAdd(obj interface{}, _ bool) {
    key, _ := cache.MetaNamespaceKeyFunc(obj)
    c.queue.Add(key)          // 只入队，微秒级返回
}
// 真正的业务逻辑放在 worker goroutine 里，从缓存取对象处理
```

---

### Q13：Informer 的 goroutine 模型是怎样的？一个进程有多少 goroutine？

粗略模型（每个 informer）：

- 1 个 `Reflector.Run` goroutine（List/Watch 循环）
- 1 个 `processLoop` goroutine（Pop 并分发）
- **每个注册的 handler 1 个** `processorListener.run` + 1 个 `pop` goroutine
- 用户自起的 N 个 worker goroutine

**面试考点**：注册 handler 越多，goroutine 越多；且**每个 handler 都要完整遍历一遍所有事件**，所以 handler 数量直接放大 CPU 开销。大规模下应尽量复用同一个 handler/controller。

---

### Q14：`dynamic` / `unstructured` / `metadata-only` informer 的区别与取舍？

| 类型 | 返回对象 | 内存 | 类型安全 | 适用 |
| --- | --- | --- | --- | --- |
| Typed informer | `*v1.Pod` 等 | 中（但可 Transform） | ✅ | 常规控制器（首选） |
| Dynamic informer | `*unstructured.Unstructured` | **大**（map\[string\]interface{}，比结构体膨胀数倍） | ❌ | 需要处理任意 CRD（如多集群同步、通用审计） |
| Metadata-only | `*metav1.PartialObjectMetadata` | **极小**（仅 metadata） | 部分 | 只关心存在性/标签/命名空间（如配额统计、NS 生命周期） |

**大规模建议**

- 只关心 metadata 的场景用 `DynamicSharedInformerFactory` + `metadata` GVR（`metav1.APIResource{Name:"pods", ...}` 走 `metadata.k8s.io` 的 PartialObjectMetadata）→ 请求头带 `Accept: ...;as=PartialObjectMetadataList`，**APIServer 侧和客户端侧都省**。
- 避免用 unstructured 缓存大资源（内存和 GC 压力显著）。

---

### Q15：controller-runtime 的 `Manager` / `Cache` 与 client-go informer 的关系？

- controller-runtime 的 `cache.Cache` 是对 `SharedInformerFactory` 的**封装**，按 GVK 懒创建 informer（`GetInformer(ctx, obj)`）。
- `DelegatingClient`：
  - **读（Get/List）→ 走 Cache（本地内存）**，零 API 请求
  - **写（Create/Update/Delete/Patch）→ 走 APIServer**
- `manager.Start()` 内部会启动所有 informer 并 `WaitForCacheSync`。
- **Leader 选举**：controller-runtime 默认**只有成为 leader 后才启动 cache**（`LeaderElection` + `LeaderElectionReleaseOnCancel`）。非 leader 副本不启动 informer → **大规模下避免了 N 副本 × 全量缓存的浪费**（对比：controller-runtime 之前的版本所有副本都 watch）。

**关键配置（1.15+）**

```go
cache.Options{
    ByObject: map[client.Object]cache.ByObject{
        &corev1.Pod{}: {
            Label: labels.SelectorFromSet(labels.Set{"app": "myapp"}),  // 只缓存部分
            Transform: func(obj interface{}) (interface{}, error) { ... }, // 裁剪字段
        },
    },
    DefaultTransform: ...,
}
```

---

### Q16：Informer 的事件顺序与一致性保证到什么程度？

- **单 key 顺序保证**：DeltaFIFO 保证同一 key 的事件按到达顺序处理。
- **跨 key 无顺序保证**：Pod A 和 Pod B 的处理顺序不确定，控制器**不能依赖跨对象顺序**。
- **不保证"每个变更都被 handler 看到"**：高频变更对象的中间状态可能被合并（但同 key 的 delta 列表会依次处理，仍是最终一致）。
- **最终一致**：断连期间的变更由 relist + `Replace` 兜底对齐。
- **缓存不是强一致**：读本地缓存可能读到几百毫秒前的旧值；对一致性要求高的操作（如写前校验）应直接读 APIServer（`client.APIReader` 或 `cache.Uncached`）。

---

## Part 2：事件与生命周期语义（Q17–Q30）

### Q17：List 阶段 informer 是怎么做的？分页吗？

- Reflector 用 `pager` 做**分页 List**（【默认值】每页 500 个对象；可通过 `Reflector.WatchListPageSize` 覆盖，**默认值请核对源码**）。
- 请求带 `resourceVersion=0` → 让 APIServer **走 watch cache**，不压 etcd（这是关键设计：informer 的初始化应该命中缓存而非打穿到 etcd）。
- 全量结果 → `DeltaFIFO.Replace(list, rv)` → 之后用该 `rv` 建 watch。

**大规模影响**

- 15 万 Pod / 500 = **300 次 List 往返**；单次 List 大对象会吃内存。
- 调大页大小可减少往返次数，但会增大单次内存峰值 —— **【需实测】权衡**。

---

### Q18：resync 和 relist 的区别？（经典高频，最容易答错）

| 维度 | **resync（重同步）** | **relist（重列）** |
| --- | --- | --- |
| 触发 | 定时器（`resyncPeriod`） | watch 断连且 RV 过期（410 / ErrExpired）、首次启动 |
| 是否请求 APIServer | ❌ **完全不请求** | ✅ 全量 List 请求 |
| 数据来源 | 本地 Indexer 中已有对象 | APIServer |
| 事件类型 | `Sync` | `Replaced` / `Sync`（旧版本） |
| 目的 | 周期性把对象重新投递给 handler，兜底"事件丢失/处理失败" | 重新建立基线，对齐断连期间的变更 |
| 代价 | 本地 CPU + 队列压力 | **APIServer / etcd 压力**（惊群风险） |

> **面试必背**：**resync 不产生任何 APIServer 请求**。很多人误以为 resync 会重新 List，这是错的。

---

### Q19：resync 设成多久合适？设成 0 行不行？

- **设 0（关闭）**：完全事件驱动。优点是无额外开销；缺点是**漏事件无法自愈**（例如 handler 处理失败但没重新入队、外部系统状态漂移）。
- **常见做法**：【建议值】**数小时量级**（controller-runtime 的 `SyncPeriod` 默认约 **10 小时**，【默认值】）。
- **反模式**：设成秒级（如 10s）→ 15 万对象每 10 秒全部重新入队 → **reconcile 风暴**，CPU 和队列被打爆，虽然不打 APIServer，但控制器自己会被压垮。
- **更优实践**：resync 设为 0 或很大，改为在控制器内部做**定期全量对账**（可控节奏 + 可观测）。

---

### Q20：删除事件丢失会怎样？Informer 如何兜底？

**场景**：watch 连接中断 30 秒，期间某对象被删除 → 重连后不会收到这条 DELETED。

**兜底机制**

1. 若能用原 RV 续传：APIServer 会正常补齐（etcd 有历史 revision）。
2. 若 RV 已过期 → relist → `DeltaFIFO.Replace()` → 对比 `KnownObjects`（本地缓存）与新列表：**缓存里存在但新列表中没有的对象，补发 Deleted 事件**。
3. 若补发时已拿不到最终对象 → 投递 `DeletedFinalStateUnknown`（只有 key）。

**业务代码必须做的**

```go
func (c *Ctrl) OnDelete(obj interface{}) {
    if d, ok := obj.(cache.DeletedFinalStateUnknown); ok {
        obj = d.Obj                 // 可能为 nil，只能用 d.Key
        key = d.Key
    }
    ...
}
```

并且：所有对外部系统（云盘、DNS、数据库）的删除操作必须**幂等**，且要能通过"定期全量对账"兜底（对比 K8s 侧与外部侧的存量）。

---

### Q21：为什么 watch 会"假死"？Bookmark 在这里的作用？

- **假死**：TCP 连接未断（LB/防火墙未超时），但**实际上已无数据流动**（例如中间设备静默丢包）。客户端以为连接正常，实际已经收不到事件 → **informer 静默不更新**，控制器看不到任何变化。
- **Bookmark 事件**：APIServer 周期性发送一个**只含 resourceVersion、无对象体**的事件。客户端（需 `allowWatchBookmarks=true`）用它：
  1. 更新本地已知最新 RV（重连时可续传，避免 relist）
  2. **检测连接存活** —— 超过预期时间没收到任何事件（含 bookmark）→ 判定连接异常，主动断开重建
- **实践**：client-go 较新版本在 watch 初始化时会请求 bookmark；LB / 云厂商的 **idle timeout** 必须大于 bookmark 间隔，否则会被静默切断。

---

### Q22：informer 的 stop channel 与生命周期？能不能重启？

- `factory.Start(stopCh)`：`stopCh` 关闭 → 所有 informer 的 Reflector、processor、processLoop 都退出。
- `SharedInformer`** 不支持重启**：一旦停止，必须**重新创建 factory 和 informer**（源码层面没有 restart 语义）。
- **优雅关闭顺序**（生产级）：
  1. 关闭业务 worker（等待处理中的 reconcile 完成或超时）
  2. `queue.ShutDownWithDrain()`（排空队列）
  3. 关闭 informer 的 stopCh
  4. 释放外部连接
- **反模式**：只 `close(stopCh)` 而不排空 workqueue → 在途任务丢失（对有外部副作用的控制器是数据丢失）。

---

### Q23：`OnUpdate` 被调用就一定说明对象变了吗？

**不一定**。两种"假更新"：

1. **resync**：会无条件对所有对象触发 `OnUpdate(old, new)`，且 `old` 与 `new` 内容相同。
2. **无意义变更**：如 `managedFields`、`resourceVersion`、`lastAppliedConfig`、`status` 的无关抖动。

**正确做法：在 reconcile 入口做"有意义变更"判断**

```go
// 1) resync 场景：内容相同直接跳过
if apiequality.Semantic.DeepEqual(oldObj.Spec, newObj.Spec) &&
   oldObj.GetDeletionTimestamp() == nil {
    return
}
// 2) reconcile 入口：比较 Spec/关键注解，而非整对象
// 3) 更彻底：给对象打 hash 注解（spec-hash），变了才处理
```

**大规模收益**：这一层过滤能砍掉绝大部分无效 reconcile，**【需实测】但通常是数量级差异**。

---

### Q24：如何只 watch 部分对象（label/field selector）？这对大规模有多重要？

```go
factory := informers.NewSharedInformerFactoryWithOptions(
    clientset, 10*time.Hour,
    informers.WithNamespace("prod"),                        // 只看一个命名空间
    informers.WithTweakListOptions(func(o *metav1.ListOptions) {
        o.LabelSelector = "team=platform"
        o.FieldSelector = "spec.nodeName=node-1"             // 仅部分字段可用
    }),
)
```

**价值（大规模集群最重要的优化之一）**

- 内存从"全集群对象"降到"我关心的那部分"
- 事件量同比例下降
- **注意**：selector 下推到 APIServer 侧过滤，但 watch cache 只对部分字段建索引，复杂 selector 会在缓存后遍历（观察 `apiserver_cache_list_fetched_objects_total` 判断是否浪费）。

---

### Q25：缓存里的对象能不能直接修改？

**不能**。Indexer 中的对象是共享的，直接修改会：

1. 污染本地缓存（其他 handler 读到被改过的对象）
2. 导致后续 `OnUpdate` 的 `oldObj` 与真实历史不一致
3. **如果误传给 Update 请求，可能把本地修改写进集群**（严重事故）

**正确做法**

```go
pod := informer.GetIndexer().GetByKey(key).(*corev1.Pod).DeepCopy()  // 必须 DeepCopy
pod.Labels["x"] = "y"
client.Update(ctx, pod)
```

---

### Q26：Informer 的 List 请求会不会打穿到 etcd？

- 默认带 `resourceVersion=0` → APIServer 从 **watch cache** 返回，**不打 etcd**。
- 若显式指定非 0 的 `resourceVersion` → 走 **quorum read**（强一致，压 etcd）。Reflector 首次 List 用的是 "0"，所以是缓存读。
- **大规模意义**：informer 重启/relist 时，只要 APIServer 的 watch cache 还有数据，就不会打穿 etcd。**这也意味着调大 **`--watch-cache-sizes`** 能显著降低 relist 的 etcd 压力**。
- WatchList 特性（较新版本）：直接用 `watch + sendInitialEvents` 完成初始化，进一步减少 etcd 全量读（**特性开关与版本支持情况请以目标版本为准**）。

---

### Q27：多个副本（多 Pod）的控制器，informer 怎么工作？

**两种模式**

| 模式 | 行为 | 代价 | 适用 |
| --- | --- | --- | --- |
| **全部副本都 watch** | 每个副本一份全量缓存 | N × 内存、N × watch 连接 | 需要每个副本都能快速接管、读多写少 |
| **仅 leader watch**（controller-runtime 默认） | 只有 leader 启动 cache | 切主时有冷启动 List 开销 | 大规模首选 |

**切主代价**：新 leader 需要**全量 List + 建立缓存**，大资源下可能要几十秒到数分钟 → 期间控制器不可用。

- 缓解：缩小缓存范围（selector / Transform / metadata-only）、预留足够内存、监控冷启动耗时。

---

### Q28：Informer 与 APF（限流）的关系？被限流会怎样？

- Informer 的 List 请求属于某个 FlowSchema（通常按 ServiceAccount/用户匹配，如 `service-accounts`、`kube-controller-manager`）。
- 被限流 → List 变慢或返回 **429** → 控制器**启动变慢**（`WaitForCacheSync` 超时）、relist 变慢、事件延迟。
- **危险正反馈**：APIServer 压力大 → 限流 → 客户端 relist 变慢 → 客户端数据更旧 → 客户端重试更多 → APIServer 压力更大。
- **缓解**：为核心控制器配置专属 FlowSchema/高优先级（见 APIServer 篇 Part 6.1），并为客户端设置合理的 `QPS/Burst`。

---

### Q29：如何给 Informer 相关逻辑做单元测试？

| 层次 | 工具 | 说明 |
| --- | --- | --- |
| reconcile 逻辑 | `sigs.k8s.io/controller-runtime/pkg/client/fake` | 纯内存客户端，测业务逻辑 |
| 需要真实 APIServer 语义 | **envtest**（`setup-envtest`） | 启动真实 kube-apiserver + etcd，测 informer/webhook/CRD 行为 |
| 事件驱动逻辑 | 直接构造 `DeltaFIFO` / 手动调用 handler | 无需集群 |
| 集成/规模验证 | kind + kubemark / 压测集群 | 验证规模行为 |

**注意**：fake client 不支持 watch 语义，测不出"事件丢失/重放"相关问题，这类必须上 envtest 或真实集群。

---

### Q30：Informer 相关的可观测指标有哪些？

**客户端侧（需在控制器中自行暴露）**

```
# workqueue（controller-runtime 默认暴露）
workqueue_depth{name}                 # 队列深度 —— 最直接的积压指标
workqueue_adds_total / workqueue_retries_total
workqueue_queue_duration_seconds     # 在队列中待了多久
workqueue_work_duration_seconds      # 处理耗时
workqueue_unfinished_work_seconds

# controller-runtime
controller_runtime_active_workers{controller}
controller_runtime_reconcile_total{controller,result}     # result=error/requeue/success
controller_runtime_reconcile_time_seconds
controller_runtime_max_concurrent_reconciles
```

**客户端进程侧**

```
process_resident_memory_bytes        # 缓存对象数量直接反映在内存上
go_goroutines                        # handler/goroutine 泄漏
```

**APIServer 侧（判断 informer 造成的压力）**

```
apiserver_request_total{verb="list",resource=...,client=...}
apiserver_request_total{verb="watch"}
apiserver_current_inflight_requests                   # watch 长连接占用
apiserver_flowcontrol_rejected_requests_total         # 是否被限流
apiserver_watch_events_sizes_bucket                   # 事件体积
```

**建议告警（【建议值】，需按 SLO 调整）**

- `workqueue_depth` 持续 \> 1000 或单调增长 → 处理不过来
- `controller_runtime_reconcile_total{result="error"}` 错误率 \> 1%
- 客户端 RSS 超过预期的缓存规模（说明有泄漏或缓存范围失控）

---

## Part 3：大规模集群高频问题（Q31–Q46）

### Q31：大规模集群下，Informer 的三大问题是什么？

| 问题 | 表现 | 根因 |
| --- | --- | --- |
| **① 内存** | 控制器 Pod RSS 数 GB、OOMKilled | 本地缓存 = 全量对象副本，且 `进程数 × 对象数 × 单对象大小` |
| **② 惊群** | APIServer 重启后 etcd 读 QPS 暴涨、List P99 飙升 | 所有 reflector 同时断连 → 同时 relist |
| **③ 事件风暴** | CPU 打满、workqueue 积压、reconcile 延迟分钟级 | 高频变更资源（Pod status/Lease/Event）+ resync 过密 + 无意义 Update |

---

### Q32：量化 Informer 的内存开销（面试给算式，别给死数字）

```
单进程缓存内存 ≈ Σ(每种资源的对象数 × 该资源单对象平均大小 × 膨胀系数)
                 + DeltaFIFO 队列 + workqueue + Go 运行时开销

膨胀系数（经验）：
  typed struct    ≈ 1.0（结构体本身）
  unstructured    ≈ 3~5x（map[string]interface{} 开销）【需实测】
  metadata-only   ≈ 0.05~0.1x【需实测】
```

**举例（示意，不是结论）**：全集群 15 万 Pod，单 Pod 对象按结构化表示约数 KB 量级 → 单个控制器进程仅 Pod 缓存就可能是**数 GB**。再乘以副本数（若非 leader-only）→ 直接 OOM。

**测量方法（必须实测，不要猜）**

```bash
kubectl top pod <controller> -n <ns>
# 或运行时取 heap
go tool pprof http://<pod>:<port>/debug/pprof/heap
# 看 inuse_space 的 top，通常 cache.threadSafeStore / unstructured 占大头
```

---

### Q33：如何降低 Informer 的内存？（综合题，必考）

按收益排序：

| 手段 | 做法 | 收益 |
| --- | --- | --- |
| **1. 缩小缓存范围** | `WithNamespace` + `LabelSelector` + `FieldSelector` | **最高**（对象数直接下降） |
| **2. Transform 裁剪字段** | controller-runtime `cache.Options.ByObject[].Transform`，只保留 metadata + 需要的 spec 字段 | **很高**（单对象大小可降一个量级） |
| **3. metadata-only** | 只要 metadata 时用 PartialObjectMetadata | 高 |
| **4. 避免 unstructured** | 用 typed 对象 | 高 |
| **5. 仅 leader 启动 cache** | controller-runtime 默认行为 | 中高（减少副本倍数） |
| **6. 按 namespace/租户分片** | 多控制器分片各自 watch 一部分 | 中高 |
| **7. 不缓存大资源** | `client.Options{Cache: &client.CacheOptions{Unstructured: false}}` / `cache.Uncached(...)` | 中 |
| **8. 减少 handler 数量** | 复用 SharedInformer | 中 |

**Transform 示例见 Part 6.2。**
**注意**：Transform 会**丢失字段**，被裁剪的对象**不能**用于 Update/Patch（会把裁剪后的对象写回 APIServer，导致字段丢失！）。写入时必须重新 Get（走 APIReader）或确保该类型不使用裁剪缓存。

---

### Q34：什么是"惊群（Thundering Herd）"？怎么缓解？（必考）

**场景**：APIServer 滚动升级 / 网络抖动 / etcd leader 切换 → 所有 Reflector 的 watch 同时断开 → 同时重试 → 大量同时 relist（全量 List）→ APIServer 和 etcd 压力暴涨 → 进一步失败 → **雪崩**。

**缓解手段（分层）**

| 层 | 手段 |
| --- | --- |
| 客户端 | client-go 自带**指数退避** + **随机 watch 超时（5–10min）**；确保使用共享 informer（减少连接数） |
| 客户端 | relist 时确保带 `resourceVersion=0`（走 watch cache 而非 etcd） |
| 客户端 | 减少控制器副本数 / 仅 leader 启动 cache |
| APIServer | 调大 `--watch-cache-sizes`，保证 relist 命中缓存 |
| APIServer | `--goaway-chance` 打散长连接，避免负载不均 |
| APIServer | 滚动升级 + `--shutdown-delay-duration`，避免多副本同时下线 |
| 架构 | 用 WatchList 特性（较新版本）让初始化走 watch 而非 List |
| 架构 | 降低"客户端数量"：DaemonSet agent 合并、禁止在业务 Pod 里起 informer |

**验证方式**：演练（非生产）：同时 kill 所有 APIServer → 观察 `apiserver_request_total{verb="list"}` 峰值与恢复时间；目标：**不出现 etcd 读 QPS 超过基线 N 倍**（N 需按集群容量定义）。

---

### Q35：为什么"禁止在业务 Pod 里起 informer"？

- 每个起 informer 的 Pod = **1 条 watch 长连接 + 1 份全量缓存**。
- 1000 个业务 Pod 都 watch 全集群 Pod → **1000 条连接 + 1000 份缓存** → APIServer 连接数和内存直接被打爆，客户端侧内存也爆炸。
- **正确架构**：**控制器模式** —— 中心化部署 1–3 个副本的控制器做 watch + reconcile，业务 Pod 只通过自己的 API/CR 交互。
- **例外**：确实需要本地感知的 agent（如 CNI、CSI node plugin）→ 必须**严格限制 selector**（只看本节点相关对象）+ metadata-only。

**面试加分**：这是"控制器模式 vs 分布式 agent 模式"的架构权衡题，能说出爆炸半径和连接数模型就很加分。

---

### Q36：高频变更资源（Pod status / Lease / Event）怎么处理？

**问题**：这些资源变更极频繁，informer 的事件量和 CPU 消耗会被它们主导，而业务往往只关心 spec/存在性。

**对策**

1. **不要 watch Event**（改用 EventRecorder 写入，或按需查）。
2. **Lease 只在 node/leader-election 场景 watch**，普通控制器不碰。
3. **Pod 用 Transform 裁掉 **`status`（只保留 metadata + spec.nodeName 等）。
4. **用 **`spec-hash`** 注解 / DeepEqual 过滤**：只处理 spec 真正变化的事件。
5. **Event handler 侧做过滤**：`OnUpdate` 里比较 `old.ResourceVersion` 不够（每次都变），要比较业务字段。

---

### Q37：控制器分片（sharding）怎么做？

**场景**：单一控制器无法处理全量对象（CPU/内存瓶颈）。

**方案**

| 方案 | 做法 | 优点 | 代价 |
| --- | --- | --- | --- |
| **按 namespace 分片** | 每个副本负责一组 namespace（一致性 hash 或静态划分） | 简单、缓存范围小 | 需处理 namespace 增删与再平衡 |
| **按对象 key hash 分片** | 所有副本 watch 全量（或用 selector 无法分片），但 reconcile 时按 key hash 判断归属 | 天然负载均衡 | **缓存仍是全量**，只省 CPU 不省内存 |
| **按 label/租户分片** | 每个副本 watch 不同 label | 缓存和 CPU 都降 | 需要 label 体系支撑 |
| **一致性 hash + 协调器** | 用 CR 记录分片归属 | 动态再平衡 | 复杂 |

**关键**：**只有缩小了 watch 范围的方案才省内存**；仅按 key hash 分 reconcile 不省内存（缓存还是全量）。这是常见的认知误区。

---

### Q38：Informer 冷启动慢（WaitForCacheSync 超时）怎么排查和优化？

**排查**

- 观察 List 请求耗时：`histogram_quantile(0.99, rate(apiserver_request_duration_seconds_bucket{verb="list",resource="pods"}[5m]))`
- 对象数量：`kubectl get pods -A --no-headers | wc -l`
- 是否被限流：`apiserver_flowcontrol_rejected_requests_total`

**优化**

1. 缩小缓存范围（selector / namespace）← **最有效**
2. Transform 减小单对象大小
3. 调大 List 页大小（减少往返，\*\*【需实测】\*\*内存换时间）
4. 确保客户端使用 **protobuf** 而非 JSON（client-go 对内置类型默认 protobuf；自定义客户端需显式配置 `ContentConfig.ContentType`）
5. 提高客户端 `QPS/Burst`（注意别超过 APF 配额）
6. **放宽 **`WaitForCacheSync`** 的超时**，并**处理超时返回值**（不能假成功）

---

### Q39：Informer 与控制器"无意义 reconcile"的成本有多高？

**成本模型**

```
每次 reconcile ≈ 1 次缓存读（微秒）+ 1 次状态判断
              + 概率性的 1~N 次 APIServer 写（毫秒级 + etcd 压力）
```

- 若 15 万对象每 10 秒 resync 一次 → **1.5 万次/秒**的 reconcile → CPU 打满，且任何一次"误判需要写"都会放大成 etcd 写风暴。
- **对策**（见 Q23）：resync 设 0 或数小时；reconcile 入口做语义比较；写前用 `Semantic.DeepEqual` 判断状态是否真的变了。

---

### Q40：什么是"幽灵对象"？如何避免基于过期缓存做危险操作？

- **幽灵对象**：对象在集群中已删除，但本地缓存还没收到删除事件（watch 断连、事件积压）→ 控制器认为它还存在。
- **危险场景**：控制器基于幽灵对象去**删除外部资源**（云盘、DNS）或**创建替代对象**（ReplicaSet 看到 Pod 消失就新建 → 实际 Pod 还在 → 超量创建）。
- **对策**
  1. 写操作前对关键对象做**直读 APIServer 校验**（`APIReader`，绕过缓存）
  2. 所有删除外部资源的操作必须 **幂等 + 确认存在性**
  3. 利用 `DeletionTimestamp` + finalizer 保证顺序
  4. 定期**全量对账**（对账周期按业务容忍度设定）

---

### Q41：多集群场景下 Informer 怎么用？

- 每个集群一个 `Manager` / `SharedInformerFactory` + 一个 rest.Config。
- **注意点**
  - 缓存爆炸：N 个集群 × 全量对象 → 必须用 selector/Transform 或 metadata-only
  - 事件风暴：多集群事件汇聚到一个控制面，workqueue 需限速
  - 故障隔离：单集群 APIServer 不可用不应拖垮全局控制器（每个集群独立的 stopCh + 独立的健康检查）
  - 常用库：controller-runtime 的 `mcsv1alpha1` / Cluster API / Karmada / OCM

---

### Q42：Informer 会不会导致 APIServer 的 watch 连接数打爆？

会。模型：

```
watch 连接数 ≈ Σ(每个客户端进程 × 该进程 watch 的资源类型数)
```

- 100 个控制器 Pod × 各 watch 5 种资源 = 500 条连接（可接受）
- 1000 个业务 Pod × 各 watch 3 种 = 3000 条（危险，尤其配合长连接内存开销）
- **APIServer 侧每个 watch = 1 goroutine + buffer**，连接数过多会导致内存上涨和调度开销。

**对策**：共享 informer、减少客户端数量、中心化控制器、监控 `apiserver_current_inflight_requests`（长连接计入）。

---

### Q43：大规模下 CRD 控制器的特殊注意点？

1. **CR 数量可能远大于内置资源**（百万级）→ 缓存内存是第一瓶颈 → 必须 Transform / 分片 / metadata-only。
2. **CRD 的 status 更新会触发 informer 自己产生的事件** → 若控制器"改 status → 触发 OnUpdate → 又改 status"会形成**无限自激循环**（hot loop）。
   - 必须：只在状态真的变化时写；或用 `Generation` / `observedGeneration` 判断。
3. **conversion webhook 参与**：CR 多版本时每次读写都可能触发 webhook 调用（同步、有 P99 成本）。
4. **finalizer 卡死** → 大量 Terminating CR 堆积在缓存里。

---

### Q44：Informer 与"控制器重启后 CPU 飙高"是什么关系？

**现象**：控制器重启后 CPU 和 APIServer 压力明显高于稳态，持续数分钟。

**原因**

1. **冷启动全量 List**（大资源慢）
2. **全量事件回放**：首次 List 的 N 个对象全部以 `Added` 事件触发 handler → N 次 reconcile 一次性涌入 workqueue
3. resync（若配置）叠加

**优化**

- 用 `isInInitialList` 区分存量对象：启动时只做**轻量登记**，不做重业务动作
- 对存量对象做**分批/限速入队**（`AddAfter` 打散）
- 提高 worker 并发但**限制对 APIServer 的写速率**（避免启动时写风暴）
- 冷启动期设置"静默期"（等待缓存稳定 + 事件消化完成再开始有副作用的操作）

---

### Q45：如何为 Informer 驱动的控制器设定 SLO？

| SLO | 建议目标 | 测量 |
| --- | --- | --- |
| 调谐时延（对象变更 → 处理完成） | P99 \< 30s（【建议值】，按业务定） | 需自行埋点：事件发生时间 → reconcile 完成时间 |
| 调谐成功率 | \> 99.9% | `controller_runtime_reconcile_total{result}` |
| 队列深度 | 稳态 \< 100，且不单调增长 | `workqueue_depth` |
| 冷启动时间 | 缓存同步 + 存量处理 \< N 分钟 | 启动日志埋点 |
| 控制器可用性 | 99.9%（含切主时间） | leader 选举 + readiness |

**错误预算**：调谐失败率超预算 → 冻结控制器发版，先修稳定性。

---

### Q46：讲一个你优化过 Informer 的案例（开放式题框架）

STAR + 数据：

1. **S**：规模（对象数、控制器副本、资源类型）
2. **P**：用指标描述（"控制器 RSS 从 1.2GB 涨到 6GB 并 OOMKilled；workqueue\_depth 峰值 8 万；冷启动 12 分钟"）
3. **A**：分层（缩小范围 → Transform → 关闭 resync → 分批入队 → 限速），每步给代码与灰度比例
4. **R**：前后对比（RSS、启动时间、队列深度、APIServer List QPS），并说明回滚条件
5. **L**：沉淀规范（CRD 上线评审必填"预估对象数 + 缓存策略"）

---

## Part 4：故障排查实战（Q47–Q56）

### Q47：`WaitForCacheSync` 一直返回 false / 超时？

**排查顺序**

1. 是否 `Start()` 之后才调 `WaitForCacheSync()`？（最常见：顺序写反）
2. **RBAC**：客户端有没有该资源的 `list` + `watch` 权限？缺权限会持续失败（日志里有 `forbidden`）。
3. 是否被 APF 限流（429）？看 `apiserver_flowcontrol_rejected_requests_total`。
4. List 是否超时（对象太多 / APIServer 慢）？
5. CRD 是否处于 `Established`？CRD 未就绪 → List 报 404 → 永远同步不了。
6. `stopCh` 是否已被提前关闭（context 取消）。

**验证**：`kubectl auth can-i list pods --as=system:serviceaccount:ns:sa`；看控制器日志的 reflector 报错。

---

### Q48：控制器内存持续上涨直至 OOM？

**定位**

```bash
# 1) 看是不是缓存本身大（正常）还是泄漏
go tool pprof -inuse_space http://pod:port/debug/pprof/heap
# 2) 常见泄漏点
```

| 原因 | 特征 | 修复 |
| --- | --- | --- |
| **缓存范围过大**（最常见，非泄漏） | heap top 是 `threadSafeStore` / `unstructured` | selector / Transform / metadata-only |
| **processorListener buffer 堆积** | heap 里有大量 notification | handler 只 enqueue，不做慢操作 |
| **workqueue 无限增长** | `workqueue_depth` 单调增长 | 检查是否漏 `Done()` / 处理速度不足 |
| **goroutine 泄漏** | `go_goroutines` 单调增长 | 检查未关闭的 stopCh、未退出的循环 |
| **unstructured 缓存** | 内存是 typed 的数倍 | 改用 typed 或 Transform |
| **finalizer 卡死导致对象不释放** | 缓存里 Terminating 对象堆积 | 修复 finalizer 处理逻辑 |

---

### Q49：控制器"卡住"，对象变了但没反应？

**决策树**

```
控制器无反应
├─ workqueue_depth > 0 但 worker 不动？
│    → 漏了 Done(key)？worker panic？并发数=0？
├─ workqueue_depth == 0 且缓存没更新？
│    → watch 假死（见 Q21）：检查是否收到 bookmark
│    → LB idle timeout 静默切断
│    → Reflector 在退避重试中（看日志 backoff）
├─ 缓存更新了但 handler 没触发？
│    → handler 注册前 informer 已 Start？
│    → OnUpdate 里被"无意义变更"过滤掉了？
└─ 事件有但 reconcile 失败？
     → 看 reconcile_total{result="error"} 与日志
```

**watch 假死验证**：对比 APIServer 侧 `apiserver_request_duration_seconds{verb="watch"}` 的连接存活情况，或直接看客户端是否长时间无任何事件（含 bookmark）。

---

### Q50：日志大量 `too old resource version` / `410 Gone`？

- RV 过期 → 触发 relist。
- **偶发**：正常（尤其长时间无变更的资源，bookmark 未启用时）。
- **频繁**：说明客户端经常长时间不消费事件，或 APIServer watch cache 太小 / compaction 太激进。
- **处理**：
  1. 调大对应资源的 `--watch-cache-sizes`
  2. 确保客户端开启 bookmark（`allowWatchBookmarks`）
  3. 排查客户端为什么卡住（handler 慢、GC 停顿、CPU 打满）
  4. **大规模危害**：大量客户端同时 410 → 同时 relist → 惊群

---

### Q51：日志出现 `DeletedFinalStateUnknown` 怎么处理？

- **含义**：删除事件到达时已拿不到对象的最终状态，只有 key。
- **成因**：watch 断连期间的删除，relist 时通过 Replace 补发，但拿不到对象体。
- **必须处理**：

```go
if d, ok := obj.(cache.DeletedFinalStateUnknown); ok {
    // 只有 key，没有 Obj
    namespace, name, _ := cache.SplitMetaNamespaceKey(d.Key)
    // 用 key 去清理外部资源（必须幂等）
}
```

- **反模式**：类型断言只处理 `*v1.Pod` 而忽略这种情况 → **外部资源泄漏**（云盘不释放是最典型的线上事故）。

---

### Q52：控制器启动时把存量对象"重做了一遍"，造成事故？

**场景**：控制器升级/重启 → 15 万存量对象全部走 `OnAdd` → 触发全量业务动作（如全量重启 Pod、全量调用外部 API）。

**对策**

1. 用 `OnAdd(obj, isInInitialList)` 区分：存量对象只做**登记/对账**，不做破坏性动作。
2. 首次同步完成后再开启"有副作用"的处理（用 `HasSynced()` 门控）。
3. 对存量处理做**限速 + 分批**（`AddAfter` 打散）。
4. 幂等：即使重放也不会产生副作用。

---

### Q53：APIServer 侧看到大量来自某控制器的 List 请求？

**排查**

```
sum by (client,resource) (rate(apiserver_request_total{verb="list"}[5m]))
```

**常见根因**

| 根因 | 特征 | 修复 |
| --- | --- | --- |
| 频繁 relist（410 循环） | list 与 410 交替出现 | 见 Q50 |
| 每个 handler 一份 informer（没共享） | 同资源多份 | 用 SharedInformerFactory |
| 业务代码绕过缓存直接 List APIServer | 代码里有裸 `client.List` | 改为从缓存读（或用 `APIReader` 但控制频率） |
| 控制器反复崩溃重启 | restartCount 高 | 修复崩溃根因 |
| 多副本都启动 cache | 副本数 × 全量 | 仅 leader 启动 |

---

### Q54：`reconcile` 出现大量 409 Conflict？

- 正常现象（乐观锁），但**比例过高**说明：
  - 多个控制器/副本在改同一对象（分片逻辑有问题，多个副本都认为自己负责）
  - 缓存太旧（Update 用的 resourceVersion 过期）→ 应先重新 Get
  - 热对象被高频并发写
- **处理**：`retry.RetryOnConflict` + 重新 Get（从 **APIReader 直读**，不要用可能陈旧的缓存对象）。
- **监控**：统计 409 占写请求比例，突增报警。

---

### Q55：控制器切主（leader 切换）后长时间不可用？

- **原因**：新 leader 需要冷启动缓存（全量 List）+ 消化存量事件。
- **优化**：缩小缓存范围、Transform、提高 List 并发/页大小、预留内存避免 GC 抖动。
- **监控**：埋点"从成为 leader 到首次成功 reconcile"的耗时，设告警。
- **兜底**：切主期间保持 readiness=false，避免被判定为健康但不干活。

---

### Q56：如何优雅地重启/升级一个 Informer 控制器？

**标准流程**

1. 新版本 Pod 启动 → 先 `WaitForCacheSync`（readiness 未就绪，不接活）
2. 缓存同步完成 → readiness 就绪 → 参与 leader 竞选
3. 旧 Pod 收到 SIGTERM：
   - 停止接收新任务
   - `queue.ShutDownWithDrain()` 排空在途任务（带超时）
   - leader 释放（`LeaderElectionReleaseOnCancel=true`，让新副本快速接管）
   - 关闭 informer stopCh
4. **回滚触发条件**：新版本 `reconcile_total{result="error"}` 错误率 \> 基线 2 倍，或冷启动超时 → 回滚镜像。
5. **注意**：若控制器有外部副作用，升级前确认**幂等性**，避免新旧版本双写（确保 leader 释放及时）。

---

## Part 5：反模式与风险清单

| # | 反模式 | 证据（现象/配置） | 等级 | 修复动作 | 验证方式 |
| --- | --- | --- | --- | --- | --- |
| 1 | 在业务 Pod 里起全量 informer | 每 Pod 一份缓存 + 一条 watch；APIServer watch 连接数 = Pod 数 | 高 | 改为中心化控制器；agent 必须用 selector + metadata-only | `apiserver_current_inflight_requests` 与客户端 RSS 下降 |
| 2 | 未使用 SharedInformerFactory | 同进程多个控制器各起一份 informer | 高 | 统一 factory + 共享 informer | 进程 RSS 与 goroutine 数下降 |
| 3 | handler 里做慢业务逻辑 | `processorListener` buffer 堆积；内存线性上涨 | 高 | handler 只 enqueue；业务放 worker | buffer 深度归零、内存平稳 |
| 4 | 漏调 `queue.Done(key)` | 该 key 永久卡住，不再入队 | 高 | 用 `defer queue.Done(key)` | `workqueue_depth` 不出现"有 depth 但无进展" |
| 5 | resync 设成秒级 | reconcile QPS 与对象数成正比，CPU 打满 | 中高 | 设 0 或数小时；改为定期全量对账 | CPU 与 reconcile\_total 下降 |
| 6 | 忽略 `DeletedFinalStateUnknown` | 外部资源（云盘/DNS）泄漏 | 高 | 显式处理该类型，删除操作幂等 | 演练：断网删除对象后恢复，外部资源被正确回收 |
| 7 | 直接修改缓存对象 | 缓存被污染，甚至误写回集群 | 中高 | 取出即 `DeepCopy()` | Code review + 单测 |
| 8 | 用 unstructured 缓存大资源 | 内存是 typed 的数倍 | 中高 | 改 typed / Transform | heap profile 对比 |
| 9 | 缓存全集群对象但只关心一小部分 | heap top 是无关对象 | 高 | namespace + label/field selector | 缓存对象数（自埋点）下降 |
| 10 | 启动时对存量对象做破坏性动作 | 升级后全量重启 Pod / 全量调用外部 API | 高 | `isInInitialList` 区分 + 限速分批 | 演练：重启控制器，无破坏性副作用 |
| 11 | 控制器自激循环（改 status → 触发 → 再改） | APIServer 写 QPS 持续高位、无限事件 | 高 | 只在状态真变时写；用 observedGeneration 门控 | 写 QPS 回落到预期 |
| 12 | Transform 裁剪后仍用该缓存对象做 Update | 字段被写丢 | 中高 | 写入前直读或该类型不裁剪 | 对比写入前后对象字段完整性 |
| 13 | 多副本全部启动全量 cache | N × 内存 | 中 | 仅 leader 启动（controller-runtime 默认） | 副本内存总量下降 |
| 14 | WaitForCacheSync 不检查返回值 | 空缓存状态下开始 reconcile → 误删/误建 | 中高 | 检查返回值，false 则退出或重试 | 故障注入：RBAC 撤权后控制器应拒绝启动 |
| 15 | 无 workqueue/控制器指标 | 出问题只能猜 | 中 | 暴露 workqueue + reconcile 指标并告警 | 有 dashboard 与告警 |

---

## Part 6：可直接套用的代码与配置

### 6.1 生产级 SharedInformerFactory 骨架

```go
factory := informers.NewSharedInformerFactoryWithOptions(
    clientset,
    10*time.Hour,                        // resyncPeriod【建议值】；设 0 关闭
    informers.WithNamespace(""),         // 空=全命名空间；按需收敛
    informers.WithTweakListOptions(func(o *metav1.ListOptions) {
        o.LabelSelector = "app.kubernetes.io/managed-by=my-operator"
    }),
)

podInformer := factory.Core().V1().Pods()

podInformer.Informer().AddEventHandler(cache.ResourceEventHandlerFuncs{
    AddFunc: func(obj interface{}) {
        if key, err := cache.MetaNamespaceKeyFunc(obj); err == nil {
            queue.Add(key)         // 只入队
        }
    },
    UpdateFunc: func(old, new interface{}) {
        oldPod := old.(*corev1.Pod)
        newPod := new.(*corev1.Pod)
        // 关键：过滤无意义变更（resync 与 status 抖动）
        if oldPod.GetResourceVersion() != newPod.GetResourceVersion() &&
           apiequality.Semantic.DeepEqual(oldPod.Spec, newPod.Spec) &&
           oldPod.GetDeletionTimestamp().IsZero() == newPod.GetDeletionTimestamp().IsZero() {
            return
        }
        if key, err := cache.MetaNamespaceKeyFunc(new); err == nil {
            queue.Add(key)
        }
    },
    DeleteFunc: func(obj interface{}) {
        // 必须处理 DeletedFinalStateUnknown
        var key string
        if d, ok := obj.(cache.DeletedFinalStateUnknown); ok {
            key = d.Key
        } else if k, err := cache.MetaNamespaceKeyFunc(obj); err == nil {
            key = k
        } else {
            return
        }
        queue.Add(key)
    },
})

factory.Start(ctx.Done())
// 必须检查返回值！
if !cache.WaitForCacheSync(ctx.Done(), podInformer.Informer().HasSynced) {
    return fmt.Errorf("failed to sync pod informer cache")
}
```

### 6.2 controller-runtime：缩小缓存 + Transform 裁剪

```go
mgr, err := ctrl.NewManager(cfg, ctrl.Options{
    Cache: cache.Options{
        // 默认同步周期【建议值】数小时；0 表示不自动 resync
        SyncPeriod: ptr.To(10 * time.Hour),
        ByObject: map[client.Object]cache.ByObject{
            // Pod 只保留需要的字段，大幅降内存
            &corev1.Pod{}: {
                Transform: func(obj interface{}) (interface{}, error) {
                    pod, ok := obj.(*corev1.Pod)
                    if !ok { return obj, nil }
                    // 注意：Transform 后的对象不能用于 Update！
                    trimmed := &corev1.Pod{
                        ObjectMeta: metav1.ObjectMeta{
                            Name:            pod.Name,
                            Namespace:       pod.Namespace,
                            Labels:          pod.Labels,
                            Annotations:     map[string]string{}, // 只留需要的 key
                            UID:             pod.UID,
                            ResourceVersion: pod.ResourceVersion,
                        },
                        Spec: corev1.PodSpec{
                            NodeName: pod.Spec.NodeName,
                        },
                    }
                    return trimmed, nil
                },
            },
            // 只缓存带特定标签的 ConfigMap
            &corev1.ConfigMap{}: {
                Label: labels.SelectorFromSet(labels.Set{"my-operator/cache": "true"}),
            },
        },
        // 完全不缓存的大资源（读走直连）
        // Uncached 通过 client.Options 配置
    },
})
```

### 6.3 workqueue 限速器与正确用法

```go
queue := workqueue.NewTypedRateLimitingQueue(
    workqueue.NewTypedWithMaxWaitRateLimiter(
        workqueue.NewTypedItemExponentialFailureRateLimiter(5*time.Millisecond, 1000*time.Second),
        workqueue.NewTypedItemBucketRateLimiter(10, 100),
        2*time.Second,     // 最大等待上限【建议值】，避免过长退避
    ),
)

func (c *Ctrl) runWorker(ctx context.Context) {
    for c.processNext(ctx) {}
}

func (c *Ctrl) processNext(ctx context.Context) bool {
    key, shutdown := c.queue.Get()
    if shutdown { return false }
    defer c.queue.Done(key)           // 必须！否则该 key 永久卡死

    if err := c.reconcile(ctx, key); err != nil {
        c.queue.AddRateLimited(key)   // 失败：退避重试
        return true
    }
    c.queue.Forget(key)               // 成功：清除退避计数
    return true
}

// 优雅关闭
func (c *Ctrl) Shutdown() {
    c.queue.ShutDownWithDrain()       // 排空在途任务，避免丢失
}
```

### 6.4 自定义 Indexer（按 nodeName 索引 Pod）

```go
informer := factory.Core().V1().Pods().Informer()
err := informer.AddIndexers(cache.Indexers{
    "byNode": func(obj interface{}) ([]string, error) {
        pod, ok := obj.(*corev1.Pod)
        if !ok || pod.Spec.NodeName == "" {
            return nil, nil
        }
        return []string{pod.Spec.NodeName}, nil
    },
})
if err != nil { return err }

// 使用
objs, err := informer.GetIndexer().ByIndex("byNode", "node-1")
```

> 注意：Indexer 在 informer **启动前**添加；启动后 AddIndexers 会报错。

### 6.5 分片控制器（按 namespace hash）

```go
// 简单一致性分片：只处理属于自己的 key
func (c *Ctrl) owns(key string) bool {
    h := fnv.New32a()
    h.Write([]byte(key))
    return h.Sum32()%c.totalShards == c.shardIndex
}

// 更省内存的做法：每个分片用 WithNamespace 只 watch 自己负责的 namespace
factory := informers.NewSharedInformerFactoryWithOptions(
    clientset, resync,
    informers.WithNamespace(myNamespaces...),
)
```

### 6.6 客户端 QPS/Burst（避免被限速拖慢冷启动）

```go
cfg := rest.CopyConfig(origCfg)
cfg.QPS = 50        // 【建议值】需结合对象规模与 APF 配额实测
cfg.Burst = 100
```

> **注意**：提高客户端 QPS 会加大 APIServer 压力，大规模场景必须**同时**确认 APF 的 FlowSchema 配额，避免出现"客户端提速 → 被 APF 拒绝 → 更慢"的负反馈。

### 6.7 客户端侧指标埋点（建议暴露）

```go
var (
    reconcileTotal = prometheus.NewCounterVec(prometheus.CounterOpts{
        Name: "myctrl_reconcile_total",
    }, []string{"result"})
    reconcileLatency = prometheus.NewHistogramVec(prometheus.HistogramOpts{
        Name:    "myctrl_reconcile_duration_seconds",
        Buckets: prometheus.DefBuckets,
    }, []string{})
    cachedObjects = prometheus.NewGaugeVec(prometheus.GaugeOpts{
        Name: "myctrl_cached_objects",
    }, []string{"resource"})
)
```

> `cached_objects` 是最容易被忽略但极有价值的指标：**它直接暴露缓存范围是否失控**，比内存更早发现问题。

---

## Part 7：面试自检清单

- [ ] 能画出 Reflector → DeltaFIFO → Controller → Indexer → Processor 的数据流
- [ ] 说清 DeltaFIFO 存在的三个理由（去重/顺序/删除兜底）
- [ ] **明确指出 resync 不产生任何 APIServer 请求**，并说清它与 relist 的区别
- [ ] 知道 watch 超时是随机的（5–10min）及其惊群缓解目的
- [ ] 能写出"handler 只 enqueue"的正确骨架，并知道漏 `Done()` 的后果
- [ ] 知道 `DeletedFinalStateUnknown` 必须处理，否则外部资源泄漏
- [ ] 能列出降低缓存内存的 5 种手段，并知道哪些省内存、哪些只省 CPU
- [ ] 说清"按 key hash 分片不省内存"（缓存仍全量）
- [ ] 知道 controller-runtime 默认只有 leader 启动 cache
- [ ] 能给出内存开销的量化算式（对象数 × 单对象大小 × 膨胀系数）
- [ ] 能说出惊群的成因与 4 层以上缓解手段
- [ ] 知道 Transform 后的对象不能用于 Update
- [ ] 能给出冷启动优化与"存量对象破坏性动作"的防护
- [ ] 能把任意优化配上下灰度比例、回滚触发条件与验证指标

---

**免责与边界**：本文为面试复习与架构评审材料，不针对任何具体生产集群直接执行操作。所有生产变更请走：变更评审 → 灰度 → 观察 → 回滚预案。文中【默认值】请以目标版本 client-go / controller-runtime 源码为准，【需实测】项请以你的监控与 profile 数据为准。
