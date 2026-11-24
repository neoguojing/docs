/home/neo/workspace/vllm/vllm/v1/core/single_type_kv_cache_manager.py
/home/neo/workspace/vllm/vllm/v1/kv_cache_spec_registry.py
/home/neo/workspace/vllm/vllm/v1/kv_cache_interface.py
/home/neo/workspace/vllm/vllm/v1/core/single_type_kv_cache_manager.py

kv_cache_specs = self.model_executor.get_kv_cache_specs()
def get_kv_cache_specs(self) -> list[dict[str, KVCacheSpec]]:
        return self.collective_rpc("get_kv_cache_spec")

self.model_executor.get_supported_kv_cache_layouts(),
def get_supported_kv_cache_layouts(self) -> list[list[str]]:
        """Layouts each worker's backends support, most preferred first."""
        return self.collective_rpc("get_supported_kv_cache_layouts")
self.model_executor.set_kv_cache_layout(layout.name)

available_gpu_memory = self.model_executor.determine_available_memory()
def determine_available_memory(self) -> list[int]:  # in bytes
        return self.collective_rpc("determine_available_memory")
self.model_executor.initialize_from_config(kv_cache_configs)
def initialize_from_config(self, kv_cache_configs: list[KVCacheConfig]) -> None:
        """Initialize the KV caches on the underlying workers."""
        self.collective_rpc("initialize_from_config", args=(kv_cache_configs,))
        
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
下面整理成一份可直接放入 GitHub 的 Markdown 文档。重点突出 **注册机制 → Spec 类型与容量计算 → Manager 核心接口**，保留关键参数和公式。

# vLLM KV Cache Spec 与 Manager 设计

## 1. KVCacheSpec 注册机制

vLLM 使用 Registry Pattern 注册不同类型的 `KVCacheSpec`，建立 **Spec 类型、Manager 类型、统一类型基类** 之间的映射关系。

注册入口：

```
register_all_kvcache_specs(vllm_config)
```

核心注册逻辑：

```
_REGISTRY_KVCACHESPEC_LIST[kvcache_spec_cls] = (
    KVCacheSpecMetadata(
        kvcache_spec_cls,
        manager_class,
        uniform_type_base_spec,
    )
)
```

注册信息：

| 信息                       | 作用                      |
| ------------------------ | ----------------------- |
| `kvcache_spec_cls`       | KV Cache 格式描述类          |
| `manager_class`          | 对应的 Cache 管理器           |
| `uniform_type_base_spec` | 判断不同 Spec 是否属于同一统一类型的基类 |

注册时会检查：

- `manager_class` 必须存在。
- `uniform_type_base_spec` 必须是 Spec 类的父类或自身。
- 重复注册时，Manager 和统一类型基类必须一致。

**注册的目的：** 将不同 Cache 格式与管理策略解耦，后续根据 Spec 类型查询对应 Manager，并通过统一类型基类判断不同层能否合并为同一个 Cache Group。

注意：注册只建立类型映射，不负责创建 Cache 或分配显存。

## 2. KVCacheSpec 类型与容量计算

`KVCacheSpec` 描述 Cache 的存储格式、容量需求和分组约束。

通用容量公式：

[\
N\_{\text{blocks}} =\
\left\lceil \frac{N\_{\text{tokens}}}{B\_{\text{block}}} \right\rceil\
]

[\
S\_{\text{cache}} =\
N\_{\text{blocks}} \times S\_{\text{page}}\
]

其中：

- (B\_{\text{block}})：每个 Block 容纳的 Token 数。
- (S\_{\text{page}})：一个 Block 对应的字节数，即 `page_size_bytes`。
- (N\_{\text{tokens}})：需要保存的 Token 数。

### 2.1 Attention 类 Cache

