
下面基于你上传的 **2348 行&#x20;**`GPUModelRunner`**&#x20;源码**整理成一份完整的 GitHub Markdown 技术文档。

文档重点不是逐行翻译代码，而是从源码中提炼出：**架构定位、核心类关系、初始化流程、KV Cache 初始化、模型执行流程、采样流程、并行机制、关键数据结构以及源码阅读路线**。图全部使用 ASCII 文本，可以直接在 GitHub Markdown 中展示。

# vLLM GPUModelRunner 源码分析

> 源码文件：`vllm/v1/worker/gpu/model_runner.py`\
> 核心类：`GPUModelRunner`\
> 主要职责：管理 GPU 上的模型执行、输入准备、Attention 元数据构建、KV Cache 访问、模型前向计算、Token 采样及请求状态更新。

---

## 1. GPUModelRunner 的架构定位

### 1.1 核心职责

`GPUModelRunner` 是 vLLM V1 GPU Worker 内部负责模型执行的核心组件。

它不是 Scheduler，也不是模型本身，而是连接调度结果和 GPU 模型执行的运行时管理器。

主要职责包括：

1. 加载模型及初始化模型运行状态。
2. 初始化 GPU KV Cache、Attention Backend 和 BlockTables。
3. 根据 Scheduler 输出准备 Batch 输入。
4. 构建 Attention Metadata 和 Slot Mapping。
5. 选择 CUDA Graph、UBatch 或 Eager 模式执行模型。
6. 执行模型前向计算，得到 Hidden States。
7. 执行 Logits 计算和 Token Sampling。
8. 更新请求状态、KV Cache 相关状态及 Speculative Decoding 状态。
9. 支持 Pipeline Parallel、Data Parallel、Context Parallel、Expert Parallel 等并行能力。
10. 支持多模态、LoRA、Pooling、Speculative Decoding 等模型执行场景。

### 1.2 在 vLLM 中的位置

```
                         API / Engine
                              |
                              v
                         Scheduler
                              |
                              | SchedulerOutput
                              v
                         GPUWorker
                              |
                              v
                       GPUModelRunner
                              |
             +----------------+----------------+
             |                |                |
             v                v                v
         InputBatch       BlockTables       ModelState
             |                |                |
             v                v                v
       Input Preparation  KV Cache       Model Inputs
             |                |                |
             +----------------+----------------+
                              |
                              v
                       Attention Metadata
                              |
                              v
                       Model Forward
                              |
                              v
                         Hidden States
                              |
                              v
                           Sampler
                              |
                              v
                       Sampled Tokens
                              |
                              v
                     Request State Update
                              |
                              v
                       ModelRunnerOutput
```

需要注意：

- Scheduler 决定本轮哪些请求执行，以及每个请求执行多少 Token。
- GPUWorker 负责 Worker 级别的执行入口和 GPU 运行环境协调。
- GPUModelRunner 负责将调度结果转化为 GPU 模型执行所需要的数据和操作。
- 模型负责实际的 Transformer 前向计算。
- Sampler 负责根据模型输出生成 Token。

### 1.3 GPUModelRunner 不负责什么

| 组件                | 职责                      | 是否属于 GPUModelRunner |
| ----------------- | ----------------------- | ------------------- |
| Scheduler         | 请求调度、Token 预算分配         | 否                   |
| GPUWorker         | GPU Worker 入口和设备级管理     | 否                   |
| GPUModelRunner    | GPU 模型执行管理              | 是                   |
| Transformer Model | 模型网络结构和前向计算             | 外部模型对象              |
| Attention Backend | Attention Kernel 和元数据执行 | 外部组件                |
| KV Cache          | 保存历史 K/V 或其他模型状态        | 外部存储组件              |
| Sampler           | Logits 处理和 Token 采样     | Runner 管理的组件        |

---

## 2. 整体架构设计

GPUModelRunner 本身是一个大型协调类，但它并不直接实现所有功能。

从源码 Import 可以看到，大量能力被拆分到独立模块中。

### 2.1 核心模块划分

| 模块           | 关键类 / 函数                    | 职责                      |
| ------------ | --------------------------- | ----------------------- |
| 请求状态         | `RequestState`              | 保存请求运行时状态               |
| 输入管理         | `InputBuffers`、`InputBatch` | 管理本轮模型输入                |
| Batch 请求状态   | `BatchReqState`             | 保存 CPU 侧 Batch 请求信息     |
| KV Cache     | `BlockTables`               | 管理请求与 KV Block 的映射      |
| Attention    | `init_attn_backend`         | 初始化 Attention Backend   |
| 模型状态         | `init_model_state`          | 初始化模型特定执行状态             |
| CUDA Graph   | `ModelCudaGraphManager`     | 管理 CUDA Graph           |
| 模型加载         | `get_model_loader`          | 加载模型                    |
| 采样           | `Sampler`                   | Token 采样                |
| 拒绝采样         | `RejectionSampler`          | Speculative Decoding 验证 |
| 结构化输出        | `StructuredOutputsWorker`   | Grammar 约束              |
| 流水线并行        | `PPHandler`                 | PP 中间数据传递               |
| Batch 分片     | `BatchSharder`              | 分片 Logits 计算            |
| 双 Batch 执行   | `UBatchRunner`              | Micro-batch 执行          |
| KV Connector | `KVConnector`               | KV Cache 外部连接和传输        |
| 多模态          | `EncoderCache`              | 多模态 Encoder Cache       |
| 专家负载均衡       | `EPLBController`            | Expert Parallel 负载均衡    |
| 投机解码         | `DraftModelSpeculator`      | Draft Token 生成          |

### 2.2 架构关系图

```
GPUModelRunner
|
+-- Configuration
|   +-- VllmConfig
|   +-- ModelConfig
|   +-- CacheConfig
|   +-- ParallelConfig
|   +-- SchedulerConfig
|   +-- CompilationConfig
|   +-- SpeculativeConfig
|
+-- Request Management
|   +-- RequestState
|   +-- InputBuffers
|   +-- InputBatch
|   +-- BatchReqState
|
+-- Model Execution
|   +-- model
|   +-- ModelState
|   +-- Attention Backend
|   +-- Attention Metadata
|   +-- BlockTables
|   +-- KV Cache
|
+-- Execution Optimization
|   +-- ModelCudaGraphManager
|   +-- UBatchRunner
|   +-- FastPrefillHelper
|
+-- Sampling
|   +-- Sampler
|   +-- RejectionSampler
|   +-- PromptLogprobsWorker
|   +-- StructuredOutputsWorker
|
+-- Parallelism
|   +-- PPHandler
|   +-- BatchSharder
|   +-- PCPManager
|   +-- EPLBController
|
+-- Advanced Features
    +-- Speculator
    +-- KVConnector
    +-- EncoderCache
    +-- LoRA State
    +-- PoolingRunner
```

