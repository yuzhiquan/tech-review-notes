# 技术复习笔记 / Tech Review Notes

个人技术复习资料集：一部分是**云原生与 LLM 基础设施的源码级流程图**，另一部分是**语言与算法的复习手册**（Go / Python / LeetCode）。内容以中文为主。

A personal collection of technical review material: **source-level architecture and
flow diagrams** for cloud-native and LLM infrastructure, plus **language and
algorithm review handbooks** (Go / Python / LeetCode). Mostly in Chinese.

```
diagrams/
  kubernetes/     Kubernetes 核心组件源码级流程图
  service-mesh/   Istio 控制面 / 数据面
  llm/            LLM 训练与推理流程
  distributed/    分布式计算框架
notes/
  go/             Go 语言核心特性
  python/         Python 核心特性
  algorithms/     LeetCode 刷题模板与 Hot 100 题解
```

---

## 一、流程图 / Diagrams

### Kubernetes

> 基于 Kubernetes v1.29+ / v1.32+ 源码 · Based on Kubernetes v1.29+ / v1.32+ sources.

#### Scheduler 调度器

| 图 / Diagram | 说明 / Description |
| --- | --- |
| [`scheduler-full-flow.png`](diagrams/kubernetes/scheduler-full-flow.png) | 一个 Pod 从调度队列到绑定 Node 的完整过程：PreFilter → Filter → PostFilter → Score → Reserve → PreBind → Bind，含各阶段内置插件与默认权重 |
| [`scheduler-source-flow.png`](diagrams/kubernetes/scheduler-source-flow.png) | 调度器工作机制源码流程图：调度队列 + Scheduling Framework 全部扩展点、`scheduleOne` 主流程、时序图、关键源码文件 |
| [`scheduler-queues.png`](diagrams/kubernetes/scheduler-queues.png) | 三个调度队列（ActiveQ / BackoffQ / UnschedulableQ）的作用、进出条件、内部实现与退避机制 |

#### API Server

| 图 / Diagram | 说明 / Description |
| --- | --- |
| [`apiserver-source-flow.png`](diagrams/kubernetes/apiserver-source-flow.png) | 从请求接入到响应的完整链路：AuthN → AuthZ → Admission → Storage → Response，含 Watch Cache、etcd3 存储、内部控制循环与关键源码路径 |

#### Informer / client-go

| 图 / Diagram | 说明 / Description |
| --- | --- |
| [`informer-overview.png`](diagrams/kubernetes/informer-overview.png) | Informer 机制全景图：Reflector、DeltaFIFO、Controller、Processor、LocalStore、ResourceEventHandler 及启动时序 |
| [`informer-detail.png`](diagrams/kubernetes/informer-detail.png) | 基于 Watch 的事件驱动缓存与分发机制，含组件协作关系与事件处理详细时序 |
| [`informer-source-flow.png`](diagrams/kubernetes/informer-source-flow.png) | 源码级流程图：`SharedIndexInformer`、`ListerWatcher`、`Reflector.processLoop`、DeltaFIFO 去重合并、Indexer/Store 索引机制 |

#### kubelet

| 图 / Diagram | 说明 / Description |
| --- | --- |
| [`kubelet-pod-startup.png`](diagrams/kubernetes/kubelet-pod-startup.png) | 从 kubelet 发现 Pod 到 Pod Running/Ready：`syncLoop` → `handlePodSyncs` → `syncPod`，含 Volume/镜像/Sandbox/CNI/容器启动与探针，及 Pod 状态流转 |

#### 存储与网络 / Storage & Networking

| 图 / Diagram | 说明 / Description |
| --- | --- |
| [`csi-source-flow.png`](diagrams/kubernetes/csi-source-flow.png) | CSI 控制面 + 节点面协作完成卷生命周期：Create → Attach → Stage/Publish → Unpublish → Detach → Delete，含 sidecar、gRPC 接口分类与源码路径 |
| [`cni-source-flow.png`](diagrams/kubernetes/cni-source-flow.png) | 从 Pod 创建到网络就绪的完整流程：kubelet → CRI Sandbox → CNI ADD/DEL、veth/IPAM/路由配置、.conflist 与 STDIN/STDOUT 示例 |

### 服务网格 / Service Mesh

| 图 / Diagram | 说明 / Description |
| --- | --- |
| [`istio-components.png`](diagrams/service-mesh/istio-components.png) | Istio (1.20+) 各组件工作机制与源码流程：控制面 istiod（discovery / push / xds / Citadel / Galley / Sidecar Injector）、数据面 Envoy（iptables 劫持、Listener/RDS/CDS/EDS/SDS）、XDS 与 mTLS 流程，含请求全链路时序图与 `istio/istio` 源码目录结构 |

