# k8s-gpu-llm-interview-qa

# Kubernetes × GPU / 大模型面试全解析：GPU 硬件 × 调度 × 训练网络 × 存储 × 推理 × 可靠性

> 适用场景：AI Infra / 云原生平台 / SRE 面试（近年大模型方向高频）
> 姊妹篇：《Scheduler》《Kubelet》《网络/CNI》《控制面》《存储/CSI》《Linux/容器底层》
> 说明：本篇会**交叉引用**前六篇的结论（调度扩展点、调度队列、CNI 性能、CSI 流程、cgroup、PSI 等），但不修改它们。
>
> **数字约定（重要）**：
>
> - **厂商公开规格**（H100/A100/MI300X 等）：我给出的是业界公开资料中的量级，**请以厂商官方 Datasheet 复核后再引用**；
> - **推演估算**（显存/带宽/checkpoint 公式）：我会写清公式与假设，标注为"**估算，需用真实框架 profile 验证**"；
> - **实测数据**：本篇不提供任何未经实测的性能数字，一律给出**测量方法**。

---

## 0. 使用说明

### 0.1 GPU/AI Infra 面试的考察层次

| 层级 | 考点 | 表现 |
| --- | --- | --- |
| L1 会用 | 会在 Pod 里写 `nvidia.com/gpu: 1` | 只会跑起来 |
| L2 懂机制 | 说得清 device plugin / CDI / DRA / runtime hook / MIG / 拓扑 | 能画 GPU 接入链路 |
| L3 懂作业语义 | 理解 Gang/Coscheduling、DDP/TP/PP/ZeRO、checkpoint、RDMA 与 NCCL | 能与算法同学对话 |
| L4 有平台/事故经验 | 能讲"MFU 只有 30% 怎么查""训练任务 hang 在 NCCL init 怎么定位""checkpoint 写爆共享存储怎么办"、能做容量与成本设计 | 有指标、有公式、有灰度、有演练 |

### 0.2 术语速查（面试里混用这些词会立刻暴露）

| 缩写 | 含义 |
| --- | --- |
| SM | Streaming Multiprocessor，GPU 的计算单元组 |
| HBM | High Bandwidth Memory，GPU 显存 |
| NVLink / NVSwitch | 卡间高速互联 / 交换芯片 |
| MIG | Multi-Instance GPU，硬件级切分 |
| MPS | Multi-Process Service，多进程共享同一 CUDA context |
| MFU / HFU | Model / Hardware FLOPs Utilization，模型算力利用率 |
| TP / PP / DP / SP / EP | Tensor / Pipeline / Data / Sequence / Expert 并行 |
| ZeRO / FSDP | 优化器/梯度/参数分片的显存优化技术 |
| GDR / GDS | GPUDirect RDMA（网卡直达显存） / GPUDirect Storage（存储直达显存） |
| RDMA / RoCE / IB | 远程直接内存访问 / 基于以太网的 RDMA / InfiniBand |
| PFC / ECN / DCQCN | 无损以太网的流控与拥塞控制 |
| XID | NVIDIA GPU 的错误码（\_driver 层事件） |
| FA / Quantization | FlashAttention / 量化（W8A8、W4A16 等） |
| vLLM / TRT-LLM / SGLang / TGI | 主流推理引擎 |
| TTFT / TPOT / ITL | 首 token 延迟 / 每 token 输出时间 / token 间隔 |

### 0.3 一张 AI 平台组件地图（先有大画面）

```
┌──────────────────────── 平台层（K8s 之上）─────────────────────────┐
│ 作业编排：Training Operator(PyTorchJob) / MPI Operator / Kubeflow    │
│ 队列与配额：Kueue（ClusterQueue/LocalQueue/ResourceFlavor）/ Volcano  │
│ 批调度：Volcano / scheduler-plugins Coscheduling / YuniKorn          │
│ 推理编排：Deployment+LWS / KServe / vLLM Production Stack / AIBrix   │
│ 网关路由：Ingress / Gateway API + Inference Extension（缓存/队列感知） │
├──────────────────────── 资源与设备层 ───────────────────────────────┤
│ NVIDIA GPU Operator（driver / toolkit / device plugin / DCGM / mig） │
│ NVIDIA Network Operator（MOFED 驱动 + 多网卡 CNI + RDMA 设备插件）    │
│ AMD GPU Operator / ROCm · Intel Gaudi · Habana · AWS Neuron / TPU    │
│ NFD / GPU Feature Discovery（给节点打拓扑与型号标签）                 │
│ DRA（Dynamic Resource Allocation）与 CDI（Container Device Interface）│
├──────────────────────── 硬件拓扑层 ─────────────────────────────────┤
│ 节点内：8 卡 HGX + NVSwitch / PCIe Gen5 MIG / NUMA                    │
│ 节点间：IB 或 RoCEv2 无损以太网 + rail-optimized 拓扑 + 高速交换机     │
│ 存储：并行文件系统（Lustre/GPFS/Weka） + 对象存储 + 本地 NVMe 缓存    │
└─────────────────────────────────────────────────────────────────────┘
```

---

# Part A · GPU 硬件与软件栈（14 题）

## A1. GPU 跟 CPU 到底有什么本质区别？

| 维度 | CPU | GPU |
| --- | --- | --- |
| 设计目标 | 低延迟、复杂控制流、通用 | **高吞吐、大规模并行、简单控制流** |
| 执行模型 | 少量强核心，乱序执行、分支预测 | 海量简单核心，**SIMT（单指令多线程）** |
| 调度单位 | 线程/进程 | **warp（32 线程一组）** → 送到 SM |
| 缓存/内存 | 大 cache 换低延迟 | 小 cache + **极高带宽显存（HBM）** |
| 内存带宽 | 数十\~数百 GB/s | **TB/s 量级** |
| 典型瓶颈 | 单核串行延迟 | **访存带宽、占用率（occupancy）、并行度不足** |

**必答的结构**：

```
GPU = 多个 SM（每个 SM 内有 CUDA Core / Tensor Core / 寄存器文件 / 共享内存）
   → 每个 SM 同时跑多个 warp
   → 寄存器文件大小决定 occupancy（能同时驻留多少 warp 来隐藏访存延迟）
   → Tensor Core 专门做矩阵乘（这就是 LLM 的算力来源）
   → HBM 提供带宽；PCIe/NVLink 提供互联带宽
```

**面试题补刀**：为什么大模型训练看的是 **TFLOPS + HBM 带宽 + 互联带宽** 三者，而不是单看 TFLOPS？

> 因为训练是"把巨大的矩阵搬运到计算单元里算完再搬回来"，**带宽不够时 Tensor Core 会空转**。这也是为什么 H100 相对 A100 的提升里，HBM3 带宽与 NVLink 带宽提升同样关键（具体数值请以厂商 Datasheet 为准）。

## A2. 一张卡要看哪几个规格？（挑卡必答题）

| 指标 | 为什么重要 | 典型（公开资料中的量级，请复核） |
| --- | --- | --- |
| **显存容量（HBM）** | 决定"能不能放下模型/优化器/batch" | A100 40/80GB；H100/H200 80GB / 更高；MI300X 192GB 级别 |
| **显存带宽** | 决定访存密集算子的速度 | A100 约 2 TB/s；H100 约 3.35 TB/s |
| **算力（BF16/FP8 TFLOPS）** | 决定 GEMM 速度 | 不同代次差异巨大，**以 Datasheet 为准** |
| **互联带宽（NVLink vs PCIe）** | 决定**卡间**集合通信成本 | NVLink 远高于 PCIe；**跨节点则看 IB/RoCE** |
| **功耗（TDP）** | 决定机房/机架密度与设计 | 主流训练卡 350W\~700W 级 |
| **ECC / 可靠性特性** | 长时间训练的静默数据风险 | 数据中心卡通常支持 ECC |

**必答的两条"选型常识"**：

1. **游戏卡/消费级卡（如 RTX 40 系）不适合数据中心**：无 ECC、显存小、驱动与保修限制、无 NVLink、机架形态与供电不匹配——但在**极小预算的推理/实验**场景会被用到，面试要能说出取舍；
2. **同一名字可能有多个 SKU**：H100 有 SXM（NVSwitch 全互联，带宽高）与 PCIe（互联受限）两种形态，**采购时看互联，不只是看算力**；中国区特供型号（如 H800/A800 之类）的互联或算力有削减，务必核对官方规格。

## A3. NVLink / NVSwitch / PCIe 与为什么要"8 卡一台"

```
同一个 NVSwitch 域（典型 8 卡 HGX）：
   GPU0..GPU7 之间经过 NVSwitch 高速互联，带宽远高于 PCIe
   ⇒ 张量并行（TP）这种"每层都要 AllReduce"的切法通常限制在单机内（NVLink domain）

跨机：
   走网卡 → IB / RoCEv2 → 交换机
   ⇒ 带宽通常显著低于机内 NVLink ⇒ **把通信量大的并行切法留在机内**
```

**面试金句**：

> "大模型并行策略的第一性原理是：**通信代价最大的切法放在带宽最高的那一层**（NVLink \> PCIe \> RDMA 网络 \> 跨地域）。TP 放机内、PP/DP 跨机，就是这么来的。"

**GB200 / NVL72 这类"整机架 NVLink 域"是趋势**（更大的机内域意味着可以抬高 TP 度），具体规格请以厂商资料为准。

## A4. GPU 里的显存都去哪儿了？（训练显存账，必考）

**全参数微调（混合精度 + Adam）的经验公式**：

```
每参数显存（bytes）≈
    2   (bf16/fp16 权重)
  + 2   (梯度，通常同 dtype)
  + 4   (fp32 主权重 master weight)
  + 4   (Adam 一阶动量 m)
  + 4   (Adam 二阶动量 v)
= 16 bytes / parameter
（另有激活值 activation、临时 workspace、NCCL buffer、碎片）

⇒ 7B 模型约 112GB 起步（优化器状态部分），70B 约 1.1TB 量级 —— **这是估算，需用真实 profile 验证**
```

**推理显存**：

```
权重：bf16 ≈ 2 bytes/param      ⇒ 7B ≈ 14GB，70B ≈ 140GB
KV cache（每 token 每层）：2(K,V) × kv_heads × head_dim × dtype_bytes
       ⇒ 整序列： × layers × seq_len × batch
例（某 70B 级 GQA 模型，layers=80、kv_heads=8、head_dim=128、bf16）：
       每 token ≈ 2×8×128×2 = 4096 B/层 ⇒ 整序列 ≈ 80层 × 4KB ≈ 320KB/token
       ⇒ 4k 上下文单序列约 1.2~1.3GB
```

**隐形的占用（面试加分）**：

- **激活值/recompute**：不做 activation checkpointing 时随 batch×seq 线性暴涨 ⇒ 用重计算换显存（以算力换显存）；
- **cuDNN/cuBLAS autotune workspace**：启动时/形状变化时会申请额外显存；
- **碎片**：PyTorch caching allocator 会造成"明明还有空闲却 OOM" ⇒ `PYTORCH_CUDA_ALLOC_CONF=expandable_segments:True` 等策略；
- **NCCL buffer**：集合通信自身的 buffer 也要占显存。

## A5. 训练并行策略：DP / TP / PP / SP / EP 各自解决什么？

| 策略 | 切什么 | 通信类型 | 放哪一层 | 代价 |
| --- | --- | --- | --- | --- |
| **DP（数据并行/DDP）** | 切 batch，每卡一份完整参数 | **AllReduce 梯度** | 跨机 OK | 显存最浪费（每份都要优化器状态） |
| **ZeRO-1/2/3 / FSDP** | **把优化器状态/梯度/参数分片** | AllGather + ReduceScatter | 跨机 OK | 通信换显存 |
| **TP（张量并行）** | 切单个算子/权重矩阵 | **每层多次 AllReduce/ReduceScatter** | **强烈建议机内 NVLink** | 通信最密集，但显存最省 |
| **PP（流水并行）** | 切层（不同卡算不同层） | 点到点传 activation（小） | 跨机 OK | **流水线气泡（bubble）** ⇒ 靠 microbatch 摊销 |
| **SP / Ulysses / Ring Attention** | 切序列（长上下文） | AllToAll / Ring 通信 | 依赖拓扑 | 长上下文场景 |
| **EP（专家并行，MoE）** | 切专家 | **AllToAll**（最吃网络） | 依赖拓扑 | AllToAll 对网络最敏感 |

**必答的组合逻辑**：

> 业内常见组合是 **DP × TP × PP**（三维并行）+ activation checkpointing + ZeRO/FSDP。做法是：**TP 限制在单机 NVLink 域内**（通常 8），**PP 跨节点**，**DP 最外层**；这样把最贵的通信压在最快的链路上。

## A6. 常见的显存优化手段（为什么有时候 1 卡能跑，有时候不行）