设计特点：

**GPUModelRunner 负责组织执行流程，具体功能通过独立组件实现。**

例如：

- Attention Kernel 不在 GPUModelRunner 中实现。
- CUDA Graph 的捕获和 Replay 不在 GPUModelRunner 中实现。
- 具体模型结构不在 GPUModelRunner 中实现。
- 采样算法由 Sampler 实现。
- KV Block 映射由 BlockTables 管理。

---

## 3. 核心类关系

### 3.1 GPUModelRunner 类定义

```
class GPUModelRunner(LoRAModelRunnerMixin):
```

继承 `LoRAModelRunnerMixin`，主要用于复用 LoRA 相关能力。

GPUModelRunner 通过组合方式持有大部分功能组件。

### 3.2 核心类图

```
                  LoRAModelRunnerMixin
                           |
                           |
                           v
                    GPUModelRunner
                           |
       +-------------------+-------------------+
       |                   |                   |
       v                   v                   v
   RequestState         ModelState          Sampler
       |                   |                   |
       v                   v                   v
   InputBatch          Model Forward       Sampling
       |
       v
   BlockTables
       |
       v
    KV Cache


                    GPUModelRunner
                           |
       +-------------------+-------------------+
       |                   |                   |
       v                   v                   v
 ModelCudaGraphManager  PPHandler          UBatchRunner
       |                   |                   |
       v                   v                   v
 CUDA Graph             Pipeline           Micro Batch
                         Parallel           Execution


                    GPUModelRunner
                           |
       +-------------------+-------------------+
       |                   |                   |
       v                   v                   v
   Speculator          KVConnector       EncoderCache
       |                   |                   |
       v                   v                   v
 Speculative            KV Transfer       Multimodal
 Decoding                                  Embeddings
```

### 3.3 关键数据结构

#### RequestState

```
self.req_states = RequestState(
    max_num_reqs=self.max_num_reqs,
    max_model_len=self.max_model_len,
    max_num_batched_tokens=self.max_num_tokens,
    num_speculative_steps=self.num_speculative_steps,
    vocab_size=self.vocab_size,
    device=self.device,
    num_prefill_lookahead=num_prefill_lookahead,
)
```

负责保存请求的长期运行状态。

主要包括：

- 已经计算的 Token 数量。
- 请求 Token ID。
- 最近采样的 Token。
- Prompt 长度。
- 请求运行过程中的 Draft Token。
- Speculative Decoding 相关状态。

可以理解为：

```
RequestState
    |
    +-- Request A
    |     +-- Token IDs
    |     +-- Computed Tokens
    |     +-- Last Sampled Token
    |     +-- Draft Tokens
    |
    +-- Request B
          +-- Token IDs
          +-- Computed Tokens
          +-- Last Sampled Token
          +-- Draft Tokens
```

它是跨多个执行 Step 保存的请求状态，而不是当前 Batch 的临时输入。

#### InputBuffers

```
self.input_buffers = InputBuffers(
    max_num_reqs=self.max_num_reqs,
    max_num_tokens=self.max_num_tokens,
    device=self.device,
)
```

负责维护输入准备过程中使用的预分配 Buffer。

它主要用于避免每个执行 Step 都重新申请大量 GPU Tensor。

#### InputBatch

`InputBatch` 是本轮实际送入模型执行的 Batch 数据描述。

它将多个请求组织成一次模型执行所需的数据结构。

例如：

```
InputBatch
    |
    +-- req_ids
    +-- idx_mapping
    +-- num_tokens
    +-- query_start_loc
    +-- logits_indices
    +-- input token information
    +-- padding information
```

它与 RequestState 的区别：

| 数据结构          | 生命周期          | 作用          |
| ------------- | ------------- | ----------- |
| RequestState  | 跨多个 Step      | 保存请求长期状态    |
| InputBuffers  | Runner 生命周期   | 复用输入 Buffer |
| InputBatch    | 当前执行 Step     | 描述本轮 Batch  |
| BatchReqState | 当前 Batch 准备阶段 | CPU 侧请求汇总信息 |

#### BlockTables

```
self.block_tables = BlockTables(...)
```

负责管理请求与 KV Cache Block 的映射关系。

```
Request A
    |
    v
Logical Block Table
    |
    +----> Physical Block 5
    +----> Physical Block 8
    +----> Physical Block 12

Request B
    |
    v
Logical Block Table
    |
    +----> Physical Block 2
    +----> Physical Block 9
```

需要区分：

- KV Cache：实际保存 K/V 数据的 GPU Tensor。
- BlockTables：描述请求使用哪些物理 Block。
- Slot Mapping：描述当前 Token 应该写入哪个 KV Cache 位置。

三者并不是同一个东西。

#### ModelState

通过：

```
self.model_state = init_model_state(
    self.vllm_config,
    self.model,
    self.encoder_cache,
    self.device,
)
```

初始化。

ModelState 封装模型执行过程中的模型特定逻辑。

例如：

- 输入准备。
- Attention Metadata 准备。
- 模型状态预处理。
- 模型状态后处理。
- 多模态相关处理。
- 模型特定采样行为。

设计目的：

让通用 GPUModelRunner 不需要直接包含大量模型特定逻辑。

---

## 4. GPUModelRunner 初始化流程

GPUModelRunner 的初始化分为几个阶段：

1. 构造 Runner。
2. 加载模型。
3. 初始化 ModelState 和 Sampler。
4. 初始化 KV Cache。
5. 执行 Profile 和 Warmup。
6. 捕获 CUDA Graph。

### 4.1 构造阶段：`__init__`

主要初始化：

```
__init__
    |
    +-- 保存 VllmConfig
    |
    +-- 初始化设备和模型参数
    |
    +-- 初始化并行配置
    |     +-- PP
    |     +-- DP
    |     +-- DCP
    |
    +-- 初始化请求状态
    |     +-- RequestState
    |     +-- InputBuffers
    |
    +-- 初始化多模态组件
    |     +-- EncoderCache
    |     +-- ECConnector
    |
    +-- 初始化投机解码组件
    |     +-- Speculator
    |     +-- DraftTokensHandler
    |
    +-- 初始化 LoRA 状态
    |
    +-- 初始化 EPLB
    |
    +-- 初始化 ExecuteModelState
```

注意：

