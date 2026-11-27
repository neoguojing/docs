

- /home/neo/workspace/vllm/vllm/v1/core/single_type_kv_cache_manager.py
- /home/neo/workspace/vllm/vllm/v1/kv_cache_spec_registry.py
- /home/neo/workspace/vllm/vllm/v1/kv_cache_interface.py
- /home/neo/workspace/vllm/vllm/v1/core/single_type_kv_cache_manager.py
- /home/neo/workspace/vllm/vllm/model_executor/layers/attention/attention.py
- /home/neo/workspace/vllm/vllm/v1/kv_cache_interface.py
- 
- kv_cache_specs = self.model_executor.get_kv_cache_specs()
self.collective_rpc("get_kv_cache_spec")
GPUModelRunner.get_kv_cache_spec
基于以下配置，获取所有的attenion layer和对象：
vllm_config.compilation_config.static_forward_context
'language_model.model.layers.0.self_attn.attn':Attention(head_size=256, num_heads=32, num_kv_heads=16)
Attention.get_kv_cache_spec 返回SlidingWindowSpec 或者 FullAttentionSpec
最终返回：格式'language_model.model.layers.0.self_attn.attn'： SlidingWindowSpec(block_size=16, num_kv_heads=16, head_size=256, dtype=torch.float16, head_size_v=256, kv_quant_mode=<KVQuantMode.NONE: 0>, page_size_padded=None, num_head_slots=None, state_content_bytes=None, tokens_per_state=1, sliding_window=1024, extra_retained_tokens=0)

- self.model_executor.get_supported_kv_cache_layouts(),
def get_supported_kv_cache_layouts(self) -> list[list[str]]:
        """Layouts each worker's backends support, most preferred first."""
        return self.collective_rpc("get_supported_kv_cache_layouts")
WorkerBase.get_supported_kv_cache_layouts
layout = <KVCacheLayout.LBNHC: (0, 1, 3, 2, 4)>

- self.model_executor.set_kv_cache_layout(layout.name)
WorkerBase.set_kv_cache_layout
设置config的layout

- available_gpu_memory = self.model_executor.determine_available_memory()
def determine_available_memory(self) -> list[int]:  # in bytes
        return self.collective_rpc("determine_available_memory")
Worker.determine_available_memory
torch.compile + Capturing CUDA graphs
available_gpu_memory = [6435619021]
determine_available_memory() 本质上是一个 GPU 显存预算计算器：先测量模型运行时的非 KV Cache 显存开销，再扣除 CUDA Graph 等额外开销，最终得到可用于 KV Cache Block 分配的显存容量。
```
# Run multimodal encoder.
                        dummy_encoder_outputs = self.model.embed_multimodal(
                            **batched_dummy_mm_inputs
                        )
 hidden_states, last_hidden_states = self._dummy_run(
            self.max_num_tokens, is_profile=True
        )
else:
                output = self._dummy_sampler_run(last_hidden_states)
[开始] determine_available_memory() (@torch.inference_mode())
  │
  ▼
maybe_apply_startup_plan(self)
  │
  ▼
是否配置了 kv_cache_memory_bytes ?
  ├── 是 (手动指定模式)
  │     │
  │     ▼
  │   self.model_runner.profile_run()  ──(触发编译/预热，跳过自动显存估算)
  │     │
  │     ▼
  │   logger.info(...)  ──(提示跳过 profiling、不遵循 gpu_memory_utilization)
  │     │
  │     ▼
  │   返回 reserve_mm_ipc_gpu_memory(kv_cache_memory_bytes, multimodal_config, _api_process_count)
  │
  └── 否 (自动 Profiling 模式)
        │
        ▼
      with memory_profiling(self.init_snapshot, weights_memory=...)
        │
        ▼
      self.model_runner.profile_run()  ──(dummy 输入前向传播，统计显存占用)
        │
        ▼
      退出上下文 -> 生成 profile_result
        │
        ▼
      (current_platform.is_cuda_alike() 且 cudagraph_mode != NONE) ?
        ├── 是 ─▶ cudagraph_memory_estimate = self.model_runner.profile_cudagraph_memory()
        └── 否 ─▶ cudagraph_memory_estimate = 0
        │
        ▼
      envs.VLLM_MEMORY_PROFILER_ESTIMATE_CUDAGRAPHS 是否开启 ?
        ├── 是 ─▶ cudagraph_memory_estimate_applied = cudagraph_memory_estimate
        └── 否 ─▶ cudagraph_memory_estimate_applied = 0
        │
        ▼
      读取 init_free_memory 与 profile_result.after_profile.free_memory
        │
        ▼
      rocm_fallback = maybe_rocm_profiling_fallback(profile_result)
        │
        ▼
      rocm_fallback is None ?
        ├── 是 (标准路径) ─▶ assert init_free_memory >= free_gpu_memory
        └── 否 (ROCm 回退) ─▶ profile_result.total_consumed = rocm_fallback
                                non_kv_cache_memory = total_consumed + transient_peak_headroom
        │
        ▼
      更新状态与计算 KV Cache 显存:
        • total_consumed = profile_result.total_consumed
        • cudagraph_memory_estimate = cudagraph_memory_estimate
        • peak_activation_memory = transient_peak_headroom + cudagraph_memory_estimate_applied
        • available_kv_cache_memory_bytes = requested_memory - non_kv_cache_memory - cudagraph_memory_estimate_applied
        │
        ▼
      输出 debug / info 日志 (初始空闲、请求显存、profiling 后空闲、可用 KV 显存)
        │
        ▼
      cudagraph_memory_estimate > 0 ?
        ├── 是 ─▶ VLLM_MEMORY_PROFILER_ESTIMATE_CUDAGRAPHS 已启用 ?
        │             ├── 是 ─▶ logger.info(...) (说明等效 util 与建议上调 util)
        │             └── 否 ─▶ logger.warning(...) (警告未计入 CUDA Graph 显存及建议值)
        └── 否 ─▶ (跳过 CUDA Graph 提示)
        │
        ▼
      返回 reserve_mm_ipc_gpu_memory(int(available_kv_cache_memory_bytes), multimodal_config, _api_process_count)
```
- self.model_executor.initialize_from_config(kv_cache_configs)
def initialize_from_config(self, kv_cache_configs: list[KVCacheConfig]) -> None:
        """Initialize the KV caches on the underlying workers."""
        self.collective_rpc("initialize_from_config", args=(kv_cache_configs,))
