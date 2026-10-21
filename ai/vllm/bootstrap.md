# vLLM 启动全景 · 总纲

> 一张全景图看懂启动主线，各阶段只讲「是什么 / 为什么」，关键名词逐条解析

**一句话总纲**：vLLM 的启动不是「加载模型」，而是在构建一个高性能推理运行时——依次完成 配置模型 → 选执行后端 → 建并行环境 → 建结构 → 加载/量化权重 → 规划 CPU/GPU 权重 → 多模态 profiling → 编译计算图 → 捕获 CUDA Graph → 规划 KV Cache → Warmup，最终 Engine Ready。

## 一、启动全景流程

主线来自真实启动日志（vLLM V1 Engine + Gemma 多模态 + AWQ-4bit + Prefetch Offload + Triton Attention + torch.compile + CUDA Graph + KV Cache）：

```
vllm serve
 │  ① API Server：解析参数 → EngineArgs → 各类 Config
 ▼  ② EngineCore：创建 V1 LLM Engine（独立进程，与 API 层解耦）
 ▼  ③ Parallel/Device：初始化 TP/PP/DP 与 NCCL
 ▼  ④ Model Runner：按 config 构建模型结构
 ▼  ⑤ Weight Loader：safetensors → GPU，AWQ 走 Marlin Kernel
 ▼  ⑥ Offload：CPU↔GPU 权重预取，为 GPU 腾显存
 ▼  ⑦ Multimodal Warmup：Vision Encoder profiling
 ▼  ⑧ torch.compile：Dynamo → Inductor 生成优化 kernel
 ▼  ⑨ CUDA Graph Capture：按 batch/shape 批量捕获图
 ▼  ⑩ KV Cache Profile：测算 GPU 可容纳 token 数
 ▼  JIT Warmup → 第二轮 CUDA Graph → Engine Ready → OpenAI API
```

## 二、各阶段核心概念速查（是什么 · 为什么）

| 阶段 | 是什么 | 为什么 |
| --- | --- | --- |
| **① API Server 参数解析** | 入口层。命令行参数 → EngineArgs → 各类 Config，再交给 EngineCore。此时还没真正跑 Transformer。 | 把用户输入统一成内部配置：模型/长度/dtype/量化/GPU 数/并行/KV Cache/backend 等，Engine 才能据此决策。 |
| **② 解析 Model** | 读 config.json → 解析 architecture → 命中对应实现类（Gemma4ForConditionalGeneration）。 | vLLM 不靠模型名猜结构，而是按声明的 architecture 动态选择模型实现，便于扩展新模型。 |
| **③ Scheduler 与 Chunked Prefill** | max_model_len=单请求最大长度(16000)；max_num_batched_tokens=一次调度最多处理的 token(2496)。二者勿混淆。 | 一次吞下超长 prompt 会占满算力并阻塞 decode。把大 prefill 切成 chunk，让 prefill 与 decode 更易混排调度。 |
| **④ Attention Backend** | Attention 有多种后端实现（FlashAttention / FlashInfer / Triton…），按 GPU、head dim、dtype、KV layout 选择。 | Gemma 异构 head dim（sliding 256 / full 512）→ FA4 不支持 → fallback 到 TRITON_ATTN。 |
| **⑤ EngineCore（进程解耦）** | APIServer 与 EngineCore 是两个进程（如 pid 79016 / 79241）：前者管 HTTP/OpenAI，后者管 GPU 推理。 | V1 核心思想：API 层与执行层解耦，请求调度与 GPU 计算互不阻塞。 |
| **⑥ Parallel State** | 初始化 TP/PP/DP 并行维度（本例均为 1）与通信后端 NCCL。 | 即便单卡也统一初始化并行抽象：抽象统一、便于扩到多卡，通信层一致。 |
| **⑦ Model Runner / 建模型结构** | "Loading from scratch" 不是从头训练，而是按 config 建 PyTorch Module，再灌入 checkpoint 权重。 | 结构与权重分离：先有骨架（Vision Tower/Embedding/Layers/Attention/MLP/LM Head），再填参数。 |
| **⑧ 量化 Kernel** | AWQ-4bit 权重 → Compressed-Tensors → MarlinLinearKernel → GPU GEMM，而非普通 FP16 Linear。 | 既省权重显存，又用专用 CUDA kernel 保持量化 Linear 的速度。 |
| **⑨ Weight Loader + dtype 转换** | safetensors 分片(4 shards)经 mmap 加载到 GPU。大量 FP16→BF16 warning = checkpoint dtype ≠ 参数 dtype。 | 加载时按模型参数 dtype 做转换对齐；本例多集中在 vision_tower 部分，属正常匹配，非报错。 |
| **⑩ Prefetch Offload** | 不把全部权重常驻 GPU；部分权重留在 CPU，执行前预取(prefetch)上 GPU，用完让位下一组。 | 24GB 显存装不下 19.47GiB 权重 + KV Cache + activation + CUDA Graph，用 CPU 换 GPU 显存，代价是 PCIe 传输延迟。 |
| **⑪ Multimodal Warmup** | 模型是 Image/Video → Vision Encoder → 多模态 Embedding → LLM，启动时预建 encoder 并分配 encoder cache、profile。 | 多模态输入规模不确定，需提前 warmup 定好预算（本例 encoder cache 预算 2496 tokens），避免首请求抖动。 |
| **⑫ torch.compile** | Dynamo 抓取计算图 → Inductor 生成优化 kernel（本例耗时 72s，是最大耗时点之一）。 | 消除 Python/dispatcher 逐算子开销，做算子融合与 kernel 优化，直接提升吞吐、降低延迟。 |
| **⑬ CUDA Graph** | 把一串 kernel 打包成图一次提交；需按不同 batch/token shape 各捕获一张图（1,2,4,…,512）。 | 省掉每次 kernel launch 的 CPU 开销。运行时按当前 batch size 找对应图直接 replay。 |
| **⑭ KV Cache Profile** | 把过去 token 的 K/V 缓存复用，按 block 分块管理（block size 32）。启动测算可得 KV 容量(本例 17059 tokens)。 | KV Cache 是 LLM 推理核心瓶颈。分块管理实现细粒度、可共享的显存复用；决定最大并发（本例 ≈1.07× 一个满长请求）。 |