此时并不意味着模型已经完成加载，也不意味着 KV Cache 已经分配。

部分组件需要等到 `load_model()` 或 `initialize_kv_cache()` 才能创建。

### 4.2 模型加载：`load_model()`

核心代码：

```
model_loader = get_model_loader(self.vllm_config.load_config)

self.model = model_loader.load_model(
    vllm_config=self.vllm_config,
    model_config=self.vllm_config.model_config,
)
```

模型加载后继续初始化：

```
self.model_state = init_model_state(
    self.vllm_config,
    self.model,
    self.encoder_cache,
    self.device,
)
```

然后初始化 Sampler：

```
self.sampler = Sampler(**sampler_kwargs)
```

如果启用了 Speculative Decoding：

```
self.rejection_sampler = RejectionSampler(
    self.sampler,
    self.speculative_config,
    self.device,
)
```

如果是 Pooling 模型：

```
self.pooling_runner = PoolingRunner(
    self.model,
    self.vllm_config,
)
```

整体流程：

```
load_model()
    |
    v
get_model_loader()
    |
    v
model_loader.load_model()
    |
    v
加载模型权重
    |
    v
LoRA 初始化（如果启用）
    |
    v
Speculator 初始化（如果启用）
    |
    v
init_model_state()
    |
    v
初始化 ModelState
    |
    v
初始化 Sampler
    |
    +----> RejectionSampler
    |
    +----> PromptLogprobsWorker
    |
    +----> StructuredOutputsWorker
    |
    +----> PoolingRunner（Pooling 模型）
    |
    v
初始化 PP Intermediate Tensors
    |
    v
模型加载完成
```

这里有一个重要区别：

**模型加载和 KV Cache 初始化是两个不同阶段。**

模型权重加载完成，并不代表运行时 KV Cache 已经完成初始化。

### 4.3 KV Cache 初始化：`initialize_kv_cache()`

核心入口：

```
initialize_kv_cache(
    kv_cache_config,
    is_profiling=False,
    kv_cache_allocation_context=None,
)
```

该方法负责建立模型运行所需要的 KV Cache 执行环境。

主要步骤：

```
initialize_kv_cache()
    |
    +-- 保存 KVCacheConfig
    |
    +-- 计算 Block Table 参数
    |
    +-- init_attn_backend()
    |      |
    |      +-- Attention Groups
    |      +-- Kernel Block Sizes
    |      +-- CUDA Graph Support
    |
    +-- 初始化 Adaptive Verification
    |
    +-- 创建 BlockTables
    |
    +-- 创建 PCPManager
    |
    +-- 创建 UBatchRunner
    |
    +-- 初始化 Mamba SSU Backend
    |
    +-- 确定 CUDA Graph Mode
    |
    +-- 创建 ModelCudaGraphManager
    |
    +-- 初始化 FastPrefill（如果需要）
    |
    +-- init_kv_cache()
    |      |
    |      +-- 分配实际 KV Cache Tensor
    |
    +-- 保存 KV Cache
    |
    +-- 初始化 KVConnector
```

核心代码：

```
self.block_tables = BlockTables(
    block_sizes=block_sizes,
    max_num_reqs=self.max_num_reqs,
    max_num_batched_tokens=self.max_num_tokens,
    max_num_blocks_per_group=max_num_blocks_per_group,
    device=self.device,
    kernel_block_sizes=self.kernel_block_sizes,
    slot_mapping_enabled=slot_mapping_enabled,
    cp_size=self.dcp_size,
    cp_rank=self.dcp_rank,
    cp_interleave=self.cp_interleave,
)
```

实际 Cache 初始化：

```
kv_caches_dict = init_kv_cache(
    self.compilation_config.static_forward_context,
    self.kv_cache_config,
    self.device,
    self.kernel_block_sizes,
    self.vllm_config,
    kv_cache_allocation_context=kv_cache_allocation_context,
    block_tables=self.block_tables,
)
```

最终：

```
self.kv_caches = [
    cache
    for cache in kv_caches_dict.values()
    if cache.device == self.device
]
```

### 4.4 KV Cache 相关对象的区别

| 对象                 | 职责                          |
| ------------------ | --------------------------- |
| KVCacheConfig      | 描述 KV Cache 的配置             |
| KV Cache Tensor    | 实际保存模型状态                    |
| BlockTables        | 管理逻辑请求到物理 Block 的映射         |
| Slot Mapping       | 当前 Token 对应的 Cache 写入位置     |
| Attention Metadata | 提供 Attention Kernel 执行所需的信息 |
| Attention Backend  | 负责具体 Attention 实现           |

可以将它们理解为：

```
Request
   |
   v
BlockTables
   |
   v
Physical KV Block
   |
   v
KV Cache Tensor
   |
   v
Attention Kernel
```

Slot Mapping 则用于定位当前 Token 在 KV Cache 中的写入位置。

---

## 5. 核心执行流程：`execute_model()`

这是 GPUModelRunner 最重要的方法之一。

它负责将 SchedulerOutput 转换为 GPU 模型执行。

### 5.1 输入和输出

输入：

```
execute_model(
    scheduler_output,
    intermediate_tensors=None,
    ...
)
```

核心输入是 `SchedulerOutput`。

它描述本轮 Scheduler 已经决定的执行任务。

例如：

```
SchedulerOutput
    |
    +-- req_id -> scheduled_tokens
    |
    +-- finished_req_ids
    |
    +-- total_num_scheduled_tokens
    |
    +-- KV Cache 相关调度信息
    |
    +-- 请求相关调度信息
```

输出：

- 最后一个 PP Rank：通常返回 `None`，将执行结果保存到 `execute_model_state`，供后续 `sample_tokens()` 使用。
- 非最后一个 PP Rank：返回 `IntermediateTensors`，用于 Pipeline Parallel 传递。

### 5.2 完整执行流程

```
                  SchedulerOutput
                         |
                         v
                   execute_model()
                         |
                         v
              更新 Pipeline Decode 请求
                         |
                         v
                   finish_requests
                         |
                         v
                    free_states
                         |
                         v
                    add_requests
                         |
                         v
                   update_requests
                         |
                         v
             block_tables.apply_staged_writes
                         |
                         v
                  检查调度 Token 数
                         |
             +-----------+-----------+
             |                       |
         无调度 Token              有调度 Token
             |                       |
             v                       v
      Connector No-Forward      gather_batch_req_state
             |                       |
             |                       v
             |                DP Sync / CUDA Graph
             |                       |
             |                       v
             |                  prepare_inputs
             |                       |
             |                       v
             |                  prepare_attn
             |                       |
             |                       v
             |               ModelState Preprocess
             |                       |
             |                       v
             |               ModelState Prepare Attn
             |                       |
             |                       v
             |               ModelState Prepare Inputs
             |                       |
             |                       v
             |                 Execute Model
             |                       |
             |         +-------------+-------------+
             |         |             |             |
             |         v             v             v
             |      FULL CG       PIECEWISE       EAGER
             |         |             |             |
             |         +-------------+-------------+
             |                       |
             |                       v
             |                 Finish Forward
             |                       |
             |                       v
             |              Extract Hidden States
             |                       |
             |                       v
             |             Save ExecuteModelState
             |                       |
             +-----------------------+
                         |
                         v
                 返回执行结果 / PP 数据
```