| Spec 类型                     | 用途 / 为什么需要                                                          | 关键参数                                                                          | 容量计算与特点                                                                                   |
| --------------------------- | ------------------------------------------------------------------- | ----------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------- |
| `FullAttentionSpec`         | 普通全量 Attention，保留完整历史 K/V                                           | `num_kv_heads`、`head_size`、`head_size_v`、`dtype`、`kv_quant_mode`              | (N=\lceil L/B\rceil)，(S=N\times S\_{\text{page}})。Page 大小由 KV Head、维度和数据类型决定              |
| `SlidingWindowSpec`         | 滑动窗口 Attention，只保留窗口内历史 KV，降低长序列显存占用                                | `sliding_window`、`extra_retained_tokens`                                      | 有效保留量受窗口限制；(N\_{\text{admission}}=\lceil(\min(W+T\_{\text{flight}},L))/B\rceil)，具体计算以实现为准 |
| `ChunkedLocalAttentionSpec` | 分块局部 Attention，限制每次计算需要保留的历史范围                                      | `attention_chunk_size`                                                        | (N=\lceil\min(C+T\_{\text{flight}},L)/B\rceil)，其中 (C) 为 Chunk 大小                          |
| `CrossAttentionSpec`        | Encoder-Decoder 模型缓存 Encoder 输出，避免重复计算                              | 继承 Attention 基础维度                                                             | (S=\lceil L\_{\text{encoder}}/B\rceil\times S\_{\text{page}})                             |
| `EncoderOnlyAttentionSpec`  | Encoder-only 层，不需要 Decoder KV Cache                                 | 继承字段                                                                          | (S=0)                                                                                     |
| `MLAAttentionSpec`          | MLA 使用 Latent Cache，避免普通 K/V 分离存储                                   | `cache_dtype_str`、`alignment`、`model_version`、`cache_role`、`tokens_per_state` | (S=N\_{\text{states}}\times S\_{\text{state}})，Page 大小受对齐和 MLA 存储布局影响                     |
| `SlidingWindowMLASpec`      | MLA 与滑动窗口结合，限制 Latent Cache 保留范围                                    | `sliding_window`、`extra_retained_tokens`、`alignment`、`model_version`          | 窗口限制 Block 数，同时按 MLA 格式计算 Page 大小                                                         |
| `SinkFullAttentionSpec`     | Sink Attention，保留指定 Sink Token 与全量 Attention Cache 相关状态             | `sink_len`                                                                    | 继承 Full Attention 的 Page 计算，额外考虑 Sink Token 保留策略                                          |
| `RSWASpec`                  | Reference Sliding Window Attention，Prefill 保留完整可见历史，Decode 回收中间 Gap | `rswa_window`                                                                 | 总 Block 数仍按 Token 位置计算；Decode 阶段可回收中间 Gap，降低实际持有量                                         |
| `HiddenStateCacheSpec`      | 为 `extract_hidden_states` 提供专用 Hidden State Cache 标记                | 继承 `MLAAttentionSpec`                                                         | 继承 MLA 的容量和 Page 计算逻辑                                                                     |

### 2.2 特殊 Cache

| Spec 类型                | 用途 / 为什么需要                              | 关键参数                                                                                       | 容量计算与特点                                                                                                                 |
| ---------------------- | --------------------------------------- | ------------------------------------------------------------------------------------------ | ----------------------------------------------------------------------------------------------------------------------- |
| `MambaSpec`            | Mamba / 状态空间模型保存状态张量，而非普通 Attention K/V | `shapes`、`dtypes`、`mamba_type`、`mamba_cache_mode`、`num_speculative_blocks`、`tp_replicated` | (S\_{\text{state}}=\sum_i \operatorname{prod}(shape_i)\times sizeof(dtype_i))；(S\_{\text{page}}) 可使用 `page_size_padded` |
| `CircularBufferSpec`   | Token Group 压缩期间保存原始 Key，使用环形缓冲区        | 继承 Attention 维度                                                                            | 每请求固定 1 个 Block：(N=1)，(S=S\_{\text{page}})                                                                              |
| `KpoolTailSpec`        | Kpool Indexer 保存原始 Tail 的临时环形缓存         | 继承 `SlidingWindowSpec`                                                                     | 每请求固定 1 个 Block：(N=1)                                                                                                   |
| `HiSparseHotSpec`      | HiSparse 请求级临时热缓存                       | `page_size`、`blocks_per_request`                                                           | (S=blocks_per_request\times page_size)                                                                                  |
| `HiSparseResidentSpec` | HiSparse GPU 常驻页缓存                      | `page_size`                                                                                | 按模型最大长度和 Block Size 计算容量，Page 大小为 `page_size`                                                                           |

