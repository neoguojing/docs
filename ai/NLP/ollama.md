# Ollama 与 llama.cpp：架构、重点技术与算法面试文档

> 面向：LLM 推理 / AI Infra / 后端架构师 / 高级工程师面试  
> 重点：**架构、推理链路、模型加载、量化、KV Cache、批处理、并发、GPU/CPU、源码阅读、性能优化**  
> 核心关系：**llama.cpp 更偏底层推理运行时；Ollama 更偏“模型管理 + 本地推理服务 + 开发体验”的上层封装。**

---

# 1. 一句话定位

| 项目 | 定位 | 核心价值 |
|---|---|---|
| **llama.cpp** | C/C++ LLM 推理运行时 | 在 CPU/GPU/Apple Silicon 等多种硬件上高效运行量化模型 |
| **Ollama** | 本地 LLM 运行与管理平台 | 模型下载、管理、配置、运行、API 服务、生命周期管理 |
| **GGUF** | 模型文件格式 | 将模型权重、Tokenizer、量化信息、架构元数据等统一封装 |
| **GGML** | 张量计算/后端基础设施 | 提供 tensor、算子、计算图、CPU/GPU backend 等能力 |

llama.cpp 的目标是以较少依赖在广泛硬件上进行高性能 LLM/VLM inference；当前支持多种量化格式以及 CUDA、HIP、Metal、Vulkan、SYCL 等后端。citeturn0search5

---

# 2. 两者的关系

最容易被问：

> **Ollama 和 llama.cpp 是什么关系？**

可以理解为：

```text
                    Ollama
        ┌───────────────────────────┐
        │ Model Registry / Pull      │
        │ Modelfile / Manifest       │
        │ API / Chat                 │
        │ Model lifecycle            │
        │ Process / Runner Manager   │
        └─────────────┬─────────────┘
                      │
                Runner / Runtime
                      │
        ┌─────────────▼─────────────┐
        │      llama.cpp / GGML      │
        │ Model loading              │
        │ Tensor / Graph             │
        │ Quantization               │
        │ KV Cache                   │
        │ Sampling                   │
        │ CPU / CUDA / Metal ...     │
        └─────────────┬─────────────┘
                      │
                Hardware Backend
       CPU / NVIDIA / Apple / AMD / ...
```

**面试表达：**

> llama.cpp 解决的是“模型如何高效执行”；Ollama 解决的是“用户如何方便地管理和运行模型”。两者关注层级不同。

注意：不要简单说成“新版 Ollama 就是 llama.cpp 的 HTTP 包装器”。Ollama 自身还有模型管理、runner 生命周期、API、配置和资源管理等逻辑；具体 runner 实现也会随版本演进。

---

# 3. llama.cpp 总体架构

```text
                    Application
                         │
                  llama-server / CLI
                         │
                  ┌──────▼──────┐
                  │ llama API   │
                  └──────┬──────┘
                         │
             ┌───────────▼───────────┐
             │      llama runtime     │
             │ context / sequence     │
             │ KV cache / sampler     │
             │ model / tokenizer      │
             └───────────┬───────────┘
                         │
                  ┌──────▼──────┐
                  │    GGML     │
                  │ tensor/graph│
                  │ operators   │
                  └──────┬──────┘
                         │
        ┌────────────────┼────────────────┐
        ▼                ▼                ▼
       CPU             CUDA             Metal
      AVX/AMX          GPU             Apple GPU
```

当前 llama-server 的服务端结构包括 `server_context`、`server_slot`、路由、HTTP、任务队列和响应队列；并支持并行解码、continuous batching、speculative decoding 等能力。citeturn0search0turn0search1

---

# 4. llama.cpp 核心模块

## 4.1 Model

负责：

- GGUF 解析
- tensor metadata
- 权重加载
- tokenizer
- architecture detection
- quantization information
- tensor 到 backend 的映射

典型关系：

```text
GGUF
 │
 ├── metadata
 │    ├── architecture
 │    ├── tokenizer
 │    ├── context length
 │    └── quantization
 │
 └── tensors
      ├── embedding
      ├── attention
      ├── FFN
      └── norm
```

---

## 4.2 Context

Context 是一次推理运行的核心状态。

可以理解为：

```text
llama_context
 ├── model
 ├── KV cache
 ├── computation state
 ├── sequence state
 ├── backend buffers
 └── decoding state
```

Context 不等于“整个模型”。

**模型权重可以共享；context 通常包含请求相关状态。**

---

## 4.3 Sequence / Slot

多用户并发时：

```text
Model
  │
  └── llama context
        ├── sequence A
        │     └── KV
        ├── sequence B
        │     └── KV
        └── sequence C
              └── KV
```

server 中可以把一个 slot 理解成一个并行请求/sequence 的管理抽象。官方 server 开发文档也明确把 `server_slot` 描述为对应 llama.cpp 中单个 sequence 的抽象。citeturn0search0

---

# 5. 一次完整推理流程