| 手段 | 用"什么"换"什么" |
| --- | --- |
| **Activation checkpointing / recompute** | 算力 → 显存 |
| **ZeRO / FSDP 分片** | 通信 → 显存 |
| **混合精度 bf16/fp16 + fp32 master weight** | 数值稳定性风险 → 显存与速度 |
| **8-bit 优化器 / Adafactor** | 优化器精度 → 显存 |
| **Gradient accumulation** | 时间 → 显存（等效 batch 不变） |
| **LoRA / QLoRA（微调）** | 冻结主权重 → 显存大幅下降（训练侧可几十倍下降，取决于实现） |
| **Offload（优化器到 CPU/NVMe）** | PCIe/NVMe 带宽 → 显存（慢，慎用） |
| **FlashAttention** | 显存与访存 → 速度双赢（IO-aware 实现） |

## A7. 推理侧的核心概念：prefill / decode / batching

```
prefill：把整段 prompt 一次性算完 ⇒ **算力密集**，决定 TTFT（首 token 时间）
decode ：逐 token 生成，每步读 KV cache ⇒ **带宽密集**，决定 TPOT/ITL
```

| 概念 | 作用 |
| --- | --- |
| **Continuous batching** | 请求逐个加入批次，避免静态 batch 的空等（vLLM 的核心之一） |
| **PagedAttention / 分页 KV** | 把 KV cache 分页管理，**降低碎片、提升并发** |
| **Prefix caching** | 相同前缀（system prompt、长文档）复用 KV ⇒ 大幅降 first-token 成本 |
| **抢占/换出（preemption & swap/recompute）** | 显存不足时怎么处理在途请求 |
| **Chunked prefill / Prefill-Decode 分离** | 让两种不同特性的计算**分开部署/分开调度**（当下趋势） |
| **投机解码（speculative decoding）** | 小模型起草 + 大模型验证，降延迟（牺牲少量算力） |
| **量化（W8A8 / W4A16 / FP8 / AWQ / GPTQ）** | 省带宽省显存，换精度 |

**必答的关键设计**：**prefill 与 decode 的资源瓶颈不同**（算力 vs 带宽），因此大规模服务里会做 **分离部署（disaggregated）**，把两种阶段放在不同 GPU 池，并通过高速网络传 KV —— 这也是为什么"推理集群"的网络设计也开始重要。

## A8. CUDA 软件栈：driver / toolkit / runtime 都是啥？

```
应用 → CUDA Runtime API (libcudart) → CUDA Driver API (libcuda) → 内核态驱动模块 → GPU
其中：
  - nvidia-driver Container（容器化驱动）：把驱动打包成容器，节点启动时由 GPU Operator 装载
  - CUDA Runtime/库（cuBLAS、cuDNN、NCCL）：传统做法是**打进镜像**
  - Forward Compatibility：`compat` 包允许容器内 toolkit 版本 > 宿主 driver 版本（有条件限制，请以官方说明为准）
```

**必答的三条**：

1. **容器里不需要装驱动，只需要 userspace CUDA 库**（由 nvidia-container-toolkit 从宿主注入驱动节点与库）；
2. **驱动版本 ↔ CUDA toolkit 版本有兼容矩阵**，混用会报 `CUDA driver version is insufficient for CUDA runtime version` 之类；GPU Operator 的价值之一就是统一管理这套矩阵；
3. **NCCL 是集合通信库**，训练和推理的多卡/多机都依赖它，它直接决定"能不能跑起来、跑多快"（见 C 篇）。

## A9. GPU 是怎么被放进容器的？（运行时链路）

三条代际演进：

| 代际 | 机制 | 说明 |
| --- | --- | --- |
| 老 | **nvidia-docker2 / nvidia-container-runtime** hook | 依赖 docker，逐步退场 |
| 主流 | **nvidia-container-toolkit + libnvidia-container**，通过 **NRI / containerd **`runtime_class`** 的 prestart hook** | containerd/CRI-O 下把 `/dev/nvidia*`、驱动库、CLI 注入容器，受环境变量控制（如 `NVIDIA_VISIBLE_DEVICES`） |
| 未来 | **CDI（Container Device Interface）** | 标准化设备声明，运行时读 `/etc/cdi/*.json` 注入，**与 DRA 配合**是新一代路径 |

```bash
# CDI 规格文件示意（由 device plugin / GPU Operator 生成）
cat /etc/cdi/nvidia.yaml | head -40
```

**必答**：容器的 GPU 可见性由 `NVIDIA_VISIBLE_DEVICES`** / **`CUDA_VISIBLE_DEVICES` 控制；device plugin 通过 **CDI 或 hook** 把设备与驱动库塞进容器。面试能说出"CDI 是 DRA 之后的设备注入标准"会明显加分。

## A10. MIG 为什么比"时间片共享"更高级？

| 维度 | 时间片共享（time-slicing） | **MIG** |
| --- | --- | --- |
| 隔离层级 | 仅调度时间片，**显存与错误域共享** | **硬件级切分**（SM + L2 + 显存控制器都分开） |
| QoS 保证 | 无 | 有（算力与显存都隔离） |
| 显存 | **不隔离**（一个进程 OOM 会拖累同卡其它进程） | 每实例独立显存 |
| 灵活性 | 运行时即可用 | 需要设置 **MIG strategy（none/single/mixed）**，切换要排空节点 |
| 限制 | 无硬件要求 | **仅部分型号支持**（如 A100/H100 系列），且**实例几何组合有限制** |

```yaml
# GPU Operator / device plugin 的 MIG 配置示意（请以所用版本文档为准）
config:
  name: mig-config
  default: all-balanced
```

常见的应对策略：

- **none**：整卡直通（训练场景默认，MIG 会损失部分算力与带宽效率）；
- **single/mixed**：按统一的 MIG 几何切分（推理场景）。

**必答坑**：一个节点上的 MIG 几何**必须统一规划**，混配置会导致可分配资源碎片化、调度困难；切换 MIG 策略通常要**排空节点**（这是重操作）。

## A11. 时间片共享（time-slicing）什么时候用？有什么雷？

**适合**：开发/测试环境、小模型推理、利用率低但需要"能跑"的场景。

**雷**：

1. **无显存隔离**：一个 Pod 占满显存会让同卡 Pod 全部 OOM；
2. **上下文切换开销**：GPU 计算被切来切去，**延迟敏感推理会明显抖动**；
3. **K8s 侧无法知道"实际用了多少显存"** ⇒ 只能靠上层约束（requests 里写假的 gpu 数量）。

```yaml
# device plugin 的 time-slicing 配置示意
apiVersion: v1
kind: ConfigMap
metadata:
  name: nvidia-device-plugin-config
data:
  config.yaml: |
    version: v1
    sharing:
      timeSlicing:
        resources:
          - name: nvidia.com/gpu
            replicas: 4            # 建议初值：按业务并发与延迟要求实测调整
```

> 想做到"显存也隔离"，原生 K8s 做不到 —— 需要 **HAMi / k8s-vGPU 这类第三方方案**（通过 admission + 运行时拦截做显存配额），或者使用 **MIG**。

## A12. MPS 是什么？和 MIG/时间片有什么不同？

MPS（Multi-Process Service）让多个进程共享同一个 **CUDA context**，减少上下文开销、提高小 kernel 的利用率。

| 方案 | 隔离 | 适用场景 |
| --- | --- | --- |
| 时间片 | 差（易互相影响） | 轻量共享、测试 |
| **MPS** | 中（共享 context，一个进程崩溃可能影响同 context 其它进程） | 多小任务、推理并发 |
| **MIG** | 强（硬件级） | 需要 QoS 保证的场景 |

**实践中**：MPS + MIG 可以叠加使用（在 MIG 实例内再用 MPS），具体支持请以厂商文档为准。

## A13. GPU 健康：XID 错误与常见故障信号

`XID` 是 NVIDIA 驱动上报的错误码（具体码值的含义以厂商文档为准），常见需要关注的类别：

| 类别 | 典型表现 |
| --- | --- |
| 显存类（如 ECC / DBE / row remap 相关码） | 建议**排空节点并做 GPU reset**，反复出现需报修/换卡 |
| NVLink / 互联类错误 | 机内集合通信失败、训练 hang ⇒ 通常是硬件或固件问题 |
| GPU fallen off the bus（掉卡） | 节点上看不到该卡 ⇒ 通常**必须重启/reset** |
| 应用级非法访问/异常 | 多数是应用层 bug，但也可能是 ECC/固件 |

```bash
dmesg -T | grep -i -E "NVRM|Xid"
journalctl -k | grep -i xid
nvidia-smi -q | grep -i -A5 "ECC Errors"
nvidia-smi --query-gpu=index,serial,ecc.errors.uncorrected.aggregate.total --format=csv
nvidia-smi -q -d TEMPERATURE,POWER,UTILIZATION,MEMORY
```

**平台侧建议（必答）**：

1. 采集 **DCGM 指标 + XID 事件**，配置分级告警；
2. **自动排空**：检测到不可恢复错误 ⇒ cordon + drain + 触发维护流程；
3. **不要在"疑似坏卡"的节点上继续接收新任务**（避免浪费几个小时的训练时间）；
4. 硬件层还要关注：**温度/功耗墙（power throttle）、掉电、风扇、PCIe 降速（带宽从 x16 掉到 x8/x4）**。

## A14. 常见"看起来像软件问题，其实是硬件"的现象

| 现象 | 可能的硬件/固件原因 |
| --- | --- |
| 训练速度忽然集体变慢 | 某卡 **thermal/power throttling**、PCIe 链路降级 |
| NCCL hang / timeout | NVLink down、网卡丢包、 infiniband link flapping |
| 反复 OOM 但账算得通 | ECC 错误导致显存减少 / row remap |
| 同一批任务里个别 rank 慢 | 该节点存在 **straggler**（硬盘/网卡/GPU 之一有问题） |
| 偶发 `illegal memory access` | 有可能是硬件，也可能是应用层越界 ⇒ **先换节点复现**定位归属 |

# Part B · Kubernetes GPU 调度机制（16 题）

## B1. 一个 Pod 想用 GPU，K8s 这条链路上到底发生了什么？

```
① 节点侧：GPU Operator 装载驱动 / nvidia-container-toolkit / CDI 配置
② NVIDIA device plugin 通过 /var/lib/kubelet/device-plugins/kubelet.sock 注册
③ 它周期性 ListAndWatch ⇒ kubelet 更新 Node allocatable：nvidia.com/gpu: 8
④ 用户创建 Pod：resources.limits["nvidia.com/gpu"]: 1
⑤ kubelet: 调度器只是在 NodeResourcesFit 里做"数字减一"（它不懂 GPU 拓扑）
⑥ 节点上的 kubelet 通过 CRI 调 DevicePlugin.Allocate({"nvidia.com/gpu":["GPU-UUID-xxx"]})
⑦ device plugin 返回：要挂哪些设备节点、环境变量（如 NVIDIA_VISIBLE_DEVICES）
⑧ containerd + nvidia-container-toolkit（或 CDI）把 /dev/nvidia* + 驱动库注入容器
⑨ 容器里 nvidia-smi 就能看到这一张卡
```

**必答的两个"不理解的事"**：

1. **调度器不知道 GPU 的型号、拓扑、NVLink 关系**，只知道"还剩几张卡"；想要拓扑感知必须靠 **标签 + 亲和 + 自定义插件**；
2. **extended resource 只能是整数**，且 **requests 必须等于 limits（或只写 limits）** —— 写 requests=1/limits=2 会报错，也不支持"半张卡"（除非用 MIG 或第三方份额方案）。

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: train
spec:
  containers:
    - name: c
      image: pytorch/pytorch:latest
      command: ["python", "train.py"]
      resources:
        limits:
          nvidia.com/gpu: 1        # 只写 limits 即可（extended resource 惯例）
```

## B2. device plugin 的能力边界在哪？（为什么它不够用）

| 能力 | device plugin |
| --- | --- |
| 上报数量 | ✅ |
| 选择具体设备 | ❌（kubelet 按顺序取，不过有 `GetPreferredAllocation` 接口可给出建议） |
| 表达"型号/显存/拓扑/带宽" | ❌（只能编码成资源名或依赖 NFD 标签） |
| 表达"可共享/带属性" | ❌ |
| 让**调度器**参与选择 | ❌（只有 kubelet 侧 Decide） |

**结论**：device plugin 只是"节点侧的设备登记与分配"，**它不是调度系统**；大模型场景需要的"按拓扑、按份额、按型号分配"，它表达不了 ⇒ 催生了 **DRA**。

## B3. DRA vs device plugin：新一代设备分配的差异？（必考）

| 维度 | device plugin | **DRA（Dynamic Resource Allocation）** |
| --- | --- | --- |
| 资源模型 | 整数计数（nvidia.com/gpu: 1） | **结构化参数**：ResourceClaim + DeviceClass + ResourceSlice（带属性选择器） |
| 谁选择设备 | kubelet 侧 | **调度器参与**（structured parameters 路径） |
| 表达"我要 40G 显存且 NVLink 互联的 2 张卡" | 做不到 | **可以**（用选择器表达） |
| 共享同一设备给多个 Pod | 做不到 | 支持 shareable / 多次分配 |
| 网络附加设备 | 不适合 | 原生支持（network-attached devices） |
| 成熟度 | 稳定通用 | **仍在演进**（各版本 API 有差异，请以所用版本 Release Notes 为准） |

```yaml
# DRA 结构化资源（示意，请以所用版本的 API 为准）
apiVersion: resource.k8s.io/v1
kind: DeviceClass
metadata:
  name: gpu-h100
