# ai-infra-hpc 仓库 × 知乎面试清单：对比总结与你的精读路线

> 定制对象：于志泉 —— 12 年云原生 SRE / 小米弹性调度组 K8s 架构师 + 技术 Leader → AI Infra / GPU 算力平台岗
> 参考资料：知乎回答（作者 jinboom 的 9 章面试清单）、`github.com/jinbooooom/ai-infra-hpc`（实际文件树核对至 2026-10）
> 与既有资料的关系：**本文件是补丁包的主控表**，主干仍是你已有的《LLM 岗位面试总计划》10 周表 + 4 套 GPU 题库 +《SRE→AIInfra 手册》

---

## 0. 先给结论（三句话）

1. **知乎那篇就是这份仓库的"考试大纲"**，仓库是"教材正文"。9 章 ↔ 仓库目录基本 1:1 映射，不存在谁更全的问题——大纲告诉你"面试会问什么"，仓库告诉你"标准答案的原文在第几节"。
2. **但它的目标画像是"校招/实习、想做 CUDA-HPC 工程师的人"**（第 8、9 章明确强调手写 kernel + OriginDL 项目）。你不是这个画像：你的护城河在 **K8s/调度/成本/稳定性**（仓库完全没覆盖），短板在 **硬件拓扑 + 网络 + 通信**。
3. 因此对你的正确用法是 **倒着读、挑着读、只买它最硬的三块**：`03topo`（拓扑）、`08infiniband`（RDMA/RoCE）、`05ccl/02nccl`（NCCL）。`05cuda` 126KB 只当字典，**不要**按作者建议精读全部、更不要去写 OriginDL（理由见 §7）。

---

## 1. 逐章对照表（知乎 ↔ 仓库 ↔ 你该怎么处理）

| 知乎章节 | 仓库路径 | 真实体量 | 内容性质 | 你的优先级 | 处理建议 |
| --- | --- | --- | --- | --- | --- |
| 1 芯片 & 算力 | `01chip/GPU.md` 59K、`CPU.md` 69K、`NPU.md` 15K | 架构演进史（Fermi→Blackwell） | ★★★ | **GPU.md 选读**：Volta / Ampere / Hopper / Blackwell + 末尾"CUDA Core & Tensor Core / NVLink / NVSwitch"小结。CPU.md **不读**。NPU.md **速览 15 分钟**（异构/国产化话题备用） |
| 2 CUDA & HPC | `02hpc/01openmp/` **0 字节**、`02simd/` **0 字节**、`05cuda/readme.md` 126K + 40+ 个 `.cu` 样例 | 仓库最硬的一块，但只对"写 kernel 的人"是核心 | ★★ | **选读 4 节 + 跑 2 个样例**（§6 W6）。注意：知乎提了 02hpc，但 openmp/simd 是**空占位文件** |
| 3 互联 & 拓扑 | `01noc` 4.7K、`02pcie` 4.9K、`03topo/readme.md` 27K | NVLink/NVSwitch 发展 + 8 卡服务器拓扑实例 | ★★★★★ | **03topo 全读**（W1）；02pcie 速览 30 分钟；01noc 跳过 |
| 4 GPUDirect | `03link/05gpuDirect/readme.md` 44K | GDS/P2P/GDR + "三个关键问题" + 内存管理 | ★★★★ | **只读 3 块**：三种形态概览、"三个关键问题"、GPU 内存管理（W5） |
| 5 RDMA & IB | `08infiniband/rdma.md` 135K、`roce.md` 13K、`RDMA_Aware_Programming_user_manual.md` **418K** | 体系最完整的一份中文 RDMA 笔记 | ★★★★★ | rdma.md **选读 10 节**；roce.md **全读**（13K，全仓库对你 ROI 最高的 13K）；418K 手册**只当字典查** |
| 6 集合通信 NCCL | `05ccl/02nccl/readme.md` 40K、`01mpi` 6.4K + 源码 | 算法 + 建图 + channel + 协议 + 千卡经验 | ★★★★★ | **nccl 全读**（W3）；mpi 跳过（知乎没提，你也用不上） |
| 7 训练 & 推理 | `06trainAndInfer/readme.md` 31K、`dl.md` 2K、`infer.md` 2K | 显存换算、MFU、并行策略、ZeRO、DeepEP | ★★★★ | **全读**（W4 顺带），直接支撑"MFU 怎么算""1B 参数多少显存" |
| 8 工程实践 | 仓库无独立章节（散落各章排障小节） | — | — | 你自带，**最强项**，不用补 |
| 9 OriginDL 项目 | 不在这仓库（作者另一个项目） | — | — | **建议不做**，见 §7 |
| （知乎未提） | `04storage/` 0 字节、`07eployment/llama.cpp/` 6.7K、`99industryReport/` 243B | 基本空 | ★ | 略过；llama.cpp 与你的 W4–W6 推理计划有交集，可 10 分钟翻一下 |