```text
HTTP Request
    │
    ▼
Chat Template
    │
    ▼
Tokenizer
    │
    ▼
Token IDs
    │
    ▼
Prompt Cache / KV Cache
    │
    ▼
Prefill
    │
    ▼
Transformer Layers
    │
    ├── RMSNorm
    ├── QKV Projection
    ├── RoPE
    ├── Attention
    ├── Residual
    ├── FFN / MoE
    └── Residual
    │
    ▼
Logits
    │
    ▼
Sampling
    │
    ▼
Next Token
    │
    └──────► repeat Decode
```

最重要的性能分界：

```text
Prompt
  ↓
Prefill  ── 计算密集 / 高并行
  ↓
Decode   ── 单 token / 内存带宽 + KV 访问敏感
```

---

# 6. Prefill vs Decode

## 6.1 Prefill

假设：

```text
prompt = 4096 tokens
```

Prefill 可以一次处理大量 token：

```text
4096 tokens
    │
    ▼
large matrix multiplication
    │
    ▼
GPU utilization 高
```

因此通常：

- 更适合 GPU
- batch 大
- GEMM 利用率高
- FLOPS 更重要

---

## 6.2 Decode

生成阶段：

```text
t1 → t2 → t3 → t4 → ...
```

每次主要生成一个 token：

```text
1 token
  ↓
所有 Transformer layers
  ↓
1 token
```

因此更容易受：

- memory bandwidth
- KV cache
- kernel launch
- synchronization
- batch scheduling

影响。

**经典面试题：**

> 为什么 LLM decode 阶段通常比 prefill 更难把 GPU 算力吃满？

答：

> 因为 decode 每一步新增 token 很少，矩阵规模小，计算并行度低，同时每层仍需要读取大量模型权重和 KV cache，因此容易从 compute-bound 转向 memory-bound。

---

# 7. GGUF

GGUF 是 llama.cpp 生态的重要模型文件格式。

逻辑：

```text
GGUF
├── Header
├── Metadata
│   ├── model architecture
│   ├── tokenizer
│   ├── rope parameters
│   └── quantization info
└── Tensor Data
    ├── weights
    ├── scales
    └── quantized blocks
```

优势：

- 单文件分发
- metadata 完整
- mmap 友好
- 支持量化
- 支持多种模型架构
- tokenizer 与模型信息一起携带

---

# 8. mmap：为什么模型启动可以很快？

典型：

```text
GGUF file
   │
 mmap
   ▼
Virtual Memory
   │
   ├── page cache
   ├── demand paging
   └── physical memory
```

mmap 不意味着：

> “整个模型瞬间加载到 RAM。”

更准确：

> 建立虚拟地址空间映射，实际物理页按访问需求加载。

优点：

- 减少显式 memcpy
- OS page cache 可复用
- 多进程共享只读物理页
- 大模型加载体验较好

缺点：

- 首次访问可能产生 page fault
- 随机访问会造成 IO 抖动
- 内存压力下可能产生回收

llama-server 当前也提供 mmap / mlock 等模型加载相关选项。citeturn0search1

---

# 9. Quantization：核心技术

量化的目标：

```text
FP16
 ↓
INT8 / Q8
 ↓
INT6 / Q6
 ↓
INT5 / Q5
 ↓
INT4 / Q4
```

降低：

- 模型体积
- 内存占用
- memory bandwidth

代价：

- 精度损失
- 量化/反量化开销
- 某些层对量化敏感

llama.cpp 当前支持从 1.5-bit 到 8-bit 等多种整数/低比特量化方案。citeturn0search5

---

# 10. 为什么量化可以提升 Decode 性能？

Decode 常常受 memory bandwidth 限制。

假设：

```text
模型权重 = 14 GB
memory bandwidth = 500 GB/s
```

每生成一个 token，大量权重需要参与计算。

如果：

```text
FP16 → 14 GB
Q4   → 4 GB
```

理论上需要搬运的数据量显著下降。

所以：

```text
量化
 ↓
模型更小
 ↓
memory traffic ↓
 ↓
cache / bandwidth 压力 ↓
 ↓
decode TPS ↑
```

但不是简单的：

> 4bit = 4 倍速度。

因为实际性能还受到：

- dequant kernel
- tensor shape
- GPU architecture
- batch size
- KV cache
- backend
- kernel implementation

影响。

---

# 11. Quantization 的核心原理

简单线性量化：

```text
x ≈ scale × q
```

例如：

```text
q = round(x / scale)
```

反量化：

```text
x ≈ q × scale
```

实际 LLM 量化通常以 block/group 为单位：

```text
weights
┌─────────────────────┐
│ block 1              │ → scale + quantized values
├─────────────────────┤
│ block 2              │ → scale + quantized values
├─────────────────────┤
│ block 3              │
└─────────────────────┘
```

这样可以在压缩率与精度之间取得平衡。

---

# 12. Attention

标准：

```text
Q = XWq
K = XWk
V = XWv

Attention(Q,K,V)
 = softmax(QKᵀ / √d)V
```

推理阶段最重要的问题不是公式，而是：

> **K/V 如何保存？**

答案就是 KV Cache。

---

# 13. KV Cache

如果每次生成 token 都重新计算历史 token：

```text
token1
token1 + token2
token1 + token2 + token3
...
```

计算量会快速增加。

KV Cache：

