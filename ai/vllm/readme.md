# vLLM 原理、架构与面试指南

> 面向：LLM 推理 / AI Infra / 后端架构师 / 推理平台岗位  
> 重点：**架构 → 请求生命周期 → KV Cache → 调度 → Attention/Kernels → Prefix Cache → Speculative Decoding → 分布式 → 性能优化 → 面试**
>
> 本文以当前 vLLM **V1 架构**为主。V1 已成为主线架构，核心系统对 Scheduler、KV Cache Manager、Worker、Sampler、API Server 等进行了统一重构。

---

# 1. vLLM 是什么？

vLLM 是一个面向大语言模型推理与 Serving 的高性能推理引擎。

它解决的核心问题不是“如何把 Transformer 跑起来”，而是：

> **如何在有限 GPU 显存和计算资源下，同时服务大量不同长度、不同生命周期的请求，并获得高吞吐、低延迟和高 GPU 利用率。**

核心优化集中在：

- KV Cache 高效管理
- Continuous Batching
- PagedAttention
- Chunked Prefill
- Prefix Caching
- 高性能 Attention / GEMM Kernel
- CUDA Graph
- Quantization
- Speculative Decoding
- Tensor / Pipeline / Data / Expert / Context Parallel
- Prefill / Decode 分离
- KV Cache Offload / Connector

当前 vLLM 官方文档将 PagedAttention、Continuous Batching、Chunked Prefill、Prefix Caching、CUDA/HIP Graph、量化、高性能 Attention/GEMM Kernel、Speculative Decoding、P/D Disaggregation 等列为核心能力。

---

# 2. LLM 推理基础

## 2.1 Prefill 和 Decode

一次生成请求可以拆成两个阶段：

```text
Prompt
  │
  ▼
Prefill
  │
  │  一次处理大量输入 Token
  │  计算 Q/K/V
  │  建立 KV Cache
  ▼
Decode
  │
  │  每次生成 1 个 Token
  │  使用历史 KV Cache
  ▼
Token → Token → Token → ...
```

### Prefill

特点：

- 输入 Token 数量多
- 可以并行计算
- 计算密集型
- 主要影响 TTFT（Time To First Token）

### Decode

特点：

- 一次通常生成一个 Token
- 每一步都需要访问历史 KV Cache
- 更容易受显存带宽影响
- 主要影响 TPOT / ITL（Time Per Output Token）

因此：

```text
Prefill：Compute Bound 更明显
Decode ：Memory Bandwidth Bound 更明显
```

这是理解 vLLM 所有优化的基础。

---

# 3. Transformer 推理中的 KV Cache

对于 Attention：

```text
Attention(Q,K,V) = softmax(QKᵀ / √d)V
```

自回归生成时，历史 Token 的 K/V 不需要重复计算，因此保存：

```text
K1,V1
K2,V2
K3,V3
...
```

形成 KV Cache。

如果序列长度为 S：

```text
KV Cache ∝ S × Layers × KV Heads × Head Dimension
```

因此上下文越长：

- KV Cache 越大
- 单请求占用显存越多
- 能同时服务的请求越少

---

# 4. 为什么传统 KV Cache 管理效率低？

最简单的方法是：

```text
Request A → 连续显存
Request B → 连续显存
Request C → 连续显存
```

问题：

1. 每个请求长度未知
2. 请求不断进入/结束
3. 需要动态扩容
4. 容易产生内部碎片
5. 长请求可能导致大块连续显存难以分配
6. Batch 中请求长度差异巨大

例如：

```text
GPU Memory

[A A A A A A A A]
[B B B B]
[C C C C C C]
[D D]
```

请求结束以后：

```text
[A A A A A A A A]
[        free    ]
[C C C C C C]
[  free  ]
```

存在碎片。

---

# 5. PagedAttention

## 5.1 核心思想

PagedAttention 借鉴操作系统虚拟内存思想：

> **逻辑 KV Cache 与物理 GPU KV Cache 解耦。**

将 KV Cache 切成固定大小 Block：

```text
Logical Sequence

Token 0 ... 15   → Block 0
Token 16 ... 31  → Block 1
Token 32 ... 47  → Block 2
Token 48 ... 63  → Block 3
```

物理显存：

```text
Physical KV Cache

Block 7
Block 2
Block 19
Block 4
```

逻辑 Block 不要求连续映射到物理显存。

---

## 5.2 Block Table

核心结构：

```text
Request
   │
   ▼
Block Table

logical 0 → physical 7
logical 1 → physical 2
logical 2 → physical 19
logical 3 → physical 4
```

Attention Kernel 根据 Block Table 找到真正的 K/V。

---

## 5.3 PagedAttention 解决什么问题？

### ① 减少显存碎片

固定 Block 分配，不需要大块连续显存。

### ② 动态增长

请求生成新 Token：