## 三、核心名词解析（是什么 · 为什么）

> 上面速查表把名词「用」了一遍，这里把它们逐条「讲清」。按所属阶段归类，仍只讲是什么 / 为什么。

### 1. Attention 后端类（对应阶段 ④）

| 名词 | 是什么 | 为什么 |
| --- | --- | --- |
| **Attention Backend** | 计算注意力（Attention）的具体 GPU kernel 实现，vLLM 里做成可插拔「后端」。 | 不同硬件、head dim、dtype、KV 内存布局下最优实现不同，所以需要多套后端并按条件择优/回退。 |
| **FlashAttention（FA2/FA3/FA4）** | 一种「IO-aware」的精确注意力算法/kernel：把序列分块(tiling)在片上缓存 SRAM 里逐块算 softmax，不把 N×N 注意力大矩阵写回显存 HBM。 | 显存占用从 O(N²) 降到 O(N)、大幅减少访存往返，长序列提速明显；代价是对 head dim、内存布局、硬件版本有白名单约束。 |
| **FlashInfer** | 面向 LLM 推理的高性能 Attention / KV-Cache kernel 库，支持变长(varlen)、分页(paged)、量化 KV，并可按需定制 kernel。 | 针对 prefill/decode 不同特性做了更激进的优化，提供比通用实现更高的吞吐，是 FA 之外的另一条高性能后端路线。 |
| **Triton / TRITON_ATTN** | Triton 是 OpenAI 开源的 GPU DSL（用 Python 方言写 CUDA kernel）；TRITON_ATTN 指用 Triton 实现的那套 attention 后端。 | 当 FA4 不支持某些 head dim（如本例 sliding 256 / full 512 异构）时，Triton 能灵活生成兼容 kernel，作为通用回退方案。 |
| **head dim / 异构 head dim** | 每个注意力头的向量维度（head_dim）。Gemma 的滑窗层与全注意力层 head_dim 不同（256 vs 512），即「异构」。 | 后端只对特定 head_dim 做了向量化优化并列入白名单，异构值不在支持列表就会触发 fallback。 |
| **SRAM / HBM** | SRAM=SM 上的片上高速缓存（小、快）；HBM=高带宽显存（大、相对慢），即平时说的「显存」。 | FlashAttention 的核心思路就是尽量在 SRAM 内完成计算、少往返 HBM，这是它省显存提速的物理基础。 |

### 2. 量化类（对应阶段 ⑧）