```text
Token 1 → K1 V1
Token 2 → K2 V2
Token 3 → K3 V3
```

下一次：

```text
Q_new
  │
  ├── K_cache
  └── V_cache
       ↓
Attention
```

只计算新 token 的 Q/K/V，再复用历史 K/V。

---

# 14. KV Cache 内存估算

近似：

```text
KV ≈
2 × layers × seq_len × kv_heads × head_dim × bytes
```

例如：

```text
layers = 32
seq = 8192
KV heads = 8
head_dim = 128
FP16 = 2 bytes
```

则：

```text
2 × 32 × 8192 × 8 × 128 × 2
≈ 1.07 GB
```

所以：

> **长上下文、多并发时，KV Cache 很容易成为显存瓶颈。**

---

# 15. KV Cache 与 GQA

传统 MHA：

```text
Q heads = 32
K heads = 32
V heads = 32
```

GQA：

```text
Q heads = 32
K heads = 8
V heads = 8
```

因此：

```text
KV Cache ↓ 4x
```

这也是现代 LLM 大量采用 GQA/MQA 的重要原因。

---

# 16. Prompt Cache / Prefix Cache

这是 Ollama / llama.cpp 面试很容易问的点。

如果：

```text
Request A:
SYSTEM + long_prompt + question_A

Request B:
SYSTEM + long_prompt + question_B
```

前缀相同：

```text
SYSTEM + long_prompt
^^^^^^^^^^^^^^^^^^^^^^
      common prefix
```

可以复用：

```text
tokens
   ↓
prefix match
   ↓
已有 KV state
   ↓
只计算新 suffix
```

llama.cpp server 的 prompt cache 会比较之前 completion 的 prompt，并在 `cache_prompt` 开启时只处理未见过的 suffix；服务端也提供 idle slot / unified KV 等缓存相关能力。citeturn0search1turn0search7

---

# 17. 前缀为什么能判断“相同”？

不要说：

> “比较字符串。”

真正关键是：

```text
raw prompt
 ↓
chat template
 ↓
tokenizer
 ↓
token IDs
 ↓
prefix token sequence comparison
```

例如：

```text
A = [101, 203, 55, 90, 18, 9]
B = [101, 203, 55, 90, 77, 4]

common prefix =

[101, 203, 55, 90]
```

因此：

> **缓存命中通常发生在 tokenized representation / execution state 层，而不是原始字符串层。**

这是非常重要的面试答案。

---

# 18. Continuous Batching

传统：

```text
Request A
████████████████

Request B
                ███████████████
```

必须等待整个 batch。

Continuous batching：

```text
Time →
A: █████████████████
B:    █████████████
C:       ███████████████
D:          █████████
```

调度器每一步重新决定：

```text
哪些 sequence 需要 decode？
哪些 sequence 已完成？
哪些新请求加入？
```

目标：

```text
GPU utilization ↑
throughput ↑
latency ↓
```

llama-server 当前明确支持 continuous batching 和多用户并行解码。citeturn0search1

---

# 19. Batch 与 Micro-batch

可以理解成：

```text
Request Queue
     │
     ▼
Scheduler
     │
     ▼
Batch
 ┌────┬────┬────┬────┐
 A    B    C    D
 └────┴────┴────┴────┘
     │
     ▼
GPU
```

需要平衡：

```text
batch ↑
→ throughput ↑

batch ↑
→ latency / memory pressure ↑
```

所以服务端核心问题之一是：

> 在吞吐、TTFT、TPOT、显存之间寻找平衡。

---

# 20. TTFT / TPOT

面试建议熟练：

### TTFT

Time To First Token：

```text
request
 ↓
tokenize
 ↓
prefill
 ↓
first token
```

反映：

- 排队
- 模型加载
- prefill
- prompt cache

---

### TPOT

Time Per Output Token：

```text
token1
 ↓
token2
 ↓
token3
```

反映：

- decode
- memory bandwidth
- KV cache
- sampling
- scheduling

---

# 21. Speculative Decoding

核心思想：

```text
Small Draft Model
       │
       ▼
t1 t2 t3 t4 t5
       │
       ▼
Large Target Model
       │
       ▼
一次验证多个 token
```

如果 draft 预测正确：

```text
accepted tokens ↑
target model calls ↓
```

llama.cpp 当前支持多种 speculative decoding，包括 draft model、EAGLE-3、MTP、n-gram 等实现。citeturn0search2

---

# 22. Speculative Decoding 为什么有效？

普通：

```text
Target
  ↓
token1
  ↓
Target
  ↓
token2
  ↓
Target
  ↓
token3
```

Speculative：

```text
Draft
 ↓
t1 t2 t3 t4

Target
 ↓
一次验证 t1~t4
```

因为：

> GPU 更擅长一次计算多个 token 的 batch。

因此 speculative decoding 的本质是：

> **用便宜模型产生候选 token，再用大模型并行验证，把 autoregressive decoding 转化为更高效的批量计算。**

---

# 23. Sampling

模型最终输出：

```text
logits
  ↓
temperature
  ↓
top-k
  ↓
top-p
  ↓
repetition penalty
  ↓
sample
  ↓
token
```

### Temperature

```text
P_i = softmax(logit_i / T)
```