```text
Block 0 → Block 1 → Block 2 → ...
```

需要多少分配多少。

### ③ 支持共享

多个请求可以指向同一个物理 Block：

```text
Request A ─┐
           ├──→ Block 10
Request B ─┘
```

这为 Prefix Cache 和 Copy-on-Write 等能力提供基础。

---

# 6. Continuous Batching

传统 Static Batching：

```text
Batch
 ├── Request A
 ├── Request B
 ├── Request C
 └── Request D

必须等待整个 Batch
```

如果：

```text
A → 10 tokens
B → 100 tokens
C → 20 tokens
D → 200 tokens
```

短请求结束后 GPU 仍然可能被长请求占据。

---

## 6.1 Continuous Batching

vLLM 不要求所有请求同时进入/退出。

```text
Step 1:
A B C

Step 2:
A B C D

Step 3:
A B D E

Step 4:
A D E F
```

每个 Engine Step：

1. Scheduler 查看等待队列
2. 查看运行中的请求
3. 查看 KV Cache 可用 Block
4. 决定本轮执行哪些 Token
5. 执行模型
6. 完成请求退出
7. 新请求进入

形成：

```text
Request Queue
      │
      ▼
 Scheduler
      │
      ▼
KV Cache Manager
      │
      ▼
Model Runner
      │
      ▼
GPU
```

---

# 7. vLLM V1 架构

当前 V1 可以从进程层理解：

```text
                    Client
                      │
                      ▼
              ┌───────────────┐
              │   API Server  │
              └───────┬───────┘
                      │ ZMQ
                      ▼
              ┌───────────────┐
              │ Engine Core   │
              │               │
              │ Scheduler     │
              │ KV Cache Mgmt │
              │ Request Mgmt  │
              └───────┬───────┘
                      │
             ┌────────┴────────┐
             ▼                 ▼
      GPU Worker 0       GPU Worker 1
             │                 │
             ▼                 ▼
        Model Runner       Model Runner
             │                 │
             └────────┬────────┘
                      ▼
                     GPU
```

官方 V1 架构中：

- API Server 负责 API / 请求接入
- Engine Core 运行 Scheduler、KV Cache 管理和执行协调
- GPU Worker 负责 GPU 模型执行
- Data Parallel 场景下，每个 DP rank 对应 Engine Core
- API Server 与 Engine Core 通过 ZMQ 通信

---

# 8. vLLM 核心模块

## 8.1 API Server

负责：

- OpenAI Compatible API
- 请求解析
- 参数校验
- Streaming
- 多模态输入
- 请求生命周期管理

典型入口：

```text
/v1/chat/completions
/v1/completions
/v1/embeddings
```

---

## 8.2 Engine Core

Engine Core 是 vLLM V1 的核心控制面。

主要负责：

```text
Request Management
        │
        ▼
Scheduler
        │
        ├── KV Cache Manager
        │
        ├── Prefix Cache
        │
        └── Speculative Decode Scheduling
        │
        ▼
Model Execution
```

Engine Core 持续运行调度循环：

```text
while running:

    receive requests

    schedule requests

    allocate KV blocks

    prepare model inputs

    dispatch GPU work

    collect outputs

    update request states
```

---

# 9. Scheduler

Scheduler 是 vLLM 最重要的模块之一。

它决定：

> **这一轮 GPU 到底执行哪些 Request、多少 Token。**

---

## 9.1 Scheduler 输入

主要考虑：

- Waiting Requests
- Running Requests
- KV Cache 剩余 Block
- Prompt 长度
- 已生成 Token 数
- 最大 Batch Token 数
- 最大并发 Request 数
- Chunked Prefill
- Prefix Cache 命中
- Speculative Decoding
- 请求优先级
- 资源约束

---

## 9.2 为什么 Scheduler 很重要？

GPU 的目标不是：

> 尽可能多放 Request

而是：

> **在显存、计算量和延迟约束下最大化整体吞吐。**

例如：

```text
Request A：Prompt 32K
Request B：Prompt 1K
Request C：Decode 1 token
Request D：Decode 1 token
```

如果一次全部 Prefill A：

```text
GPU 被 A 长时间占用
```

可能导致：

```text
B/C/D 延迟上升
```

因此需要 Chunked Prefill：

```text
A: 32K
↓
8K
↓
8K
↓
8K
↓
8K
```

同时穿插 Decode：

```text
Step 1: A-prefill + C-decode
Step 2: A-prefill + D-decode
Step 3: A-prefill + B
...
```

---

# 10. Chunked Prefill

长 Prompt 可以拆成多个 Chunk：

```text
32K Prompt

[0~8K]
[8K~16K]
[16K~24K]
[24K~32K]
```

好处：

- 避免长 Prefill 独占 GPU
- 降低 Decode 被阻塞的时间
- 提高整体 GPU 利用率
- 改善多租户场景下的延迟