### 5.3 请求状态更新

执行模型前，首先需要更新请求状态。

涉及：

```
update_pp_decode_requests()
finish_requests()
free_states()
add_requests()
update_requests()
```

主要作用：

| 方法                            | 作用                             |
| ----------------------------- | ------------------------------ |
| `update_pp_decode_requests()` | 更新 Pipeline Parallel Decode 请求 |
| `finish_requests()`           | 处理已经完成的请求                      |
| `free_states()`               | 释放请求相关状态                       |
| `add_requests()`              | 添加新请求                          |
| `update_requests()`           | 更新正在运行的请求                      |

然后：

```
self.block_tables.apply_staged_writes()
```

将暂存的 Block Table 修改应用到实际状态。

### 5.4 Batch 请求整理

通过：

```
gather_batch_req_state()
```

整理当前 Batch 请求信息。

源码定义：

```
class BatchReqState(NamedTuple):
    req_ids: list[str]
    num_scheduled_tokens: np.ndarray
    num_tokens: int
    idx_mapping_np: np.ndarray
    prefill_len_np: np.ndarray
    num_computed_prefill_tokens_np: np.ndarray
    is_prefilling_np: np.ndarray
    has_prefill: bool
```

它的作用是将 Scheduler 输出和 Runner 内部请求状态组合成当前执行所需的 CPU 侧 Batch 信息。

注意：

`BatchReqState` 不是 GPU 输入 Tensor，而是 Batch 准备阶段的请求汇总信息。

### 5.5 DP 同步与 CUDA Graph 分发

源码调用：

```
dispatch_cg_and_sync_dp(...)
```

负责处理：

- Data Parallel 之间的执行同步。
- CUDA Graph 执行模式选择。
- Batch 执行描述生成。
- 不同 DP Rank 的 Token 数量协调。

这里需要注意：

Data Parallel 的执行同步不等于把一个 Batch 的 Token 全部切分后再汇总生成结果。

DP Rank 通常拥有自己的模型执行副本和请求执行负载，具体同步逻辑由 DP 调度和相关通信机制完成。

### 5.6 输入准备：`prepare_inputs()`

主要工作：

- 准备输入 Token。
- 准备位置编码相关信息。
- 整理请求索引。
- 构造 InputBatch。
- 准备 Query Start Location。
- 准备 Logits Indices。
- 处理 Padding。
- 处理 Prefill / Decode 输入差异。

得到：

```
input_batch
```

### 5.7 Attention 准备：`prepare_attn()`

Attention 准备阶段主要处理：

1. Block Table 信息。
2. Slot Mapping。
3. Attention Metadata 的构建所需输入。
4. 不同 KV Cache Group 的映射关系。

核心关系：

```
InputBatch
    |
    v
BlockTables
    |
    v
Slot Mapping
    |
    v
Attention Metadata
    |
    v
Attention Backend
```

需要特别理解：

**Slot Mapping 和 Attention Metadata 不一样。**

Slot Mapping 解决当前 Token 的 KV Cache 写入位置问题。

Attention Metadata 则提供 Attention Kernel 执行需要的上下文信息。

### 5.8 ModelState 准备

源码通过 ModelState 执行模型相关的预处理和输入准备。

主要涉及：

```
model_state.preprocess_state()
          |
          v
model_state.prepare_attn()
          |
          v
model_state.prepare_inputs()
```

这些方法将通用输入和模型特定状态转换为模型可以执行的输入。

例如 Mamba 等具有 Recurrent State 的模型，可能需要在执行前进行状态迁移或整理。

### 5.9 模型前向执行

最终进入模型执行阶段。

源码根据 `batch_desc.cg_mode` 选择执行方式。

```
if batch_desc.cg_mode == CUDAGraphMode.FULL:
    model_output = self.cudagraph_manager.run_fullgraph(batch_desc)
else:
    ...
```

在非 FULL 模式下：

```
if ubatch_state is not None:
    model_output = self.ubatch_runner.run(
        self.model,
        model_inputs,
        ubatch_state,
    )
elif batch_desc.cg_mode == CUDAGraphMode.PIECEWISE:
    model_output = self.cudagraph_manager.run_pw_graph(
        self.model,
        model_inputs,
    )
else:
    model_output = self.model(**model_inputs)
```

执行路径：

```
                    Model Inputs
                         |
                         v
                    CG Mode 判断
                         |
       +-----------------+------------------+
       |                 |                  |
       v                 v                  v
    FULL CG          PIECEWISE             NONE
       |                 |                  |
       v                 v                  v
  run_fullgraph     run_pw_graph       model(**inputs)
       |                 |                  |
       +-----------------+------------------+
                         |
                         v
                    Model Output
```

三种模式的区别：

| 模式        | 执行方式                 | 特点              |
| --------- | -------------------- | --------------- |
| NONE      | 直接调用模型               | Eager 执行        |
| PIECEWISE | 分段 CUDA Graph        | 部分模型区域 Graph 执行 |
| FULL      | 完整 CUDA Graph Replay | 复用完整捕获图         |

另外，源码还支持 UBatch 执行路径。

它与 CUDA Graph Mode 不是完全相同维度的概念：UBatch 是一种 Micro-batch 执行组织方式，可以参与具体执行路径选择。

### 5.10 Forward 结果处理

执行完成后：

```
self.kv_connector.finish_forward()
```

然后根据 PP Rank 判断输出类型。

最后一个 PP Rank：

```
Model Output
     |
     v
Hidden States
     |
     v
ExecuteModelState
```

非最后一个 PP Rank：

```
Model Output
     |
     v
IntermediateTensors
     |
     v
PPHandler
     |
     v
传递给下一个 PP Rank
```

最后保存：