- T < 1：更集中
- T > 1：更随机

### Top-K

只保留概率最高的 K 个 token。

### Top-P

保留累计概率达到 P 的 token 集合。

---

# 24. RoPE

Rotary Positional Embedding。

核心思想：

> 不直接把 position embedding 加到 token embedding，而是在 Q/K 空间做旋转。

二维形式：

```text
[x1]
[x2]

        rotate(theta)

[x1']
[x2']
```

位置变化：

```text
position
   ↓
rotation angle
   ↓
Q/K
```

好处：

- 相对位置信息自然进入 attention
- 不需要显式 position embedding table
- 支持一定的 context extension 技术

面试不要只背公式，要能解释：

> RoPE 的核心是让 Q/K 的内积显式依赖 token 的相对位置。

---

# 25. GGML 的核心思想

GGML 可以理解成：

> **面向推理的 tensor + graph + backend abstraction。**

核心对象：

```text
Tensor
 ├── shape
 ├── dtype
 ├── data
 ├── strides
 └── backend buffer
```

计算：

```text
Tensor A
   │
   ├── MatMul
   │
Tensor B
   │
   ▼
Tensor C
```

进一步形成：

```text
Computation Graph

Input
  ↓
MatMul
  ↓
RoPE
  ↓
Attention
  ↓
FFN
  ↓
Output
```

---

# 26. 为什么需要 Computation Graph？

因为可以提前知道：

```text
有哪些 op
↓
tensor 依赖
↓
内存需求
↓
backend
↓
执行顺序
```

有利于：

- memory planning
- graph optimization
- backend dispatch
- 减少动态分配
- CPU/GPU execution

---

# 27. Backend Abstraction

核心思想：

```text
GGML Operator
      │
      ▼
Backend
 ┌────┼────┬────┬────┐
CPU  CUDA Metal Vulkan ...
```

上层不应该关心：

```text
CUDA kernel
Metal shader
AVX instruction
```

而是：

```text
ggml_mul_mat(...)
```

然后由 backend 执行。

这也是 llama.cpp 可以支持多硬件的重要原因。

---

# 28. CPU 优化

CPU 推理主要依赖：

```text
SIMD
 ├── AVX
 ├── AVX2
 ├── AVX512
 └── AMX

Threading
Cache locality
Memory bandwidth
```

典型：

```text
Quantized weights
       ↓
SIMD load
       ↓
dequant / dot product
       ↓
accumulate
```

llama.cpp 当前针对 x86 的 AVX/AVX2/AVX512/AMX，以及 Apple Silicon 的 NEON/Accelerate/Metal 等进行了优化。citeturn0search5

---

# 29. GPU Offload

如果模型：

```text
VRAM < Model Size
```

可以：

```text
CPU
 ├── Layer 0
 ├── Layer 1
 └── Layer 2

GPU
 ├── Layer 3
 ├── ...
 └── Layer N
```

形成：

```text
CPU + GPU Hybrid Inference
```

llama.cpp 官方 README 明确支持 CPU+GPU hybrid inference，使模型可以部分大于总 VRAM 容量时仍运行。citeturn0search5

---

# 30. Ollama 架构

可以抽象成：

```text
                  Client
                    │
              Ollama HTTP API
                    │
              ┌─────▼─────┐
              │   Server  │
              └─────┬─────┘
                    │
       ┌────────────┼────────────┐
       ▼            ▼            ▼
   Model Mgmt    Scheduler    API/Chat
       │            │
       │            ▼
       │       Runner Manager
       │            │
       │            ▼
       │         Runner
       │            │
       │            ▼
       │       llama.cpp/GGML
       │            │
       ▼            ▼
    Model Store   CPU/GPU
```

---

# 31. Ollama 主要解决什么？

如果直接使用 llama.cpp：

```text
下载 GGUF
配置参数
启动 server
管理模型
处理模型生命周期
```

开发者需要自己做很多工作。

Ollama 将这些统一：

```text
ollama pull
ollama run
ollama list
ollama show
ollama rm
```

并提供 API：

```text
Client
  ↓
Ollama
  ↓
Runner
  ↓
Model
```

所以 Ollama 更接近：

> **Local LLM Platform / Runtime Manager**

而 llama.cpp 更接近：

> **Inference Runtime**

---

# 32. Ollama Model 管理

核心对象可以理解为：

```text
Model Name
   ↓
Manifest
   ↓
Model Blob
   ↓
GGUF / weights
   ↓
Runner
```

类似容器镜像的思想：

```text
Model
 ├── metadata
 ├── layers/blobs
 └── configuration
```

因此：

```text
ollama pull
```

本质不是简单：

```text
wget model.bin
```

而是模型资源的下载、缓存、校验、组织和运行准备。

---

# 33. Ollama Runner

Ollama 的核心设计之一：

```text
Ollama Server
      │
      ├── Model A
      │      └── Runner
      │
      ├── Model B
      │      └── Runner
      │
      └── Model C
             └── Runner
```

Server 管理：

- 模型
- 请求
- runner
- 生命周期
- 资源

Runner 管理：

- 模型加载
- inference
- context
- KV
- token generation