代价：

- 调度复杂度增加
- KV Cache 管理复杂
- Prefill 被拆分后需要维护中间状态

---

# 11. Automatic Prefix Caching

如果两个请求：

```text
Request A:
System Prompt + Document + Question A

Request B:
System Prompt + Document + Question B
```

前缀：

```text
System Prompt + Document
```

完全相同。

第一次：

```text
Prefix
  ↓
Prefill
  ↓
KV Cache
```

第二次：

```text
Prefix
  ↓
直接复用 KV Cache
  ↓
只计算 Question B
```

官方称为 Automatic Prefix Caching（APC）。

---

# 12. Prefix Cache 如何判断前缀一样？

核心思想：

```text
Token IDs
   │
   ▼
Block
   │
   ▼
Hash
```

可以理解为：

```text
Block Hash =
Hash(
    Parent Block Hash,
    Token IDs,
    Block Metadata
)
```

形成链式 Hash：

```text
Block0
  │
  ▼
Hash0

Block1
  │
  ├── Hash0
  └── Token IDs
       ↓
     Hash1

Block2
  │
  ├── Hash1
  └── Token IDs
       ↓
     Hash2
```

这样：

```text
A:
System → Block1 → Block2 → QuestionA

B:
System → Block1 → Block2 → QuestionB
```

如果 Block1、Block2 Hash 相同：

```text
B 可以复用 A 的物理 KV Block
```

---

# 13. Prefix Cache 的重要边界

Prefix Cache 主要优化：

```text
Prefill
```

而不是：

```text
Decode
```

因此：

```text
TTFT ↓
Prefill Compute ↓
```

但：

```text
Output Token 生成速度
```

并不会因为 Prefix Cache 本身而直接提升。

如果请求：

```text
Prompt = 100K
Output = 1K
```

Prefix Cache 价值可能非常高。

如果：

```text
Prompt = 1K
Output = 100K
```

主要瓶颈在 Decode，Prefix Cache 收益就有限。

---

# 14. KV Cache Manager

KV Cache Manager 负责：

```text
Block Allocation
Block Free
Block Reuse
Prefix Cache
Reference Management
KV Cache Metadata
```

可以理解成：

```text
Scheduler
    │
    ▼
KVCacheManager
    │
    ├── Free Blocks
    ├── Allocated Blocks
    ├── Prefix Cache
    ├── Block Table
    └── Ref Count
```

---

# 15. KV Cache 生命周期

一个 Request：

```text
NEW
 │
 ▼
WAITING
 │
 ▼
RUNNING
 │
 ├── Prefill
 │
 ├── Decode
 │
 └── Decode
      │
      ▼
   FINISHED
      │
      ▼
Free KV Blocks
```

如果命中 Prefix Cache：

```text
WAITING
   │
   ▼
Prefix Match
   │
   ▼
Reuse KV Blocks
   │
   ▼
Only compute suffix
```

---

# 16. Attention Kernel

vLLM 不自己从头实现整个 Transformer。

真正影响 GPU 性能的核心之一是 Kernel。

主要包括：

- Attention
- GEMM
- MoE
- Quantization
- Sampling
- Communication

vLLM 会根据硬件、模型和场景选择不同 Attention Backend，例如：

```text
FlashAttention
FlashInfer
Triton
TRTLLM-GEN
FlashMLA
```

---

# 17. FlashAttention

普通 Attention：

```text
QKᵀ
 ↓
Attention Matrix
 ↓
Softmax
 ↓
× V
```

如果直接物化 Attention Matrix：

```text
O(S²)
```

显存访问压力很大。

FlashAttention 的核心思想：

> **利用 Tiling + SRAM/Shared Memory，减少 HBM 与中间结果的读写。**

概念：

```text
Q Block ──────┐
              ├──→ GPU SRAM
K Block ──────┘
              │
              ▼
          Online Softmax
              │
              ▼
              V
```

核心收益：

```text
减少 HBM IO
减少中间矩阵
提高 GPU Memory Locality
```

---

# 18. Decode Attention 为什么特殊？

Decode 时：

```text
Q = 1 Token
K/V = 历史所有 Token
```

所以：

```text
Q 很小
KV 很大
```

计算量并不一定特别大，但需要大量读取 KV Cache：

```text
GPU HBM
  │
  ├── K Cache
  ├── V Cache
  ├── K Cache
  └── V Cache
```

因此 Decode 常常更接近：

> **Memory Bandwidth Bound**

这也是为什么：

```text
KV Cache Layout
Block Size
Memory Coalescing
Kernel Fusion
```

非常重要。

---

# 19. CUDA Graph

普通 CUDA：

```text
CPU
 │
 ├── cuda kernel 1
 ├── cuda kernel 2
 ├── cuda kernel 3
 ├── cuda kernel 4
 └── ...
```