spec:
  selectors:
    - cel:
        expression: 'device.driver == "gpu.nvidia.com" && device.attributes["gpu.nvidia.com"].model == "H100"'
---
apiVersion: resource.k8s.io/v1
kind: ResourceClaim
metadata:
  name: two-linked-gpus
spec:
  devices:
    requests:
      - name: gpus
        deviceClassName: gpu-h100
        count: 2
---
apiVersion: v1
kind: Pod
spec:
  resourceClaims:
    - name: gpu
      resourceClaimName: two-linked-gpus
  containers:
    - name: c
      image: pytorch/pytorch:latest
      resources:
        claims: [{name: gpu}]
```

> **面试口径**：知道 DRA 在解决"**表达力 + 调度介入**"就够了；不要夸大到"生产已全面替代 device plugin"，当前大模型集群主流仍是 device plugin + 拓扑标签/NFD。

## B4. NFD 与 GPU Feature Discovery：拓扑标签从哪来？

- **NFD（Node Feature Discovery）**：发现 CPU/内核/PCI 设备等特征打标签；
- **GPU Feature Discovery (GFD)**：专门发现 GPU 型号、显存、驱动/CUDA 版本、MIG 配置等，写标签（label 前缀请以所用版本为准）；
- 标签通常会被分类 many 个 label: e.g. `nvidia.com/gpu.product=NVIDIA-H100-80GB-HBM3`、`nvidia.com/gpu.count=8`、`nvidia.com/gpu.memory=81920`、`nvidia.com/gpu.driver-version=...`。

**用途**：让 Pod 能用 `nodeSelector/nodeAffinity` 选机型、避免把 H100 的任务调度到 A100 上。

```yaml
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
          - matchExpressions:
              - key: nvidia.com/gpu.product
                operator: In
                values: ["NVIDIA-H100-80GB-HBM3"]
```

**坑（必答）**：GFD 只是"打标签"，**不知道 NVLink 拓扑的详细结构**；真正的"挑一组同 NVLink 域的卡"需要 topology manager / vendor 的拓扑感知插件 / 或者 job-level 自定义调度。

## B5. 为什么大模型训练必须要 Gang Scheduling？（死锁题）

**经典死锁场景**：

```
集群剩 8 张卡分布在 2 台机器上（每台 4 张）
有两个 8 卡任务 A、B 先后启动：
   A 占了 4 张（machine1），B 占了 4 张（machine2）
   A 等剩下 4 张，B 也等剩下 4 张
   ⇒ 双方都持有部分资源互相等待 ⇒ 死锁，且资源被空占（GPU 不释放）
```

**解法：All-or-Nothing（Gang Scheduling）** —— 只有凑齐全部 minMember 才真正绑定，否则**谁都不给**。

实现方式（放在调度框架的哪个扩展点，是与《Scheduler 篇》的呼应）：

| 方案 | 机制 |
| --- | --- |
| **scheduler-plugins Coscheduling** | **Permit** 扩展点：持有已通过 Filter/Score 的 Pod 等待配额；`PostFilter` 超时整组放弃；`QueueSort` 按 PodGroup 创建时间排序防饿死 |
| **Volcano** | PodGroup CRD + 专属调度器 `volcano`，支持 Gang、公平份额、抢占、网络拓扑感知 |
| **YuniKorn** | 面向大数据/批处理的层级队列与公平性 |
| **Kueue** | 更偏**准入/配额队列**（见 B8），常与上面三者搭配 |

```yaml
# Volcano PodGroup + Job 示意
apiVersion: scheduling.volcano.sh/v1beta1
kind: PodGroup
metadata:
  name: training-job
spec:
  minMember: 16
  minResources:
    nvidia.com/gpu: "16"
  queue: default
---
apiVersion: batch.volcano.sh/v1alpha1
kind: Job
metadata:
  name: pytorch-job
spec:
  schedulerName: volcano
  minAvailable: 16
  tasks:
    - name: worker
      replicas: 8
      template:
        spec:
          containers:
            - name: pytorch
              image: pytorch/pytorch:latest
              resources:
                limits: {nvidia.com/gpu: 2}
```

**必答的副作用（Permit 阶段的含义）**：

> Pod 在 **Permit 等待期已经 assume 了资源**（《Scheduler 篇》Q2 的乐观绑定），所以"等待"是真的占着资源；因此必须配**超时**与**上限**，否则会变成资源泄漏。

## B6. Training Operator / PyTorchJob 到底提供了什么？

| 组件 | 作用 |
| --- | --- |
| **Kubeflow Training Operator**（CRD：`PyTorchJob`、`TFJob`、`MPIJob`…） | 描述"训练作业"这个高层语义，自动创建 master/worker Pod，注入 `RANK/WORLD_SIZE/MASTER_ADDR/MASTER_PORT` 之类的 rendezvous 环境变量 |
| **MPI Operator** | MPI 语义（如 Horovod 场景） |
| **torchrun / torchelastic** | 容错与弹性 rendezvous（worker 挂了可重启/缩容） |
| **LeaderWorkerSet（LWS）** | **多节点的 leader/worker 组**（常见用于**多机推理**与大模型的 TP 组），保证组整体滚动更新与调度一致性 |

```yaml
# LWS 示意（API 版本请以所用集群为准）
apiVersion: leaderworkerset.x-k8s.io/v1
kind: LeaderWorkerSet
metadata:
  name: vllm
spec:
  replicas: 4                 # 4 个"组"
  leaderWorkerTemplate:
    size: 2                   # 每组 1 leader + 1 worker（如 TP=2）
    restartPolicy: RecreateGroupOnPodRestart
    leaderTemplate:
      spec:
        containers:
          - name: vllm
            image: vllm/vllm-openai:latest
            resources:
              limits: {nvidia.com/gpu: 1}
```

**必答**：LWS 解决的是"**一组 Pod 必须整体创建/整体更新/整体调度**"的问题，是多机推理（TP 跨机）编排的关键组件。

## B7. 拓扑感知调度：怎么把通信压在最快的链路上？

调度能做的层次（从粗到细）：

| 层次 | 手段 |
| --- | --- |
| **节点层级**（最常用） | `topologySpreadConstraints` 让同一作业的 Pod 尽量**集中在少数节点**（减少跨机）；或反向打散提高容灾 |
| **机型/域层级** | nodeAffinity 限定同一机型、同一 rack/zone 标签 |
| **NUMA/PCIe 层级** | kubelet `topologyManagerPolicy: single-numa-node`，让 GPU 与 CPU/NIC 在同一 NUMA（避免跨 socket PCIe 访问） |
| **NVLink/交换机层级** | 依赖厂商/社区的拓扑感知插件或 job-level 自定义分配（部分由 Volcano / 厂商 operator 提供） |

```yaml
spec:
  topologySpreadConstraints:
    - maxSkew: 1
      topologyKey: kubernetes.io/hostname
      whenUnsatisfiable: DoNotSchedule
      labelSelector:
        matchLabels: {job-name: train-job-123}
```

**实践建议（需实测校准）**：

- 训练作业：**倾向聚拢**（maxSkew 小 + 尽量少的主机数）⇒ 更少的跨机通信；
- 推理服务：**倾向打散**（跨可用区/机架）⇒ 更好的可用性。

## B8. Kueue：为什么"队列/配额"在大模型集群里是必需品？

GPU 是稀缺且昂贵的共享资源，必须有"谁能用多少、谁先谁后"的规则。Kueue 的核心对象：

| 对象 | 作用 |
| --- | --- |
| **ResourceFlavor** | 描述一类资源（如 h100-80g、a100-40g、spot 卡池），可带 node label/taint 匹配 |
| **ClusterQueue** | 集群级配额与准入（quota、borrowing/lending、抢占策略） |
| **LocalQueue** | 命名空间级入口，业务只看到它 |
| **AdmissionCheck** | 额外的准入检查（如等待外部 Provisioning 完成） |
| **Cohort** | 一组 ClusterQueue 之间互相借用配额 |

```yaml
apiVersion: kueue.x-k8s.io/v1beta1
kind: ResourceFlavor
metadata:
  name: h100-80g
spec:
  nodeLabels:
    nvidia.com/gpu.product: NVIDIA-H100-80GB-HBM3
---
apiVersion: kueue.x-k8s.io/v1beta1
kind: ClusterQueue
metadata:
  name: research-cq
spec:
  namespaceSelector: {}
  resourceGroups:
    - coveredResources: ["cpu", "memory", "nvidia.com/gpu"]
      flavors:
        - name: h100-80g
          resources:
            - name: "nvidia.com/gpu"
              nominalQuota: 64          # 建议按采购容量分配，需结合业务排期校准
---
apiVersion: kueue.x-k8s.io/v1beta1
kind: LocalQueue
metadata:
  name: research-lq
  namespace: research
spec:
  clusterQueue: research-cq
```

```yaml
# 作业侧声明队列（Kueue 会先"准入"再真正调度，避免进入 scheduler 后白白 Pending）
metadata:
  labels:
    kueue.x-k8s.io/queue-name: research-lq
```

**必答的价值**：

1. **准入层排队** ⇒ 避免成千上万个不可调度的 Pod 堆积在调度队列里拖垮调度器（《Scheduler 篇》L3/L4 的治理）；
2. **配额与借用** ⇒ 多团队共享 GPU 池但保证底线；
3. **队列内优先级/抢占** ⇒ 让紧急任务可以先拿到资源。

## B9. GPU 资源超卖 / 共享的正确姿势

从弱到强：

| 方案 | 隔离能力 | 适用 |
| --- | --- | --- |
| 时间片共享（device plugin replicas） | 差 | 开发/轻量推理 |
| **HAMi / k8s-vGPU 类**（webhook + 运行时拦截显存/算力） | **中**（软限制，超额仍可能 OOM） | 多租户推理、教学平台 |
| **MIG** | **强**（硬件） | 有 QoS 要求的推理场景 |
| DRA 的份额模型 | 演进中 | 未来方向 |

```yaml
# HAMi 风格的资源写法示意（请以所用项目文档为准）
resources:
  limits:
    nvidia.com/gpu: 1
    nvidia.com/gpumem: 8192        # 显存 MiB
    nvidia.com/gpucores: 30        # 算力百分比