Worker.initialize_from_config
GPUModelRunner.initialize_kv_cache
/home/neo/workspace/vllm/vllm/v1/worker/gpu/attn_utils.py：init_kv_cache
```
init_kv_cache()
│
├── KVCacheConfig
│   ├── KVCache Groups
│   ├── KVCache Specs
│   └── Cache Layout / Block 信息
│
├── VllmConfig
│   ├── attention_config
│   │   └── hisparse_config
│   ├── cache_config
│   │   └── get_resolved_kv_cache_layout()
│   └── model_config
│       └── hf_config.model_type
│
├── allocate_kv_cache() 
│   └── 普通 KV Cache Tensor 分配
       这套机制涉及三个层次：
        1. 物理内存层：通过 torch.zeros() 分配一块大的 GPU Buffer。
        2. 逻辑 Tensor 层：通过 torch.as_strided() 创建不同 Layer 的 KV Cache View。
        3. 模型访问层：通过 kv_caches[layer_name] 将对应 View 绑定到 Attention 层。
│
├── init_hisparse_kv_cache()
│   └── HiSparse 特殊缓存初始化
│
├── get_shared_kv_cache_layers()
│   └── Cache 共享映射
│
├── forward_context
│   └── Attention Runtime 对象注册表
│
└── bind_kv_cache_to_layers() 
    └── Cache 与 Attention 对象绑定： 设置Attention对象的self.kv_cache = kv_cache

torch.zeros()
    │
    ▼
PyTorch Tensor 创建逻辑
    │
    ▼
CUDA Device Allocator
    │
    ▼
CUDA 显存分配
    │
    ▼
返回 CUDA Tensor

view = torch.as_strided(
    buffer,
    size=(3, 4),
    stride=(4, 1),
)
shape：逻辑上有多少维，每一维有多大。
stride：某一维索引增加 1，需要在底层存储中跨过多少个元素。
访问 view[1, 2]：
底层偏移 = 1 * 4 + 2 * 1
         = 6

view_5d [10, 256, 16, 16, 1024]
│
├── Layer 0
│   ├── Block 0
│   │   ├── Head 0
│   │   │   └── Token 0～15
│   │   │       └── Channel 0～1023
│   │   ├── Head 1
│   │   └── ... Head 15
│   │
│   ├── Block 1
│   └── ... Block 255
│
├── Layer 1
├── ...
└── Layer 9

L	Layer	10	Transformer 层
B	Block	256	每层的 KV Cache Block
H	KV Head	16	KV 注意力头
N	Token	16	每个 Block 的 Token 槽位
C	Channel	1024	每个 Head、Token 对应的字节数据

kvcache的格式： dict[layername:tensor()]
```