```
self.execute_model_state = ExecuteModelState(
    input_batch=input_batch,
    attn_metadata=attn_metadata,
    slot_mappings_by_layer=slot_mappings_by_layer,
    hidden_states=hidden_states,
    aux_hidden_states=aux_hidden_states,
    dp_sync=dp_sync,
    finished_req_ids=finished_req_ids,
    ec_connector_output=ec_connector_output,
    routed_experts=routed_experts,
    cudagraph_stats=cudagraph_stats,
)
```

这一步非常重要。

**execute_model() 负责前向执行，而不是直接完成整个 Token 生成过程。**

它将执行结果保存到 `ExecuteModelState`，供后续采样阶段使用。

---

## 6. ExecuteModelState：Forward 和 Sampling 之间的桥梁

源码：

```
class ExecuteModelState(NamedTuple):
    input_batch: InputBatch
    attn_metadata: dict[str, Any] | None
    slot_mappings_by_layer: dict[str, torch.Tensor] | None
    hidden_states: torch.Tensor | None
    aux_hidden_states: list[torch.Tensor] | None
    dp_sync: DPSyncState | None
    finished_req_ids: set[str]
    ec_connector_output: ECConnectorOutput | None
    routed_experts: RoutedExpertsTensors | None
    cudagraph_stats: CUDAGraphStat | None
```

为什么需要它？

因为模型前向和 Token Sampling 被拆成两个阶段。

```
execute_model()
      |
      v
Model Forward
      |
      v
Hidden States
      |
      v
ExecuteModelState
      |
      | 保存中间执行上下文
      v
sample_tokens()
      |
      v
Logits
      |
      v
Sampler
      |
      v
Sampled Tokens
```

它保存：

| 字段                       | 作用                           |
| ------------------------ | ---------------------------- |
| `input_batch`            | 本轮 Batch 输入                  |
| `attn_metadata`          | Attention 执行上下文              |
| `slot_mappings_by_layer` | 每层 Slot Mapping              |
| `hidden_states`          | 模型输出 Hidden States           |
| `aux_hidden_states`      | 辅助 Hidden States             |
| `dp_sync`                | DP 同步信息                      |
| `finished_req_ids`       | 本轮完成的请求                      |
| `ec_connector_output`    | Encoder-Decoder Connector 输出 |
| `routed_experts`         | MoE 路由相关信息                   |
| `cudagraph_stats`        | CUDA Graph 统计信息              |

可以将其理解为：

**Forward 阶段产生的结果包，Sampling 阶段消费这个结果包。**

---

## 7. Token Sampling：`sample_tokens()`

这是 GPUModelRunner 的另一个核心方法。

主要负责：

1. 读取 ExecuteModelState。
2. 处理 Pipeline Parallel。
3. 恢复需要的 Hidden States。
4. 执行 Token Sampling。
5. 广播采样结果。
6. 计算 Prompt Logprobs。
7. 构造异步输出。
8. 更新 RequestState。
9. 执行 Speculative Decoding Draft Token 生成。
10. 执行 KV Connector 后处理。

### 7.1 整体流程

```
                 sample_tokens()
                        |
                        v
              读取 ExecuteModelState
                        |
                        v
              清空 execute_model_state
                        |
                        v
                  判断 PP Rank
                        |
           +------------+-------------+
           |                          |
           v                          v
       非最后 Rank                  最后 Rank
           |                          |
           v                          v
      PPHandler.receive()       恢复 Hidden States
           |                          |
           v                          v
   更新本地 RequestState           sample()
           |                          |
           v                          v
    KVConnector 后处理          PPHandler.broadcast()
           |                          |
           v                          v
        返回输出                 Prompt Logprobs
                                      |
                                      v
                                AsyncOutput
                                      |
                                      v
                             postprocess_sampled()
                                      |
                                      v
                              Speculator.propose()
                                      |
                                      v
                              更新 Draft Tokens
                                      |
                                      v
                             KVConnector 后处理
                                      |
                                      v
                                   返回
```

### 7.2 读取 ExecuteModelState

```
input_batch = self.execute_model_state.input_batch
hidden_states = self.execute_model_state.hidden_states
attn_metadata = self.execute_model_state.attn_metadata

self.execute_model_state = None
```

注意：

先读取状态，再将成员变量清空。

这样可以避免旧的执行结果被重复消费。

如果：

```
if self.execute_model_state is None:
    return None
```

说明前一次执行可能失败，当前没有可供采样的有效执行状态。

### 7.3 执行采样

```
sampler_output, num_sampled, num_rejected = self.sample(
    hidden_states,
    input_batch,
    grammar_output,
)
```

这里调用 `sample()` 完成真正的 Token 采样。

### 7.4 Prompt Logprobs

```
prompt_logprobs_dict = (
    self.prompt_logprobs_worker.compute_prompt_logprobs(
        self.model.compute_logits,
        hidden_states,
        input_batch,
        self.req_states.all_token_ids.gpu,
        self.req_states.num_computed_tokens.gpu,
        self.req_states.prompt_len.np,
    )
)
```

Prompt Logprobs 用于计算 Prompt Token 对应的概率信息。

它和生成下一个 Token 的采样逻辑不同。

### 7.5 AsyncOutput

源码：

```
async_output = AsyncOutput(
    model_runner_output=model_runner_output,
    sampler_output=sampler_output,
    num_sampled_tokens=num_sampled,
    main_stream=self.main_stream,
    copy_stream=self.output_copy_stream,
    check_ep_fault=self.check_ep_fault,
    routed_experts=routed_experts,
)
```

这里采用异步输出设计。

主要目的是让 GPU 到 CPU 的输出复制与后续计算尽可能重叠。

```
GPU Main Stream
       |
       v
   Sampling
       |
       v
   Sampler Output
       |
       +----------------------+
       |                      |
       v                      v
  后续 GPU 计算          Output Copy Stream
       |                      |
       v                      v
 Speculator Proposal       D2H Copy
       |                      |
       +----------+-----------+
                  |
                  v
             Async Output
```

异步输出不是简单地把 Tensor 转成 Python 数据，而是通过独立 Stream 和异步结果对象管理输出准备过程。

### 7.6 postprocess_sampled()

负责将采样结果反映到请求状态中。

主要涉及：

- 更新已计算 Token 数。
- 更新最近采样 Token。
- 更新请求 Token ID。
- 更新总长度。
- 更新输出 Token 计数。
- 更新模型状态。

调用形式：

```
self.postprocess_sampled(
    input_batch.idx_mapping,
    sampler_output.sampled_token_ids,
    num_sampled,
    num_rejected,
    input_batch.query_start_loc,
)
```

它是 Sampling 之后的状态提交阶段。

---

## 8. `sample()`：Logits 与 Token 采样