每次 Kernel Launch 都有 CPU/GPU 调度开销。

CUDA Graph：

```text
Capture
   │
   ▼
Kernel1 → Kernel2 → Kernel3 → Kernel4
   │
   ▼
Replay
```

减少：

- CPU Launch Overhead
- Kernel Launch Latency
- Host-side scheduling overhead

特别适合：

```text
Decode
Small Batch
Repeated Execution
```

但动态 Batch / 动态 Shape 会增加 Graph 管理复杂度。

因此 vLLM 使用 CUDA/HIP Graph、piecewise/full graph 等方式适配不同执行场景。

---

# 20. Quantization

量化主要减少：

```text
Model Weight Memory
KV Cache Memory
Memory Bandwidth
```

常见：

```text
FP16 / BF16
FP8
INT8
INT4
GPTQ
AWQ
MXFP
NVFP4
```

核心思想：

```text
FP16
  ↓
FP8 / INT8 / INT4
  ↓
Memory ↓
Bandwidth ↓
```

但代价可能是：

```text
Accuracy ↓
Kernel Complexity ↑
Dequantization Overhead
```

因此量化不是简单的：

> 精度越低越快

真正结果取决于：

```text
模型
硬件
Kernel
Batch Size
Memory Bandwidth
Compute Capability
```

---

# 21. Speculative Decoding

核心思想：

> 用一个便宜的小模型一次预测多个 Token，再让大模型验证。

传统：

```text
Large Model
   ↓
Token1
   ↓
Large Model
   ↓
Token2
   ↓
Large Model
```

Speculative：

```text
Small Draft Model
      ↓
T1 T2 T3 T4
      │
      ▼
Large Target Model
      │
      ▼
Verify
```

例如：

```text
Draft:
A B C D

Target:
A B C X
```

那么：

```text
A B C 接受
D 拒绝
```

下一轮继续。

---

# 22. Speculative Decoding 为什么可能加速？

传统：

```text
1 iteration → 1 token
```

Speculative：

```text
1 target forward
→ 验证多个 token
→ 接受多个 token
```

如果：

```text
平均每轮接受 3 tokens
```

理论上可以减少 Target Model 的执行轮数。

但收益依赖：

- Draft Model 质量
- Acceptance Rate
- Draft Model 成本
- Batch Size
- GPU 利用率
- KV Cache 开销

当前 vLLM 支持多种 speculative decoding 方法，包括 n-gram、suffix、EAGLE、DFlash 等。

---

# 23. Distributed Inference

单 GPU 放不下模型时，需要多 GPU。

常见并行方式：

```text
Tensor Parallel
Pipeline Parallel
Data Parallel
Expert Parallel
Context Parallel
```

---

## 23.1 Tensor Parallel

一个 Layer 的 Tensor 拆到多个 GPU：

```text
             Layer
               │
       ┌───────┼───────┐
       ▼       ▼       ▼
     GPU0     GPU1    GPU2
```

适合：

```text
模型太大
需要降低单 GPU Weight Memory
```

代价：

```text
GPU ↔ GPU Communication
```

通常依赖：

```text
NCCL
NVLink
InfiniBand
PCIe
```

---

# 24. Pipeline Parallel

按 Layer 切：

```text
GPU0:
Layer 0~20

GPU1:
Layer 21~40

GPU2:
Layer 41~60

GPU3:
Layer 61~80
```

优点：

```text
降低单 GPU 模型容量
```

问题：

```text
Pipeline Bubble
通信
调度复杂
```

---

# 25. Data Parallel

多个 Engine Replica：

```text
           Load Balancer
          /      |      \
         ▼       ▼       ▼
       GPU组A  GPU组B  GPU组C
```

每个 Replica 都有完整模型。

优点：

```text
吞吐线性扩展较容易
```

缺点：

```text
每个 Replica 都需要模型副本
```

适合：

```text
模型能够放入单个 GPU/TP Group
请求量很大
```

---

# 26. Expert Parallel

MoE 模型：

```text
Router
  │
  ├── Expert 0
  ├── Expert 1
  ├── Expert 2
  └── ...
```

Expert Parallel：

```text
GPU0 → Expert 0,1
GPU1 → Expert 2,3
GPU2 → Expert 4,5
GPU3 → Expert 6,7
```

核心问题：

```text
Token Routing
All-to-All
Load Balance
```

---

# 27. Prefill / Decode 分离

传统：

```text
GPU
 ├── Prefill
 └── Decode
```

P/D Disaggregation：

```text
Prefill Cluster
      │
      │ KV Cache Transfer
      ▼
Decode Cluster
```

好处：

- Prefill 和 Decode 分别扩容
- 计算资源隔离
- 更容易控制 TTFT / TPOT
- 适合生产级大规模 Serving

难点：

```text
KV Cache Transfer
Network Bandwidth
Scheduling
Failure Recovery
Cache Consistency
```