- self.model_executor.compile_or_warm_up_model()
Worker.compile_or_warm_up_model
```
compile_or_warm_up_model()
1. 编译尺寸准备
   
   确定需要执行模型编译和预热的 Batch Token Size。
2. 模型 Dummy Run
   
   _dummy_run() 使用虚拟输入执行模型，触发编译和相关初始化。
3. Kernel Warmup
   
   kernel_warmup() 预热模型执行所需的 CUDA Kernel。
4. V2 Model Runner 额外预热
   
   warmup_kernels() 提前调整 Workspace，避免 CUDA Graph 捕获后的内存失效。
5. CUDA Graph Capture
   
   capture_model() 捕获模型执行图，供后续推理重放。
6. KV Cache 显存预算评估
   
   根据实际显存使用情况，计算 KV Cache 可使用的显存并给出配置建议。
7. Sampler / Pooler 预热
   
   V1 Model Runner 提前分配采样及相关 Tensor 缓冲区。
8. 运行时环境初始化
   
   固定随机种子、触发 Inductor Lazy Init、启动 JIT 监控。
9. 运行时优化
   
   Freeze GC Heap、开启 GPU Sync Check、调整 Torch 线程数。
```
def collective_rpc(  # type: ignore[override]
        self,
        method: str | Callable,
        timeout: float | None = None,
        args: tuple = (),
        kwargs: dict | None = None,
        non_block: bool = False,
        single_value: bool = False,
    ) -> Any:
        if kwargs is None:
            kwargs = {}

        if not non_block:
            result = run_method(self.driver_worker, method, args, kwargs)
            if isinstance(result, AsyncModelRunnerOutput):
                result = result.get_output()
            return result if single_value else [result]

        try:
            result = run_method(self.driver_worker, method, args, kwargs)
            if isinstance(result, AsyncModelRunnerOutput):
                return AsyncOutputFuture(result, single_value)
            future = Future[Any]()
            future.set_result(result if single_value else [result])
        except Exception as e:
            future = Future[Any]()
            future.set_exception(e)
        return future

```
Request ID
    |
    v
req_to_blocks
    |
    +-- Request A -> [B0, B1, B2]
    |
    +-- Request B -> [B0, B3, B4]
    |
    +-- Request C -> [B5, B6, B7]
                       |
                       v
                   BlockPool
                       |
                       v
               实际 Block 资源管理

Request
   |
   v
计算 Request Block Hash
   |
   v
get_cached_block()
   |
   v
查询 cached_block_hash_to_block
   |
   +----------+
   |          |
   v          v
 Cache Hit  Cache Miss
   |          |
   v          v
 touch()   get_new_blocks()
   |          |
   v          v
ref_cnt++   分配新 Block
   |          |
   +-----+----+
         |
         v
     执行模型推理
         |
         v
    生成 KV Cache
         |
         v
 cache_full_blocks()
         |
         v
 注册 Prefix Cache Hash

           BlockPool
               |
       +-------+-------+
       |               |
       v               v
  get_new_blocks   free_blocks
       |               |
       v               v
   popleft_n()      ref_cnt--
       |               |
       v               v
  检查旧 Hash       ref_cnt == 0
       |               |
       v          +----+----+
  驱逐旧缓存       |         |
       |          v         v
       v        无 Hash    有 Hash
   ref_cnt++      |         |
       |          v         v
       v       prepend    append
    使用         队首      队尾
                  |         |
                  +----+----+
                       |
                       v
                   等待分配


                 BlockPool
                     |
          +----------+----------+
          |                     |
          v                     v
 cached_block_hash_to_block  cached_block_hashes_by_block
          |                     |
          v                     v
      Hash -> Block          Block -> Hash Set
          |                     |
          +----------+----------+
                     |
                     v
                KVCacheBlock
                     |
          +----------+----------+
          |          |          |
          v          v          v
       block_id   ref_cnt   block_hash
```
# **vLLM KV Cache 初始化流程**

## **1. 整体流程**

KV Cache 初始化由 EngineCore 协调，通过 ModelExecutor 分发到 Worker，最终由 GPUModelRunner 完成缓存分配和 Attention 绑定。

```
_initialize_kv_caches()
    |
    +-- 1. get_kv_cache_specs()
    |      获取所有 Attention 层的 KV Cache Spec
    |
    +-- 2. get_supported_kv_cache_layouts()
    |      获取支持的 Cache Layout
    |
    +-- 3. set_kv_cache_layout()
    |      设置 Cache Layout
    |
    +-- 4. determine_available_memory()
    |      Profiling，计算可用 KV Cache 显存
    |
    +-- 5. 生成 KVCacheConfig
    |
    +-- 6. initialize_from_config()
    |      |
    |      v
    |    GPUModelRunner.initialize_kv_cache()
    |      |
    |      v
    |    init_kv_cache()
    |      |
    |      +-- allocate_kv_cache()
    |      +-- init_hisparse_kv_cache()
    |      +-- get_shared_kv_cache_layers()
    |      +-- bind_kv_cache_to_layers()
    |
    +-- 7. compile_or_warm_up_model()
```

