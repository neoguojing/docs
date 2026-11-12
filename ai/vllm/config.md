我检查了你提供的 GitHub 分支。原文档目前只有配置组的概览，真正的配置参数被三个自动生成指令隐藏了：

| 原文中的折叠位置                           | 对应内容                              | 主要源码                       |
| ---------------------------------- | --------------------------------- | -------------------------- |
| `gen:serve-args`                   | `vllm serve` 全部 CLI 参数            | `vllm/engine/arg_utils.py` |
| `gen:engine-args`                  | `EngineArgs`、`AsyncEngineArgs` 参数 | `vllm/engine/arg_utils.py` |
| `vllm/envs.py:env-vars-definition` | 环境变量名称、说明和默认值                     | `vllm/envs.py`             |

其中，`arg_utils.py` 不只是参数声明，还负责根据配置类的类型注解、默认值和文档字符串动态生成 CLI 参数。配置类本身才是理解参数含义的核心依据。

[image](https://www.google.com/s2/favicons?domain=https://raw.githubusercontent.com\&sz=32)

GitHub+1



我建议将文档扩展为一份完整的 vLLM 配置参数参考手册，而不是简单地把 `--help` 的输出复制进来。每个参数都应说明：

1. 是什么：参数的含义、类型和默认值。
2. 为什么：它解决什么问题，影响哪个模块。
3. 如何使用：常见取值、配置示例及参数之间的关系。
4. 影响：对显存、吞吐、延迟、启动时间或并行执行的影响。

另外，需要严格区分你指定的 GitHub 分支与官方最新版文档，避免把其他版本新增或删除的参数混入当前分支。

## 一、配置体系与执行关系

首先需要理解 vLLM 的配置不是一组平铺的 CLI 参数，而是由多个配置对象组合形成的配置体系。

用户配置入口

`vllm serve` / `LLM(...)` / YAML / 环境变量

EngineArgs / AsyncEngineArgs

参数解析、类型转换、默认值处理

VllmConfig

聚合各个 Config 对象，进行配置校验和初始化

ModelConfig

LoadConfig

ParallelConfig

CacheConfig

SchedulerConfig

CompilationConfig

AttentionConfig

KernelConfig

MultiModalConfig

ObservabilityConfig

Engine Runtime

模型加载、分布式执行、KV Cache、请求调度、推理计算

### 1.1 三种配置入口

| 入口         | 作用       | 典型用途           |
| ---------- | -------- | -------------- |
| CLI        | 命令行参数    | `vllm serve`   |
| YAML       | 集中管理参数   | 生产部署、配置复用      |
| Python API | 通过对象传入参数 | `LLM()`、Engine |

### 1.2 配置对象与职责

| 配置对象                    | 核心职责              | 主要影响              |
| ----------------------- | ----------------- | ----------------- |
| Frontend                | HTTP/API 服务       | 服务接入、请求处理         |
| ModelConfig             | 模型、Tokenizer、模型行为 | 模型结构和计算           |
| LoadConfig              | 权重加载              | 启动时间、加载内存         |
| AttentionConfig         | Attention 后端      | Prefill/Decode 性能 |
| MambaConfig             | Mamba/SSM 后端      | 状态空间模型计算          |
| StructuredOutputsConfig | 结构化输出             | JSON、Grammar 约束解码 |
| ParallelConfig          | 分布式并行             | GPU、节点间计算         |
| CacheConfig             | KV Cache          | 显存、上下文、并发         |
| OffloadConfig           | CPU/GPU 数据卸载      | 显存与内存交换           |
| MultiModalConfig        | 多模态处理             | 图片、视频、音频          |
| LoRAConfig              | LoRA 适配器          | 多模型适配器服务          |
| LoggingConfig           | 日志                | 日志输出和排障           |
| ObservabilityConfig     | 可观测性              | Metrics、Tracing   |
| SchedulerConfig         | 请求调度              | 并发、吞吐、延迟          |
| CompilationConfig       | 编译和 CUDA Graph    | 推理性能、启动时间         |
| KernelConfig            | 内核实现选择            | 算子性能              |
| VllmConfig              | 全局配置聚合            | 配置协调和运行时          |

这几个配置组的职责与原文的划分一致。

[image](https://www.google.com/s2/favicons?domain=https://github.com\&sz=32)

GitHub+1



## 二、vllm serve 参数展开

### 2.1 通用入口参数

| 参数                      | 作用                | 说明                     |
| ----------------------- | ----------------- | ---------------------- |
| `model_tag` / `--model` | 指定模型              | Hugging Face 模型名称或本地目录 |
| `--help`                | 显示帮助              | 查看支持的参数                |
| `--config`              | YAML 配置文件         | 从配置文件读取 CLI 参数         |
| `--headless`            | 无头模式              | 用于分布式部署中的非前端节点         |
| `--api-server-count`    | API Server 数量     | 控制前端服务进程数量             |
| `--grpc`                | 启动 Python gRPC 服务 | 替代 HTTP OpenAI 兼容服务入口  |
| `--shutdown-timeout`    | 关闭超时              | 控制服务退出等待时间             |

### 2.2 Frontend：服务入口配置

职责： 管理 HTTP Server、API 路由、请求协议、鉴权、Chat Template、响应格式等。

| 参数                               | 作用       | 说明                         |
| -------------------------------- | -------- | -------------------------- |
| `--host`                         | 监听地址     | 默认通常为 `localhost`          |
| `--port`                         | HTTP 端口  | 默认 8000                    |
| `--api-key`                      | API 鉴权   | 验证请求中的 API Key             |
| `--served-model-name`            | 对外模型名称   | API 请求使用的模型名称              |
| `--chat-template`                | 对话模板     | 将 OpenAI Message 转换为模型输入格式 |
| `--chat-template-content-format` | 内容格式     | `auto`、`openai`、`string`   |
| `--response-role`                | 响应角色     | 指定生成消息的角色                  |
| `--enable-request-id-headers`    | 请求 ID    | 在响应头中返回请求标识                |
| `--cors-allow-origins`           | CORS 来源  | 允许跨域请求的来源                  |
| `--cors-allow-methods`           | CORS 方法  | 允许的 HTTP 方法                |
| `--cors-allow-headers`           | CORS 请求头 | 允许的请求头                     |
| `--ssl-keyfile`                  | TLS 私钥   | HTTPS 服务配置                 |
| `--ssl-certfile`                 | TLS 证书   | HTTPS 服务配置                 |
| `--enable-log-requests`          | 请求日志     | 输出请求信息                     |
| `--disable-log-stats`            | 关闭统计日志   | 不再输出周期性运行统计                |

关键理解：

Frontend 主要处理协议层工作：

```
Client
  │
  ▼
HTTP Server
  │
  ▼
API Router
  │
  ▼
Request Validation
  │
  ▼
Chat Template / Tokenization
  │
  ▼
Engine
  │
  ▼
Response Serialization
```

Frontend 和 Engine 是两个不同的职责层。调整 HTTP 端口不会改变 GPU Worker 的分布式通信地址。

### 2.3 ModelConfig：模型配置

职责： 决定加载什么模型、使用什么 Tokenizer、支持什么任务，以及模型的计算行为。

| 参数                           | 作用                   | 说明                          |
| ---------------------------- | -------------------- | --------------------------- |
| `--model`                    | 模型路径                 | 模型名称或本地目录                   |
| `--tokenizer`                | Tokenizer 路径         | 默认使用模型对应的 Tokenizer         |
| `--tokenizer-mode`           | Tokenizer 模式         | 控制 Tokenizer 实现             |
| `--trust-remote-code`        | 信任远程代码               | 允许执行模型仓库自定义代码               |
| `--dtype`                    | 权重数据类型               | FP16、BF16、FP32、Auto 等       |
| `--seed`                     | 随机种子                 | 控制随机性                       |
| `--max-model-len`            | 最大上下文长度              | Prompt + Output 的最大 Token 数 |
| `--quantization`             | 量化方法                 | AWQ、GPTQ、FP8 等，取决于模型和硬件支持   |
| `--revision`                 | 模型版本                 | 指定模型仓库 revision             |
| `--code-revision`            | 代码版本                 | 指定远程代码版本                    |
| `--hf-config-path`           | 模型配置路径               | 覆盖默认 Hugging Face Config    |
| `--config-format`            | 配置格式                 | 控制模型配置文件读取格式                |
| `--served-model-name`        | 服务模型名称               | 对外暴露的模型标识                   |
| `--runner`                   | 执行任务类型               | `generate`、`pooling` 等      |
| `--convert`                  | 模型转换方式               | 适配不同任务的模型结构                 |
| `--skip-tokenizer-init`      | 跳过 Tokenizer 初始化     | 适用于特定输入形式                   |
| `--enable-prompt-embeds`     | 支持 Prompt Embeddings | 允许输入嵌入向量                    |
| `--disable-sliding-window`   | 关闭滑动窗口               | 控制模型滑动窗口行为                  |
| `--allowed-local-media-path` | 本地媒体访问路径             | 限制多模态文件访问范围                 |
| `--allowed-media-domains`    | 媒体域名白名单              | 限制远程媒体访问域名                  |

重点参数说明：

- `dtype`：决定模型权重和部分计算的数据精度，直接影响显存占用和计算性能。
- `max-model-len`：限制单条请求的最大上下文长度，间接影响 KV Cache 容量需求。
- `quantization`：主要用于压缩权重，降低权重显存占用，但不代表 KV Cache 也自动使用相同的量化精度。
- `runner`：决定模型执行任务的类型，不是简单的生成参数。

### 2.4 LoadConfig：模型加载配置

职责： 控制模型权重从哪里来、以什么格式加载，以及如何提高加载效率。

| 参数                                   | 作用               | 说明                 |
| ------------------------------------ | ---------------- | ------------------ |
| `--load-format`                      | 权重加载格式           | 自动选择或指定加载器         |
| `--download-dir`                     | 下载缓存目录           | 模型文件下载位置           |
| `--safetensors-load-strategy`        | Safetensors 加载策略 | 控制权重文件读取方式         |
| `--safetensors-prefetch-num-threads` | 预取线程数            | OS Page Cache 预取线程 |
| `--safetensors-prefetch-block-size`  | 预取块大小            | 每次预取的数据量           |
| `--model-loader-extra-config`        | 加载器扩展配置          | 向具体 Loader 传递额外参数  |
| `--ignore-patterns`                  | 忽略文件规则           | 加载模型时跳过匹配文件        |
| `--use-tqdm-on-load`                 | 加载进度条            | 控制加载进度显示           |
| `--pt-load-map-location`             | PyTorch 设备映射     | 控制 checkpoint 加载设备 |
| `--shuffle-safetensors-files`        | 文件随机打乱           | 分布式加载时改善 I/O 负载分布  |

为什么需要 LoadConfig？

大型模型权重可能包含数十甚至数百个文件，加载过程涉及：

1. 文件下载与缓存。
2. Safetensors/PyTorch 文件读取。
3. 权重映射与设备转换。
4. GPU 显存分配。
5. 多 GPU 权重分发。

LoadConfig 主要影响模型启动阶段，而不是每次 Token Decode 的计算逻辑。

### 2.5 AttentionConfig：Attention 后端

职责： 选择 Attention 计算实现，控制 Attention Kernel 的执行策略。

核心参数：

| 参数                               | 作用                       |
| -------------------------------- | ------------------------ |
| `--attention-backend`            | 指定 Attention 后端          |
| `--attention-config`             | 传递 Attention 相关配置        |
| `flash_attn_version`             | FlashAttention 版本控制      |
| `backend_per_kind`               | 不同 Attention 类型选择不同后端    |
| `use_trtllm_attention`           | 控制是否使用 TRT-LLM Attention |
| `mla_prefill_backend`            | MLA Prefill 后端           |
| `use_prefill_query_quantization` | Prefill Query 量化策略       |
| `indexer_kv_dtype`               | Indexer KV 数据类型          |
| `use_non_causal`                 | 控制非因果 Attention 行为       |

常见后端及其特点：

- FlashAttention：优化 Attention 的访存与计算效率。
- FlashInfer：面向高性能推理的 Kernel 库。
- Triton：通过 Triton 编写和编译 GPU Kernel。
- TRT-LLM：使用 TensorRT-LLM 相关实现。

注意：具体可用后端取决于 GPU 架构、模型 Attention 类型、数据类型和 vLLM 当前分支支持情况。

### 2.6 MambaConfig：状态空间模型配置

职责： 配置 Mamba 等 SSM 模型的计算后端与状态缓存。

| 参数                                         | 作用                        |
| ------------------------------------------ | ------------------------- |
| `--mamba-backend`                          | Mamba 计算后端                |
| `--mamba-ssu-algorithm`                    | Selective State Update 算法 |
| `--mamba-prefill-backend`                  | Prefill 阶段 SSD Scan 后端    |
| `--enable-mamba-cache-stochastic-rounding` | 启用随机舍入                    |
| `--mamba-cache-philox-rounds`              | Philox 随机数生成轮数            |

与 Transformer 的 KV Cache 不同，Mamba 主要依赖状态空间状态缓存。随机舍入用于降低 FP16 状态缓存写入时的舍入偏差。

### 2.7 StructuredOutputsConfig：结构化输出

职责： 保证模型输出符合指定格式或语法约束。

| 参数                            | 作用        |
| ----------------------------- | --------- |
| `--structured-outputs-config` | 结构化输出整体配置 |
| `--reasoning-parser`          | 推理内容解析器   |
| `--additional-config`         | 平台相关扩展配置  |

主要应用：

- JSON Schema 约束输出。
- Grammar 约束输出。
- 正则表达式约束。
- Reasoning 内容解析。

`backend` 用于选择约束解码后端，`disable_additional_properties` 等配置则影响 JSON Schema 的约束行为。

### 2.8 ParallelConfig：并行配置

职责： 控制模型在单 GPU、多 GPU、多节点之间如何分布执行。

这是 vLLM 最重要的配置组之一。

| 并行方式                  | 核心参数                       | 作用                      |
| --------------------- | -------------------------- | ----------------------- |
| Tensor Parallel       | `--tensor-parallel-size`   | 将单层模型计算拆分到多 GPU         |
| Pipeline Parallel     | `--pipeline-parallel-size` | 将不同模型层分配给不同 GPU         |
| Data Parallel         | `--data-parallel-size`     | 多个模型副本处理不同请求            |
| Context Parallel      | Prefill/Decode CP 参数       | 沿上下文维度分割计算              |
| Expert Parallel       | `--enable-expert-parallel` | 将 MoE Experts 分布到不同 GPU |
| Expert Load Balancing | EPLB 参数                    | 动态或策略性调整专家负载            |

其他重要参数：

| 参数                               | 作用                             |
| -------------------------------- | ------------------------------ |
| `--distributed-executor-backend` | 分布式执行器选择，如 Ray、Multiprocessing |
| `--nnodes`                       | 节点数量                           |
| `--node-rank`                    | 当前节点编号                         |
| `--master-addr`                  | 分布式主节点地址                       |
| `--master-port`                  | 分布式初始化端口                       |
| `--distributed-timeout-seconds`  | 分布式通信超时                        |
| `--all2all-backend`              | MoE All-to-All 通信后端            |
| `--enable-dbo`                   | 启用 Dual Batch Overlap          |
| `--ubatch-size`                  | 微批次配置                          |
| `--enable-elastic-ep`            | 启用弹性专家并行                       |
| `--max-parallel-loading-workers` | 并行权重加载 Worker 数量               |

需要区分：

- TP：解决单层模型太大、单卡无法容纳或计算的问题。
- PP：解决模型层数较多时跨设备分配的问题。
- DP：通过多个模型副本提升请求处理能力。
- EP：将 MoE 专家分布到不同设备。
- CP：分割上下文维度，支持更长上下文和分布式 Attention。

### 2.9 CacheConfig：缓存配置

职责： 管理 KV Cache 的分配、数据类型、块管理、前缀复用和卸载。

| 参数                           | 作用              | 影响               |
| ---------------------------- | --------------- | ---------------- |
| `--block-size`               | KV Block 大小     | 内存管理粒度           |
| `--gpu-memory-utilization`   | GPU 显存使用比例      | GPU 内存预算         |
| `--kv-cache-memory-bytes`    | KV Cache 显存预算   | Cache 实际容量       |
| `--kv-cache-dtype`           | KV Cache 数据类型   | Cache 显存与精度      |
| `--enable-prefix-caching`    | 前缀缓存            | 复用相同 Prompt 的 KV |
| `--prefix-caching-hash-algo` | 前缀 Hash 算法      | Prefix Block 标识  |
| `--swap-space`               | CPU Swap 空间     | 缓存交换能力           |
| `--cpu-offload-gb`           | CPU Offload 容量  | 显存与内存交换          |
| `--enable-chunked-prefill`   | 分块 Prefill      | 控制长 Prompt 处理    |
| `--num-gpu-blocks-override`  | 覆盖 GPU Block 数量 | 调试和测试            |
| `--mamba-cache-dtype`        | Mamba Cache 类型  | SSM 状态缓存精度       |

重点理解：

KV Cache 是推理过程中保存历史 Token 的 K、V 张量，避免 Decode 阶段重复计算历史 Token 的 Attention。

显存预算可粗略理解为：

\\[ M\_{GPU}=M\_{Weights}+M\_{KV}+M\_{Activations}+M\_{Other} \\]

其中 `gpu-memory-utilization` 是整体显存预算控制，`kv-cache-memory-bytes` 是对 KV Cache 容量的直接控制，两者不能简单理解成两个可以独立叠加的显存池。

### 2.10 OffloadConfig：卸载配置

职责： 将部分模型权重或状态从 GPU 转移到 CPU 等存储位置，缓解显存压力。

主要涉及：

- `offload-backend`：卸载后端。
- `cpu-offload-gb`：CPU Offload 容量。
- Prefetch Offload：预取卸载数据。
- UVA Offload：利用统一虚拟寻址等机制访问卸载数据。

适用于 GPU 显存不足的场景，但可能增加 CPU-GPU 数据传输开销。

### 2.11 MultiModalConfig：多模态配置

职责： 控制图片、视频、音频等输入的预处理和编码过程。

主要配置方向：

| 参数类别            | 作用             |
| --------------- | -------------- |
| 图片限制            | 最大图片数量、分辨率等    |
| 视频限制            | 视频帧数、解码参数      |
| 音频限制            | 音频长度、解码和预处理    |
| Processor Cache | 多模态预处理缓存       |
| MM Encoder      | 多模态 Encoder 并行 |
| Media Cache     | 媒体数据缓存         |
| IPC             | 多模态数据进程间传递     |

多模态模型的资源消耗不仅包括语言模型 KV Cache，还包括图像、视频和音频 Encoder 的计算及中间结果。

### 2.12 LoRAConfig：LoRA 配置

职责： 支持基础模型加载后动态使用多个 LoRA 适配器。

| 参数                        | 作用            |
| ------------------------- | ------------- |
| `--enable-lora`           | 启用 LoRA       |
| `--max-loras`             | 最大并发 LoRA 数量  |
| `--max-lora-rank`         | 最大 LoRA 秩     |
| `--max-cpu-loras`         | CPU 侧 LoRA 数量 |
| `--lora-modules`          | 注册 LoRA 模块    |
| `--lora-extra-vocab-size` | 额外词表空间        |
| `--fully-sharded-loras`   | LoRA 权重分片     |
| `--max-lora-chunk-size`   | LoRA 分块大小     |

用途：一个基础模型服务多个微调任务，减少重复加载完整模型的成本。

### 2.13 LoggingConfig：日志配置

职责： 控制 vLLM 内部日志输出。

主要配置方向：

- 日志级别：DEBUG、INFO、WARNING、ERROR。
- 日志格式：文本格式、结构化格式。
- 日志文件：日志配置文件及输出位置。
- 请求日志：请求内容、参数和请求 ID。
- 统计日志：吞吐量、Token 数、延迟等。

### 2.14 ObservabilityConfig：可观测性

职责： 提供运行时性能指标和分布式追踪。

主要监控内容：

| 指标                | 作用               |
| ----------------- | ---------------- |
| TTFT              | 首 Token 延迟       |
| TPOT              | 每输出 Token 时间     |
| E2E Latency       | 请求端到端延迟          |
| Running Requests  | 正在运行的请求数         |
| Waiting Requests  | 等待调度的请求数         |
| GPU Cache Usage   | GPU KV Cache 使用率 |
| Prompt Tokens     | 输入 Token 数       |
| Generation Tokens | 输出 Token 数       |

Tracing 可以进一步记录请求在 Frontend、Scheduler、Worker 等模块的处理时间。

### 2.15 SchedulerConfig：请求调度配置

职责： 控制请求进入模型执行阶段的调度方式。

| 参数                               | 作用                     |
| -------------------------------- | ---------------------- |
| `--max-num-seqs`                 | 最大并发序列数量               |
| `--max-num-batched-tokens`       | 每轮调度的 Token 预算         |
| `--max-num-scheduled-tokens`     | 每次调度 Token 数量上限        |
| `--max-num-partial-prefills`     | 并行进行的部分 Prefill 数量     |
| `--max-long-partial-prefills`    | 长 Prompt 部分 Prefill 数量 |
| `--long-prefill-token-threshold` | 长 Prompt 判定阈值          |
| `--scheduling-policy`            | 调度策略                   |
| `--enable-chunked-prefill`       | 分块 Prefill             |
| `--preemption-mode`              | 抢占方式                   |

核心原理：

Scheduler 决定每轮哪些请求可以进入执行，以及每个请求分配多少 Token 计算预算。

主要影响：

- 请求吞吐量。
- 请求等待时间。
- GPU 利用率。
- Prefill 和 Decode 的资源竞争。

### 2.16 CompilationConfig：编译配置

职责： 控制模型计算图编译、算子优化以及 CUDA Graph 捕获。

主要配置方向：

| 参数                           | 作用                       |
| ---------------------------- | ------------------------ |
| `--compilation-config`       | 编译总配置                    |
| `mode`                       | 编译模式                     |
| `backend`                    | 编译后端                     |
| `custom_ops`                 | 自定义算子选择                  |
| `splitting_ops`              | 图拆分算子                    |
| `cudagraph_mode`             | CUDA Graph 模式            |
| `cudagraph_capture_sizes`    | CUDA Graph 捕获 Batch Size |
| `max_cudagraph_capture_size` | 最大捕获规模                   |
| `cudagraph_num_of_warmups`   | Graph 预热次数               |
| `compile_sizes`              | 编译使用的输入规模                |
| `inductor_compile_config`    | Torch Inductor 编译配置      |
| `inductor_passes`            | 编译优化 Pass                |

为什么需要 CUDA Graph？

普通 CUDA 执行过程中，CPU 需要不断提交 Kernel，存在 CPU Launch Overhead。

CUDA Graph 将一组 GPU 操作捕获成计算图，后续通过 Graph Replay 减少重复提交开销。

### 2.17 KernelConfig：算子配置

职责： 选择具体算子的实现后端和优化策略。

主要包括：

- Attention Kernel。
- MoE Kernel。
- Linear Kernel。
- RMSNorm Kernel。
- All-Reduce / All-to-All 通信算子。
- Triton / FlashInfer 等实现选择。
- Kernel Autotuning 和 Warmup。

注意：AttentionConfig 和 KernelConfig 不完全相同。前者主要决定 Attention 计算后端，后者是更广泛的 Kernel 实现与优化配置。

### 2.18 VllmConfig：全局配置

职责： 将模型、缓存、调度、并行、编译等配置组合成完整的运行时配置。

它不是单纯的另一个 CLI 参数集合，而是 Engine Runtime 使用的配置聚合对象。

典型组成：

```
VllmConfig(    model_config=...,    cache_config=...,    parallel_config=...,    scheduler_config=...,    load_config=...,    attention_config=...,    compilation_config=...,    kernel_config=...,    observability_config=...,    logging_config=...,)
```

具体字段以当前分支 `vllm/config/vllm.py` 为准。

以上配置组的展开依据是原文的分组，以及分支 `arg_utils.py` 中的参数定义。

[image](https://www.google.com/s2/favicons?domain=https://raw.githubusercontent.com\&sz=32)

GitHub+2



## 三、Engine 参数展开

`EngineArgs` 和 `AsyncEngineArgs` 是 Engine 配置的两个入口。

| 类                 | 使用场景 | 说明              |
| ----------------- | ---- | --------------- |
| `EngineArgs`      | 离线推理 | 为 `LLM()` 提供配置  |
| `AsyncEngineArgs` | 异步推理 | 为异步 Engine 提供配置 |

两者共享大量配置定义，但使用场景不同。

### 3.1 EngineArgs 的主要参数类别

| 类别        | 主要参数                                        | 用途              |
| --------- | ------------------------------------------- | --------------- |
| 模型        | model、tokenizer、dtype                       | 模型初始化           |
| 加载        | load_format、download_dir                    | 模型权重加载          |
| Cache     | kv_cache_dtype、block_size                   | KV Cache        |
| 并行        | tensor_parallel_size、pipeline_parallel_size | 分布式执行           |
| 调度        | max_num_seqs、max_num_batched_tokens         | 请求调度            |
| 编译        | cudagraph_capture_sizes                     | CUDA Graph      |
| Attention | attention_config                            | Attention 后端    |
| LoRA      | lora_config                                 | 适配器管理           |
| 多模态       | multimodal_config                           | 多模态输入           |
| 可观测性      | observability_config                        | Metrics、Tracing |
| 日志        | logging_config                              | 日志管理            |

### 3.2 EngineArgs 与 vllm serve 的关系

```
# 离线推理from vllm import LLMllm = LLM(    model="Qwen/Qwen3-8B",    tensor_parallel_size=2,    max_model_len=8192,    gpu_memory_utilization=0.9,)outputs = llm.generate(["Hello"])
```

```
# 在线服务
vllm serve Qwen/Qwen3-8B \
    --tensor-parallel-size 2 \
    --max-model-len 8192 \
    --gpu-memory-utilization 0.9
```

两种方式最终都会进入 Engine 配置体系，只是外部调用方式不同。

### 3.3 JSON 配置参数

对于复杂配置对象，可以使用 JSON 传参。

例如：

```
vllm serve Qwen/Qwen3-8B \
  --compilation-config '{"mode":3,"cudagraph_capture_sizes":[1,2,4,8]}'
```

也支持展开形式：

```
vllm serve Qwen/Qwen3-8B \
  --compilation-config.mode 3 \
  --compilation-config.cudagraph_capture_sizes 1 2 4 8
```

这种机制允许将配置类中的嵌套字段直接映射为 CLI 参数。

[image](https://www.google.com/s2/favicons?domain=https://raw.githubusercontent.com\&sz=32)

GitHub+1



## 四、YAML 配置展开

vLLM 支持通过 YAML 统一管理服务配置。

### 4.1 配置示例

```
model: Qwen/Qwen3-8B

# Frontend
host: "0.0.0.0"
port: 8000
served-model-name: qwen3

# ModelConfig
dtype: bfloat16
max-model-len: 8192
trust-remote-code: true

# ParallelConfig
tensor-parallel-size: 1
pipeline-parallel-size: 1

# CacheConfig
gpu-memory-utilization: 0.85
kv-cache-dtype: auto
enable-prefix-caching: true

# SchedulerConfig
max-num-seqs: 16
max-num-batched-tokens: 8192

# CompilationConfig
compilation-config:
  mode: 3
```

### 4.2 配置优先级

按照原文：

1. 命令行参数优先于 YAML 参数。
2. YAML 未指定的参数使用默认值。

例如：

```
vllm serve --config config.yaml --port 9000
```

如果 YAML 中配置了 `port: 8000`，最终使用命令行指定的 9000。

## 五、环境变量展开

环境变量与 CLI 参数的区别：

- CLI：主要配置某次 Engine 或 Server 实例。
- 环境变量：控制进程级别的运行环境、底层组件行为及平台实现。

当前分支的 `vllm/envs.py` 定义了大量环境变量，涉及 CUDA、NCCL、日志、分布式通信、编译、多模态等。

[image](https://www.google.com/s2/favicons?domain=https://raw.githubusercontent.com\&sz=32)

GitHub



### 5.1 环境变量分类

| 类别          | 代表变量                           | 用途               |
| ----------- | ------------------------------ | ---------------- |
| 网络          | `VLLM_HOST_IP`                 | vLLM 内部通信 IP     |
| 网络          | `VLLM_PORT`                    | 内部通信端口           |
| 网络          | `VLLM_RPC_BASE_PATH`           | RPC 相关路径         |
| GPU         | `CUDA_VISIBLE_DEVICES`         | 指定可见 GPU         |
| GPU         | `VLLM_TARGET_DEVICE`           | 目标计算设备           |
| GPU         | `VLLM_NCCL_SO_PATH`            | NCCL 动态库路径       |
| Cache       | `VLLM_CACHE_ROOT`              | 缓存根目录            |
| Cache       | `VLLM_ASSETS_CACHE`            | 资源缓存             |
| Logging     | `VLLM_LOGGING_LEVEL`           | 日志级别             |
| Logging     | `VLLM_LOG_STATS_INTERVAL`      | 统计日志周期           |
| Distributed | `LOCAL_RANK`                   | 本地进程 Rank        |
| Distributed | `VLLM_DP_RANK`                 | DP Rank          |
| Distributed | `VLLM_DP_SIZE`                 | DP 总规模           |
| Compilation | `VLLM_USE_AOT_COMPILE`         | AOT 编译控制         |
| Compilation | `VLLM_DISABLE_COMPILE_CACHE`   | 关闭编译缓存           |
| Compilation | `VLLM_COMPILE_ON_ONE_RANK`     | 单 Rank 编译控制      |
| Multimodal  | `VLLM_IMAGE_FETCH_TIMEOUT`     | 图片获取超时           |
| Multimodal  | `VLLM_VIDEO_FETCH_TIMEOUT`     | 视频获取超时           |
| Multimodal  | `VLLM_AUDIO_FETCH_TIMEOUT`     | 音频获取超时           |
| Backend     | `VLLM_USE_RUST_FRONTEND`       | Rust Frontend 控制 |
| Backend     | `VLLM_WORKER_MULTIPROC_METHOD` | Worker 多进程启动方式   |

### 5.2 特别注意：内部通信地址

原文特别强调：

```
VLLM_HOST_IP
VLLM_PORT
```

这两个变量用于 vLLM 内部通信，不是 API Server 的监听地址和端口。

例如：

```
export VLLM_HOST_IP=192.168.1.100
export VLLM_PORT=29500

vllm serve Qwen/Qwen3-8B \
    --host 0.0.0.0 \
    --port 8000
```

这里：

- `VLLM_HOST_IP`、`VLLM_PORT`：内部通信环境。
- `--host`、`--port`：对外 HTTP 服务监听。

两者不能混淆。

### 5.3 环境变量的使用方式

```
# 日志
export VLLM_LOGGING_LEVEL=DEBUG

# 指定 GPU
export CUDA_VISIBLE_DEVICES=0,1

# 缓存目录
export VLLM_CACHE_ROOT=/data/vllm

# 编译缓存
export VLLM_DISABLE_COMPILE_CACHE=0

# 启动服务
vllm serve Qwen/Qwen3-8B
```

## 六、配置参数之间的关联

这是配置文档中建议重点补充的部分。单独理解某个参数并不足够，实际部署中多个配置会共同影响推理性能。

| 关联关系               | 参数组合                                          | 影响              |
| ------------------ | --------------------------------------------- | --------------- |
| 模型与显存              | dtype + quantization + gpu-memory-utilization | 权重显存占用          |
| 上下文与缓存             | max-model-len + kv-cache-dtype                | KV Cache 需求     |
| 并发与缓存              | max-num-seqs + max-num-batched-tokens         | 请求并发与显存压力       |
| 并行与通信              | TP + PP + DP + EP                             | GPU 分布和通信开销     |
| Attention 与 Kernel | attention-backend + kernel-config             | 算子执行性能          |
| 编译与执行              | compilation-config + cudagraph                | 启动时间与推理性能       |
| 多模态与显存             | multimodal-config + KV Cache                  | Encoder 和媒体处理资源 |
| 调度与延迟              | scheduler-config + chunked-prefill            | TTFT、TPOT 和吞吐   |

### 6.1 一个实际部署配置

例如单机双 GPU 部署 Qwen 模型：

```
vllm serve Qwen/Qwen3-8B \
  --host 0.0.0.0 \
  --port 8000 \
  --dtype bfloat16 \
  --tensor-parallel-size 2 \
  --max-model-len 8192 \
  --gpu-memory-utilization 0.85 \
  --max-num-seqs 16 \
  --max-num-batched-tokens 8192 \
  --enable-prefix-caching
```

参数配置逻辑：

1. ModelConfig：指定模型、精度和上下文长度。
2. ParallelConfig：使用两张 GPU 做 TP。
3. CacheConfig：确定 KV Cache 显存预算及前缀缓存。
4. SchedulerConfig：限制并发序列和每轮 Token 预算。
5. Frontend：提供 HTTP 服务。

## 七、建议补充的源码索引

如果这份文档同时承担源码学习入口的作用，建议增加以下源码索引。

| 文件                           | 核心内容                                |
| ---------------------------- | ----------------------------------- |
| `vllm/engine/arg_utils.py`   | EngineArgs、AsyncEngineArgs、CLI 参数解析 |
| `vllm/config/vllm.py`        | VllmConfig 全局配置                     |
| `vllm/config/model.py`       | ModelConfig                         |
| `vllm/config/cache.py`       | CacheConfig                         |
| `vllm/config/parallel.py`    | ParallelConfig                      |
| `vllm/config/scheduler.py`   | SchedulerConfig                     |
| `vllm/config/compilation.py` | CompilationConfig                   |
| `vllm/config/attention.py`   | AttentionConfig                     |
| `vllm/config/load.py`        | LoadConfig                          |
| `vllm/config/multimodal.py`  | MultiModalConfig                    |
| `vllm/config/kernel.py`      | KernelConfig                        |
| `vllm/envs.py`               | 环境变量定义                              |

建议阅读顺序：

```
arg_utils.py
    │
    ├── EngineArgs
    ├── AsyncEngineArgs
    ├── 参数解析 get_kwargs()
    │
    ▼
vllm/config/
    │
    ├── ModelConfig
    ├── ParallelConfig
    ├── CacheConfig
    ├── SchedulerConfig
    ├── CompilationConfig
    │
    ▼
vllm/config/vllm.py
    │
    ▼
VllmConfig
    │
    ▼
Engine Runtime
```

源码中的参数定义和说明应优先于人工整理的文档。特别是默认值、参数类型、平台支持和弃用状态，需要以目标分支为准。

## 八、参考文档与维护建议

以下是这次扩展所依据的资料：

1. [你提供的 configuration.md](https://github.com/neoguojing/vllm/blob/codespace-scaling-halibut-7q499q7wxqcwq57/docs/zh/configuration.md?utm_source=chatgpt.com)：文档结构与配置分组。
2. [arg_utils.py](https://github.com/neoguojing/vllm/blob/codespace-scaling-halibut-7q499q7wxqcwq57/vllm/engine/arg_utils.py?utm_source=chatgpt.com)：CLI 参数和 EngineArgs 定义。
3. [config 源码目录](https://github.com/neoguojing/vllm/blob/codespace-scaling-halibut-7q499q7wxqcwq57/vllm/config/?utm_source=chatgpt.com)：各配置类的详细定义。
4. [envs.py](https://github.com/neoguojing/vllm/blob/codespace-scaling-halibut-7q499q7wxqcwq57/vllm/envs.py?utm_source=chatgpt.com)：环境变量定义。
5. [官方 Serve CLI Reference](https://docs.vllm.ai/en/latest/cli/serve/?utm_source=chatgpt.com)：参数说明和用法。
6. [官方 Engine Arguments](https://docs.vllm.ai/en/latest/configuration/engine_args/?utm_source=chatgpt.com)：EngineArgs 和 AsyncEngineArgs。

有一个重要的完整性说明： 上面已经展开配置体系、主要参数及核心关联，但尚不是该分支全部 CLI 字段逐项逐字展开的最终版本。尤其是 Frontend、ParallelConfig、MultiModalConfig 和 CompilationConfig 中，还有大量平台相关及实验性参数，不能将本稿中的代表性参数表当成完整的 `--help` 参数清单。

我的建议是保留原文自动生成机制，把自动生成的参数表完整展开，再将上面这些原理、用途和关联说明放在各组参数表前面。这样既能保持与源码同步，又能让文档真正成为可用于源码阅读、部署和面试复习的配置手册。