这是一种典型的：

> **Control Plane + Data Plane**

设计。

---

# 34. Ollama 为什么需要 Runner 进程？

如果模型推理和 server 完全处于一个进程：

```text
Server
 └── Model
      └── GPU
```

可能产生：

- 模型切换困难
- 崩溃影响 API server
- 多模型资源隔离困难
- GPU 生命周期管理困难

拆开后：

```text
Server
  │
  ├── Runner A
  ├── Runner B
  └── Runner C
```

Server：

> 控制

Runner：

> 执行

这就是典型控制面/数据面分离。

---

# 35. Ollama 调度

假设：

```text
Model A loaded
Model B unloaded
GPU memory = 24GB
```

请求：

```text
Request A → A
Request B → B
Request C → A
```

调度器需要考虑：

```text
Model loaded?
   ↓
已有 runner?
   ↓
显存够吗？
   ↓
是否复用？
   ↓
是否 unload？
   ↓
启动新 runner
```

因此模型服务的调度并不只是：

> round-robin。

它更像：

```text
Request Scheduling
+
Model Residency
+
GPU Memory Management
+
Runner Lifecycle
```

---

# 36. Ollama 的模型加载优化

关键目标：

```text
cold start
 ↓
model load
 ↓
warm inference
```

优化：

```text
Model Cache
+
Runner Reuse
+
Memory Residency
+
Keep Alive
```

因此面试可以回答：

> 本地 LLM 服务的性能不只是 TPS，还包括模型冷启动时间、模型切换成本和显存驻留策略。

---

# 37. Ollama vs llama.cpp

| 维度 | Ollama | llama.cpp |
|---|---|---|
| 定位 | LLM 本地运行平台 | 推理 runtime |
| 语言 | Go + native runtime | C/C++ |
| 模型管理 | 强 | 弱/基础 |
| API | 强 | 有 llama-server |
| 模型下载 | 完整 | 支持，但定位不同 |
| GGUF | 支持 | 核心 |
| Quantization | 使用 | 核心实现之一 |
| KV Cache | runner/runtime 层 | 核心 runtime |
| CPU 推理 | 支持 | 核心优势 |
| GPU | 支持 | CUDA/Metal/HIP/Vulkan/SYCL 等 |
| 调度 | 上层负责 | server/runtime 负责 |
| 源码学习价值 | 系统工程 | 推理底层 |

---

# 38. 最重要的性能优化公式

面试时建议形成这套思维：

```text
LLM Performance
=
Model Size
+
Compute
+
Memory Bandwidth
+
KV Cache
+
Batching
+
Scheduling
+
Kernel
+
I/O
```

对于 Decode：

```text
TPS
≈
f(
memory bandwidth,
model size,
quantization,
KV cache,
batch size,
kernel efficiency
)
```

对于 Prefill：

```text
TTFT
≈
queueing
+
tokenization
+
prompt processing
+
model computation
```

---

# 39. 显存估算

粗略：

```text
Weight Memory
≈ Parameters × Bytes per Parameter
```

例如：

```text
7B FP16
≈ 7B × 2
≈ 14GB
```

Q4：

```text
≈ 7B × 0.5
≈ 3.5GB
```

实际会增加：

```text
metadata
scales
alignment
temporary buffers
KV cache
runtime buffers
```

所以不能直接认为：

```text
7B Q4 = 3.5GB exactly
```

---

# 40. 总显存

实际：

```text
VRAM
=
Weights
+
KV Cache
+
Activations
+
Workspace
+
CUDA/Backend Buffers
+
Fragmentation
```

因此：

```text
模型能加载
≠
模型能高并发运行
```

---

# 41. 长上下文为什么贵？

KV Cache：

```text
KV ∝ sequence_length
```

所以：

```text
4K
 ↓
8K
 ↓
16K
 ↓
32K
```

显存近似线性增长。

如果：

```text
context × concurrency
```

同时增大：

```text
KV memory
≈
O(context × concurrent_sequences)
```

这就是本地多用户 LLM 服务的核心瓶颈之一。

---

# 42. 多用户并发时最关键的问题

假设：

```text
24GB GPU

Model = 8GB
KV A = 2GB
KV B = 2GB
KV C = 2GB
...
```

随着并发：

```text
KV cache ↑
```

最终：

```text
OOM
```

所以调优不是单纯：

> 把 context 设置到最大。

而是：

```text
Model size
×
Context length
×
Concurrency
×
KV dtype
```

一起考虑。

---

# 43. Prompt Cache vs KV Cache

必须区分。

### KV Cache

当前 sequence 的历史：

```text
A B C D E
↓
K/V
```

### Prompt Cache

不同请求之间复用共同 prefix：

```text
Request A:
SYSTEM A B C D X

Request B:
SYSTEM A B C D Y

             ^^^^^
             reused
```

因此：

```text
KV Cache
= sequence state

Prompt Cache
= reusable prefix state
```

---

# 44. Context Shift

上下文达到上限：

```text
[old tokens................new tokens]
                     ↑
                   limit
```

如果继续生成，需要：

```text
discard old tokens
+
preserve recent context
```

也就是 context shift / sliding-window 类策略。

注意：