---

# 28. KV Cache Offload

GPU KV Cache 不够时，可以扩展到：

```text
GPU HBM
   ↓
CPU DRAM
   ↓
Remote KV Cache
   ↓
SSD
```

本质是：

> **把 KV Cache 从单机 GPU Memory 扩展成多级缓存系统。**

关键问题：

```text
Cache Hit
Cache Eviction
Bandwidth
Latency
Consistency
Placement
```

对于 Agent 场景尤其重要，因为：

```text
System Prompt
Tool Definitions
Conversation History
Long Context
```

往往高度复用。

---

# 29. Hybrid Attention / KV Cache

现代模型不一定所有 Layer 都是 Full Attention。

可能混合：

```text
Full Attention
Sliding Window Attention
Mamba / State Space
Sparse Attention
MLA
```

因此 KV Cache Manager 不能简单假设：

```text
所有 Layer 都使用相同 KV Cache
```

需要：

```text
KV Cache Group
Attention Type
Layer-specific Allocation
不同 Prefix Cache 规则
```

vLLM 当前已经有 Hybrid KV Cache Manager，用于支持不同 Attention 类型的混合模型。

---

# 30. vLLM 一次请求完整流程

假设：

```text
POST /v1/chat/completions
```

流程：

```text
Client
  │
  ▼
API Server
  │
  │ Tokenize
  ▼
Engine Core
  │
  ▼
Request Queue
  │
  ▼
Scheduler
  │
  ├── Prefix Cache Lookup
  │
  ├── KV Cache Allocation
  │
  └── Scheduling Decision
  │
  ▼
Model Runner
  │
  ▼
Attention / GEMM / MoE Kernel
  │
  ▼
Sampler
  │
  ▼
Next Token
  │
  ├── Stop?
  │     │
  │     ├── Yes → Finish
  │     └── No
  │
  ▼
Scheduler
  │
  ▼
Next Decode
```

---

# 31. 性能指标

面试必须区分：

## TTFT

Time To First Token

```text
Request
  ↓
First Token
```

主要受：

- Prefill
- Prefix Cache
- Queue Waiting
- Scheduler
- Model Size
- Input Length

影响。

---

## TPOT

Time Per Output Token

```text
Output Token 之间平均间隔
```

主要受：

- Decode
- KV Cache
- Attention Kernel
- Memory Bandwidth
- Batch Size

影响。

---

## ITL

Inter Token Latency

与 TPOT 类似，用于描述生成阶段 Token 间延迟。

---

## Throughput

常见：

```text
tokens / second
requests / second
```

生产系统通常需要同时看：

```text
TTFT
TPOT
Throughput
P50
P95
P99
GPU Utilization
KV Cache Utilization
Prefix Cache Hit Rate
```

---

# 32. 性能优化方法论

不要一上来就：

```text
加 GPU
```

应该按层定位。

```text
                    性能问题
                       │
          ┌────────────┼────────────┐
          ▼            ▼            ▼
        Queue         Model         GPU
          │            │            │
       Scheduler     Kernel       Memory
          │            │            │
       KV Cache      Attention     NCCL
          │            │            │
       Prefix        GEMM          PCIe
          │            │            │
       Batch         Quant         NVLink
```

---

# 33. TTFT 高怎么排查？

```text
TTFT 高
 │
 ├── Queue Time 高？
 │
 ├── Prompt 太长？
 │
 ├── Prefix Cache 命中率低？
 │
 ├── Prefill Batch 太大？
 │
 ├── Chunked Prefill 配置？
 │
 ├── GPU Compute 不足？
 │
 └── Scheduler 被长请求阻塞？
```

---

# 34. TPOT 高怎么排查？

```text
TPOT 高
 │
 ├── Decode Batch 太大？
 ├── KV Cache 太大？
 ├── Memory Bandwidth 不够？
 ├── Attention Kernel 不合适？
 ├── CUDA Graph 是否生效？
 ├── Quantization？
 ├── GPU 利用率？
 └── Multi-GPU Communication？
```

---

# 35. GPU Utilization 低怎么办？

不要直接认为：

> GPU 不够快。

先区分：

```text
CPU Bound
GPU Kernel Launch Bound
Memory Bound
Compute Bound
Communication Bound
Scheduler Bound
```

例如：

```text
GPU Util 低
+
CPU 很忙
=
可能 Host / Scheduler / Launch Overhead

GPU Util 低
+
HBM Bandwidth 接近上限
=
可能 Memory Bound

GPU Util 高
+
Latency 高
=
可能 Batch / Model / Kernel 本身成为瓶颈
```

---

# 36. vLLM 为什么吞吐高？

可以总结成四层：

## 第一层：Memory Management

```text
PagedAttention
KV Cache Manager
Prefix Cache
```

解决：

```text
显存碎片
KV Cache 浪费
重复计算
```

