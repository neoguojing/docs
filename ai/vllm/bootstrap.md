# vLLM 启动全景 · 总纲

> 一张全景图看懂启动主线，各阶段只讲「是什么 / 为什么」

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

## 三、启动完成后的显存账（GPU Memory Utilization=0.9 → 21.2 GiB）

利用率不是「模型大小 / 显存」，而是权重 + Runtime + Activation + CUDA Graph + KV Cache 综合决定。

| 占用项 | 容量 |
| --- | --- |
| 模型权重 + 非 torch 开销 | 13.61 GiB |
| Activation 峰值 | 1.38 GiB |
| CUDA Graph | 0.44 GiB |
| KV Cache | 6.21 GiB |

## 四、源码地图（日志可直接对应的模块）

最值得深入的 5 个核心对象链：**EngineCore → GPUWorker → ModelRunner → Scheduler → KV Cache**。

| 入口 / 配置 | arg_utils.py · config.py · models/model.py（Resolved architecture） |
| --- | --- |
| Engine / 并行 | v1 core · parallel_state.py |
| 执行 / 模型 | gpu_worker.py · model_runner.py · model_executor/models/ |
| 权重 / 量化 | default_loader.py · weight_utils.py · compressed_tensors · Marlin |
| Attention | attention backend · Triton kernel |
| Offload / 多模态 | prefetch.py · encoder_runner.py |
| 编译 / 图 / 缓存 | torch.compile(Dynamo+Inductor) · CUDA Graph · kv_cache_utils.py · scheduler.py |

## 五、源码学习路径（自顶向下，按调用链追）

1. **启动主链**：vllm serve → APIServer → EngineCore → GPUWorker → ModelRunner：搞清一个 serve 怎么走到 ModelRunner。
2. **模型加载**：load_model() → Model → WeightLoader → safetensors → GPU：结构何时建、权重何时灌、量化在哪介入。
3. **显存与缓存**：Weights + Activation + CUDA Graph + KV Cache：重点 KV Cache / Block / Block Table / Cache Manager / Scheduler。
4. **执行优化**：torch.compile → Inductor → CUDA Graph → Kernel，再进 Triton / FlashAttention / FlashInfer / Marlin。
5. **请求执行**：HTTP → APIServer → EngineCore → Scheduler → KV Cache → ModelRunner → Sampler → Response，此时再看 continuous batching / chunked prefill / prefix caching。