| 名词 | 是什么 | 为什么 |
| --- | --- | --- |
| **量化（Quantization）/ 4-bit** | 把权重从 16-bit（FP16/BF16）压到更低比特（如 4-bit）表示。 | 显存占用与访存带宽成倍下降，推理更快更省；难点是压完不掉精度、且要跑得比反量化再算更快。 |
| **AWQ（Activation-aware Weight Quantization）** | 一种 4-bit 权重量化方法：依据激活值分布，识别并「保护」少数关键权重通道（给更高精度/缩放），其余统一低比特。 | 在几乎不掉精度前提下把权重压到约 1/4，是当前主流 4-bit 量化算法之一。 |
| **Compressed-Tensors** | 一种量化模型的标准化打包与加载格式/框架（Neural Magic 提出）：用元数据描述量化方案、scale、group size 等。 | 不同量化算法（AWQ/GPTQ 等）的 checkpoint 描述五花八门，统一格式后 vLLM 能据元数据还原量化结构并选对 kernel。 |
| **Marlin / MarlinLinearKernel** | 针对 INT4（AWQ/GPTQ）权重 + FP16 激活的高性能 GEMM CUDA kernel：在 kernel 内部直接处理量化权重做混合精度矩阵乘。 | 若先「反量化成 FP16 再矩阵乘」会多一次大张量读写、反而变慢；Marlin 让量化 Linear 的速度逼近甚至超过 FP16，真正把量化变成提速而非纯省显存。 |
| **GEMM** | General Matrix Multiply，通用矩阵乘（Linear/全连接层的计算本质）。 | 矩阵乘是 LLM 前向的计算主体，量化带来的显存/带宽收益最终主要落在 GEMM 上，所以量化 kernel 的质量直接决定性能。 |
| **Linear 层（FP16 Linear）** | 全连接矩阵乘层，是模型里数量最多的可计算层。 | 量化版与普通版的差别就体现在这里——同一层是用普通 FP16 Linear，还是走 Marlin 这类量化 kernel。 |

### 3. 权重加载类（对应阶段 ⑤⑨）

| 名词 | 是什么 | 为什么 |
| --- | --- | --- |
| **safetensors** | HuggingFace 推出的安全权重文件格式：张量名→张量的扁平映射，不含 pickle、支持 mmap/懒加载与分片。 | 比传统 .bin/pickle 更安全（不执行任意代码）、加载更快、天然支持分片与按需读取。 |
| **shard / 分片（4 shards）** | 权重太大时拆成多个 safetensors 文件（如 model-00001-of-00004…）。 | 单文件过大不利分发与并行加载，分片后可多线程/多进程并行读、降低单次内存峰值。 |
| **mmap（内存映射）** | 把文件按页映射进地址空间，访问到哪页才读哪页。 | 不必一次性把整份权重读进内存，加载更快、内存峰值更低，适合超大 checkpoint。 |
| **dtype / FP16 / BF16** | 数据精度类型。FP16=半精度（10 位尾数，范围小）；BF16=「脑浮点」（8 位指数位，动态范围与 FP32 同、尾数更短）。 | checkpoint 存的 dtype 可能与模型参数 dtype 不一致，加载时需转换对齐（大量 FP16→BF16 warning 即源于此）。BF16 动态范围大、不易溢出，是 Ampere+ GPU 的常用训练/推理精度，属正常匹配而非报错。 |
| **Weight Loader / default_loader** | 负责把 checkpoint 读到 GPU、并把张量正确映射进模型各参数的组件。 | 结构与权重分离，需要一个统一的「灌权重」环节处理分片、dtype 转换、量化还原、CPU/GPU 放置等杂务。 |
| **Loading from scratch** | 先按 config 搭一个空的模型结构，再把 checkpoint 权重灌进去——不是「从头训练」。 | 强调「结构」与「权重」是两件事：日志里的 scratch 指先建骨架，随后才加载参数，避免误解为重新训练。 |

### 4. 编译与计算图类（对应阶段 ⑧⑨）