`sample_tokens()` 负责整体采样阶段的调度，而 `sample()` 负责具体采样过程。

核心流程：

```
Hidden States
     |
     v
logits_indices
     |
     v
选择需要采样的 Hidden States
     |
     v
model.compute_logits()
     |
     v
Logits
     |
     v
Structured Outputs Mask
     |
     v
Sampler / RejectionSampler
     |
     v
Sampled Token IDs
     |
     v
Gather / Batch Sharding（如果启用）
     |
     v
SamplerOutput
```

### 8.1 Hidden States 选择

不是所有 Hidden States 都需要参与采样。

通过 `input_batch.logits_indices` 选择需要计算 Logits 的位置。

### 8.2 Logits 计算

```
logits = self.model.compute_logits(hidden_states)
```

模型前向输出 Hidden States，而不是直接输出最终 Token。

Logits 由模型的输出层计算得到。

### 8.3 Structured Outputs

如果启用了结构化输出，则需要根据 Grammar 生成 Mask。

通过：

```
self.structured_outputs_worker
```

将不符合约束的 Token 屏蔽。

例如 JSON 输出约束：

```
当前允许的 Token：
    {
    "
    ...

当前不允许的 Token：
    不符合 JSON 语法的 Token
```

### 8.4 Sampler 和 RejectionSampler

普通生成：

```
Logits
   |
   v
Sampler
   |
   v
Sampled Tokens
```

投机解码：

```
Target Model
     |
     v
Target Logits
     |
     v
RejectionSampler
     |
     +---- Draft Tokens
     |
     +---- Target Distribution
     |
     v
接受 / 拒绝 Draft Tokens
     |
     v
最终 Sampled Tokens
```

区别：

- `Sampler`：普通 Token 采样。
- `RejectionSampler`：负责 Speculative Decoding 中 Draft Token 的验证和接受/拒绝逻辑。

---

## 9. Speculative Decoding

GPUModelRunner 中存在：

```
self.speculator
self.rejection_sampler
self.draft_tokens_handler
```

它们分别承担不同职责。

| 组件                 | 职责                 |
| ------------------ | ------------------ |
| Speculator         | 生成 Draft Tokens    |
| RejectionSampler   | 验证 Draft Tokens    |
| DraftTokensHandler | 管理和传递 Draft Tokens |

整体流程：

```
              Target Model
                    |
                    v
                Hidden States
                    |
                    v
             Speculator.propose()
                    |
                    v
               Draft Tokens
                    |
                    v
              Target Model
                    |
                    v
              Target Logits
                    |
                    v
             RejectionSampler
                    |
             +------+------+
             |             |
             v             v
           Accept        Reject
             |             |
             +------+------+
                    |
                    v
             最终采样结果
                    |
                    v
             更新 RequestState
```

需要理解：

Speculative Decoding 并不是简单地让 Draft Model 直接生成最终输出。

Draft Model 负责提出候选 Token，Target Model 负责验证。

GPUModelRunner 通过 `sample_tokens()` 将验证后的结果和下一轮 Draft Token 生成衔接起来。

---

## 10. Pipeline Parallel

源码中：

```
self.use_pp = self.parallel_config.pipeline_parallel_size > 1
```

如果开启 PP，则初始化：

```
self.pp_handler = PPHandler(...)
```

对于非首个 PP Rank，还会创建：

```
self.intermediate_tensors = self.model.make_empty_intermediate_tensors(
    batch_size=self.max_num_tokens,
    dtype=self.model_config.dtype,
    device=self.device,
)
```

### 10.1 PP 执行流程

```
                 Input Tokens
                      |
                      v
                 PP Rank 0
                      |
                      v
                Transformer Layers
                      |
                      v
              IntermediateTensors
                      |
                      v
                 PP Rank 1
                      |
                      v
                Transformer Layers
                      |
                      v
              IntermediateTensors
                      |
                      v
                 PP Rank 2
                      |
                      v
                 Final Hidden
                      |
                      v
                    Sampler
                      |
                      v
                 Sampled Tokens
```

### 10.2 为什么只有最后一个 Rank 负责采样？

源码中：

```
if self.is_last_pp_rank:
    ...
    self.sampler = Sampler(...)
```

模型的最后一个 Pipeline Stage 才能得到完整模型输出 Hidden States。

因此：

- 非最后 Rank：执行自己的模型分段，并传递中间 Tensor。
- 最后 Rank：执行最后一段模型，并负责采样。

### 10.3 PP Token 广播

采样完成后：

```
self.pp_handler.broadcast(
    sampler_output.sampled_token_ids,
    num_sampled,
    num_rejected,
    input_batch,
)
```

最后一个 Rank 将采样结果广播给其他 Rank。

其他 Rank 需要同步更新请求状态，以保证后续模型执行的一致性。

---

## 11. CUDA Graph 与 UBatch

### 11.1 ModelCudaGraphManager

通过：

```
self.cudagraph_manager = ModelCudaGraphManager(...)
```

创建 CUDA Graph 管理器。

GPUModelRunner 并不自己实现 Graph Capture 和 Replay，而是将这些工作委托给 `ModelCudaGraphManager`。

### 11.2 CUDA Graph 模式

```
CUDAGraphMode
    |
    +-- NONE
    |
    +-- PIECEWISE
    |
    +-- FULL
```

| 模式        | 特征              |
| --------- | --------------- |
| NONE      | 普通 Eager 执行     |
| PIECEWISE | 分段 Graph 执行     |
| FULL      | 完整 Graph Replay |

### 11.3 UBatchRunner

初始化：

```
self.ubatch_runner = maybe_build_ubatch_runner(...)
```

执行：

```
model_output = self.ubatch_runner.run(
    self.model,
    model_inputs,
    ubatch_state,
)
```

UBatch 的主要目标是将一个 Batch 拆分为多个 Micro-batch 进行执行。

```
             Original Batch
                   |
          +--------+--------+
          |                 |
          v                 v
       UBatch 0          UBatch 1
          |                 |
          v                 v
      Model Forward     Model Forward
          |                 |
          +--------+--------+
                   |
                   v
              Batch Result
```

注意：

UBatch 和 DP 不同。

- DP 是不同数据并行 Rank 之间的并行执行。
- UBatch 是单个执行流程内部的 Micro-batch 划分机制。

---

## 12. 多模态支持

GPUModelRunner 通过：

```
self.mm_registry = MULTIMODAL_REGISTRY
```

判断模型是否支持多模态输入。

```
self.supports_mm_inputs = (
    self.mm_registry.supports_multimodal_inputs(
        self.model_config
    )
)
```