### LLM 基础设施 / LLM Infrastructure

| 图 / Diagram | 说明 / Description |
| --- | --- |
| [`llm-training-inference-overview.png`](diagrams/llm/llm-training-inference-overview.png) | LLM 训练与推理的完整流程与核心组件（总览）：数据准备 → 模型构建 → 并行策略 (DP/TP/PP/ZeRO) → 训练循环 → 评估与 Checkpoint；推理侧 Serving、Prefill/Decode、KV Cache、PagedAttention、Continuous Batching 与部署形态 |
| [`llm-training-inference-detail.png`](diagrams/llm/llm-training-inference-detail.png) | 同一主题的详解版：训练侧含 Transformer Block 内部结构（RMSNorm/MHA/FFN/残差）、损失与反向传播细节、LoRA/RLHF 辅助模块、训练监控；推理侧含 Prefill/Decode 两阶段展开、KV Cache 读写、采样策略 (Top-k/Top-p/Temperature)、后处理与可观测性 |

### 分布式计算 / Distributed Computing

| 图 / Diagram | 说明 / Description |
| --- | --- |
| [`ray-architecture.png`](diagrams/distributed/ray-architecture.png) | Ray (≥ 2.x) 架构、组件与源码级工作流程：GCS、Raylet、Object Manager、Plasma Store、Worker，含任务提交时序与关键数据结构 |

---

## 二、复习笔记 / Notes

### Go

| 文档 / Document | 说明 / Description |
| --- | --- |
| [`go-language-review.pdf`](notes/go/go-language-review.pdf) | Go 语言核心特性复习指南（35 页）：语法、类型系统、并发与 GMP 调度、内存与 GC、接口与组合、工程实践，标注易错点与易考点 |

### Python

| 文档 / Document | 说明 / Description |
| --- | --- |
| [`python-core-features-review.pdf`](notes/python/python-core-features-review.pdf) | Python 核心特性复习手册（50 页）：从语言语义到 CPython、并发、工程实践与性能优化，面向已有编程经验的工程师 |

### 算法 / Algorithms

以「模式」而非「题目」为单位组织：Hot 100 被压缩为约 14 个可迁移模式（哈希、双指针、滑动窗口、前缀和、链表、树、图、回溯、二分、单调栈、堆、贪心、DP、技巧），并要求能在 Go 与 Python 间快速切换实现。

| 文档 / Document | 说明 / Description |
| --- | --- |
| [`leetcode-hot100-quick-review.pdf`](notes/algorithms/leetcode-hot100-quick-review.pdf) | LeetCode Hot 100 Go + Python 双语言快速复习手册（62 页）：模式地图（看到什么信号 → 想到什么算法）、核心思路、易错点、扩展题与 60 天复习路径 |
| [`leetcode-hot100-full-solutions.pdf`](notes/algorithms/leetcode-hot100-full-solutions.pdf) | LeetCode Hot 100 全量题解手册（207 页）：100 题逐题图解，每题固定结构 —— 题意抽象 → 核心观察 → 不变量/状态 → 手推图 → 算法步骤 → 双语言实现 → 复杂度 → 易错点 → 扩展 |
| [`go-leetcode-templates.pdf`](notes/algorithms/go-leetcode-templates.pdf) | Go 刷题模板速查：双指针、哈希表、栈、二叉树 DFS、二分查找等常用套路，附 Go 刷题易错点 |

---

## 使用建议 / How to use

- 流程图分辨率较高，建议点开原图查看（GitHub 上点击图片 → 查看原始文件）。
- 图中蓝色文字多为源码路径（如 `pkg/scheduler/framework/plugins/`），可直接在对应版本的源码树中定位。
- 流程图内容基于特定版本源码整理，不同版本的插件集合、队列实现和扩展点可能有差异，请以实际源码为准。

## 关于准确性 / Accuracy

这些图与笔记由 AI 辅助生成，用于快速建立整体认知，**不是权威文档**。其中的源码路径、插件列表、函数名、语言细节可能与你所用版本存在差异或错误。使用前请对照官方文档与对应版本的源码核对。

These diagrams and notes were AI-generated as a fast way to build a mental model —
they are **not authoritative documentation**. Source paths, plugin lists, function
names, and language details may be inaccurate or drift from the version you're on.
Verify against official docs and the actual source before relying on any detail.
Corrections via issues/PRs are welcome.

## License

内容以 [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/) 授权，可自由使用与修改，请保留署名。

Content is licensed under [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/).