## **2. 获取 KV Cache Spec**

**调用链：**

```
get_kv_cache_specs()
    → GPUModelRunner.get_kv_cache_spec()
    → Attention.get_kv_cache_spec()
```

通过 `static_forward_context` 获取模型中的 Attention 层：

```
vllm_config.compilation_config.static_forward_context
```

例如：

```
Attention(    head_size=256,    num_heads=32,    num_kv_heads=16)
```

调用 `Attention.get_kv_cache_spec()`，根据 Attention 类型返回：

- `FullAttentionSpec`
- `SlidingWindowSpec`

示例：

```
SlidingWindowSpec(    block_size=16,    num_kv_heads=16,    head_size=256,    dtype=torch.float16,    sliding_window=1024)
```

**作用：** 描述每个 Attention 层需要的 KV Cache 规格，尚未分配实际显存。

## **3. 确定 Cache Layout**

**调用链：**

```
get_supported_kv_cache_layouts()
    → set_kv_cache_layout()
```

获取 Worker 支持的 Layout，选择优先级合适的布局并设置配置。

示例：

```
KVCacheLayout.LBNHC
```

对应维度排列：

```
(0, 1, 3, 2, 4)
```

**作用：** 确定 KV Cache Tensor 的内存布局，影响后续 Tensor 的组织方式。

## **4. GPU 显存预算**

**调用链：**

```
determine_available_memory()
    → Worker.determine_available_memory()
    → model_runner.profile_run()
```

主要通过 Dummy Run 模拟模型运行，统计非 KV Cache 显存开销，并考虑 CUDA Graph 显存估算。

核心流程：

```
determine_available_memory()
    |
    v
判断 kv_cache_memory_bytes
    |
    +-- 手动指定
    |      |
    |      v
    |   profile_run()
    |
    +-- 自动 Profiling
           |
           v
       memory_profiling()
           |
           v
       profile_run()
           |
           v
       profile_cudagraph_memory()
           |
           v
       计算可用 KV Cache 显存
```

核心计算关系：

```
available_kv_cache_memory_bytes
    = requested_memory
    - non_kv_cache_memory
    - cudagraph_memory_estimate_applied
```

示例：

```
available_gpu_memory = [6435619021]
```

表示对应 Worker 可用于 KV Cache 的显存预算，单位为 Byte。

## **5. KV Cache 实际初始化**

**调用链：**

```
initialize_from_config()
    → GPUModelRunner.initialize_kv_cache()
    → init_kv_cache()
```

初始化主要包含：

| 方法                             | 作用                     |
| ------------------------------ | ---------------------- |
| `allocate_kv_cache()`          | 分配普通 KV Cache Tensor   |
| `init_hisparse_kv_cache()`     | 初始化 HiSparse 特殊缓存      |
| `get_shared_kv_cache_layers()` | 获取缓存共享映射               |
| `bind_kv_cache_to_layers()`    | 绑定 Cache 与 Attention 层 |

### **5.1 三层内存结构**

```
物理内存
    |
    v
torch.zeros()
    |
    v
分配 GPU Buffer
    |
    v
逻辑 Tensor
    |
    v
torch.as_strided()
    |
    v
创建不同 Layer 的 Tensor View
    |
    v
模型访问
    |
    v
kv_caches[layer_name]
    |
    v
绑定 Attention 对象
```

**torch.zeros()：** 创建底层物理 Tensor，分配 GPU 显存。

**torch.as_strided()：** 基于底层 Buffer 创建不同 Shape、Stride 的 Tensor View。

例如：

```
view = torch.as_strided(    buffer,    size=(3, 4),    stride=(4, 1),)
```

访问：

```
view[1, 2]

offset = 1 * 4 + 2 * 1
       = 6
```

- Shape：每个维度的大小。
- Stride：对应维度索引增加 1 时，底层跨越的元素数量。

### **5.2 KV Cache 五维结构**

示例：

```
view_5d [10, 256, 16, 16, 1024]
```

| 维度 | 含义      | 大小   |
| -- | ------- | ---- |
| L  | Layer   | 10   |
| B  | Block   | 256  |
| H  | KV Head | 16   |
| N  | Token   | 16   |
| C  | Channel | 1024 |

逻辑结构：