如果支持多模态输入，并且当前是第一个 PP Rank：

```
self.encoder_cache = EncoderCache()
```

然后创建：

```
self.ec_connector = get_ec_connector(
    vllm_config,
    self.encoder_cache,
)
```

多模态处理的逻辑：

```
Image / Audio / Video
        |
        v
Multimodal Encoder
        |
        v
Encoder Embeddings
        |
        v
EncoderCache
        |
        v
Model Input Preparation
        |
        v
LLM Forward
```

EncoderCache 用于管理多模态 Encoder 相关缓存。

多模态 Encoder 和 LLM Decoder 的执行逻辑并不完全相同，具体处理由 ModelState 和 Encoder Runner 等组件负责。

---

## 13. KVConnector

初始化：

```
self.kv_connector = NO_OP_KV_CONNECTOR
```

在 KV Cache 初始化完成后，根据配置获取实际 Connector：

```
self.kv_connector = get_kv_connector(
    self.vllm_config,
    kv_caches_dict,
)
```

模型执行前后会调用：

```
self.kv_connector.pre_forward(...)
```

```
self.kv_connector.finish_forward()
```

```
self.kv_connector.post_forward(finished_req_ids)
```

执行关系：

```
KVConnector
    |
    +-- pre_forward()
    |       |
    |       v
    |   Forward 前处理
    |
    +-- finish_forward()
    |       |
    |       v
    |   Forward 后处理
    |
    +-- post_forward()
            |
            v
        Step 后处理
```

它为 KV Cache 的外部连接、传输等能力提供统一接口。

具体的数据传输策略取决于实际配置的 Connector 实现。

---

## 14. Pooling 模型

GPUModelRunner 不仅支持 Generate 模型，也支持 Pooling 模型。

```
self.is_pooling_model = (
    self.model_config.runner_type == "pooling"
)
```

如果是 Pooling 模型，并且当前是最后一个 PP Rank：

```
self.pooling_runner = PoolingRunner(
    self.model,
    self.vllm_config,
)
```

执行流程：

```
InputBatch
    |
    v
Model Forward
    |
    v
Hidden States
    |
    v
PoolingRunner.pool()
    |
    v
Pooling Output
    |
    v
AsyncPoolingOutput
```

Pooling 与 Generate 的区别：

| 类型       | 输出                                 |
| -------- | ---------------------------------- |
| Generate | Sampled Token IDs                  |
| Pooling  | Pooling Embedding / Pooling Result |

所以 GPUModelRunner 在模型前向阶段可以复用很多通用逻辑，但最终输出阶段不同。

---

## 15. Dummy Run 和 Profile

源码提供：

```
_dummy_run()
_dummy_sampler_run()
_dummy_pooler_run()
profile_run()
profile_cudagraph_memory()
needs_cudagraph_capture()
```

### 15.1 `_dummy_run()`

通过构造 Dummy SchedulerOutput 模拟模型执行。

```
_dummy_run()
    |
    v
构造 Dummy SchedulerOutput
    |
    v
execute_model(dummy_run=True)
    |
    v
模型前向执行
    |
    v
返回 Hidden States
```

用途：

- 初始化阶段的模型执行验证。
- 显存 Profiling。
- CUDA Graph Capture 前的准备。
- 预热模型执行路径。

### 15.2 `profile_run()`

主要流程：

```
profile_run()
    |
    v
多模态 Encoder Profiling（如果需要）
    |
    v
_dummy_run()
    |
    v
Dummy Sampler / Pooler
    |
    v
GPU Synchronize
    |
    v
清理临时状态
```

它并不是正常请求生成流程，而是初始化和 Profiling 阶段的辅助执行。

---

## 16. 其他重要方法

| 方法                               | 职责                       |
| -------------------------------- | ------------------------ |
| `update_max_model_len()`         | 更新最大模型长度                 |
| `init_routed_experts_capturer()` | 初始化 MoE Expert 路由捕获      |
| `get_supported_tasks()`          | 获取模型支持的任务类型              |
| `get_model()`                    | 获取当前模型                   |
| `get_draft_model()`              | 获取 Draft Model           |
| `reload_weights()`               | 委托 V1 Runner 执行权重重载      |
| `update_config()`                | 委托 V1 Runner 更新配置并同步配置引用 |
| `get_encoder_timing_stats()`     | 获取 Encoder 执行时间统计        |
| `get_kv_cache_spec()`            | 获取 KV Cache 规格           |
| `_init_kv_zero_meta()`           | 初始化 KV Block Zeroing 元数据 |
| `reset_mm_cache()`               | 重置多模态缓存                  |
| `reset_encoder_cache()`          | 重置 Encoder Cache         |
| `shutdown()`                     | 释放模型、KV Cache 和 GPU 资源   |
| `setup_eplb_from_mapping()`      | 根据映射初始化 EPLB             |

特别值得注意：

`reload_weights()` 和 `update_config()` 在当前代码中仍然委托给 V1 GPUModelRunner 的实现。

这说明当前代码存在 V1/V2 迁移兼容逻辑，不能简单认为所有方法都已经完全迁移到 V2 实现。

---

## 17. GPUModelRunner 的关键数据流

### 17.1 请求数据流

```
SchedulerOutput
      |
      v
BatchReqState
      |
      v
InputBatch
      |
      v
Model Inputs
      |
      v
Hidden States
      |
      v
Logits
      |
      v
Sampler
      |
      v
Sampled Tokens
      |
      v
RequestState Update
      |
      v
ModelRunnerOutput
```

### 17.2 KV Cache 数据流

```
Scheduler KV Allocation Result
             |
             v
         BlockTables
             |
             v
       Slot Mapping
             |
             v
       Attention Metadata
             |
             v
       Attention Backend
             |
             v
          KV Cache
```

### 17.3 Forward 到 Sampling 的状态流

```
execute_model()
      |
      v
InputBatch
      |
      v
Model Forward
      |
      v
Hidden States
      |
      v
ExecuteModelState
      |
      v
sample_tokens()
      |
      v
Sampler
      |
      v
postprocess_sampled()
      |
      v
RequestState
```

---

## 18. 最重要的概念辨析

