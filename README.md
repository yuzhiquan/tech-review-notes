# Kubernetes 组件源码级流程图 / K8s Components Source-Level Diagrams

一组 Kubernetes 核心组件的**源码级**工作流程图，标注了关键源码文件、函数调用链、数据结构和扩展点。适合用来对照阅读 `kubernetes/kubernetes` 与 `client-go` 源码。

A collection of **source-level** architecture and flow diagrams for Kubernetes core
components, annotated with key source files, call chains, data structures, and
extension points. Meant to be read alongside the `kubernetes/kubernetes` and
`client-go` source trees. Diagram text is in Chinese.

除 Kubernetes 之外，也收录了少量相关分布式系统与 LLM 基础设施的流程图（见「其他」一节）。

A few diagrams cover adjacent distributed-systems and LLM infrastructure topics
rather than Kubernetes itself — see the "Other" section.

> 基于 Kubernetes v1.29+ / v1.32+ 源码 · Based on Kubernetes v1.29+ / v1.32+ sources.

## 目录 / Contents

### Scheduler 调度器

| 图 / Diagram | 说明 / Description |
| --- | --- |
| [`scheduler-full-flow.png`](scheduler-full-flow.png) | 一个 Pod 从调度队列到绑定 Node 的完整过程：PreFilter → Filter → PostFilter → Score → Reserve → PreBind → Bind，含各阶段内置插件与默认权重 |
| [`scheduler-source-flow.png`](scheduler-source-flow.png) | 调度器工作机制源码流程图：调度队列 + Scheduling Framework 全部扩展点、`scheduleOne` 主流程、时序图、关键源码文件 |
| [`scheduler-queues.png`](scheduler-queues.png) | 三个调度队列（ActiveQ / BackoffQ / UnschedulableQ）的作用、进出条件、内部实现与退避机制 |

### API Server

| 图 / Diagram | 说明 / Description |
| --- | --- |
| [`apiserver-source-flow.png`](apiserver-source-flow.png) | 从请求接入到响应的完整链路：AuthN → AuthZ → Admission → Storage → Response，含 Watch Cache、etcd3 存储、内部控制循环与关键源码路径 |

### Informer / client-go

| 图 / Diagram | 说明 / Description |
| --- | --- |
| [`informer-overview.png`](informer-overview.png) | Informer 机制全景图：Reflector、DeltaFIFO、Controller、Processor、LocalStore、ResourceEventHandler 及启动时序 |
| [`informer-detail.png`](informer-detail.png) | 基于 Watch 的事件驱动缓存与分发机制，含组件协作关系与事件处理详细时序 |
| [`informer-source-flow.png`](informer-source-flow.png) | 源码级流程图：`SharedIndexInformer`、`ListerWatcher`、`Reflector.processLoop`、DeltaFIFO 去重合并、Indexer/Store 索引机制 |

### kubelet

| 图 / Diagram | 说明 / Description |
| --- | --- |
| [`kubelet-pod-startup.png`](kubelet-pod-startup.png) | 从 kubelet 发现 Pod 到 Pod Running/Ready：`syncLoop` → `handlePodSyncs` → `syncPod`，含 Volume/镜像/Sandbox/CNI/容器启动与探针，及 Pod 状态流转 |

### 存储与网络 / Storage & Networking

| 图 / Diagram | 说明 / Description |
| --- | --- |
| [`csi-source-flow.png`](csi-source-flow.png) | CSI 控制面 + 节点面协作完成卷生命周期：Create → Attach → Stage/Publish → Unpublish → Detach → Delete，含 sidecar、gRPC 接口分类与源码路径 |
| [`cni-source-flow.png`](cni-source-flow.png) | 从 Pod 创建到网络就绪的完整流程：kubelet → CRI Sandbox → CNI ADD/DEL、veth/IPAM/路由配置、.conflist 与 STDIN/STDOUT 示例 |

### 服务网格 / Service Mesh

| 图 / Diagram | 说明 / Description |
| --- | --- |
| [`istio-components.png`](istio-components.png) | Istio (1.20+) 各组件工作机制与源码流程：控制面 istiod（discovery / push / xds / Citadel / Galley / Sidecar Injector）、数据面 Envoy（iptables 劫持、Listener/RDS/CDS/EDS/SDS）、XDS 与 mTLS 流程，含请求全链路时序图与 `istio/istio` 源码目录结构 |

### 其他 / Other

| 图 / Diagram | 说明 / Description |
| --- | --- |
| [`ray-architecture.png`](ray-architecture.png) | Ray (≥ 2.x) 架构、组件与源码级工作流程：GCS、Raylet、Object Manager、Plasma Store、Worker，含任务提交时序与关键数据结构 |
| [`llm-training-inference-flow.png`](llm-training-inference-flow.png) | LLM 训练与推理的完整流程与核心组件：数据准备 → 模型构建 → 并行策略 (DP/TP/PP/ZeRO) → 训练循环 → 评估与 Checkpoint；推理侧 Serving、Prefill/Decode、KV Cache、PagedAttention、Continuous Batching 与部署形态 |

## 使用建议 / How to use

- 图片分辨率较高，建议点开原图查看（GitHub 上点击图片 → 查看原始文件）。
- 图中蓝色文字多为源码路径（如 `pkg/scheduler/framework/plugins/`），可直接在对应版本的源码树中定位。
- 流程图内容基于特定版本源码整理，不同版本的插件集合、队列实现和扩展点可能有差异，请以实际源码为准。

## 关于准确性 / Accuracy

这些图由 AI 辅助生成，用于快速建立整体认知，**不是权威文档**。图中的源码路径、插件列表、函数名可能与你所用版本存在差异或错误。使用前请对照对应版本的源码树核对。

These diagrams were AI-generated as a fast way to build a mental model — they are
**not authoritative documentation**. Source paths, plugin lists, and function names
may be inaccurate or drift from the version you're on. Verify against the actual
source tree before relying on any detail. Corrections via issues/PRs are welcome.

## License

图表以 [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/) 授权，可自由使用与修改，请保留署名。

Diagrams are licensed under [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/).