| 名词 | 是什么 | 为什么 |
| --- | --- | --- |
| **torch.compile** | PyTorch 2.x 的图编译入口，把逐行执行的 eager 代码编译成优化后的 kernel。 | 消除 Python/dispatcher 逐算子调度开销、做算子融合，提升吞吐降低延迟；本例耗时约 72s，是启动最大耗时点之一。 |
| **Dynamo** | torch.compile 的前端：抓取 Python 字节码，构建出可优化的计算图。 | 把动态、带分支的 Python 代码「翻译」成可优化的静态图，是后续 kernel 生成的前提。 |
| **Inductor** | torch.compile 的后端代码生成器，产出 Triton / C++ 优化 kernel。 | 在图层面做算子融合与内存规划，生成高效 kernel，是真正产生性能收益的环节。 |
| **kernel / 算子融合** | kernel=GPU 上执行的一段计算程序；融合=把多个连续算子合并进一个 kernel。 | 减少 kernel launch 次数、避免中间结果反复往返 HBM，是 Inductor 提速的主要手段。 |
| **CUDA Graph** | 把一串固定的 kernel 序列「捕获」成一张图，之后一次性「replay」提交执行。 | 省掉每次单独 launch 的 CPU 开销；小 kernel 数量多、batch 小的时候收益尤为明显。因图内 shape 固定，需按不同 batch/shape 各捕获一张图（1,2,4,…,512）。 |
| **capture / replay / kernel launch** | launch=CPU 向 GPU 提交一个 kernel；capture=录制一串 kernel 成图；replay=整图一次提交。 | 逐 kernel launch 的 CPU 开销在 decode 小 batch 时会成为瓶颈，CUDA Graph 用「一次提交」摊薄这部分开销。 |
| **JIT Warmup** | just-in-time 预热：先空跑/真实跑一遍，触发延迟编译与缓存填充。 | 把编译、内存分配等一次性开销放到启动阶段完成，避免首个真实请求时卡顿抖动。 |

### 5. KV Cache 与调度类（对应阶段 ③⑩⑭）

| 名词 | 是什么 | 为什么 |
| --- | --- | --- |
| **KV Cache** | 缓存历史 token 计算出的 Key/Value 张量，下一步生成时直接复用、不重复算。 | decode 每步只需算「新 token」的 K/V，这是自回归提速的关键；代价是它随序列长度增长、最吃显存，是 LLM 推理核心瓶颈。 |
| **PagedAttention / block / block size 32 / Block Table** | 像操作系统分页那样管理 KV Cache：切成固定大小 block（本例 32），用「页表」Block Table 记录每个请求用了哪些物理块。 | 彻底消除按连续大块预分配造成的碎片与浪费，做到按需分配、块间可共享，从而提升显存利用率与并发。 |
| **prefix caching** | 复用不同请求之间「相同前缀」已有的 KV Cache。 | 多个请求共享同一 system prompt / few-shot 前缀时，省去重复的 prefill 计算，降延迟提吞吐。 |
| **Prefill / Decode** | prefill=并行处理整段 prompt、算出全部 K/V（算力密集）；decode=逐 token 自回归生成（访存密集）。 | 两阶段计算特性差异巨大，把它们高效混排调度是连续批处理与延迟控制的核心难点。 |
| **Chunked Prefill** | 把过长的 prefill 切成若干 chunk，分多个调度步处理。 | 防止一个超长 prompt 独占一步调度、阻塞正在进行的 decode，从而压低 decode 的延迟抖动。 |
| **continuous batching** | 请求级动态组批：有请求算完就立即退出，新请求随时补入同一批。 | 不必等整批齐步，GPU 利用率和吞吐更高，是 vLLM 高吞吐的关键机制之一。 |
| **Scheduler** | 决定每一步「跑哪些请求、各处理多少 token」的调度器。 | 在吞吐（多塞请求）与延迟（别饿死 decode）之间做实时权衡，是 Engine 的大脑。 |
| **max_model_len / max_num_batched_tokens** | 前者=单请求最大上下文长度(16000)；后者=单步最多调度的 token 数(2496)。 | 一个约束「单请求能有多长」，一个约束「单步算力预算多大」，混淆二者会导致对显存/吞吐的误判。 |
| **gpu_memory_utilization** | 允许 vLLM 占用的总显存比例（本例 0.9）。 | 不能设成 1.0：必须给 Activation、CUDA Graph、内存碎片留余量，否则运行期 OOM。 |

### 6. 多模态与并行/进程类（对应阶段 ⑥⑦⑪）