```

**必答风险**：任何"软限制"方案都可能因为**上层声明与实际不符**导致同卡互相影响 ⇒ 必须配：显存使用监控 + OOM 关联到具体 Pod + 明确的赔偿/调度回退策略。

## B10. GPU 节点的 bin-packing 与碎片

**矛盾**：

- 训练任务需要**整块**资源（8 卡甚至跨机）⇒ 需要**大块连续供给**；
- 推理/开发任务常常只要 1 卡 ⇒ 容易把 8 卡机切成碎片，导致训练排不上。

**治理（高分答法）**：

1. **节点池分层**：训练池（整卡、整机型）/ 推理池（可共享、可 MIG）/ 开发池（低优先级、可抢占）；
2. **打分策略分流**：训练池用 `MostAllocated`（尽快填满整机）或相反地留出整块，推理池按需；两种策略的目标不同，**必须分开 profile**；
3. **配额守底线**：用 Kueue/命名空间 ResourceQuota 保证训练不被推理吃光；
4. **MIG 只在专用池开**（MIG 一旦开启，该机就不能给训练用了）；
5. 监控"**分配率 vs 使用率**"双指标 —— 分配率高但使用率低 = 占着不用，这是最大的浪费源。

```promql
# 分配率（Requests ≠ 实际使用），配合 DCGM 的使用率对比
sum(kube_pod_container_resource_requests{resource=~"nvidia.com/gpu"}) by (node)
# 实际 GPU 利用率（需 dcgm-exporter）
DCGM_FI_PROF_GR_ENGINE_ACTIVE
```

## B11. GPU 节点的自动扩缩（CA / Karpenter）

**难点**：GPU 节点**扩容慢**（实例创建 + 驱动初始化 + GPU Operator 就绪往往需要分钟级），而且**昂贵**。

实践要点：

- GPU 池**单独配置 ASG/MachineDeployment**，并用 taint 隔离，避免无 GPU 的 Pod 落到上面；
- 缩容要考虑**长训练任务**：必须 cordon + 等待任务结束，而不是硬性驱逐 ⇒ 通常需要上层作业系统配合（这也是很多团队选择 "常驻 GPU 池 + 队列" 而非按需伸缩的原因）；
- 推理侧更适合自动扩缩（无状态、可快速重建），但要解决**权重加载慢**问题（见 E 篇）。

## B12. GPU 相关的 kubelet/system 层设置（与两篇联动）

| 项 | 建议 |
| --- | --- |
| `topologyManagerPolicy` | 若要 NUMA/PCIe 亲和，考虑 `single-numa-node` / `restricted`（可用性成本要压测） |
| `cpuManagerPolicy: static` | 绑核场景才开，且必须配 `reservedSystemCPUs` |
| `reservedSystemCPUs` | 给 GPU 节点的系统进程留 CPU，**尤其重要**（NCCL 通信需要 CPU 参与） |
| `memoryManagerPolicy` | 与 NUMA 亲和配合，视版本支持使用 |
| cgroup v2 / swap | GPU 节点通常**不建议依赖 swap**（Host-Device pinning 与延迟敏感） |
| Hugepages / hosts | RDMA/DPDK 类需要大页时才有必要配置 |

> 参考《Kubelet 篇》L10 的 NUMA/绑核陷阱，GPU 节点是这些坑最集中的地方。

## B13. GPU Operator 到底管了什么？（部署必答）

一份典型的 GPU Operator 会包含这些组件（具体组件名请以所用版本为准）：

| 组件 | 作用 |
| --- | --- |
| **Driver DaemonSet（容器化驱动）** | 在节点上装载内核模块 |
| **Container Toolkit / CDI 配置** | 让容器能真正使用 GPU |
| **Device Plugin DaemonSet** | 向 kubelet 上报 GPU |
| **DCGM Exporter** | 暴露 GPU 指标 |
| **GPU Feature Discovery** | 打 GPU 相关标签 |
| **MIG Manager** | 统一管理 MIG 几何配置 |
| **Node Status Exporter / 健康检查相关组件** | 把 GPU 健康暴露到 Node condition |
| **验证用的 validator Pod** | 部署后自检能否真跑 GPU 负载 |

**面试提示**：把 GPU Operator 理解成"**GPU 节点的标准化交付方式**"，它把驱动/工具链/设备发现/监控一次性收敛，避免手工装驱动导致的版本漂移（漂移是 GPU 集群第一大事故源）。

## B14. GPU × 安全隔离：多租户能不能安全共享一张 GPU？

| 风险 | 说明 |
| --- | --- |
| 显存数据残留 | 容器退出后显存内容不一定清零 ⇒ 结合隔离方案与可信域管理 |
| 错误传播 | time-slicing 下一个进程异常可能影响同卡其它进程 |
| 设备节点过大授权 | `privileged` 或粗暴开放 `/dev/nvidia*` 会让容器能力过大 |
| 多租户是否伦理 | 建议：**一张物理卡在同一时间只服务同一租户**（或仅服务可信内部租户）；真正的多租户 LLM 服务应在**服务层**（推理框架）做隔离与配额，而不是共享 CUDA 进程 |

## B15. 怎么判断"GPU 真的在被有效使用"？（利用率误区必考）

`nvidia-smi` 的 "GPU-Util 100%" 极具误导性：它只表示**有 kernel 在跑**，不代表 Tensor Core 在高效算有用的东西。更靠谱的是：

| 指标 | 含义 |
| --- | --- |
| `DCGM_FI_PROF_GR_ENGINE_ACTIVE` | Graphics/Compute engine active（比 smi 更细的分帧统计） |
| `DCGM_FI_PROF_SM_ACTIVE` / `SM_OCCUPANCY` | SM 活跃比例与占用率 |
| `DCGM_FI_PROF_PIPE_TENSOR_ACTIVE` | **Tensor Core 是否活跃**（LLM 训练关注它） |
| `DCGM_FI_DEV_FB_USED` / `_FB_FREE` | 显存使用 |
| `DCGM_FI_DEV_GPU_TEMP` / `POWER_USAGE` / `SM_CLOCK` / `MEM_CLOCK` | 是否被限温/限功率/降频 |
| `DCGM_FI_DEV_XID_ERRORS` | XID 错误事件 |
| `DCGM_FI_DEV_ECC_*` | ECC 错误计数 |

```promql
# 全集群 GPU 平均算力引擎活跃度（有效利用率的粗口径）
avg(DCGM_FI_PROF_GR_ENGINE_ACTIVE)
# 有分配但几乎不用的"僵尸卡"（占着 GPU 不干活 —— 最贵的浪费）
DCGM_FI_PROF_SM_ACTIVE < 5
# 降频/限功率检查（数值需以该型号的标称时钟为基准，并实测校准）
DCGM_FI_DEV_SM_CLOCK < 1300
DCGM_FI_DEV_POWER_USAGE / DCGM_FI_DEV_POWER_MGMT_LIMIT > 0.98
```

**结论口径**：

> "GPU 利用率要看三层：**① 有没有被分配（调度/配额）② 有没有 kernel 在跑（SM/Tensor active）③ 跑得是否有效（MFU）**。绝大多数集群的浪费发生在第 3 层：SM 很忙，但 MFU 只有 30%。"

## B16. MFU 是什么？怎么算？（高阶必考题）

```
MFU = 实际达成的有效算力 / 峰值算力
    = (每步的有效 FLOPs × 每秒步数) / (GPU 数 × 单卡峰值 FLOPS)

其中每步有效 FLOPs（Transformer）常见估算：
    ≈ 6 × 参数量 × tokens      （前向+反向的近似：前向 2N，反向 4N）
   若使用 activation checkpointing 则会额外增加一次前向（约 8N）
```

**估算示例（所有数字均为"假设值"，用于演示口径；真实值必须用 Profiler 实测）**：

```
假设：70B 参数、64 张卡、全局 batch = 262144 tokens/步、单步耗时 8.7s
  有效 FLOPs/步 ≈ 6 × 70e9 × 262144 ≈ 1.1e17 FLOPs
  达成算力     ≈ 1.1e17 / 8.7s ≈ 1.27e16 FLOPS ≈ 12.7 PFLOPS
  峰值算力     ≈ 64 × 989 TFLOPS（BF16 峰值量级，待以 Datasheet 复核）≈ 63.3 PFLOPS
  ⇒ MFU ≈ 12.7 / 63.3 ≈ 20%
```

**这个公式最容易踩的三个坑（一定要在面试里主动说明）**：

1. **tokens 的口径**：要用"每步的全局 tokens"，并且注意 **TP/PP 会让单卡重复/分担计算**，不能直接乘 GPU 数；
2. **是否含重计算**：开了 activation checkpointing 时系数从 \~6N 变成 \~8N；
3. **单位的幂次**：PFLOPS/TFLOPS 差 1000 倍，是这类题最常见的算术事故。

> 业界公开分享里，良好优化的 LLM 训练 MFU 常在 **30%\~****50% 区间**（具体务必以自家 workload 的 Profiler 数据为准）。**MFU 低于 30% 就该立项优化**，那是几百万上千万的算力差。

**提升 MFU 的常见抓手**：

1. 增大 batch（提升 GEMM 效率）⇒ 但要配合 LR schedule；
2. **算子融合 + FlashAttention**；
3. **减少重计算 vs 显存的权衡**；
4. **通信/计算 overlap**（ZeRO/FSDP 的 prefetch、NCCL 调优）；
5. **降低 bubble**（PP microbatch 数足够多）；
6. 输入 shape 尽量规整（避免 padding 浪费与 kernel autotune 抖动）。

# Part C · 大模型训练网络（12 题）

> 训练集群里"网络不是配角，而是算力的一部分"。这一篇与《网络 / CNI 篇》交叉，但关注的是 **RDMA / 集合通信 / 拓扑**。

## C1. 为什么大模型训练对网络的要求和 Web 服务完全不同？

| 维度 | Web/微服务 | **大模型训练** |
| --- | --- | --- |
| 流量模型 | 南北向为主、请求多而小 | **东西向、极大规模、同步（集合通信）** |
| 延迟敏感性 | 单请求 P99 | **同步阻塞**：一次 AllReduce 慢 ⇒ 所有 rank 一起等 |
| 丢包容忍 | 靠重传，问题不大 | **几乎不允许丢包**（RDMA 对丢包极敏感） |
| 带宽量级 | 10\~100G 常见 | 单节点 200\~400G 起步，多轨聚合更高 |
| 关键指标 | QPS/延迟 | **有效带宽、通信占比、MFU** |

**必答的"同步放大效应"**：

> 分布式训练是 **BSP（批量同步并行）** 式的：一次集合通信没完成，所有 GPU 都在空转。所以"网络慢 20%" 往往直接表现为"整体算力利用率掉一大截"，而不是某个请求慢。

## C2. 集合通信量怎么估算？（会算这个非常加分）

**Ring AllReduce 的通信量公式**：

```
每个 GPU 要发送的数据量 ≈ 2 × (N-1)/N × D     （N = 参与的 GPU 数，D = 每个 GPU 持有的数据字节数）
≈ 2D（当 N 较大时）
```

**举例（估算，需实测验证）**：

```
70B 模型，bf16，DDP（不分片）：
  单卡需同步的梯度 D ≈ 70B × 2 bytes = 140GB
  每步每卡通信量 ≈ 2 × 140GB = 280GB
  若跨机有效带宽 100 Gb/s ≈ 12.5 GB/s ⇒ 仅同步就需 ≈ 22s/步
  ⇒ 这就是为什么"朴素 DDP 跑大模型"根本不现实，
     必须用 ZeRO/FSDP 分片 + 把通信量大的切法压在 NVLink 上
```

**结论**：**并行策略的选择本质上是在"重排通信量分布"**，把大流量放在高带宽层（NVLink），把小流量放在低带宽层（网络）。

## C3. NCCL 是什么？训练作业里最常调的环境变量有哪些？

NCCL 是 NVIDIA 的集合通信库，负责 AllReduce/AllGather/ReduceScatter/AllToAll 的高效实现（自动选择 ring/tree 等算法与路径：NVLink、PCIe、共享内存、IB/RoCE socket）。

**排障与调优常用环境变量（请以所用 NCCL 版本文档为准）**：

| 变量 | 用途 |
| --- | --- |
| `NCCL_DEBUG=INFO`（或 `TRACE`） | **排障第一件事**，看它选了什么路径/算法 |
| `NCCL_DEBUG_SUBSYS=INIT,ENV,GRAPH,TUNING` | 缩小日志范围 |
| `NCCL_SOCKET_IFNAME=eth0` / `NCCL_SOCKET_IFNAME=^docker,lo` | 指定/排除用于控制面通信的接口（**选错接口是 hang 的头号原因**） |
| `NCCL_IB_HCA=mlx5_0,mlx5_1` | 指定用哪些 RDMA 网卡 |
| `NCCL_IB_GID_INDEX=3` | RoCEv2 场景常需要指定 GID index |
| `NCCL_NET_GDR_LEVEL` / `NCCL_P2P_LEVEL` | 控制 GPUDirect RDMA / P2P 使用层级 |
| `NCCL_ALGO=Ring / Tree` | 强制算法（调参时用） |
| `NCCL_TOPO_FILE` | 提供自定义拓扑描述（特殊机型） |
| `NCCL_ASYNC_ERROR_HANDLING` | 错误处理行为（版本相关） |
| `NCCL_IB_TIMEOUT` / `NCCL_IB_RETRY_CNT` | RoCE 网络的超时重传参数 |

**典型 hang 场景**：多网卡机器（eth0 管理网 + 多个 RDMA 网卡 + CNI 的 veth）时，NCCL 可能选到错误的接口 ⇒ 表现为"卡在 init 或第一步不动"。

```bash
# 训练容器里第一件事
NCCL_DEBUG=INFO NCCL_DEBUG_SUBSYS=INIT,ENV,GRAPH python train.py 2>&1 | grep -i nccl | head -60
```

## C4. 训练网络的两条技术路线：IB vs RoCE

| 维度 | **InfiniBand** | **RoCEv2（无损以太网）** |
| --- | --- | --- |
| 生态 | 专网、专用交换机与 HCA，子网管理器（SM） | 以太网交换机 + RDMA 网卡 |
| 无损 | 原生支持 | **依赖 PFC + ECN + DCQCN 的精细调参** |
| 运维 | 相对"开箱即用"但需专业技能 | 成本低、可复用以太网团队，但**调参不慎会出现 PFC 风暴/死锁** |
| 规模 | 大规模成熟 | 大规模可行，但对交换机配置要求高 |

**必答的 RoCE 风险**：

- **PFC 风暴 / 死锁**：优先级流控扩散到整网 ⇒ 必须合理规划 PFC 作用范围（通常只在"服务器到 leaf" 段）、配置无损队列与缓冲区；
- **ECN/DCQCN 参数**需要与交换机侧协同；
- **MTU**：常用 9000（jumbo）降低 PPS，但**必须整条链路一致**；
- 与《网络篇》A11 呼应：**隧道封装会让 MTU 变小，而 RDMA 流量通常走**主机接口/ secondary network\*\*，不要让它进隧道\*\*。

## C5. 训练集群的典型拓扑：rail-optimized 是什么？

```
常见 8 卡服务器 + 8 张 RDMA 网卡，每台服务器的第 i 张网卡（对应第 i 张 GPU）
连到同一台 leaf 交换机的同一"轨道"（rail i）：

  rail0: leaf-A  ── node1:gpu0/nic0 ── node2:gpu0/nic0 ── ...
  rail1: leaf-B  ── node1:gpu1/nic1 ── node2:gpu1/nic1 ── ...
  ...