说明：

1. `block_size` 决定一个 Block 能容纳多少 Token；`page_size_bytes` 决定一个 Block 占多少字节。
2. Attention Cache 的 Page 大小与 KV Head 数、Head Dimension、数据类型及量化格式有关。
3. Mamba Cache 按状态张量的 Shape 和 dtype 计算，不适用普通 K/V Head 公式。
4. SWA、Chunked Local 等类型可以回收历史 Block，因此 Admission 容量不一定等于完整序列的 Block 数。
5. 表中公式是容量计算的概念表达；具体实现还可能考虑对齐、并行切分、Speculative Decoding 和额外保留状态。

## 3. SingleTypeKVCacheManager

`SingleTypeKVCacheManager` 是单一 KV Cache Group 的抽象管理器，负责请求与 Block 的映射、分配、Prefix Cache、回收及 Worker 协作。

组件职责：

| 组件                         | 职责                          |
| -------------------------- | --------------------------- |
| `KVCacheSpec`              | 描述 Cache 格式和容量需求            |
| `SingleTypeKVCacheManager` | 管理请求使用哪些 Block，以及何时分配、缓存、回收 |
| `BlockPool`                | 实际分配、释放、管理 Block 引用计数和缓存    |
| `KVCacheBlock`             | 表示一个逻辑 Cache Block          |
| Worker                     | 执行实际 GPU Cache 初始化、清零及数据复制  |

### 3.1 核心状态

| 成员                           | 作用                                            |
| ---------------------------- | --------------------------------------------- |
| `kv_cache_spec`              | 当前 Manager 对应的 Cache Spec                     |
| `block_pool`                 | 底层 Block 分配池                                  |
| `block_size`                 | Manager 实际分配粒度，DCP 大于 1 时会乘以 `dcp_world_size` |
| `scheduler_block_size`       | Scheduler 调度粒度，通常为各 Group Block Size 的 LCM    |
| `cache_hit_alignment_tokens` | Prefix Cache 命中对齐粒度                           |
| `kv_cache_group_id`          | Cache Group ID                                |
| `req_to_blocks`              | `request_id -> list[KVCacheBlock]`            |
| `num_cached_block`           | 运行请求已经缓存的 Block 数                             |
| `new_block_ids`              | 新分配、需要 Worker 清零的 Block ID                    |
| `_partial_hit_reqs`          | 部分 Prefix Cache 命中的 CoW 记录                    |
| `_pending_cow_copies`        | 等待 Worker 执行的 Block Copy                      |
| `_null_block`                | 被回收或跳过位置的空 Block 占位符                          |

### 3.2 核心接口

| 功能          | 接口                                       | 作用 / 实现原理                                                 |
| ----------- | ---------------------------------------- | --------------------------------------------------------- |
| 计算分配量       | `get_num_blocks_to_allocate()`           | 根据所需 Block、已有 Block、Prefix 命中、跳过的窗口 Block 和 CoW 额外需求计算分配量 |
| 本地缓存命中      | `add_local_computed_blocks()`            | 将本地 Prefix Cache 命中 Block 加入请求，并更新引用关系                    |
| 外部缓存命中      | `allocate_external_computed_blocks()`    | 为外部 KV Connector 已计算的 Token 分配逻辑 Block                    |
| 分配 Block    | `allocate_new_blocks()`                  | 调用 `block_pool.get_new_blocks()`，更新 `req_to_blocks`       |
| 查找缓存        | `find_longest_cache_hit()`               | 查找最长 Prefix Cache 命中；抽象方法，由不同 Manager 实现                  |
| 写入缓存        | `cache_blocks()`                         | 将已经计算完成的完整 Block 放入 Prefix Cache                          |
| 公共前缀        | `get_num_common_prefix_blocks()`         | 计算运行请求之间的公共前缀 Block 数；抽象方法                                |
| 可缓存范围       | `reachable_block_mask()`                 | 决定哪些 Block 可以参与 Prefix Cache                              |
| 回收历史 Block  | `remove_skipped_blocks()`                | 根据 Attention Window 计算不再需要的历史 Block，并释放                   |
| 跳过 Token    | `get_num_skipped_tokens()`               | 计算 Attention 不再访问的历史 Token 数，子类可重写                        |
| 释放请求        | `free()`                                 | 清理请求状态并将 Block 归还 BlockPool                               |
| CoW         | `_apply_cow()`                           | 将共享尾 Block 替换成私有 Block，记录待复制任务                            |
| Worker 清零   | `take_new_block_ids()`                   | 获取并清空需要 Worker 清零的 Block ID                               |
| Worker Copy | `take_pending_cow_copies()`              | 获取待执行的源、目标 Block Copy 对                                   |
| 外部 Offload  | `take_pending_boundary_state_offloads()` | 获取 Mamba Align 边界状态的 Offload 记录                           |