**一句话**：仓库真正有料的只有 **6 个 md 文件**（05cuda / 03topo / 05gpuDirect / rdma / 02nccl / 06trainAndInfer），其余不是空就是占位或字典。

---

## 2. 仓库有、知乎没提的增量（对你有用）

- **`01chip/NPU.md`**：AI 芯片技术路线（GPU/FPGA/ASIC、训练芯片 vs 推理芯片）→ "异构算力 / 国产卡接入调度"话题的谈资，15 分钟速览即可。
- **`08infiniband/roce.md`（13K）+ rdma.md 里的 RoCE 章节**：**你的公认最短板（PFC/ECN/DCQCN）就藏在这里**。这是全仓库对你性价比最高的一份文档，13K 半天读完，直接把"IB/RoCE 生产未操盘"从硬伤降级为"体系读过 + 知道排查顺序"。
- **`03link/02pcie`**：PCIe 是理解拓扑和 GPUDirect 的前置，速览即可。
- **`07eployment/llama.cpp`**：与推理部署计划有交集，可选。

## 3. 知乎提了、仓库其实没有的（别白找）

`02hpc/01openmp/readme.md`、`02hpc/02simd/readme.md`、`04storage/readme.md`、`01chip/readme.md` **全是 0 字节**；`99industryReport` 只有 243B。别在这些目录上耗时间。

## 4. 两边都没有、而你恰好最强的（护城河，别丢）

K8s 调度与 GPU 编排（Volcano/Kueue/HAMi/DRA）、DCGM 可观测、Xid/坏卡/straggler 定位、算力成本与配额模型、推理服务化（vLLM）、多租户隔离。

→ **这个仓库不补这些**，别指望从这儿学。你的 4 套题库 + 10 周计划才是主干，本仓库只是**短板补丁包**。

---

## 5. 路线设计：三条反直觉原则

1. **不按目录顺序读**。01→08 是"从沙子到芯片"的自底向上，适合学生打地基；你的时间贵，且你已经有"上层的房子"（调度/成本/稳定性），缺的是地基里的某几根桩。
2. **自顶向下、现象驱动**。从你能碰到的现象（AllReduce 卡住、MFU 上不去、8 卡跑不满）往下追：NCCL → RDMA → 拓扑 → GPUDirect → CUDA。**每往下挖一层，只挖到"能解释这个现象"就停**。
3. **每一章必须有面试出口**。读完必须能脱稿回答你已有题库里的某道题（B1–B6 或 Phase1 十问），否则这一章就是白读。

---

## 6. 6 周嵌入式精读表（对齐你现有 10 周计划，不另起炉灶）

> 每周 6–8h（工作日 1h × 5 + 周末 3h），总投入 **约 40–45h**。左侧"主计划动作"保持原样，右列是**新增**的精读任务。