```

**好处**：同号 GPU（做同样切分的那些 rank）之间的通信大多**只在同一 leaf 内一跳完成**，减少跨 leaf 流量与拥塞。

**面试要点**：

- 这种拓扑决定了作业的 **rank 到节点的映射策略**（把"通信最密集的 rank 对"放在同一 rail/同一 leaf）；
- 这也是"拓扑感知调度"在训练集群里的真实含义之一；
- 另一类常见拓扑是 **fat-tree/spine-leaf**（通用性好，但同号 GPU 不一定同 leaf）。

## C6. K8s 里怎么给 Pod 提供 RDMA 网络？（与 CNI 篇联动）

| 方案 | 做法 | 适用 |
| --- | --- | --- |
| **hostNetwork** | 训练 Pod 直接用宿主网络 | 最常见、性能最好、最简单（代价：端口冲突、网络策略失效） |
| **Multus + macvlan/ipvlan + SR-IOV VF** | 给 Pod 额外一张"直通"的高性能网卡 | 需要网络隔离/多租户/不能 hostNetwork 的场景 |
| **NVIDIA Network Operator** | 统一管理 MOFED/DOCA 驱动、RDMA device plugin、secondary network CRD（如 MacvlanNetwork / IPoIBNetwork / HostDeviceNetwork） | 生产标准化的做法 |
| RDMA device plugin | 上报 `rdma/hca_shared_devices` 等资源 | 配合上述方案 |

```yaml
# Multus + macvlan 给训练 Pod 加一张高速网卡（示意，请以所用版本为准）
apiVersion: k8s.cni.cncf.io/v1
kind: NetworkAttachmentDefinition
metadata:
  name: rdma-net
  namespace: default
spec:
  config: |
    {
      "cniVersion": "1.0.0",
      "type": "macvlan",
      "master": "ens5f0",
      "mode": "bridge",
      "ipam": { "type": "whereabouts", "range": "192.168.100.0/24", "exclude": ["192.168.100.1/32"] }
    }
---
# 引用
metadata:
  annotations:
    k8s.v1.cni.cncf.io/networks: rdma-net
```

**必答的坑**：

1. **IPAM 与 IP 规划**：whereabouts 需要真实网段规划，**IP 耗尽会让训练 Pod 起不来**（与《网络篇》L7 同源）；
2. **接口选择**：NCCL 必须选到这张高速网卡 ⇒ `NCCL_SOCKET_IFNAME` / `NCCL_IB_HCA` 要显式配；
3. **不要让它走隧道**：RDMA 流量不应经过 VXLAN/Geneve 封装（MTU 与性能都受损）；
4. **SR-IOV VF 与节点绑定**：VF 是硬件资源，迁移/重建会受限。

## C7. GPUDirect RDMA（GDR）的价值与前提

**作用**：让网卡**直接读写 GPU 显存**，绕过"显存→主机内存→网卡"的多次拷贝 ⇒ 降低延迟、减少 CPU 占用、提升有效带宽。

**前提**：

- 驱动栈支持（如 nvidia-peermem / OFED 的 peer memory 相关模块，具体组件名以版本为准）；
- GPU 与 NIC 在**同一 PCIe 拓扑的合适位置**（通常要求同一 PCIe switch 或至少同一 NUMA，取决于平台）；
- 这也是为什么 `topologyManagerPolicy: single-numa-node` 在训练节点常被讨论（**是否必要取决于机型拓扑，需实测**）。

**排障**：`NCCL_DEBUG=INFO` 日志里可以看到是否用了 GDR 路径；若未启用通常提示 `NET/IB : GPU Direct RDMA Disabled` 之类（具体文案以版本为准）。

## C8. 训练网络故障的现场定位顺序

```
① NCCL_DEBUG=INFO：看它选了哪些接口、是否用了 IB/GDR、报了什么错
② 确认节点 RDMA 设备存在：ibstat / ibv_devinfo / rdma link
③ 交换机侧：端口 up？速率/ MTU 是否一致？丢包/CRC 计数？PFC 暂停帧统计？
④ 端到端带宽实测：ib_write_bw / ib_send_bw（或 perftest 系列） ⇒ 与理论值对比
⑤ 单节点多卡（不走网络）是否正常 ⇒ 区分"机内问题" vs "网络问题"
⑥ NCCL 自带的测试：nccl-tests（all_reduce_perf）⇒ 直接给出 busbw
⑦ 抓包/计数器：ethtool -S、perfquery（IB）
```

```bash
# 常用基线测试（示意）
ibstat
ib_write_bw -d mlx5_0 -F --report_gbits -q 4 <server-ip>
all_reduce_perf -b 8M -e 8G -f 2 -g 1     # nccl-tests
```

**必答**：**nccl-tests 的 **`busbw`** 是训练网络的"黄金指标"**——上线新集群、改驱动、换交换机后都要跑一遍基线并归档（这属于"变更前后对比"的标准动作）。

## C9. 大规模训练网络的"隐形杀手"

| 现象 | 根因 |
| --- | --- |
| 训练速度周期性抖动 | 网络拥塞 + 同步等待；或 checkpoint 写盘抢占带宽 |
| 只有部分 rank 慢（straggler） | 某节点网卡/链路降级、PCIe 降速、GPU 降频 |
| 大规模（\>256 卡）时 MFU 明显下降 | 跨 leaf 流量增多、拥塞、PP bubble |
| 偶发 NCCL timeout | 网络抖动/丢包 + 超时参数偏小 |
| RoCE 集群整体变慢 | PFC/ECN 参数不当、缓冲区/队列配置问题 |

**治理思路**：

- 建立 **基线测试 + 定期复测**（健康巡检的一部分）；
- 训练作业与数据 IO / checkpoint **错峰**或分流到不同网络平面；
- 对 straggler 做**自动识别与隔离**（见 F 篇）；
- 关键作业保留"**最小通信域**"的排布策略（尽量同 leaf/rail）。

## C10. 推理侧网络：为什么"K8s Service + gRPC"会负载不均？（超经典）

**问题**：LLM 推理是长连接 + HTTP/2 多路复用。一个客户端到 Service 的**一条 TCP 连接**上跑很多 stream ⇒ 这条连接被分配到某个 Pod 后，**后续请求都落在同一个 Pod**，其它 Pod 闲着。

**解法（按推荐度）**：

| 方案 | 说明 |
| --- | --- |
| **限速单连接的并发 stream** | 让服务端发送 `SETTINGS_MAX_CONCURRENT_STREAMS=1` 之类配置 ⇒ 客户端被迫开新连接 ⇒ 重新负载均衡（**实践中最有效**） |
| **客户端/网关侧按请求 LB** | 用支持 per-request 负载均衡的网关（Envoy/istio 的 `LEAST_REQUEST`），或在客户端做端点选择 |
| **Headless Service + 客户端 LB** | 应用自己解析 Endpoints 并建多条连接 |
| **合理的连接生命周期** | 设置 `maxAge` 让连接周期性重建（辅助手段） |
| **Inference Gateway（Gateway API Inference Extension）** | 面向 LLM 的网关：感知模型/队列/缓存，做**有状态的端点选择**（见 E 篇） |

## C11. 推理服务的网络还要考虑什么？

| 项 | 说明 |
| --- | --- |
| **长超时** | 生成式请求动辄数十秒\~数分钟，网关/Ingress 的 `proxy_read_timeout`/`timeout` 必须放大 |
| **流式响应（SSE）** | 反向代理必须**关闭响应缓冲**（如 `proxy_buffering off`），否则 token 变得不连续 |
| **请求排队** | 显存决定并发上限 ⇒ 需要在网关层做队列与优先级（而不是无限堆积） |
| **模型预热与权重加载** | 冷启动几十秒到几分钟 ⇒ 需要预热池与就绪判定（见 E 篇） |
| **跨可用区** | 推理多副本应跨区打散，但如果做了 Prefill/Decode 分离，跨区会带来 KV 传输延迟 |

## C12. 网络侧的可观测与基线

```promql
# RDMA/网卡层（依赖 node-exporter 或厂商 exporter，指标名请核对）
rate(node_network_receive_bytes_total{device=~"ib.*|ens.*"}[5m])
rate(node_network_receive_drop_total[5m])
rate(node_network_receive_errs_total[5m])
# RDMA 特定计数（部分 exporter 提供）
rdma_rx_discards / rdma_out_of_sequence 之类（以实际 exporter 为准）
# GPU 侧"等待通信"的间接信号：SM 活跃度下降但任务在跑
DCGM_FI_PROF_SM_ACTIVE
```

```promql
# 网络基线告警（示例）
rate(node_network_receive_drop_total[5m]) > 0
rate(node_network_receive_errs_total[5m]) > 0
```

**必答**：**网络基线必须"用 nccl-tests 的 busbw 归档"**，而不是只看网卡利用率——因为"利用率为 0 但训练很慢"往往是 hang/同步等待，而不是带宽用满。

# Part D · 大模型场景的存储（10 题）

> 与《存储 / CSI 篇》交叉，但这里聚焦：**训练数据吞吐、checkpoint、模型分发**。

## D1. 训练对存储的要求：为什么"传统 PVC"常常不够？

| 阶段 | 需求特征 |
| --- | --- |
| **数据读取** | 海量小文件/打包大文件，**顺序吞吐**要求高；多 worker 并发读同一个数据集 ⇒ **元数据与聚合带宽**双压力 |
| **checkpoint 写入** | **TB 级、突发、要求可靠**；写不完会拖慢训练 |
| **模型分发** | 权重几十 GB\~数百 GB，需要**快速分发到大量节点** |
| **推理加载** | 冷启动时从共享存储/本地缓存加载权重 ⇒ **影响扩容速度** |

**核心矛盾**：**训练数据吞吐、checkpoint 带宽、模型分发速度**常常决定"GPU 有多少时间在等数据"。

## D2. 训练数据的常见组织方式与坑

| 组织方式 | 优点 | 坑 |
| --- | --- | --- |
| **大量小文件**（图片/文本分文件） | 直观 | **元数据操作爆炸**，POSIX 文件系统会被打爆 |
| **打包格式**（TFRecord / WebDataset / MDS / Parquet / Arrow） | 顺序读、元数据友好 | 需要额外预处理 |
| **对象存储直接读** | 容量无限 | **延迟高、不适合逐样本随机读**，需要批量预取 |

**必答建议（需实测）**：

- 训练数据**优先打包**；
- 多 epoch 训练时**把数据缓存到节点本地 NVMe**（见 D5 Fluid/缓存）；
- 用 `num_workers`、`prefetch_factor`、pin\_memory 等 DataLoader 参数把 IO 与计算 overlap（**否则 GPU 会等数据**）。

## D3. checkpoint：规模、频率与"最优间隔"

**规模估算（示例公式，需用真实训练验证）**：

```
checkpoint 大小 ≈ 参数量 × 每参数字节数
   全参数训练（含 fp32 优化器状态）≈ 16 bytes/param      ⇒ 70B 约 1.1TB（量级）
   只存权重（bf16）              ≈ 2 bytes/param          ⇒ 70B 约 140GB
```

**频率怎么定（高阶题，能给出模型非常加分）**：

```
若：M = 集群/任务的平均无故障时间（MTBF），δ = 一次 checkpoint 的耗时（或写放大成本）
Young/Daly 的一阶最优近似：
    最优间隔 L* ≈ sqrt(2 × δ × M)