> 这不是“无限上下文”，而是通过丢弃/压缩部分历史状态维持有限上下文窗口。

llama-server 当前提供 context-shift 相关配置。citeturn0search7

---

# 45. 为什么 LLM 推理特别依赖 Memory Bandwidth？

Decode：

```text
1 token
 ↓
Layer 1
 ↓
Layer 2
 ↓
...
 ↓
Layer N
```

每层都需要访问大量权重。

如果：

```text
Compute = 100 TFLOPS
Bandwidth = 500 GB/s
```

但实际操作需要大量搬运权重：

```text
GPU compute 很强
但数据喂不进去
```

就会：

```text
GPU utilization 低
```

因此：

> GPU utilization 低 ≠ GPU 性能差，可能是 memory-bound。

---

# 46. Roofline 思维

判断：

```text
Arithmetic Intensity
=
FLOPs / Bytes
```

低：

```text
Memory Bound
```

高：

```text
Compute Bound
```

LLM：

```text
Prefill → 更偏 Compute Bound
Decode  → 更偏 Memory Bound
```

不是绝对结论，但非常适合面试回答。

---

# 47. 常见性能指标

建议面试熟悉：

| 指标 | 含义 |
|---|---|
| TTFT | Time To First Token |
| TPOT | Time Per Output Token |
| TPS | Tokens Per Second |
| Prompt TPS | Prefill 速度 |
| Generation TPS | Decode 速度 |
| Queue Time | 排队时间 |
| Model Load Time | 模型加载时间 |
| Cache Hit Rate | Prompt/KV cache 命中率 |
| GPU Memory | 显存占用 |
| KV Memory | KV cache 占用 |

---

# 48. llama.cpp 常见源码阅读路径

建议：

```text
README
 ↓
examples/simple
 ↓
common
 ↓
src/llama-model.*
 ↓
src/llama-context.*
 ↓
src/llama-graph.*
 ↓
src/llama-kv-cache.*
 ↓
ggml/src
 ↓
tools/server
```

如果重点是服务端：

```text
tools/server/server.cpp
 ↓
server-context
 ↓
server-slot
 ↓
queue
 ↓
decode
```

当前 server 文档明确描述了 context、slot、route、HTTP、queue、response queue 等主要组件。citeturn0search0

---

# 49. Ollama 源码阅读路径

建议不要从 UI 开始。

优先：

```text
server
 ↓
request handling
 ↓
scheduler
 ↓
runner
 ↓
model loading
 ↓
runtime
```

重点理解：

```text
Request
 ↓
Scheduler
 ↓
Model Instance
 ↓
Runner
 ↓
Inference
 ↓
Streaming Response
```

---

# 50. 面试题：为什么 Ollama 很适合本地开发？

回答：

> 它把模型下载、模型版本/manifest、运行配置、模型生命周期、HTTP API 和底层 inference runtime 封装起来，使用户不需要直接管理 GGUF、backend 和底层 runner。它的价值更偏“本地 LLM 平台化”，而不是重新发明 Transformer 推理内核。

---

# 51. 面试题：为什么 llama.cpp 能在 CPU 上运行大模型？

答：

```text
1. Quantization
2. SIMD
3. Cache-friendly tensor layout
4. Multithreading
5. GGUF + mmap
6. CPU optimized kernels
7. Memory-efficient KV cache
```

核心原因：

> **减少数据搬运 + 利用 SIMD + 降低权重精度 + 高效内存布局。**

---

# 52. 面试题：为什么 Q4 模型速度可能比 FP16 快？

答：

```text
FP16
↓
权重大
↓
memory traffic 大

Q4
↓
权重小
↓
memory traffic ↓
↓
decode 更快
```

但需要考虑：

```text
dequantization
kernel efficiency
hardware
batch size
```

因此不能机械地认为：

> Q4 一定比 FP16 快。

---

# 53. 面试题：KV Cache 为什么不能无限增加？

因为：

```text
KV ∝ Context × Layers × KV Heads × Head Dim × Concurrency
```

context 和并发增长都会导致：

```text
KV memory ↑
```

最终：

```text
OOM
```

解决：

- GQA/MQA
- KV quantization
- context limit
- prefix cache
- eviction
- paging / memory management
- concurrency control

---

# 54. 面试题：如何提高本地模型 TPS？

建议按层排查：

```text
1. Model
   ↓
2. Quantization
   ↓
3. Backend
   ↓
4. GPU offload
   ↓
5. Kernel
   ↓
6. Batch
   ↓
7. KV Cache
   ↓
8. Scheduling
```

具体：

```text
Q4/Q5
+
GPU offload
+
CUDA/Metal optimized backend
+
continuous batching
+
KV cache
+
prompt cache
+
speculative decoding
```

---

# 55. 面试题：如何降低 TTFT？

拆解：

```text
TTFT
=
Queue
+
Model Load
+
Tokenization
+
Prompt Cache Miss
+
Prefill
```

优化：

### 1. Model Load

```text
warm model
keep alive
```

### 2. Prompt

```text
prefix cache
```

### 3. Prefill

```text
GPU
batch
optimized kernels
```

### 4. Queue

```text
admission control
scheduling
```