### 3.3 关键计算公式

**（1）基础 Block 需求**

[\
N\_{\text{required}}=\
\left\lceil\frac{T}{B}\right\rceil\
]

其中 (T) 是需要 Cache Slot 的 Token 数，(B) 是 Manager 的 Block Size。

**（2）普通运行请求的新增 Block**

[\
N\_{\text{new}}=\
\max(0,N\_{\text{required}}-N\_{\text{existing}})\
]

代码对应：

```
return max(num_required_blocks - num_req_blocks, 0)
```

**（3）考虑滑动窗口和 Prefix Cache 的新请求**

```
num_new_blocks = max(
    num_required_blocks
    - max(num_skipped_blocks, num_local_computed_blocks),
    0,
)
```

其中：

- `num_skipped_blocks`：Attention Window 已经跳过的 Block 数。
- `num_local_computed_blocks`：本地命中及当前已有的 Block 数。

还需要加上可驱逐命中 Block 的容量，以及部分命中时 CoW 所需的额外 Block：

[\
N\_{\text{allocate}}=\
N\_{\text{new}}+\
N\_{\text{evictable}}+\
N\_{\text{CoW}}\
]

**（4）Admission 上限**

对支持历史 Block 回收的 Spec：

[\
N\_{\text{required}}=\
\min\left(\
\left\lceil\frac{T}{B}\right\rceil,\
N\_{\text{admission-cap}}\
\right)\
]

避免长序列请求按照完整历史长度预留所有 Block。

**（5）Block 回收**

[\
N\_{\text{skipped-blocks}}=\
\left\lfloor\
\frac{T\_{\text{skipped}}}{B}\
\right\rfloor\
]

回收范围：

```
[0, num_skipped_blocks)
```

回收后用 `null_block` 替换原位置，保持 Block Table 的位置语义。

## 4. 总体关系

```
register_all_kvcache_specs()
          |
          v
 KVCacheSpec Registry
          |
          | Spec Class -> Manager Class
          v
   get_kv_cache_specs()
          |
          v
  KVCacheGroupSpec / KVCacheConfig
          |
          v
   KVCacheManager
          |
          +-----------------------+
          |                       |
          v                       v
 SingleTypeManager 0      SingleTypeManager N
          |                       |
          +-----------+-----------+
                      |
                      v
                   BlockPool
                      |
                      v
                 KVCacheBlock
                      |
                      v
               Worker / GPU Cache
```

**核心结论：**

- `KVCacheSpec` 解决 **Cache 如何存储、需要多少空间**。
- Registry 解决 **不同 Spec 对应哪个 Manager，以及如何统一分组**。
- `SingleTypeKVCacheManager` 解决 **请求如何分配、复用和释放 Block**。
- `BlockPool` 解决 **Block 的实际分配、引用计数和回收**。
- Worker 负责 **GPU 侧 Cache 的实际初始化与数据操作**。

统一 Cache 术语和大小写压缩 Spec 表格内容区分分配与实际占用