（真实环境还有重启/恢复/队列等待等成本 ⇒ 这只是"量级参考"，务必结合自身数据校准）
```

**实践**：

- 写 checkpoint **不要直接堵住训练**：常用"**本地 NVMe 先落盘 → 异步上传到共享存储/对象存储**"；
- 保留策略要兼顾成本（旧 checkpoint 往往比数据盘贵）；
- **必须定期验证 checkpoint 可读**（很多团队踩过"存了 300 个 checkpoint 没有一个能恢复"）。

## D4. 并行文件系统 vs 对象存储：怎么选？

| 维度 | 并行文件系统（Lustre/GPFS/Weka/…） | 对象存储（S3/OSS/…） |
| --- | --- | --- |
| 语义 | POSIX | 对象 API |
| 吞吐/延迟 | **高吞吐、低延迟**，适合训练 | 高延迟、适合大块顺序读写 |
| 小文件 | 相对好 | **差** |
| 成本 | 高 | 低 |
| 容量 | 有限（需扩容） | 近乎无限 |
| 典型用法 | 训练共享数据、checkpoint 热层 | **数据湖、checkpoint 归档、模型仓库** |

**推荐组合**：**对象存储做数据湖与归档 + 并行文件系统/本地 NVMe 做热数据**，通过缓存系统把热数据自动落到计算节点附近。

## D5. Fluid / 数据缓存：为什么它是 K8s 原生的解法？

Fluid 提供 `Dataset` / `Runtime` 抽象，把 Alluxio/JindoFS 等缓存引擎编排到 K8s 里，并做**数据亲和性调度**（把 Pod 调度到已有缓存的节点）。

```yaml
apiVersion: data.fluid.io/v1alpha1
kind: Dataset
metadata:
  name: imagenet
spec:
  mounts:
    - mountPoint: s3://bucket/imagenet
      name: imagenet
  accessModes: ["ReadOnlyMany"]
---
apiVersion: data.fluid.io/v1alpha1
kind: AlluxioRuntime
metadata:
  name: imagenet
spec:
  replicas: 8
  tieredstore:
    levels:
      - mediumtype: SSD
        path: /var/lib/alluxio
        quota: 500Gi
```

**价值（必答）**：

1. **把"网络/远端存储带宽"换成"本地 NVMe 带宽"**，显著降低数据等待；
2. **缓存在节点上可复用**（多 epoch、多任务共享同一数据集）；
3. 配合调度亲和 ⇒ 减少"缓存未命中导致的慢节点"。

**风险**：缓存一致性（源数据更新后缓存失效策略）、缓存容量规划、缓存节点故障后的回源风暴。

## D6. GPUDirect Storage（GDS）与为什么它重要

- GDS 让存储设备**直接 DMA 到 GPU 显存**，绕过主机内存中转；
- 对"超大 checkpoint / 超大样本加载"有收益；
- 需要驱动、文件系统、硬件链路的支持（**具体支持矩阵以厂商文档为准**），且配置复杂 ⇒ 只有当 IO 成为瓶颈时才值得投入。

## D7. K8s 侧的卷使用模式

| 场景 | 推荐 |
| --- | --- |
| 只读模型权重、多 Pod 共享 | `ReadOnlyMany` 共享文件卷，或每个节点本地缓存 |
| checkpoint 写 | 共享文件卷（RWX 或专用写入卷）+ 本地 NVMe 缓冲 |
| 训练临时空间 / shm | `emptyDir`（必要时 `medium: Memory`，注意计入 memory limit，见《Linux 篇》A19） |
| 数据集缓存 | Fluid/本地 NVMe（hostPath/local PV 由平台统一管理） |

```yaml
volumes:
  - name: shm
    emptyDir: {medium: Memory, sizeLimit: 8Gi}     # 训练 DataLoader 多 worker 常见需求
  - name: model
    persistentVolumeClaim: {claimName: model-cache, readOnly: true}
```

**必答坑**：

- **大量卷挂载会让 Pod 启动变慢**（每个卷都要 attach/mount，《存储篇》A6/C3）；
- **RWX 文件系统的并发写语义**不适合做 checkpoint 的"多写"——checkpoint 的写入者应唯一。

## D8. 模型分发：为什么不要"把权重打进镜像"

| 方案 | 问题/优势 |
| --- | --- |
| **权重打进镜像** | 镜像 10\~100GB ⇒ 拉取慢、registry 压力大、版本管理混乱、**强烈不推荐** |
| **共享文件卷 / 对象存储挂载** | 常见；但要解决并发拉取造成的带宽风暴 |
| **P2P/延迟加载**（Dragonfly、Nydus/erofs、soci 等） | 大规模分发友好 |
| **节点本地预缓存 + 预热 DaemonSet** | 推理弹性扩容的关键（见 E 篇） |

```yaml
# 推理服务用 initContainer 预热权重到本地缓存（示意）
initContainers:
  - name: prefetch
    image: my-registry/model-fetcher:latest
    command: ["sh", "-c", "aws s3 sync s3://models/qwen-72b /cache/qwen-72b"]
    volumeMounts:
      - name: cache
        mountPath: /cache
```

## D9. 存储侧的容量与性能度量

```promql
# 卷使用率与 inode（见《存储篇》E3）
kubelet_volume_stats_used_bytes / kubelet_volume_stats_capacity_bytes > 0.85
kubelet_volume_stats_inodes_used / kubelet_volume_stats_inodes > 0.9
# 节点磁盘吞吐打满（数据加载瓶颈的间接证据）
rate(node_disk_written_bytes_total[5m])
rate(node_disk_read_bytes_total[5m])
# IO 等待
rate(node_cpu_seconds_total{mode="iowait"}[5m])
```

**训练侧更直接的判断**：如果 GPU 利用率周期性掉到很低，且 `iowait` 或磁盘吞吐同时打满 ⇒ **数据/存储是瓶颈**，先做缓存与预取，而不是先调 NCCL。

## D10. 存储方案的选型决策（综合）

```
1) 数据规模与形态？
   - 海量小文件 → 打包 + 缓存
   - 大块顺序 → 对象存储直读 + 缓存
2) 是否有 POSIX 强依赖？
   - 有（多数训练框架默认）→ 并行文件系统 或 缓存引擎提供的 POSIX
   - 无（自研数据管线）→ 对象存储 + 预取更便宜
3) checkpoint 频次与规模？
   - TB 级高频 → 本地 NVMe 写 + 异步上传；共享存储要能扛突发带宽
4) 预算与运维能力？
   - 自建 Lustre/GPFS 复杂度高；托管并行文件/云文件存储省心但贵
5) 是否需要跨地域/多集群？
   - 需要 → 对象存储做统一数据湖，计算侧各自缓存
```

---

# Part E · 推理平台与弹性（10 题）

## E1. 推理服务的形态：为什么它和 Web 服务差别很大？

| 维度 | Web 服务 | **LLM 推理** |
| --- | --- | --- |
| 单请求耗时 | 毫秒 | **秒\~****分钟** |
| 资源瓶颈 | CPU/网络 | **显存（KV cache）与带宽** |
| 扩缩容速度 | 秒级 | **分钟级**（权重加载慢） |
| 弹性触发量 | QPS/CPU | **排队长度、KV cache 使用率、TTFT/TPOT** |
| 失败模式 | 5xx | 超时、OOM（显存打满）、降级（拒绝/截断） |
| 状态 | 无状态 | **有 KV cache 状态（前缀缓存亲和）** |

## E2. 推理服务在 K8s 上的编排方式

| 方案 | 适用 |
| --- | --- |
| Deployment + Service | 单卡/单实例、无 TP |
| **LeaderWorkerSet（LWS）** | **多机 TP**（一组 Pod 整体调度/更新） |
| **KServe / vLLM Production Stack / AIBrix / Dynamo** | 更高层的模型服务编排（路由、批处理、缓存、自动扩缩） |
| KubeRay | Ray 生态（vLLM 在 Ray 上的分布式部署） |

```yaml
# vLLM 部署要点示意
spec:
  containers:
    - name: vllm
      image: vllm/vllm-openai:latest
      args: ["--model", "/models/qwen-72b", "--tensor-parallel-size", "2", "--max-model-len", "32768"]
      resources:
        limits: {nvidia.com/gpu: 2}
      readinessProbe:
        httpGet: {path: /health, port: 8000}
        periodSeconds: 5
        failureThreshold: 60         # 权重加载慢 ⇒ 就绪探针必须放宽
      startupProbe:
        httpGet: {path: /health, port: 8000}
        failureThreshold: 120
        periodSeconds: 5
      terminationGracePeriodSeconds: 120    # 生成中的请求要能优雅结束
```

**必答**：推理服务的**探针必须考虑加载时间**（否则会陷入"启动即被杀"的循环，《Kubelet 篇》Q9/Q10），终止也要给足时间让在途生成结束。

## E3. 推理的自动扩缩怎么做？

| 信号 | 说明 |
| --- | --- |
| QPS / 并发请求数 | 基础 |
| **排队长度（queue depth）** | 最贴近用户体感 |
| **KV cache 使用率** | 决定还能不能再接请求（vLLM 类引擎会暴露） |
| **TTFT / TPOT 的 P99** | 直接的 SLO 信号 |

```yaml
# KEDA 基于自定义指标扩缩（示意）
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: vllm
spec:
  scaleTargetRef: {name: vllm}
  triggers:
    - type: prometheus
      metadata:
        serverAddress: http://prometheus:9090
        metricName: vllm_num_requests_waiting      # 指标名以所用引擎为准
        threshold: "3"
  advanced:
    restoreToOriginalReplicaCount: false
```

**冷启动对策（必答）**：

1. **镜像与权重预热**：节点上预缓存权重（DaemonSet 预热 + 本地缓存卷）；
2. **常驻最小副本**（避免冷启动打到首个用户）；
3. **使用延迟加载/本地 NVMe 缓存**降低加载时间；
4. 扩缩策略"**快扩慢缩**"（缩容要保守，避免反复抖动）。

## E4. 推理网关：为什么需要"模型感知"的路由？

普通的 L4/轮询路由对 LLM 很低效，因为：

- 请求差异巨大（prompt 长度、输出长度不同 ⇒ 成本不同）；
- **前缀缓存（prefix cache）命中率**取决于路由到哪个实例；
- 需要**队列与优先级**（显存决定并发上限，不能无限堆请求）；
- 需要按 LoRA/适配器、按模型版本路由。

**社区方向**：**Gateway API Inference Extension**（K8s 生态的推理网关扩展）提供"模型感知的端点选择 + 请求排队 + 关键性/优先级"，是近年面试的加分话题（**项目状态请以官方仓库为准**）。

## E5. 高可用的多副本与打散

```yaml
topologySpreadConstraints:
  - maxSkew: 1
    topologyKey: topology.kubernetes.io/zone
    whenUnsatisfiable: ScheduleAnyway
    labelSelector: {matchLabels: {app: vllm}}
---
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: vllm
spec:
  minAvailable: 2
  selector: {matchLabels: {app: vllm}}
```

**必答**：大模型推理的"**副本漂移很贵**"（加载慢），所以 **PDB 与优雅终止**尤其重要，节点维护/缩减时不能把多个副本同时干掉。

## E6. 单卡放不下怎么办？（推理并行）

| 手段 | 说明 |
| --- | --- |
| **TP（张量并行）** | 一张卡放不下权重时切分；**强烈建议单机内（NVLink）** |
| **PP（流水并行）** | 跨机推理可用（延迟增加） |
| **量化（FP8/AWQ/GPTQ）** | 降显存与带宽 ⇒ 提高吞吐 |
| **Speculative decoding** | 降延迟（用算力换延迟） |
| **Prefill/Decode 分离** | 两种阶段独立扩缩（趋势） |

## E7. 推理服务的 SLO 怎么定？（可落地）

| SLI | 建议目标（起点，需实测） |
| --- | --- |
| **TTFT**（首 token 延迟）P99 | 如 ≤ 2s（交互式）、≤ 10s（长 prompt 批处理） |
| **TPOT / ITL**（token 间隔）P99 | 如 ≤ 100ms/token（保证"看起来流畅"） |
| 请求成功率 | ≥ 99.9% |
| 拒绝率（显存/队列满） | ≤ 0.1% |
| 端到端完成时延 P99 | 按 prompt/输出长度分段定义（**不要用一个数概括**） |

> **注意**：LLM 的延迟与输出长度强相关，SLO 必须\*\*按"输入长度/输出长度分桶"\*\*定义，否则平均值毫无意义。

## E8. 批处理/离线推理怎么设计？

- 用 **Job + Kueue 排队**，用 GPU 池的"低优先级/可抢占"资源；
- 吞吐优先：大 batch、continuous batching、无严格延迟 SLO；
- 数据同样要缓存/预取，避免 IO 成为瓶颈。

## E9. 推理与训练混部的风险

| 风险 | 缓解 |
| --- | --- |
| 训练的长任务占满 GPU 池 | 节点池分层 + Kueue 配额 |
| 推理延迟被训练的通信/IO 影响 | 网络平面/存储平面分流；NUMA 与网卡亲和 |
| 抢占导致推理中断 | 推理副本设更高优先级 + PDB；训练侧支持 checkpoint 恢复 |
| 显存/温度/功耗耦合 | 监控 + 节点池隔离 |

## E10. GPU 池的"利用率治理"闭环（平台视角）

```
① 采集：DCGM 指标（SM/Tensor active、显存、温度、功耗、XID）+ 作业级标签
② 归因：按 team/model/job 聚合（标签必须落到 Pod 上）
③ 发现浪费：分配但未使用（僵尸卡）、MFU 低、排队久
④ 治理手段：队列配额、自动缩容空闲作业、抢占低优先级、开发池限时限量
⑤ 汇报：利用率看板 + 周报（showback），与预算挂钩
```

```promql
# 僵尸卡：分配出去但长时间几乎不干活（阈值需实测校准）
DCGM_FI_PROF_SM_ACTIVE < 5
# 按团队聚合的 GPU 小时（需要把 team 标签关联到 GPU 指标）
sum by (team) (DCGM_FI_PROF_GR_ENGINE_ACTIVE)
```

# Part F · 可靠性、容错与成本（10 题）

## F1. 训练任务的故障来源分布（先有概率直觉）

| 来源 | 典型 |
| --- | --- |
| **硬件** | GPU XID/ECC/掉卡、NVLink down、网卡/光模块故障、内存 ECC、电源/散热 |
| **软件/框架** | OOM、死锁、NCCL 初始化失败、数据样本异常（NaN/超长）、依赖版本冲突 |
| **平台** | 节点被驱逐、调度失败、镜像拉不动、存储/网络故障、证书过期 |
| **人为** | 参数写错、配额误配、误删卷/作业 |

**关键认知**：**训练时长越长，遭遇硬件故障的概率越高**（MTBF × 卡数 ⇒ 千卡训练几乎每天都会遇到点什么）。所以**容错不是可选项**。

## F2. 训练容错的三层机制

```
① 自动重启（框架层）：torchelastic / Job 的 restartPolicy
     ⇒ 代价：从上一个 checkpoint 重跑（丢失间隔内的进度）