| 周 | 主计划动作（不变） | 新增精读 | 具体读哪些节 | 面试出口（必须能脱稿答） |
| --- | --- | --- | --- | --- |
| **W1** | DCGM + Grafana 大盘 | `03topo` 全读 + `01chip/GPU.md` 选读 | NVLink / NVSwitch 发展、GPU 互联内部结构、初代结构、八卡 P100 拓扑、新华三 AI 串行/并行/HPC 三种配置对比、QPI 与 UPI；GPU.md 的 Volta/Ampere/Hopper/Blackwell + 末尾小结 | ①NVLink 比 PCIe 强在哪，带宽差多少、怎么验证？②8 卡服务器三种拓扑的 trade-off？③挑机器训大模型看什么？④NUMA 是什么，怎么查设备亲和性？ |
| **W2** | Volcano gang scheduling | `02pcie` 速览 30 分钟 | PCIe 代际带宽、PCIe switch 与 CPU/NUMA 的关系 | 能说出"卡插在哪个 PCIe 树、离哪个 NUMA 节点近，为什么会影响训练" |
| **W3** ★最重 | nccl-tests + MFU + 坏卡 SOP | `05ccl/02nccl` 全读 + `roce.md` 全读 + `rdma.md` 选读 10 节 | **nccl**：算法原理与实现（Broadcast/AllGather/Reduce/ReduceScatter/AllReduce 的 Ring/Tree/Bruck + 耗时总结）、机器内拓扑分析、建图、路径计算、channel 搜索、transport 建立、double binary tree、Protocol（Simple/LL/LL128）、千卡训练经验、查看拓扑命令；**rdma**：原理与优缺点、名词关系（RDMA/IB/IBoE/RoCE/iWARP）、RoCEv1 vs v2、SEND/RECV/WRITE/READ/立即数、内存注册（含注册 GPU 内存）、QP/CQ/MR/PD/MW、性能优化（小包延迟/inline/unsignaled/MTU/多 QP/NUMA）、降低延迟波动（CPU 隔离/中断绑定）、门铃机制、错误处理（PORT DOWN / PORT INIT / 64MB 锁页限制） | **B1**（straggler 检测处置）、**B2**（Ring/Tree/NVLS × DP/TP/PP/EP 放置）、**B3**（IB/RoCE/GDR/SHARP 区别与混跑隔离）全覆盖 + ⑤AllReduce 卡住/忽快忽慢从哪几条线查？⑥千卡有哪些坑？⑦LL128 为什么块更大？⑧PFC/ECN/DCQCN 原理与拥塞扩散？ |
| **W4** | vLLM 部署 + 压测 | `06trainAndInfer` 复习 30 分钟 | 推理时的显存开销、KV cache 相关段落 | ⑨1B 参数多少显存？训练 vs 推理各花在哪，为什么激活值是大头？ |
| **W5** | 推理原理 + 量化 | `05gpuDirect` 选读 + rdma.md 内存注册节 | GPUDirect 概览（GDS / P2P / GDR、IB GPUDirect Async）、**"三个关键问题"**（网卡如何直接读写 GPU 显存 / GPU 如何访问网卡寄存器与队列 / GPU 如何提交通信请求与门铃同步）、GPU 内存管理（锁页内存、零拷贝、UVA、UM）、GDRCopy、NVSHMEM 与 NCCL 的关系 | ⑩"zero-copy / RDMA 直通 GPU"到底怎么实现的？⑪pinned memory 为什么带宽更高？⑫GPUDirect P2P 与 GPUDirect RDMA 分别解决什么问题？ |
| **W6** | K8s 推理部署 + 系统设计 | `05cuda` 选读 4 节 + 跑 2 个样例 | 2 编程模型（Grid/Block/Thread 三层、函数限定符、`__global__/__device__/__host__/__shared__/__constant__/__managed__/__restrict__`）、3 执行模型（SM / Block 不跨 SM / Warp / SIMD vs SIMT / 线程束分化 / occupancy）、4 内存布局（各层内存快慢、对齐与合并访问、pinned/零拷贝/UVA/UM）、6 流与并发（CUDA 流、重叠计算与拷贝、Hyper-Q）；Nsight 与 CUDA-GDB 只看概览 | ⑬Warp 分化会踩什么坑？⑭为什么访存不合并会"体感变慢"？⑮SM / Block / Warp 三层关系？⑯拿到一个慢 kernel 的排查顺序？ |
| **W7+** | 叙事定稿 + Mock | 只回头查，不新读 | — | 把 ①–⑯ 讲成"背景—我的动作—量化—复盘"四段式 |