| 名词 | 是什么 | 为什么 |
| --- | --- | --- |
| **Vision Encoder / Vision Tower** | 把图像/视频编码成 embedding 向量的子模块（常为 ViT），输出喂给 LLM 主干。 | 多模态模型需要先把非文本输入转成与文本同空间的 embedding，再走统一 LLM。 |
| **encoder cache / Multimodal Warmup** | 缓存视觉编码结果；启动时预建 encoder 并 profile 出可分配预算（本例 2496 tokens）。 | 图像/视频 token 数不固定，需提前估算预算，避免首请求因显存/容量不确定而抖动或 OOM。 |
| **TP / PP / DP** | 张量并行(TP)/流水线并行(PP)/数据并行(DP)。 | 单卡装不下或想更快时，把大模型在维度/层/样本三个方向切到多卡。 |
| **NCCL** | NVIDIA 的多 GPU 集合通信库（all-reduce、broadcast 等）。 | TP/PP 的切分需要跨卡同步张量，NCCL 提供高效 GPU 间通信后端。 |
| **APIServer / EngineCore / 进程解耦** | APIServer 管 HTTP/OpenAI 接口，EngineCore 管 GPU 推理，二者为两个进程，经 IPC 通信。 | V1 核心设计：把网络/序列化这类 CPU 工作从 GPU 关键路径上移出，互不阻塞。 |
| **EngineArgs / Config / arg_utils** | 命令行参数汇总成的内部配置对象与解析工具。 | 所有决策（模型/长度/量化/并行/backend/KV）都源自配置，统一入口便于校验与传播。 |
| **GPUWorker / ModelRunner** | Worker 管一个 GPU 上的设备与显存，ModelRunner 负责把一批请求跑成一次 forward。 | 把「设备资源管理」与「前向执行」分层，便于多卡与不同后端复用。 |
| **architecture / Resolved architecture / config.json** | config.json 声明模型架构名，vLLM 据此解析并绑定到对应实现类。 | 不靠模型名猜结构，按声明的 architecture 动态选实现，新增模型只需注册实现类。 |
| **LM Head / Sampler** | LM Head=最后输出到词表 logits 的投影层；Sampler=按 logits 采样出下一个 token（贪心/top-k/top-p 等）。 | 一个把隐藏状态映射到词概率，一个从概率里决定实际 token，是生成「最后一环」。 |

## 四、启动完成后的显存账（GPU Memory Utilization=0.9 → 21.2 GiB）

利用率不是「模型大小 / 显存」，而是权重 + Runtime + Activation + CUDA Graph + KV Cache 综合决定。

| 占用项 | 容量 |
| --- | --- |
| 模型权重 + 非 torch 开销 | 13.61 GiB |
| Activation 峰值 | 1.38 GiB |
| CUDA Graph | 0.44 GiB |
| KV Cache | 6.21 GiB |

> 注：Activation = 前向计算时各层临时生成的中间张量（hidden states、QKV/FFN 中间结果等），随并发与序列长度变化、用完即释放；正因要给 Activation 与 CUDA Graph 留余量，KV Cache 才不能把剩余显存吃满。

## 五、源码地图（日志可直接对应的模块）

最值得深入的 5 个核心对象链：**EngineCore → GPUWorker → ModelRunner → Scheduler → KV Cache**。

| 入口 / 配置 | arg_utils.py · config.py · models/model.py（Resolved architecture） |
| --- | --- |
| Engine / 并行 | v1 core · parallel_state.py |
| 执行 / 模型 | gpu_worker.py · model_runner.py · model_executor/models/ |
| 权重 / 量化 | default_loader.py · weight_utils.py · compressed_tensors · Marlin |
| Attention | attention backend · Triton kernel |
| Offload / 多模态 | prefetch.py · encoder_runner.py |
| 编译 / 图 / 缓存 | torch.compile(Dynamo+Inductor) · CUDA Graph · kv_cache_utils.py · scheduler.py |

## 六、源码学习路径（自顶向下，按调用链追）

1. **启动主链**：vllm serve → APIServer → EngineCore → GPUWorker → ModelRunner：搞清一个 serve 怎么走到 ModelRunner。
2. **模型加载**：load_model() → Model → WeightLoader → safetensors → GPU：结构何时建、权重何时灌、量化在哪介入。
3. **显存与缓存**：Weights + Activation + CUDA Graph + KV Cache：重点 KV Cache / Block / Block Table / Cache Manager / Scheduler。
4. **执行优化**：torch.compile → Inductor → CUDA Graph → Kernel，再进 Triton / FlashAttention / FlashInfer / Marlin。
5. **请求执行**：HTTP → APIServer → EngineCore → Scheduler → KV Cache → ModelRunner → Sampler → Response，此时再看 continuous batching / chunked prefill / prefix caching。
