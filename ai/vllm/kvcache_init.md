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