---

## 7. 关于 OriginDL / 手写 CUDA 项目：我的判断是**别做**

作者建议拿 OriginDL 当面试项目，这个建议对你不成立：

1. **岗位画像不匹配**：你投的是 AI Infra 平台 / 算力调度岗，面试官问的是"8 卡机器怎么榨干""AllReduce 为什么慢""坏卡怎么定位"，不是"你写过几个 kernel"。
2. **会稀释主线**：你已有更强的项目叙事（千节点弹性调度、成本治理、混部）。从零写框架在 senior 面试官面前不加分，反而可能被解读为"方向漂移 / 项目深度不足"。
3. **时间账不划算**：写通一个 DL 框架 60–100h；同样时间你能拿到 DCGM + Volcano + vLLM + NCCL + MFU 五份产出物，命中率高得多。

**替代方案（半天拿到同样的"我动手调过"证据）**：直接用仓库现成的样例做 profiling 实验——

- `02hpc/05cuda/4.globalMemory/4.3.2readSegment.cu`（对齐 vs 未对齐访存）
- `02hpc/05cuda/3.cudaExecutionModel/3.4.2reduceInteger.cu`（reduce 的多种优化）

跑一遍 `ncu`，出一张"改前 vs 改后"的指标对比 + 三句话结论。面试说 **"我在样例 kernel 上做过 ncu 级 profiling，看过合并访问和 occupancy 的实际差别"**，比说"我写了个框架"更稳、更诚实、也更像平台工程师该有的深度。

---

## 8. 实验锚点（与精读同步，每个都要留产出物）

| # | 实验 | 关键动作 | 对应章节 | 产出物 |
| --- | --- | --- | --- | --- |
| 1 | 8 卡拓扑可视化 | `nvidia-smi topo -m` + `NCCL_TOPO_DUMP_FILE=./topo.xml`，手绘一张拓扑图并标注 NVLink/PCIe/NUMA | 03topo | 拓扑图 + 笔记 |
| 2 | NCCL 带宽基线 | `all_reduce_perf -b 8 -e 512M`，记 **busbw**，对照 NVLink/PCIe 理论带宽算效率 | 02nccl | busbw 表（含理论值对照） |
| 3 | NCCL 建图可观测 | `NCCL_DEBUG=INFO NCCL_DEBUG_SUBSYS=INIT,GRAPH` 跑一次多卡任务，读懂 channel / ring 是怎么选出来的 | 02nccl | 日志注解（贴进笔记） |
| 4 | MFU 实测 | nanoGPT / torchtitan 小训练：MFU = 模型 FLOPs ÷（卡数 × 峰值 FLOPS × step 时间） | 06trainAndInfer | MFU 表 + **口径说明**（怎么采的、对照组是谁） |
| 5 | kernel profiling | `ncu` 跑 readSegment / reduceInteger，对比对齐 vs 未对齐、shared memory 前后 | 05cuda | 前后对比截图 + 3 句结论 |
| 6 | （有 IB 环境才做）RoCE/IB 体检 | `ib_write_bw` / `perfquery` 看端口计数器与重传；没有环境就靠 roce.md 做原理笔记 | 08infiniband | 笔记，或一句诚实的边界表述 |

> 第 6 项如无环境，面试统一表述：**"RoCE 生产调优我没有操盘过，但体系我读过（PFC/ECN/DCQCN、QP/CQ/MR、NUMA 亲和、CPU 隔离），知道排查顺序是从端口计数器 → 拥塞域 → 亲和性配置往下走。"** 这比硬编安全得多。