## 第二层：Scheduling

```text
Continuous Batching
Chunked Prefill
```

解决：

```text
GPU 空转
长请求阻塞
```

## 第三层：Kernel

```text
FlashAttention
FlashInfer
Triton
CUDA Graph
GEMM/MoE Kernel
```

解决：

```text
GPU 执行效率
Memory IO
Launch Overhead
```

## 第四层：Model / System Scaling

```text
Quantization
Speculative Decoding
Tensor Parallel
Pipeline Parallel
Data Parallel
Expert Parallel
P/D Disaggregation
KV Offload
```

解决：

```text
模型规模
吞吐
延迟
容量
```

---

# 37. vLLM 与 HuggingFace Transformers 的区别

| 维度 | Transformers | vLLM |
|---|---|---|
| 定位 | 模型框架 | 推理 Serving Engine |
| 单请求推理 | 强 | 强 |
| 批量 Serving | 基础 | 核心能力 |
| Continuous Batching | 非核心 | 核心 |
| PagedAttention | 无 | 核心 |
| Prefix Cache | 有相关能力，但不是核心架构 | 核心 |
| KV Cache 管理 | 相对基础 | 高度优化 |
| API Server | 非核心 | OpenAI Compatible |
| GPU Kernel | 依赖 PyTorch/生态 | 大量专用优化 |
| 分布式 Serving | 可组合 | 原生支持多种并行 |
| 生产 Serving | 可用 | 核心定位 |

一句话：

> Transformers 更偏“模型开发/运行框架”，vLLM 更偏“高性能 LLM Serving Engine”。

---

# 38. vLLM 与 TensorRT-LLM 的区别

可以从工程定位理解：

```text
vLLM
  ↓
灵活 + 快速迭代 + Serving Ecosystem

TensorRT-LLM
  ↓
NVIDIA 深度优化 + Kernel/Graph/Runtime
```

实际生产中两者都属于高性能 LLM Inference Stack。

面试不要简单回答：

> 谁更快。

应该回答：

> 性能取决于模型、GPU、Batch、上下文长度、量化方式、并行方式和 Kernel；需要针对具体 workload benchmark。

---

# 39. vLLM 源码阅读路线

如果目标是面试和架构理解，不建议从几百万行源码直接开始。

推荐：

```text
第一阶段：整体架构
    ↓
第二阶段：Request 生命周期
    ↓
第三阶段：Scheduler
    ↓
第四阶段：KV Cache
    ↓
第五阶段：Model Runner
    ↓
第六阶段：Attention Kernel
    ↓
第七阶段：Distributed
```

---

## 39.1 第一阶段

先理解：

```text
API Server
Engine Core
Scheduler
KV Cache Manager
GPU Worker
Model Runner
```

重点回答：

> 一个请求是怎么从 HTTP 进入 GPU 的？

---

## 39.2 第二阶段：Request

重点：

```text
Request
Request Status
Waiting Queue
Running Queue
Sampling Params
Output
```

画出：

```text
NEW
 ↓
WAITING
 ↓
RUNNING
 ↓
FINISHED
```

---

## 39.3 第三阶段：Scheduler

重点阅读：

```text
vllm/v1/core/sched/
```

重点理解：

```text
schedule()
waiting
running
num_computed_tokens
num_new_tokens
KV Cache capacity
Chunked Prefill
```

---

## 39.4 第四阶段：KV Cache

重点：

```text
vllm/v1/core/kv_cache_manager
vllm/v1/kv_cache_interface
```

重点理解：

```text
Block
Block Pool
Block Table
KV Cache Spec
Prefix Cache
Allocation
Free
Reference
```

---

## 39.5 第五阶段：Model Runner

重点理解：

```text
Input Preparation
Model Forward
Attention
Sampling
CUDA Graph
```

核心问题：

> Scheduler 决定了执行什么，那么 Model Runner 是如何把 Scheduler 的结果变成 GPU Tensor 的？

---

# 40. 面试核心问题

## Q1：vLLM 为什么快？

标准回答：

> vLLM 的核心优势不是单一 Kernel，而是围绕 LLM Serving 对整个推理链路进行优化。核心包括 PagedAttention 对 KV Cache 的高效管理、Continuous Batching 提高 GPU 利用率、Chunked Prefill 改善长 Prompt 与 Decode 的调度、Prefix Caching 避免重复 Prefill，以及 FlashAttention/FlashInfer、CUDA Graph、量化、Speculative Decoding 和多 GPU 并行等底层优化。

---

## Q2：PagedAttention 解决什么问题？

回答：

> 它把逻辑 KV Cache 与物理显存解耦，把 KV Cache 切成固定 Block，通过 Block Table 建立映射。这样请求可以按需增长，不要求物理显存连续，可以显著降低碎片和 KV Cache 浪费，同时为 Prefix Cache 等共享机制提供基础。