② Checkpoint + 恢复（框架/平台层）
     ⇒ 决定 RPO（丢多少进度）
③ 节点自愈（平台层）
     ⇒ 发现坏卡/坏节点 → cordon/drain → 维护流程 → 回池
```

**必答的量化**（见 D3 的最优间隔公式）：checkpoint 太频繁 ⇒ 写放大拖慢训练；太稀疏 ⇒ 故障损失大。**这是一道可以用公式回答的工程题**。

## F3. 坏卡/坏节点怎么自动处理？

```yaml
# 思路（具体工具请以集群所用方案为准，例如 node-maintenance / medik8s 类项目）
1. DCGM/事件检测到 XID 或 ECC 不可恢复错误
2. 触发告警 + 自动 cordon（停止接收新任务）
3. 若节点上无关键长任务 → drain（配合 PDB 与作业层容忍）
4. 触发 GPU reset / 重启驱动 / 重启节点
5. 自检通过（跑一遍 GPU 健康检查 + nccl-tests）→ uncordon 回池
```

**必答**：**不要自动 drain 正在跑长训练的节点**（会毁掉几天的算力）；正确做法是 **cordon + 等待作业结束或作业层决定**。

## F4. 慢节点（straggler）怎么发现？

| 信号 | 来源 |
| --- | --- |
| 某 rank 的 step time 明显高于中位数 | 训练框架侧日志/指标 |
| 该节点的 GPU 时钟/温度/功耗异常 | DCGM |
| 该节点网卡丢包/降速 | 网络计数器 |
| PSI / iowait 偏高 | node exporter |

**处置**：先 cordon 观察；若持续异常 ⇒ 从训练池摘除并走维修流程。**识别 straggler 是千卡训练的必修课**（它把一个整体的 MFU 拖下来）。

## F5. 训练 hang 了怎么排查？

```
① 看是"卡在哪一步"：NCCL init？第一步？第 N 步（通常是数据/checkpoint）？
② NCCL_DEBUG=INFO + SUBSYS=INIT,ENV,GRAPH ⇒ 是否选错网卡、是否 GDR 未启用
③ py-spy dump / py-spy top -p <pid> ⇒ 看进程栈卡在哪（IO 还是通信）
④ nvidia-smi ⇒ 是否某卡显存暴涨/OOM、是否有进程残留
⑤ 网络：ibstat/perftest/nccl-tests 基线
⑥ 存储：df/iostat，checkpoint 目录是否写满/写慢
⑦ 若为同步等待：对比各 rank 的 step time（找 straggler）
```

```bash
py-spy dump --pid <pid>
py-spy top --pid <pid>
```

## F6. 多租户 GPU 集群的配额与公平（与 B8 联动）

| 机制 | 作用 |
| --- | --- |
| Kueue ClusterQueue / Cohort | 配额、借用/归还、优先级与抢占 |
| ResourceFlavor + taint | 把不同机型/价格/可靠性的卡池区分开 |
| Namespace ResourceQuota | `nvidia.com/gpu` 的硬上限 |
| 作业优先级 | 高优研究任务插队；低优开发任务可抢占 |

**必答**：**"抢占训练任务"代价极高**（丢进度）⇒ 通常只抢占**短任务/低优推理/开发任务**，对长训练用"排队 + 预约"而不是硬抢。

## F7. Spot / 竞价实例能不能用来训练？

| 场景 | 结论 |
| --- | --- |
| 短任务（\<1h）、可重试 | ✅ 合适 |
| 长训练 | ⚠️ 可用但**必须有 checkpoint + 自动恢复**，且要接受中断重排 |
| 推理（在线） | ❌ 一般不合适（除非多副本 + 快速重建 + 权重预热） |

**配套**：节点池区分 spot/on-demand（用 taint 隔离），作业声明容忍哪种池（Kueue ResourceFlavor + nodeSelector）。

## F8. GPU 成本怎么算与优化？（老板会问的题）

```
成本 = Σ(GPU 单价 × GPU 小时) + 网络/存储/机房
优化杠杆（按优先级）：
  1) 利用率（分配率 × 使用率 × MFU）—— 最大的一块
  2) 机型选择（同样 MFU 下更便宜的卡 / 更合适的互联）
  3) 采购与承诺（预留/承诺折扣） vs 按需 vs Spot
  4) 并行策略与精度（FP8/量化显著提升吞吐）
  5) 数据/存储开销（缓存、打包、checkpoint 保留策略）
  6) 组织手段（配额、限时限量的开发池、showback/chargeback）
```

**必答**：**最大的浪费通常不是"买贵了"，而是"分了不用"和"用了但 MFU 低"** ⇒ 监控与归因机制比砍价更重要。

## F9. 变更治理：GPU 集群哪些变更最危险？

| 变更 | 风险 | 做法 |
| --- | --- | --- |
| **驱动/CUDA 版本** | 全集群不兼容、训练直接失败 | 节点池灰度（1 台 → 10% → 全量）+ 跑 nccl-tests/训练冒烟 |
| **GPU Operator / device plugin** | 设备不上报 ⇒ 全部新任务 Pending | 先灰度一个池，保留可回滚版本 |
| **MIG 策略切换** | 需排空节点、资源形态改变 | 专用池 + 窗口期 + 通知 tenant |
| **网络固件/交换机配置** | 训练性能断崖或 hang | 变更前后跑 nccl-tests 基线对比 |
| **内核/cgroup** | 驱动装载失败 | 按《Linux 篇》C14 的节点池灰度 |

**通用规则**：**每次变更后跑"基线测试 + 冒烟训练（小步数）"，对比 busbw 与 step time**，通过才放量。

## F10. 一份"GPU 节点上线 Checklist"

```
[ ] 驱动/CUDA/toolkit 版本与预期的矩阵一致
[ ] nvidia-smi 可见全部 8 张卡，ECC 开启，无 XID 历史
[ ] NFD/GFD 标签正确（型号、显存、驱动版本）
[ ] 拓扑符合预期（GPU↔NIC 的 NUMA/PCIe 关系）
[ ] RDMA 网卡 up、速率与 MTU 正确、nccl-tests busbw 达基线
[ ] device plugin 上报数量正确；MIG 配置符合池定位
[ ] DCGM exporter 正常，指标可查
[ ] kubelet 参数（reservedSystemCPUs / topology manager）已按池配置
[ ] 存储/缓存（本地 NVMe、共享卷）可读写并有容量告警
[ ] 加入正确的节点池（taint/label），并通过 Kueue/调度器的准入
[ ] 冒烟：跑一个小训练 + 一个小推理，确认端到端 OK
```

---

# Part G · 反模式清单（可直接用于生产巡检）

| # | 反模式 | 证据 | 等级 | 修复 | 验证 |
| --- | --- | --- | --- | --- | --- |
| G1 | 权重打进镜像（10GB+） | 镜像列表体积异常、发布极慢 | 🔴 高 | 权重走共享存储/对象存储 + 预热 | 发布时间显著下降（实测） |
| G2 | 训练任务没有 Gang/Coscheduling | 部分 Pod 起来互相等待、GPU 空占 | 🔴 高 | Volcano/Kueue/coscheduling | 死锁场景不再出现 |
| G3 | 推理探针阈值沿用 Web 服务默认值 | 启动即被杀、滚动更新失败 | 🔴 高 | startupProbe + 放宽 failureThreshold | 冷启动场景能正常 Ready |
| G4 | 推理用 Service + gRPC 长连接无 per-request LB | 副本负载严重不均 | 🔴 高 | 限制 MAX\_CONCURRENT\_STREAMS / 用推理网关 | 各副本请求量均衡（实测） |
| G5 | NCCL 未指定网卡/接口 | hang 在 init、跨机走错网卡 | 🔴 高 | 显式配置 `NCCL_*` 并固化到作业模板 | nccl-tests 稳定通过 |
| G6 | 无 checkpoint 就跑长训练 | 一次故障损失数天 | 🔴 高 | checkpoint 策略 + 定期恢复演练 | 恢复演练成功且 RTO 达标 |
| G7 | 把 MIG 开在训练池 | 训练拿不到完整卡、性能受损 | 🔴 高 | 池分层：训练整卡、推理可共享 | 训练作业调度成功率提升 |
| G8 | 只看 `nvidia-smi` 的 GPU-Util | 误判"很忙"实际 MFU 很低 | 🟠 中 | DCGM SM/Tensor 指标 + MFU 计算 | 优化立项有数据支撑 |
| G9 | 无 GPU 健康（XID/ECC）监控与自动排空 | 坏卡继续接任务，浪费算力 | 🔴 高 | DCGM 告警 + 自动 cordon + 维护流程 | 注入 XID 演练能自动隔离 |
| G10 | 训练与推理混部不分池 | 推理被训练挤兑、抢占损失大 | 🟠 中 | 节点池分层 + 配额 | 推理 SLO 稳定 |
| G11 | 数据小文件直读共享存储 | iowait 高、GPU 等数据 | 🟠 中 | 打包 + 本地缓存（Fluid） | GPU 空等时间下降 |
| G12 | checkpoint 直接写远端且不节流 | 写带宽打满，训练被拖慢 | 🟠 中 | 本地 NVMe 缓冲 + 异步上传 | step time 抖动消失 |
| G13 | 队列/配额缺失（谁先提交谁占满） | 研究员抢卡、互相抱怨 | 🟠 中 | Kueue/Volcano 队列与配额 | 配额守底线，借用可控 |
| G14 | 无基线（nccl-tests / 训练冒烟）就做网络变更 | 变更后性能断崖才发现 | 🔴 高 | 变更前后的 busbw/step time 对比 | 基线归档可比对 |
| G15 | 反向代理开启响应缓冲 + 短超时 | SSE 流式输出被缓冲、请求被截断 | 🟠 中 | 关闭 buffering + 放大超时 | 流式输出连续 |

---

# Part H · 可观测性：GPU 与大模型指标与告警

## H1. 指标分层

| 层 | 指标来源 | 例子 |
| --- | --- | --- |
| **GPU 设备** | **dcgm-exporter** | `DCGM_FI_PROF_GR_ENGINE_ACTIVE`、`_SM_ACTIVE`、`_PIPE_TENSOR_ACTIVE`、`DCGM_FI_DEV_FB_USED/_FB_FREE`、`DCGM_FI_DEV_GPU_TEMP`、`DCGM_FI_DEV_POWER_USAGE`、`DCGM_FI_DEV_SM_CLOCK`、`DCGM_FI_DEV_XID_ERRORS`、`DCGM_FI_DEV_ECC_*` |
| **分配/调度** | kube-state-metrics + kubelet | `kube_pod_container_resource_requests{resource="nvidia.com/gpu"}`、`kube_pod_container_resource_limits`、`scheduler_pending_pods`、`kubeproxy_*` |
| **网络** | node exporter / 厂商 exporter / nccl-tests | 网卡吞吐、丢包、错包、RDMA 计数、交换机计数 |
| **存储** | `kubelet_volume_stats_*`、node exporter | 容量、inode、磁盘吞吐、iowait |
| **推理引擎** | vLLM/TGI 自带 metrics | 排队长度、运行中请求数、KV cache 使用率、TTFT/TPOT、吞吐(tokens/s) |
| **训练框架** | 框架导出或日志 | step time、loss、grad norm、samples/s、tokens/s |
| **作业/队列** | Kueue / Volcano | 排队时长、准入数、抢占次数、配额使用率 |

> **原则**：GPU 指标必须能**关联到 team/model/job**（通过 Pod 标签 join），否则无法归因与治理。

## H2. 告警规则（可直接粘贴，阈值需实测校准）

```yaml
groups:
  - name: gpu-platform
    rules:
      - alert: GpuXidError
        expr: increase(DCGM_FI_DEV_XID_ERRORS[10m]) > 0
        for: 2m
        labels: {severity: critical}
        annotations:
          summary: "节点 {{ $labels.instance }} GPU {{ $labels.gpu }} 出现 XID 错误，建议排空检查"

      - alert: GpuEccError
        expr: increase(DCGM_FI_DEV_ECC_DBE_VOL_TOTAL[1h]) > 0
        for: 5m
        labels: {severity: critical}
        annotations:
          summary: "出现不可纠正 ECC 错误，需 GPU reset / 换卡"

      - alert: GpuMemoryPressure
        expr: DCGM_FI_DEV_FB_USED / (DCGM_FI_DEV_FB_USED + DCGM_FI_DEV_FB_FREE) > 0.95
        for: 15m
        labels: {severity: warning}

      - alert: GpuIdleButAllocated
        expr: DCGM_FI_PROF_SM_ACTIVE < 5
        for: 6h
        labels: {severity: info}
        annotations:
          summary: "GPU 长时间空闲但被占用，检查是否有僵尸任务"

      - alert: GpuThermalOrPowerThrottle
        expr: DCGM_FI_DEV_POWER_USAGE / DCGM_FI_DEV_POWER_MGMT_LIMIT > 0.98
        for: 30m
        labels: {severity: warning}
        annotations:
          summary: "GPU 触及功率上限，可能因温度/功耗导致降频"

      - alert: GpuClockThrottle
        expr: DCGM_FI_DEV_SM_CLOCK < 1000
        for: 30m
        labels: {severity: warning}
        annotations:
          summary: "GPU 时钟异常偏低，检查散热/供电/驱动"

      - alert: TrainingJobStalled
        expr: |
          # 需从框架侧导出 step time 或用自定义指标；示意：
          histogram_quantile(0.99, rate(train_step_duration_seconds_bucket[10m])) > 300
        for: 10m
        labels: {severity: warning}
        annotations:
          summary: "训练 step 耗时异常，可能是 straggler / 网络 / IO 问题"

      - alert: GpuQuotaQueueBacklog
        expr: kueue_pending_workloads > 50        # Kueue 指标名请以所用版本核对
        for: 30m
        labels: {severity: info}
        annotations:
          summary: "GPU 队列积压较多，评估是否需要扩容或调整配额"

      - alert: InferenceQueueBacklog
        expr: vllm_num_requests_waiting > 10           # 指标名以所用引擎为准
        for: 10m
        labels: {severity: warning}
        annotations:
          summary: "推理排队积压，需扩容或降级"

      - alert: InferenceKVCacheHigh
        expr: vllm_gpu_cache_usage_perc > 0.9           # 指标名以所用引擎为准
        for: 15m
        labels: {severity: warning}
        annotations:
          summary: "KV cache 使用率过高，可能出现抢占/拒绝请求"