| 概念                 | 解决的问题                   | 关键理解         |
| ------------------ | ----------------------- | ------------ |
| SchedulerOutput    | 本轮执行什么                  | 调度结果         |
| BatchReqState      | 当前 Batch 包含哪些请求         | CPU 侧请求汇总    |
| RequestState       | 请求运行到什么状态               | 跨 Step 保存    |
| InputBatch         | 本轮模型输入是什么               | GPU 执行输入描述   |
| InputBuffers       | 如何复用输入内存                | 预分配 Buffer   |
| BlockTables        | 请求使用哪些 KV Block         | 逻辑到物理映射      |
| Slot Mapping       | 当前 Token 写到哪里           | KV 写入位置      |
| Attention Metadata | Attention 如何执行          | Kernel 执行元数据 |
| ModelState         | 模型特定状态如何处理              | 模型适配层        |
| Hidden States      | 模型前向输出什么                | 后续计算 Logits  |
| Logits             | 每个 Token 的预测分数          | Sampler 输入   |
| Sampler            | 如何选择下一个 Token           | Token 采样     |
| ExecuteModelState  | Forward 和 Sampling 如何衔接 | 中间执行上下文      |
| ModelRunnerOutput  | 如何将结果返回上层               | 执行结果         |

---

## 19. 源码阅读建议

不建议按照 2348 行代码从头读到尾。

建议按执行链路阅读。

### 第一阶段：理解整体入口

重点阅读：

```
GPUModelRunner.__init__()
GPUModelRunner.load_model()
GPUModelRunner.initialize_kv_cache()
```

需要回答：

1. Runner 初始化了哪些对象？
2. 模型什么时候真正加载？
3. KV Cache 什么时候分配？
4. Sampler 什么时候初始化？
5. CUDA Graph Manager 什么时候创建？

### 第二阶段：理解 Forward

重点阅读：

```
GPUModelRunner.execute_model()
    |
    +-- gather_batch_req_state()
    |
    +-- prepare_inputs()
    |
    +-- prepare_attn()
    |
    +-- ModelState.prepare_attn()
    |
    +-- ModelState.prepare_inputs()
    |
    +-- dispatch_cg_and_sync_dp()
    |
    +-- model(...)
```

需要回答：

1. SchedulerOutput 如何变成 InputBatch？
2. RequestState 如何参与输入准备？
3. BlockTables 如何生成 Slot Mapping？
4. Attention Metadata 如何生成？
5. CUDA Graph 如何选择？
6. 模型实际在哪里执行？

### 第三阶段：理解 Sampling

重点阅读：

```
GPUModelRunner.sample_tokens()
    |
    +-- sample()
    |
    +-- postprocess_sampled()
    |
    +-- Speculator.propose()
```

需要回答：

1. Hidden States 如何计算 Logits？
2. 普通 Sampling 和 Rejection Sampling 有什么区别？
3. Structured Outputs 如何约束 Token？
4. Sampling 结果如何更新 RequestState？
5. Speculator 如何生成下一轮 Draft Tokens？

### 第四阶段：理解优化机制

重点阅读：

```
ModelCudaGraphManager
UBatchRunner
BatchSharder
PPHandler
KVConnector
EPLBController
```

建议优先研究 CUDA Graph、KV Cache、PP 和 Speculative Decoding。

---

## 20. 建议设置的断点

| 断点位置                        | 观察内容                     | 学习目标                     |
| --------------------------- | ------------------------ | ------------------------ |
| `GPUModelRunner.__init__`   | 初始化成员变量                  | 理解 Runner 架构             |
| `load_model()`              | 模型加载过程                   | 理解 Model Loader          |
| `initialize_kv_cache()`     | KV Cache 初始化             | 理解 Cache 架构              |
| `execute_model()`           | SchedulerOutput          | 理解执行入口                   |
| `gather_batch_req_state()`  | Batch 请求信息               | 理解请求整理                   |
| `prepare_inputs()`          | InputBatch               | 理解输入构造                   |
| `prepare_attn()`            | BlockTables、Slot Mapping | 理解 KV Cache 映射           |
| `dispatch_cg_and_sync_dp()` | BatchExecutionDescriptor | 理解 DP 和 CUDA Graph       |
| `model(**model_inputs)`     | 模型输入和输出                  | 理解真正的 Forward            |
| `sample_tokens()`           | ExecuteModelState        | 理解 Forward 与 Sampling 衔接 |
| `sample()`                  | Logits、SamplerOutput     | 理解采样                     |
| `postprocess_sampled()`     | RequestState             | 理解请求状态更新                 |
| `speculator.propose()`      | Draft Tokens             | 理解投机解码                   |

### 推荐调试顺序

```
1. load_model()
       |
       v
2. initialize_kv_cache()
       |
       v
3. execute_model()
       |
       v
4. prepare_inputs()
       |
       v
5. prepare_attn()
       |
       v
6. model(**model_inputs)
       |
       v
7. sample_tokens()
       |
       v
8. sample()
       |
       v
9. postprocess_sampled()
```

---

## 21. 总结

GPUModelRunner 是 vLLM V1 GPU 侧的模型执行管理器。

它的核心工作可以归纳为三个阶段：

### 阶段一：执行准备

```
SchedulerOutput
      |
      v
RequestState Update
      |
      v
InputBatch
      |
      v
BlockTables / Slot Mapping
      |
      v
Attention Metadata
      |
      v
Model Inputs
```

### 阶段二：模型前向

```
Model Inputs
      |
      v
CUDA Graph / UBatch / Eager
      |
      v
Model Forward
      |
      v
Hidden States
      |
      v
ExecuteModelState
```

### 阶段三：采样和状态更新

```
ExecuteModelState
      |
      v
Compute Logits
      |
      v
Sampler / RejectionSampler
      |
      v
Sampled Tokens
      |
      v
postprocess_sampled()
      |
      v
RequestState Update
      |
      v
ModelRunnerOutput
```

**最重要的理解：**

1. GPUModelRunner 是执行协调器，不是模型本身。
2. `execute_model()` 负责准备输入和执行模型前向。
3. `sample_tokens()` 负责采样、结果处理和状态更新。
4. `ExecuteModelState` 是 Forward 和 Sampling 之间的桥梁。
5. `RequestState` 保存跨 Step 的请求状态，`InputBatch` 描述当前 Step 的执行输入。
6. `BlockTables`、Slot Mapping、KV Cache、Attention Metadata 是不同层次的对象。
7. CUDA Graph、UBatch、PP、DP、Speculative Decoding 等能力通过独立模块集成，而不是全部由 GPUModelRunner 自己实现。

从源码阅读角度，最值得深入的主线是：

**SchedulerOutput → InputBatch → BlockTables / Attention Metadata → Model Forward → Hidden States → Sampling → RequestState Update。**

这条主线基本覆盖了 vLLM 推理执行的核心过程。

补充源码依据与准确边界压缩重复说明和流程图增加 Prefill 与 Decode 对比