---

## Q3：Continuous Batching 和 Static Batching 区别？

回答：

> Static Batching 要等一批请求组成后统一执行，容易出现短请求等待长请求的问题。Continuous Batching 在每个 Engine Step 动态决定哪些请求执行，并允许请求随时加入和退出，更适合生成长度不可预测的 LLM Serving。

---

## Q4：为什么 Decode 比 Prefill 更容易 Memory Bound？

回答：

> Decode 每轮通常只计算一个 Query Token，但需要读取整个历史 KV Cache。计算量相对有限，而 KV Cache 的 HBM 读取量很大，所以容易受显存带宽限制。

---

## Q5：Prefix Cache 怎么实现？

回答：

> 将 Prompt 切成 Cache Block，对 Block 及其上下文信息计算 Hash，通过 Hash 找到已经存在的物理 KV Block。命中后直接复用 KV Cache，只计算未命中的 suffix。

---

## Q6：Prefix Cache 是否提升 Decode？

回答：

> 它主要减少 Prefill 阶段的重复计算，因此主要降低 TTFT。对于输出很长、主要时间消耗在 Decode 的 workload，收益有限。

---

## Q7：Chunked Prefill 为什么存在？

回答：

> 长 Prompt 的 Prefill 如果一次执行，可能长时间占用 GPU，使 Decode 请求等待。Chunked Prefill 将长 Prompt 切成多个 Chunk，在 Prefill 和 Decode 之间进行调度，从而改善整体延迟和 GPU 利用率。

---

## Q8：CUDA Graph 为什么能提升性能？

回答：

> 它把一组 Kernel 的执行图捕获下来，之后可以重复 Replay，减少 CPU 到 GPU 的 Kernel Launch 和调度开销。对于 Decode 这种大量重复、小 Kernel 的场景尤其有价值。

---

## Q9：Tensor Parallel 的瓶颈是什么？

回答：

> Tensor Parallel 可以把单层计算拆到多个 GPU，但会引入 GPU 间通信。GPU 数量增加以后，计算时间下降，而通信成本可能成为主要瓶颈，因此需要关注 NCCL、NVLink、PCIe、InfiniBand 以及通信与计算的重叠。

---

## Q10：如何提升 vLLM 的吞吐？

回答框架：

```text
1. Batch
   Continuous Batching
   Chunked Prefill

2. KV Cache
   PagedAttention
   Prefix Cache

3. Kernel
   FlashAttention / FlashInfer
   CUDA Graph
   Kernel Fusion

4. Model
   Quantization
   Speculative Decoding

5. Parallel
   TP / PP / DP / EP / CP

6. Architecture
   P/D Disaggregation
   KV Cache Offload
```

---

# 41. 高阶面试题

## Q11：如果 GPU 利用率 30%，如何排查？

建议回答：

```text
第一步：看 CPU / GPU / HBM / PCIe / NCCL

第二步：确认是：
CPU Bound
Memory Bound
Compute Bound
Communication Bound
Scheduler Bound

第三步：
看 Request Queue
看 Batch Size
看 Decode Batch
看 Kernel Launch
看 CUDA Graph
看 KV Cache
看 Prefix Hit Rate

第四步：
用 Nsight / PyTorch Profiler / vLLM Metrics 验证
```

---

## Q12：如果 P99 TTFT 很高？

拆解：

```text
TTFT
 =
Queue Time
+
Scheduler Time
+
Prefix Lookup
+
Prefill
+
First Decode
```

重点排查：

```text
Queue 是否堆积
长 Prompt 是否阻塞
Chunked Prefill
Prefix Cache Hit
GPU Saturation
Scheduler Policy
```

---

## Q13：为什么增加 GPU 后性能没有线性提升？

可能：

```text
Communication Bound
NCCL
PCIe
NVLink
Pipeline Bubble
Load Imbalance
KV Cache Bottleneck
Scheduler Bottleneck
```

所以：

```text
GPU 数量 ↑
≠
吞吐线性 ↑
```

---

# 42. Agent 场景下为什么 vLLM 特别重要？

Agent workload 有明显特点：

```text
System Prompt
+
Tool Definitions
+
Conversation History
+
Retrieved Context
+
Tool Result
+
New User Input
```

大量内容会重复。

例如：

```text
System Prompt      5K
Tool Definitions   8K
History            20K
RAG Context        30K
Question           1K
```

下一轮：

```text
System Prompt      5K   ← 重复
Tool Definitions   8K   ← 重复
History            20K  ← 重复
RAG Context        30K  ← 部分重复
Question           1K   ← 新增
```

如果 Prefix Cache 命中：

```text
70K
 ↓
Reuse KV
 ↓
只 Prefill 新增部分
```

这使得：

> **KV Cache 不再只是 GPU 内部的优化，而开始成为 Agent Infrastructure 的核心资源。**