---

## 9. 时间预算与止损线

- 总预算 **40–45h / 6 周**。任一章超出预算 1.5 倍，立刻降级为"只背出口题"。
- **硬止损**：
  - `RDMA_Aware_Programming_user_manual.md`（418KB）**不通读**，只按需查；
  - `05cuda/readme.md`（126KB）**不精读**，只读 §6 W6 标出的 4 节；
  - `01openmp` / `02simd` / `04storage` / `01noc` **不看**。
- 读不下去的章节（如 NCCL 源码分析细节）直接跳过——你的目标是**讲清楚设计意图**，不是能改 NCCL 源码。

---

## 10. 出口题自检清单（16 题，读完打勾）

| # | 问题 | 出处 |
| --- | --- | --- |
| 1 | NVLink 比 PCIe 强在哪？带宽/延迟量级？怎么验证？ | 03topo |
| 2 | NVSwitch 怎么把 8 卡织成全互联？ | 03topo |
| 3 | 8 卡服务器三种拓扑（串行/并行/HPC）的 trade-off？ | 03topo |
| 4 | 挑一台训大模型的机器看什么？（直连、CPU/GPU 亲和、IB 网卡插哪个 NUMA） | 03topo |
| 5 | NUMA 是什么？Linux 下怎么查设备 NUMA 亲和性？ | 03topo + rdma |
| 6 | AllReduce/AllGather/ReduceScatter/Broadcast 各自干嘛？Ring/Tree/Bruck 谁延迟好谁带宽好？ | 02nccl |
| 7 | NCCL 怎么利用实际拓扑建图？channel 为什么拆成一堆？ | 02nccl |
| 8 | Simple / LL / LL128 适用场景？LL128 为什么块更大？ | 02nccl |
| 9 | AllReduce 卡住/忽快忽慢，你从哪几条线查？ | 02nccl + rdma |
| 10 | 千卡规模有哪些踩坑经验？ | 02nccl |
| 11 | RDMA 绕开内核数据路径带来哪些好处？SEND/RECV vs READ/WRITE 用在哪？ | rdma |
| 12 | RDMA/IB/IBoE/RoCE/iWARP 什么关系？RoCE v1 vs v2 差异？ | rdma + roce |
| 13 | 为什么 RDMA 必须内存注册？注册 GPU 显存多哪些限制？ | rdma |
| 14 | PFC/ECN/DCQCN 怎么配合模拟无损？配错会怎样（incast/拥塞扩散）？ | roce |
| 15 | GPUDirect P2P / RDMA / GDS 分别解决什么？网卡怎么直接读写显存？门铃机制是什么？ | 05gpuDirect |
| 16 | 1B 参数多少显存？训练/推理显存各花在哪？MFU 怎么算、怎么判断是算力打不满还是网络拖后腿？ | 06trainAndInfer |

---

## 11. 三个坑（别踩）

1. **别把 RDMA 说成"我调过生产 RoCE"** —— 用"体系读过 + 本地实验 + 知道排查顺序"表述，这是加分不是减分。
2. **别在 CUDA 里陷进去** —— 你是平台岗不是 kernel 岗，概念层 + 一次 profiling 足够，写框架是负 ROI。
3. **别让这份仓库喧宾夺主** —— 你的主线仍是：DCGM 大盘 + Volcano 实操 + vLLM 压测 + NCCL/MFU 四份**产出物**，加上简历四段式改写。本仓库只负责让你在"拓扑 / 网络 / 通信"这三个追问点上不再露怯。

---

## 12. 本周只做一件事

打开 `03link/03topo/readme.md`（27K），配合机器上的 `nvidia-smi topo -m`，把实验锚点 #1 的**拓扑图**画出来。这是整个补丁包里最"看得见摸得着"的第一步——一周后你在面试里讲 NVLink/NVSwitch，就从这张图讲起。