---

# 56. 面试题：如何支持多个模型？

架构：

```text
                API
                 │
              Router
                 │
       ┌─────────┼─────────┐
       ▼         ▼         ▼
    Model A   Model B   Model C
      │          │          │
   Runner A   Runner B   Runner C
```

问题：

```text
GPU memory
Model switching
Cold start
Concurrency
Cache isolation
```

解决：

```text
LRU / residency policy
keep-alive
model unloading
resource-aware scheduling
```

llama-server 当前也提供 router mode，可以在多个 inference server instance 之间按模型路由请求。citeturn0search0turn0search1

---

# 57. 面试题：如何设计一个本地 LLM Runtime？

建议分层：

```text
API Layer
    │
Scheduler
    │
Session / Sequence Manager
    │
Model Manager
    │
Inference Runtime
    │
Tensor Runtime
    │
Backend
    │
Hardware
```

核心模块：

```text
Model Loader
Tokenizer
Graph Executor
KV Cache
Scheduler
Sampler
Memory Manager
Backend
Metrics
```

---

# 58. 面试题：如何设计 KV Cache Manager？

至少包含：

```text
KVBlock
 ├── sequence_id
 ├── token_range
 ├── layer
 ├── device
 └── ref_count
```

管理：

```text
allocate
free
reuse
evict
prefix match
```

进一步：

```text
Prefix Tree / Radix Tree
```

用于：

```text
token prefix
    ↓
cache block
```

典型思路：

```text
Tokens:
A B C D E

Cache:
A B C D

New request:
A B C D X

reuse A B C D
compute X
```

---

# 59. 面试题：如果 GPU 利用率只有 30%，怎么办？

不要直接回答：

> “增加 batch。”

先定位：

```text
GPU utilization
      │
      ├── compute bound?
      ├── memory bound?
      ├── kernel launch?
      ├── CPU bottleneck?
      ├── synchronization?
      ├── PCIe transfer?
      ├── insufficient batch?
      └── KV cache pressure?
```

工具：

```text
nvidia-smi
Nsight Systems
Nsight Compute
perf
CPU profiler
runtime metrics
```

然后判断：

```text
Prefill
vs
Decode
```

---

# 60. 面试题：CPU+GPU 混合推理有什么问题？

优点：

```text
VRAM 不够时仍可运行
```

问题：

```text
CPU ↔ GPU data transfer
PCIe bandwidth
synchronization
pipeline bubble
```

如果：

```text
Layer A CPU
 ↓ PCIe
Layer B GPU
 ↓ PCIe
Layer C CPU
```

会非常慢。

因此更合理：

```text
CPU layers
     ↓
GPU layers
     ↓
GPU layers
     ↓
GPU layers
```

减少跨设备通信。

---

# 61. 面试题：为什么多并发可能让单请求变慢？

因为：

```text
Concurrency ↑
 ↓
Batch ↑
 ↓
GPU utilization ↑
```

但：

```text
Queueing ↑
KV Cache ↑
Memory bandwidth contention ↑
```

最终：

```text
Throughput ↑
Latency ↑
```

所以服务设计需要明确：

> 是追求单请求延迟，还是整体吞吐。

---

# 62. 面试题：Ollama 和 vLLM 怎么比较？

不要简单说谁更好。

可以从架构定位比较：

| | Ollama | llama.cpp | vLLM |
|---|---|---|---|
| 重点 | 本地开发体验 | 本地/通用 inference runtime | 高吞吐服务 |
| C/C++ runtime | 依赖底层 runtime | 核心 | 否 |
| GGUF | 强 | 核心 | 非主要方向 |
| CPU | 强 | 强 | GPU 为主 |
| Paged KV | 非核心定位 | 有多种 KV/cache 能力 | 核心思想之一 |
| Continuous batching | 有相关能力 | 有 | 核心 |
| OpenAI API | 支持 | 支持 | 支持 |
| Production serving | 可用 | 可用 | 强项 |
| 本地单机 | 强 | 强 | 可以 |

---

# 63. 最值得掌握的“算法”清单

如果目标是面试，不需要把所有代码都读完，重点掌握：

```text
★★★★★
1. KV Cache
2. Prefix / Prompt Cache
3. Quantization
4. Continuous Batching
5. Speculative Decoding
6. Attention
7. GQA / MQA
8. RoPE
9. Sampling

★★★★
10. Computation Graph
11. Memory Planning
12. CPU SIMD
13. GPU Kernel
14. mmap
15. Context Shift

★★★
16. MoE inference
17. Flash Attention 思想
18. KV Quantization
19. Prefix Tree / Radix Cache
20. Model Scheduling
```

---

# 64. 面试最容易被追问的 10 个问题

## Q1：Ollama 和 llama.cpp 是什么关系？

> Ollama 偏模型管理和本地服务平台，llama.cpp 偏底层推理 runtime；两者可以通过 runner/runtime 组合起来。

## Q2：为什么 llama.cpp 快？

> 量化 + SIMD + 高效 tensor/backend + GPU kernel + mmap + KV cache + batching + 针对硬件优化。

## Q3：KV Cache 是什么？

> 缓存历史 token 的 K/V，避免每次生成都重新计算历史 attention。