---

# 43. 从系统架构角度理解 vLLM

可以把 vLLM 看成：

```text
             LLM Serving System
                     │
        ┌────────────┼────────────┐
        ▼            ▼            ▼
    Scheduler      Cache       Compute
        │            │            │
        │            │            ├── Attention
        │            │            ├── GEMM
        │            │            ├── MoE
        │            │            └── CUDA Graph
        │            │
        │            ├── KV Cache
        │            ├── Prefix Cache
        │            └── Block Manager
        │
        ├── Continuous Batching
        ├── Chunked Prefill
        └── Spec Decode
```

本质是：

> **Scheduler + Memory Manager + GPU Runtime + Distributed Runtime**

---

# 44. 面试时建议重点掌握的 10 个知识点

按优先级：

```text
★★★★★ 1. PagedAttention
★★★★★ 2. KV Cache
★★★★★ 3. Continuous Batching
★★★★★ 4. Scheduler
★★★★★ 5. Prefix Cache
★★★★★ 6. Prefill / Decode
★★★★☆ 7. FlashAttention / Kernel
★★★★☆ 8. CUDA Graph
★★★★☆ 9. Tensor Parallel
★★★★☆ 10. Speculative Decoding
```

进一步深入：

```text
Chunked Prefill
Quantization
Expert Parallel
Pipeline Parallel
P/D Disaggregation
KV Offload
Hybrid KV Cache
```

---

# 45. 最终知识图谱

```text
vLLM
│
├── 1. Serving
│   ├── OpenAI API
│   ├── Streaming
│   └── Request Management
│
├── 2. Scheduling
│   ├── Continuous Batching
│   ├── Chunked Prefill
│   ├── Decode Scheduling
│   └── Speculative Scheduling
│
├── 3. Memory
│   ├── KV Cache
│   ├── PagedAttention
│   ├── Block Table
│   ├── Prefix Cache
│   ├── KV Offload
│   └── Hybrid KV Cache
│
├── 4. Compute
│   ├── Attention
│   ├── FlashAttention
│   ├── FlashInfer
│   ├── GEMM
│   ├── MoE
│   ├── CUDA Graph
│   └── torch.compile
│
├── 5. Model Optimization
│   ├── Quantization
│   └── Speculative Decoding
│
├── 6. Distributed
│   ├── Tensor Parallel
│   ├── Pipeline Parallel
│   ├── Data Parallel
│   ├── Expert Parallel
│   └── Context Parallel
│
└── 7. Large Scale Serving
    ├── P/D Disaggregation
    ├── KV Cache Connector
    ├── Remote KV Cache
    └── Multi-node Serving
```

---

# 46. 一句话总结

如果面试官问：

> **你怎么理解 vLLM？**

可以回答：

> **vLLM 本质上是一个面向 LLM Serving 的高性能 Runtime，它围绕“请求调度、KV Cache 管理和 GPU 执行”三个核心问题展开。Scheduler 通过 Continuous Batching 和 Chunked Prefill 管理动态请求；KV Cache Manager 通过 PagedAttention、Block 管理和 Prefix Cache 提高显存利用率并减少重复计算；Model Runner 再通过 FlashAttention/FlashInfer、CUDA Graph、量化以及各种 GPU Kernel 提高单次执行效率，最后通过 TP/PP/DP/EP、P/D Disaggregation 和 KV Cache Offload 扩展到大规模分布式 Serving。**

---

# 47. 推荐源码阅读顺序

```text
vllm/
│
├── entrypoints/
│       ↓
│   API Server
│
├── v1/engine/
│       ↓
│   Engine Core
│
├── v1/core/sched/
│       ↓
│   Scheduler
│
├── v1/core/kv_cache_manager/
│       ↓
│   KV Cache
│
├── v1/worker/
│       ↓
│   Worker / Model Runner
│
├── attention/
│       ↓
│   Attention Backend
│
├── model_executor/
│       ↓
│   Model Execution
│
└── distributed/
        ↓
    TP / PP / DP / EP
```

建议实际阅读顺序：

```text
Architecture
  ↓
Request
  ↓
Scheduler
  ↓
KV Cache
  ↓
Model Runner
  ↓
Attention
  ↓
CUDA Graph
  ↓
Distributed
```

---

# 48. 官方资料

- vLLM Documentation: https://docs.vllm.ai/
- Architecture Overview: https://docs.vllm.ai/en/latest/design/arch_overview/
- V1 Guide: https://github.com/vllm-project/vllm/blob/main/docs/usage/v1_guide.md
- Automatic Prefix Caching: https://docs.vllm.ai/en/stable/features/automatic_prefix_caching/
- vLLM GitHub: https://github.com/vllm-project/vllm
- Hybrid KV Cache Manager: https://github.com/vllm-project/vllm/blob/main/docs/design/hybrid_kv_cache_manager.md