```

> 同上篇规范：**指标名必须先在目标环境 **`/metrics`** 核对**（不同版本 dcgm-exporter 的字段命名有差异）。

## H3. 建议的 SLO（**初始建议值，需按业务与实测校准**）

| SLI | 建议目标 |
| --- | --- |
| GPU 有效利用率（分配且 SM active） | ≥ 70%（平台目标，需归因口径） |
| 训练作业 MFU | ≥ 30%（低于即立项优化） |
| 作业排队时长 P90 | ≤ 数小时（按队列承诺） |
| 训练任务非计划中断率 | ≤ 1 次 / 每 100 GPU·天（按自身基线定） |
| 推理 TTFT P99 / TPOT P99 | 按分桶定义（见 E7） |
| 推理请求成功率 | ≥ 99.9% |
| 坏卡自动隔离时长 | ≤ 15min（从告警到 cordon） |

# Part I · 演练（GameDay）计划表

| 场景 | 注入方式 | 期望表现 | 验证的 SLO |
| --- | --- | --- | --- |
| 坏卡（XID/ECC） | 注入 XID 事件或人为标记 | 告警 → 自动 cordon → 维护流程；不再接新任务 | 坏卡隔离时长 |
| GPU 掉卡 | 模拟 `fallen off bus` | 节点被隔离，作业按策略失败/重排 | 容错与 RTO |
| 训练 hang | 让 NCCL 走错接口 | 能快速定位（NCCL\_DEBUG + py-spy），超时后作业重启 | MTTR |
| straggler | 人为降频/限速某节点 | 能被识别并隔离 | 训练稳定性 |
| 网络降级 | 限速某条链路/降 MTU | busbw 基线下降可被发现；训练 step time 上升可归因 | 网络基线有效性 |
| checkpoint 失败 | 让存储不可写 | 训练不静默失败；有告警；可恢复 | 容错可靠性 |
| checkpoint 恢复演练 | 从 checkpoint 重启训练 | 数据一致、loss 曲线连续、耗时达标 | RPO/RTO |
| 推理冷启动 | 从 0 扩容 | 权重预热有效，TTFT 达标 | 弹性与 TTFT |
| 推理负载不均 | 单连接压测 | 网关/连接策略生效，副本均衡 | 负载均衡有效性 |
| 推理显存打满 | 发送超长请求 | 引擎按策略拒绝/抢占，不整体崩溃 | 稳定性 |
| 配额挤兑 | 多团队同时提交 | Kueue 队列按优先级与配额准入，无饿死 | 公平性 |
| 驱动升级 | 灰度一个池 | 冒烟与 nccl 基线通过才放量 | 变更安全性 |

---

# Part J · 面试速记卡（15 条）+ 一道综合设计题

## J1. 15 张速记卡

1. **容器用 GPU 的链路**：驱动容器 → toolkit/CDI → device plugin 上报 → limits `nvidia.com/gpu: N` → kubelet Allocate → 注入 `/dev/nvidia*` 与库。
2. **扩展资源的铁律**：只能是整数、requests 必须等于 limits；调度器**只做数字减法**，不懂型号/拓扑。
3. **device plugin 的短板**：不会选设备、不能表达属性、调度器不参与 ⇒ 这就是 **DRA** 的动机。
4. **DRA 三件套**：DeviceClass（模板）+ ResourceClaim（申请）+ ResourceSlice（驱动上报的能力与属性），**调度器参与选择**。
5. **Gang Scheduling 是刚需**：否则出现"各占一半互相等待"的死锁；实现靠 **Permit 扩展点**（占位等待 = 真占资源，必须配超时）。
6. **并行策略第一性原理**：通信最贵的切法放最快链路 —— **TP 在机内 NVLink，PP/DP 跨机**。
7. **显存公式**：训练（混合精度 + Adam）≈ **16 bytes/参数**；推理权重 bf16 ≈ 2 bytes/参数；KV cache 按 `2 × kv_heads × head_dim × layers × seq × dtype` 估。
8. **MFU**：`6N × tokens / step_time / (卡数 × 峰值)`；**低于 30% 就该立项**；口径（tokens、重计算、单位）是这题最容易翻车的地方。
9. **GPU 利用率三层**：有没有被分配 → 有没有 kernel 在跑（SM/Tensor active）→ 跑得是否有效（MFU）。只看 `nvidia-smi` 会误判。
10. **XID/ECC 是坏卡信号**：必须监控 + 自动 cordon；**不要自动 drain 正在跑长训练的节点**。
11. **NCCL 排障三板斧**：`NCCL_DEBUG=INFO`、指定 `NCCL_SOCKET_IFNAME`/`NCCL_IB_HCA`、`nccl-tests` 的 busbw 基线。
12. **RDMA 网络在 K8s 里**：hostNetwork（简单常用）或 Multus+SR-IOV/macvlan（隔离）；**别让 RDMA 流量进隧道**。
13. **checkpoint 最优间隔**：`sqrt(2 × δ × M)` 量级（δ=写一次成本，M=MTBF）；长训练必须先落本地 NVMe 再异步上传。
14. **推理三件套**：探针要放宽（加载慢）+ 优雅终止要加长 + **Service+gRPC 要处理连接级负载不均**（限制 MAX\_CONCURRENT\_STREAMS 或推理网关）。
15. **成本杠杆顺序**：利用率（分配率×使用率×MFU） \> 机型/采购方式 \> 精度与批处理优化 \> 存储与数据开销。

## J2. 综合设计题：**"给你一个 512 卡（64 台 8 卡机）集群，支撑训练 + 推理，怎么设计？"**

按这个顺序答（面试官会认为你真做过）：

**① 先问清需求（SLO 与约束）**

- 训练：模型规模（7B/70B/…）、并行策略、期望 MFU、作业并发数、单作业最长时长、RPO（能接受丢多少进度）；
- 推理：模型、QPS 与并发、TTFT/TPOT 目标、是否需要多机 TP、是否流式；
- 组织：团队数、配额规则、优先级、是否有开发池；
- 预算：机型、云/自建、预留 vs 按需 vs Spot。

**② 硬件与拓扑**

- 机型统一（避免异构导致调度与性能不一致）；
- 机内 8 卡 NVLink 域 ⇒ **TP ≤ 8（机内）**；
- 跨机：**IB 或 RoCEv2** + rail-optimized 拓扑；每节点 ≥1\~2 张高速网卡（按卡数配比）；
- 存储：并行文件系统/共享存储 + 本地 NVMe（缓存与 checkpoint 缓冲）+ 对象存储（数据湖与归档）。

**③ 平台层（K8s）**

- GPU Operator 统一交付驱动/工具链/device plugin/DCGM/GFD；Network Operator 管 RDMA 与多网卡；
- 节点池分层：**训练池（整卡、禁 MIG） / 推理池（可共享或 MIG） / 开发池（低优、可抢占、限时限量）**；
- 调度：**Kueue（队列/配额/准入） + Volcano 或 coscheduling（Gang）**；
- 拓扑：`topologySpreadConstraints`（训练聚拢、推理打散）+ nodeAffinity 限定机型 + 必要时 single-numa-node。

**④ 作业层**

- 训练：Training Operator / MPIJob + torchelastic + checkpoint 策略（本地 NVMe → 异步上传 → 保留策略）+ 恢复演练；
- 推理：LWS（多机 TP）+ vLLM 类引擎 + 推理网关（队列与缓存感知）+ KEDA/Prometheus 指标扩缩 + 权重预热。

**⑤ 可观测与治理**

- DCGM 指标 + XID/ECC 告警 + 自动 cordon；
- 训练：step time、loss、MFU、straggler 检测；
- 推理：TTFT/TPOT 分桶、排队长度、KV cache 使用率；
- 网络：nccl-tests busbw 基线归档；存储：容量/inode/吞吐；
- 归因：GPU 指标必须能 join 到 team/model/job。

**⑥ 变更与演练**

- 驱动/网络/调度组件变更 → 节点池灰度 + 基线对比；
- GameDay：坏卡、训练 hang、straggler、checkpoint 恢复、推理冷启动、配额挤兑。

**⑦ 成本闭环**

- 利用率看板 + showback/chargeback；
- 开发池限时限量；僵尸卡自动回收；Spot 池只给可中断任务。

> **收尾加分句**："这个设计里最容易失败的地方不是硬件，而是**治理**——没有队列与配额，512 卡会在两周内变成'先到先得 + 大量占着不用'。"

---

## 附：文档自查清单

- [ ] 每个主题配了可直接套用的 YAML / 环境变量 / PromQL / 命令片段
- [ ] 厂商规格数字均标注"以官方 Datasheet 复核"，显存/带宽/checkpoint 计算均标注为**估算公式**
- [ ] 未提供任何未经实测的性能数字（MFU 示例明确标为假设值），一律给出**测量方法**
- [ ] 反模式清单含：证据 / 等级 / 修复 / 验证
- [ ] SLO 标为建议初值
- [ ] 高危变更（驱动/MIG/网络）给出灰度、基线对比与回滚原则
- [ ] 与《Scheduler / Kubelet / 网络 / 控制面 / 存储 / Linux》六篇做了显式交叉引用