```
KV Cache
  |
  +-- Layer 0
  |     |
  |     +-- Block 0
  |     |     +-- Head 0
  |     |     |     +-- Token 0~15
  |     |     |           +-- Channel 0~1023
  |     |     +-- Head 1
  |     |     +-- ...
  |     +-- Block 1
  |     +-- ...
  |
  +-- Layer 1
  +-- ...
  +-- Layer 9
```

注意：这里描述的是逻辑 Tensor 结构，实际物理内存排列由 Layout 和 Stride 决定。

### **5.3 Attention 绑定**

初始化后，通过 `kv_caches` 建立 Layer Name 与 Tensor 的映射：

```
kv_caches = {    "language_model.model.layers.0.self_attn.attn": tensor_0,    "language_model.model.layers.1.self_attn.attn": tensor_1,}
```

通过 `bind_kv_cache_to_layers()` 将对应 Tensor 绑定到 Attention 对象：

```
attention.self.kv_cache = kv_cache
```

最终 Attention 层可以访问自己的 KV Cache。

## **6. 编译与预热**

**调用链：**

```
compile_or_warm_up_model()
    → Worker.compile_or_warm_up_model()
```

主要操作：

| 操作                  | 作用                    |
| ------------------- | --------------------- |
| 编译尺寸准备              | 确定 Batch Token Size   |
| `_dummy_run()`      | 虚拟输入执行模型              |
| `kernel_warmup()`   | CUDA Kernel 预热        |
| `warmup_kernels()`  | V2 Model Runner 额外预热  |
| `capture_model()`   | CUDA Graph Capture    |
| Sampler / Pooler 预热 | 分配相关缓冲区               |
| 运行时优化               | Inductor 初始化、GC、线程配置等 |

## **7. 初始化与运行时管理**

两者是不同阶段：

| 对比   | 初始化阶段                          | 运行时阶段                    |
| ---- | ------------------------------ | ------------------------ |
| 目标   | 创建 KV Cache 显存                 | 管理 KV Cache Block        |
| 核心对象 | KVCacheSpec、KVCacheConfig      | BlockPool、KVCacheManager |
| 主要操作 | 分配 Tensor、创建 View、绑定 Attention | 分配、释放、复用 Block           |
| 数据结构 | `kv_caches`                    | `req_to_blocks`、Hash 映射  |

运行时的基本关系：

```
Request
   |
   v
req_to_blocks
   |
   v
BlockPool
   |
   +-- get_new_blocks()
   +-- get_cached_block()
   +-- free_blocks
   |
   v
KVCacheBlock
```

Prefix Cache 则通过 Block Hash 查找和复用已有缓存。

## **8. 核心类与方法**

| 类 / 模块         | 核心方法                           | 作用                |
| -------------- | ------------------------------ | ----------------- |
| EngineCore     | `_initialize_kv_caches()`      | 协调初始化             |
| ModelExecutor  | `get_kv_cache_specs()`         | 获取缓存规格            |
| Worker         | `determine_available_memory()` | 显存评估              |
| Worker         | `initialize_from_config()`     | 初始化分发             |
| GPUModelRunner | `get_kv_cache_spec()`          | 获取 Attention Spec |
| GPUModelRunner | `initialize_kv_cache()`        | 初始化 KV Cache      |
| Attention      | `get_kv_cache_spec()`          | 生成缓存规格            |
| Attention      | `self.kv_cache`                | 保存缓存 Tensor       |
| attn_utils     | `init_kv_cache()`              | 执行缓存初始化           |
| BlockPool      | `get_new_blocks()`             | 运行时分配 Block       |
| BlockPool      | `get_cached_block()`           | 查询缓存 Block        |

## **9. 总结**

记住 KV Cache 初始化的核心链路：

```
Spec 获取
   ↓
Layout 确定
   ↓
GPU 显存预算
   ↓
生成 KVCacheConfig
   ↓
torch.zeros() 分配物理显存
   ↓
torch.as_strided() 创建 Tensor View
   ↓
绑定 Attention
   ↓
模型编译与预热
```

**核心理解：**

- `KVCacheSpec`：定义缓存需求。
- `KVCacheConfig`：描述缓存分配配置。
- `torch.zeros()`：分配底层物理显存。
- `torch.as_strided()`：创建逻辑 Tensor View。
- `kv_caches`：维护 Layer 与 Cache Tensor 的映射。
- `BlockPool`：负责运行时 Block 生命周期管理。

资料依据：用户提供的 vLLM 源码分析记录。仅整理已提供的调用链、代码和数据结构，不补充未验证的源码实现。&#x20;

粘贴的文本 (1)(20261127-132951).txt