## Q4：KV Cache 为什么耗显存？

> KV 大小近似线性依赖 context length、layer 数和并发 sequence 数。

## Q5：Prefill 和 Decode 有什么区别？

> Prefill 批量处理 prompt，更偏计算密集；Decode 每次生成少量 token，更容易受 memory bandwidth 和 KV cache 影响。

## Q6：为什么 Q4 快？

> 权重更小，减少 memory traffic；但实际速度取决于 dequant 和 kernel。

## Q7：Prefix Cache 怎么判断前缀相同？

> 经过 tokenizer 后比较 token sequence / cache state，而不是简单比较原始字符串。

## Q8：Continuous Batching 是什么？

> 请求动态进入和退出 batch，每个 decode step 根据当前活跃 sequence 重新组织计算。

## Q9：Speculative Decoding 是什么？

> 小模型生成候选，大模型批量验证，以更少的大模型 decode step 获取多个 token。

## Q10：如何提升 LLM 推理性能？

> 先区分 TTFT 和 TPOT，再区分 Prefill 和 Decode，之后从模型量化、backend、kernel、batch、KV cache、cache hit、scheduler、GPU/CPU 通信逐层优化。

---

# 65. 一张图记住整个体系

```text
                       LLM Application
                              │
                              ▼
                    ┌─────────────────┐
                    │     Ollama      │
                    │ Model Management│
                    │ API / Scheduler │
                    │ Runner Lifecycle│
                    └────────┬────────┘
                             │
                             ▼
                    ┌─────────────────┐
                    │ llama.cpp       │
                    │ Inference       │
                    │ Runtime         │
                    └────────┬────────┘
                             │
             ┌───────────────┼───────────────┐
             ▼               ▼               ▼
          Model           Context          Server
             │               │               │
            GGUF           KV Cache       Batching
             │               │               │
             └───────────────┼───────────────┘
                             ▼
                          GGML
                             │
              ┌──────────────┼──────────────┐
              ▼              ▼              ▼
             CPU            CUDA           Metal
              │              │              │
              └──────────────┼──────────────┘
                             ▼
                         Hardware

核心优化：

Model
 ├── Quantization
 ├── GGUF
 └── mmap

Runtime
 ├── Computation Graph
 ├── KV Cache
 ├── Prefix Cache
 └── Sampling

Serving
 ├── Continuous Batching
 ├── Scheduler
 ├── Parallel Decoding
 └── Speculative Decoding

Hardware
 ├── SIMD
 ├── GPU Kernel
 ├── Memory Bandwidth
 └── CPU/GPU Offload
```

---

# 66. 架构师级回答模板

如果面试官问：

> “你怎么看 Ollama / llama.cpp？”

可以直接回答：

> 我会把它们拆成两个层次看。llama.cpp 是 inference runtime，核心解决模型加载、tensor/graph 执行、量化、KV Cache、sampling 以及 CPU/GPU backend 等问题；Ollama 更偏上层的本地 LLM 平台，负责模型管理、API、runner 生命周期和资源调度。  
>
> 从性能角度，我会重点关注 Prefill 和 Decode 两条路径。Prefill 通常更偏 compute-bound，而 Decode 更容易受 memory bandwidth、KV Cache 和 kernel efficiency 影响。因此 Q4/Q5 量化、KV Cache、continuous batching、prefix cache、GPU offload 和 speculative decoding 是几个核心优化方向。  
>
> 如果从服务化角度继续往上看，真正需要解决的是模型驻留、并发 sequence、KV memory、请求调度、TTFT/TPOT 和多模型资源隔离。这样就从一个“能跑模型”的 runtime，进一步变成了一个可服务化的 inference platform。

---

# 67. 最终知识框架

建议把整个 Ollama / llama.cpp 体系记成 **8 个问题**：

```text
1. 模型是什么？
   ↓
GGUF / Model / Tokenizer

2. 模型怎么加载？
   ↓
mmap / memory mapping / backend buffer

3. 模型怎么计算？
   ↓
GGML Tensor / Graph / Kernel

4. 模型怎么变小？
   ↓
Quantization

5. 历史 token 怎么复用？
   ↓
KV Cache / Prefix Cache

6. 多请求怎么运行？
   ↓
Batching / Continuous Batching / Scheduler

7. 怎么让生成更快？
   ↓
Quantization / Kernel / Speculative Decoding

8. 怎么变成服务？
   ↓
Ollama / llama-server / Runner / API / Model Lifecycle
```

**如果只准备面试，优先把这 8 个问题讲透，比机械阅读全部源码更重要。**

---

## 参考资料

- llama.cpp 官方仓库与 README：支持 GGUF、量化、多种 CPU/GPU backend 以及 CPU+GPU hybrid inference。citeturn0search5turn0search6
- llama.cpp server 文档：parallel decoding、continuous batching、prompt cache、speculative decoding、router mode 等。citeturn0search1
- llama.cpp server 开发架构：`server_context`、`server_slot`、routes、HTTP、queue、response queue。citeturn0search0
- llama.cpp speculative decoding：draft model、EAGLE-3、MTP、n-gram 等。citeturn0search2
