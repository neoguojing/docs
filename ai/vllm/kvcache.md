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
```
下面整理成一份可直接放入 GitHub 的 Markdown 文档。重点突出 **注册机制 → Spec 类型与容量计算 → Manager 核心接口**，保留关键参数和公式。
下面是完整修订版：**所有数学公式全部改为纯文本表达**，不使用 LaTeX、`$`、`$$`，确保可以直接复制到 GitHub Markdown 中正常显示。

# vLLM KV Cache Spec 与 Manager 设计

## 1. KVCacheSpec 注册机制

### 1.1 作用

vLLM 通过 `KVCacheSpecRegistry` 建立 KV Cache Spec 与 Manager 的映射关系，使不同类型的 Cache 可以使用对应的管理器实现。

主要解决三个问题：

1. **类型识别**：识别模型各层需要使用的 KV Cache 类型。
2. **Manager 选择**：根据 Spec 类型找到对应的 Manager。
3. **统一类型判断**：通过 `uniform_type_base_spec` 判断不同 Spec 是否属于同一类 Cache。

注册关系：

```
KVCacheSpecRegistry
        |
        | register()
        v
Spec Class -> KVCacheSpecMetadata
                    |
                    +-- kvcache_spec_cls
                    +-- manager_class
                    +-- uniform_type_base_spec
```

### 1.2 注册过程

核心入口：

```
register_all_kvcache_specs(vllm_config)
```

主要完成：

- 注册各种 KVCacheSpec 对应的 Manager。
- 建立 Spec 类型到 Metadata 的映射。
- 校验 Spec 与 uniform base 类型之间的继承关系。
- 检查重复注册时 Manager 和 uniform base 是否一致。

**注意：注册只建立类型映射，不创建 Cache，也不分配 GPU 显存。**

### 1.3 典型注册关系

| Spec 类型                   | Manager 类型                   | Uniform Base Spec         |
| ------------------------- | ---------------------------- | ------------------------- |
| FullAttentionSpec         | FullAttentionManager         | FullAttentionSpec         |
| SlidingWindowSpec         | SlidingWindowManager         | SlidingWindowSpec         |
| ChunkedLocalAttentionSpec | ChunkedLocalAttentionManager | ChunkedLocalAttentionSpec |
| CrossAttentionSpec        | CrossAttentionManager        | CrossAttentionSpec        |
| MLAAttentionSpec          | FullAttentionManager         | FullAttentionSpec         |
| SlidingWindowMLASpec      | SlidingWindowManager         | SlidingWindowMLASpec      |
| SinkFullAttentionSpec     | SinkFullAttentionManager     | FullAttentionSpec         |
| RSWASpec                  | RSWAManager                  | FullAttentionSpec         |
| HiddenStateCacheSpec      | FullAttentionManager         | FullAttentionSpec         |
| MambaSpec                 | MambaManager                 | MambaSpec                 |
| CircularBufferSpec        | CircularBufferManager        | CircularBufferSpec        |
| KpoolTailSpec             | KpoolTailManager             | KpoolTailSpec             |
| HiSparseHotSpec           | HiSparseHotManager           | HiSparseHotSpec           |
| HiSparseResidentSpec      | HiSparseResidentManager      | HiSparseResidentSpec      |

另外，Role Manager 还存在独立注册关系，例如：

```
KVCacheGroupRole.HISPARSE_SOURCE
              |
              v
      HiSparseSourceManager
```

### 1.4 Uniform Type 的作用

`uniform_type_base_spec` 用于判断一组 Spec 是否属于同一种统一类型。

相关接口：

- `KVCacheSpecRegistry.get_uniform_type_base_spec()`
- `KVCacheSpec.is_uniform_with_collection()`
- `UniformTypeKVCacheSpecs.from_specs()`
- `UniformTypeKVCacheSpecs.is_uniform_type()`

注意：Uniform Type 表示类型兼容，不代表所有 Cache 参数完全相同。实际合并仍然需要满足对应的 Spec 约束。

---

## 2. KVCacheSpec 类型与容量计算

`KVCacheSpec` 描述 Cache 的数据结构、Block 大小及容量计算方式。

不同模型结构需要不同的 Cache 管理方式：

- 普通 Attention：保存 K/V。
- Sliding Window Attention：只保留局部窗口。
- MLA：使用不同于传统 K/V 的 Cache 表示。
- Mamba：保存状态张量。
- Circular Buffer：循环复用固定容量。
- HiSparse：区分 Hot Cache 和 Resident Cache。

### 2.1 通用容量计算

KV Cache 通常以 Block 为单位管理。

定义：

- `T`：需要缓存的 Token 数量。
- `B`：一个 Block 对应的 Token 数量。
- `S_block`：一个 Block 占用的字节数。
- `N_blocks`：需要的 Block 数量。

**Block 数量：**

```
N_blocks = ceil(T / B)
```

其中 `ceil()` 表示向上取整。

**总 Cache 容量：**

```
S_cache = N_blocks * S_block
```

**传统 Attention 单个 Block 的理论容量：**

```
S_block = 2 * H_kv * B * D_head * S_dtype
```

其中：

- `2`：分别保存 Key 和 Value。
- `H_kv`：KV Head 数量。
- `B`：Block Size。
- `D_head`：Head Dimension。
- `S_dtype`：每个元素占用的字节数。

该公式适用于传统 K/V 存储模型；特殊 Cache 类型需要根据各自 Spec 计算。

### 2.2 Cache 类型对比

| Spec 类型                     | 作用与为什么需要                                                                | 关键参数                                                                                                                                                                         | 容量计算与特点                                            |
| --------------------------- | ----------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | -------------------------------------------------- |
| `FullAttentionSpec`         | 标准全量 Attention，保存历史 Token 的 K/V，支持完整上下文注意力                              | `num_kv_heads`、`head_size`、`head_size_v`、`dtype`、`block_size`                                                                                                                | `S_block = 2 * H_kv * B * D_head * S_dtype`        |
| `SlidingWindowSpec`         | 滑动窗口 Attention，只需要保留最近窗口内的 Token，降低长上下文 Cache 占用                        | `sliding_window`、`extra_retained_tokens`                                                                                                                                     | 有效 Token 数受窗口限制；`N_blocks ≈ ceil(T_effective / B)` |
| `ChunkedLocalAttentionSpec` | 分块局部 Attention，限制每次计算所需的局部历史，降低 Cache 需求                                | `attention_chunk_size`                                                                                                                                                       | 容量取决于局部 Chunk 的保留策略，不能简单按完整上下文长度计算                 |
| `CrossAttentionSpec`        | 用于 Cross Attention，保存供 Decoder 查询的 Encoder 侧 K/V                        | Attention 相关维度参数                                                                                                                                                             | Cache 需求与 Encoder 输出及其 K/V 表示有关                    |
| `MLAAttentionSpec`          | 适配 Multi-head Latent Attention，使用 MLA 特定的 Cache 表示，减少传统 K/V Cache 的存储开销 | `cache_dtype_str`、`alignment`、`model_version`、`cache_role` 等                                                                                                                 | 根据 MLA Cache 的实际表示、对齐和数据类型计算，不能直接套用传统 K/V 公式       |
| `SlidingWindowMLASpec`      | 将滑动窗口机制与 MLA Cache 结合                                                   | `sliding_window`、`extra_retained_tokens`、`alignment`、`model_version` 等                                                                                                       | 容量同时受窗口限制和 MLA Cache 布局影响                          |
| `SinkFullAttentionSpec`     | 支持 Attention Sink，保留开头的 Sink Token，同时维护后续 Attention Cache               | `sink_len`                                                                                                                                                                   | 容量受 Sink Token 与普通历史 Cache 的保留策略影响                 |
| `RSWASpec`                  | 实现 RSW​​A 窗口管理，控制历史 Cache 的保留范围                                         | `rswa_window`                                                                                                                                                                | 容量受窗口参数影响；可能通过回收中间 Block 控制占用                      |
| `HiddenStateCacheSpec`      | 管理 Hidden State Cache，满足需要缓存隐藏状态的模型执行场景                                 | 以具体 Spec 定义为准                                                                                                                                                                | 根据 Hidden State 的张量形状和数据类型计算                       |
| `MambaSpec`                 | 管理 Mamba 等状态空间模型的状态 Cache，不采用传统 Attention K/V 存储形式                      | `shapes`、`dtypes`、`page_size_padded`、`mamba_type`、`mamba_cache_mode`、`num_speculative_blocks`、`num_prefill_checkpoint_blocks`、`prefill_checkpoint_alignment`、`tp_replicated` | 按状态张量大小计算，见下文                                      |
| `CircularBufferSpec`        | 使用固定容量循环 Buffer，适用于不需要持续保存完整历史的场景                                       | Attention 相关参数、`block_size`                                                                                                                                                  | 单请求固定一个 Block，通过循环复用控制容量                           |
| `KpoolTailSpec`             | 管理 KPool 尾部 Cache，采用固定 Block 的管理方式                                      | 继承 Sliding Window 相关参数                                                                                                                                                       | 单请求固定一个 Block                                      |
| `HiSparseHotSpec`           | 管理 HiSparse 中的 Hot Cache，保存热点数据                                         | `page_size`、`blocks_per_request`                                                                                                                                             | 容量与 `blocks_per_request * page_size` 相关            |
| `HiSparseResidentSpec`      | 管理 HiSparse 中的 Resident Cache，保存驻留数据                                    | `page_size`                                                                                                                                                                  | 按 Resident Cache 的 Block 和 Page 布局计算               |

注：

1. 表中容量公式是理解 Cache 规模的通用模型，实际实现还可能考虑对齐、量化、Padding、并行切分和额外保留空间。
2. `SlidingWindowSpec`、`ChunkedLocalAttentionSpec`、`RSWASpec` 等类型的实际 Block 分配量还取决于 Manager 的保留和回收策略。
3. 特殊 Spec 的字段和容量计算应以对应源码实现为准。

### 2.3 Mamba Cache 容量计算

Mamba 不保存传统 Attention 的 K/V，而是保存状态张量。

设第 `i` 个状态张量的 Shape 为 `shape_i`，数据类型字节数为 `S_dtype_i`：

```
S_state = sum(prod(shape_i) * S_dtype_i)
```

其中：

- `prod(shape_i)`：该状态张量所有维度相乘，得到元素总数。
- `S_dtype_i`：每个元素占用的字节数。
- `sum()`：累加所有状态张量的存储大小。

如果使用 Padding 后的 Page Size：

```
S_block = S_state_padded
```

相关参数：

- `shapes`：状态张量的形状。
- `dtypes`：状态张量的数据类型。
- `page_size_padded`：Padding 后的 Page 大小。
- `mamba_cache_mode`：影响 Cache 的管理和容量策略。
- `num_speculative_blocks`：Speculative Decoding 相关 Block 数量。
- `num_prefill_checkpoint_blocks`：Prefill Checkpoint 相关 Block 数量。

Mamba Cache 的实际容量还需要结合 Cache Mode、Checkpoint 和 Speculative 配置计算。

---

## 3. SingleTypeKVCacheManager

### 3.1 核心职责

`SingleTypeKVCacheManager` 是单一 KV Cache Group 的抽象管理器。

它负责：

1. 计算请求需要多少 Cache Block。
2. 为请求分配和回收 Block。
3. 管理请求与 Block 的映射关系。
4. 处理 Prefix Cache 命中。
5. 处理 Copy-on-Write。
6. 管理新分配 Block 的通知和状态交接。

它不负责：

- 执行 Attention。
- 执行 GPU Tensor 分配。
- 执行模型推理。
- 直接计算 K/V 内容。

### 3.2 相关组件关系

```
                 Scheduler
                     |
                     v
              KVCacheManager
                     |
                     v
        SingleTypeKVCacheManager
                     |
          +----------+----------+
          |                     |
          v                     v
      BlockPool              KVCacheSpec
          |                     |
          v                     v
     KVCacheBlock         Cache 类型和容量
          |
          v
    Block 生命周期管理

Worker:
    根据 KVCacheConfig 初始化实际 GPU Cache Tensor
```

组件职责：

| 组件                         | 核心职责                          |
| -------------------------- | ----------------------------- |
| `KVCacheSpec`              | 描述 Cache 类型、数据布局和容量           |
| `KVCacheManager`           | 管理多个 KV Cache Group           |
| `SingleTypeKVCacheManager` | 管理单个 Cache Group 的 Block 生命周期 |
| `BlockPool`                | 实际分配、释放、缓存和引用计数管理             |
| `KVCacheBlock`             | 表示一个逻辑 Cache Block            |
| Worker                     | 初始化并使用 GPU 上的实际 Cache Tensor  |

### 3.3 初始化关键状态

| 成员                                  | 作用                        |
| ----------------------------------- | ------------------------- |
| `kv_cache_spec`                     | 当前 Group 的 Cache Spec     |
| `block_pool`                        | 负责 Block 分配和回收            |
| `enable_caching`                    | 是否启用 Cache                |
| `kv_cache_group_id`                 | 当前 Cache Group ID         |
| `block_size`                        | 当前 Manager 的 Block Size   |
| `scheduler_block_size`              | 所有 Group Block Size 的 LCM |
| `cache_hit_alignment_tokens`        | Cache Hit 对齐 Token 数      |
| `dcp_world_size`                    | DCP 并行规模                  |
| `pcp_world_size`                    | PCP 并行规模                  |
| `_max_admission_blocks_per_request` | 单请求最大允许使用的 Block 数        |
| `req_to_blocks`                     | Request ID 到 Block 列表的映射  |
| `num_cached_block`                  | 当前请求已缓存 Block 数量          |
| `_null_block`                       | 跳过 Block 的占位对象            |
| `new_block_ids`                     | 本轮新分配 Block ID            |
| `_record_new_block_ids`             | 是否记录需要通知 Worker 的新 Block  |
| `_partial_hit_reqs`                 | 部分 Prefix Cache 命中请求      |
| `_pending_cow_copies`               | 等待执行的 Copy-on-Write 操作    |
| `_pending_boundary_state_offloads`  | 等待处理的边界状态 Offload         |

其中，`req_to_blocks` 是请求和逻辑 Cache Block 之间最重要的映射：

```
Request A -> [Block 1, Block 2, Block 3]
Request B -> [Block 4, Block 5]
Request C -> [Block 6, Block 7, Block 8]
```

注意：`block_size` 在 DCP 并行场景下可能根据 `dcp_world_size` 调整；`scheduler_block_size` 用于协调不同 Cache Group 的调度粒度。

### 3.4 核心接口

| 功能分类          | 接口                                       | 作用                          |
| ------------- | ---------------------------------------- | --------------------------- |
| Block 计算      | `get_num_blocks_to_allocate()`           | 计算当前请求需要新增多少 Block          |
| 容量计算          | `_get_num_evictable_blocks()`            | 计算可以通过驱逐获得的 Block 数量        |
| Cache 命中      | `_has_partial_local_hit()`               | 判断是否存在部分本地 Cache 命中         |
| 跳过 Token      | `get_num_skipped_tokens()`               | 计算需要跳过的 Token 数量            |
| 本地 Cache      | `add_local_computed_blocks()`            | 将本地计算得到的 Cache Block 加入请求管理 |
| 外部 Cache      | `allocate_external_computed_blocks()`    | 分配外部计算结果对应的 Cache Block     |
| 新 Block 分配    | `allocate_new_blocks()`                  | 为请求分配新的 Cache Block         |
| Prefix Cache  | `find_longest_cache_hit()`               | 查找最长 Prefix Cache 命中，抽象接口   |
| Prefix Cache  | `cache_blocks()`                         | 将请求 Block 加入 Cache          |
| Prefix Cache  | `get_num_common_prefix_blocks()`         | 计算多个请求的公共 Prefix Block，抽象接口 |
| Block 可达性     | `reachable_block_mask()`                 | 计算 Block 的可达性 Mask          |
| Block 回收      | `remove_skipped_blocks()`                | 移除被跳过的 Block                |
| Block 回收      | `_remove_blocks_in_range()`              | 移除指定范围的 Block               |
| Block 回收      | `pop_blocks_for_free()`                  | 从请求映射中取出待释放 Block           |
| Block 回收      | `free()`                                 | 释放请求占用的 Block               |
| Copy-on-Write | `_apply_cow()`                           | 执行 CoW 相关 Block 处理          |
| Copy-on-Write | `take_pending_cow_copies()`              | 获取等待执行的 CoW 操作              |
| Worker 通知     | `take_new_block_ids()`                   | 获取新分配 Block ID              |
| Worker 通知     | `take_pending_boundary_state_offloads()` | 获取待处理的边界状态 Offload          |
| Worker 通知     | `finalize_partial_tail_offload()`        | 完成 Partial Tail Offload     |
| 生命周期          | `new_step_starts()`                      | 开始新的调度 Step                 |

其中，以下三个接口由具体 Manager 子类实现或提供差异化逻辑：

- `find_longest_cache_hit()`
- `get_num_common_prefix_blocks()`
- `get_num_skipped_tokens()`

这也是不同 Cache 类型在 Prefix Cache 和 Block 回收策略上的主要差异之一。

---

## 4. Manager 核心计算原理

### 4.1 普通 Block 分配

设：

- `T`：请求需要缓存的 Token 数量。
- `B`：Block Size。
- `N_existing`：请求当前已有 Block 数量。

需要的 Block 数量：

```
N_required = ceil(T / B)
```

需要新增的 Block 数量：

```
N_new = max(N_required - N_existing, 0)
```

### 4.2 Prefix Cache 命中与跳过 Block

当请求存在 Prefix Cache 命中或跳过 Token 时，部分历史 Token 不需要重新分配 Block。

简化理解：

```
N_new = max(N_required - max(N_skipped, N_cached), 0)
```

其中：

- `N_skipped`：跳过的 Block 数量。
- `N_cached`：已命中的本地 Cache Block 数量。

实际实现还需要考虑部分命中、Block 对齐、CoW 和驱逐等情况，不能仅用该公式替代源码中的完整分配逻辑。

### 4.3 Admission 限制

对于 Sliding Window 等具有最大保留容量的 Cache，单请求实际需要的 Block 数可能受到 Admission 上限限制。

```
N_required = min(ceil(T / B), N_admission_max)
```

这样可以避免请求长度持续增长时，Cache 占用也无限增长。

### 4.4 Copy-on-Write

多个请求可能共享相同的 Prefix Cache Block。

当其中一个请求需要修改共享 Block 时，需要执行 Copy-on-Write：

```
Request A ----+
              |
              +---- Shared Block
              |
Request B ----+

Request B 需要修改 Block：

Request A -------- Shared Block
Request B -------- New Block
```

核心思想：

- 只读共享：多个请求可以引用同一个 Block。
- 写入修改：为需要修改的请求分配独立 Block。
- 避免直接修改共享数据，保证其他请求的 Cache 内容不受影响。

因此，Block 分配时还可能需要额外预留 CoW Block。

### 4.5 Block 回收

请求完成或不再需要某些历史 Cache 时，Manager 将 Block 从请求映射中移除，再交给 `BlockPool` 处理释放。

```
Scheduler
    |
    v
KVCacheManager.free()
    |
    v
SingleTypeKVCacheManager.free()
    |
    v
pop_blocks_for_free()
    |
    v
BlockPool.free_blocks()
    |
    v
Block 回到可复用池
```

Block 是否能立即复用，还取决于引用计数、Prefix Cache 和 BlockPool 的管理状态。

---

## 5. 核心流程总结

### 5.1 请求分配 Cache

```
Scheduler
    |
    v
KVCacheManager.allocate_slots()
    |
    v
get_num_blocks_to_allocate()
    |
    +---- 计算需求 Block 数量
    +---- 检查 Prefix Cache 命中
    +---- 计算跳过 Block
    +---- 检查可驱逐容量
    +---- 考虑 CoW
    |
    v
allocate_new_blocks()
    |
    v
BlockPool.get_new_blocks()
    |
    v
更新 req_to_blocks
    |
    v
Worker 使用对应 Cache Block
```

### 5.2 请求释放 Cache

```
Scheduler
    |
    v
KVCacheManager.free()
    |
    v
SingleTypeKVCacheManager.free()
    |
    v
pop_blocks_for_free()
    |
    v
BlockPool.free_blocks()
    |
    v
Block 进入可复用状态
```

### 5.3 核心概念关系

| 概念                         | 解决的问题                             |
| -------------------------- | --------------------------------- |
| `KVCacheSpec`              | Cache 应该如何存储？                     |
| `KVCacheSpecRegistry`      | 根据 Spec 类型选择哪个 Manager？           |
| `KVCacheGroupSpec`         | 哪些模型层共享同一组 Block 管理关系？            |
| `KVCacheTensor`            | Worker 如何组织实际 Cache Tensor 的存储位置？ |
| `SingleTypeKVCacheManager` | 请求如何申请、使用和释放 Block？               |
| `BlockPool`                | Block 如何分配、复用、缓存和回收？              |
| `KVCacheBlock`             | 一个逻辑 Cache Block 如何表示？            |

**总结：**

vLLM 将 KV Cache 管理拆分为三个层次：

1. **Spec 层**：描述 Cache 数据结构和容量。
2. **Manager 层**：管理请求与 Block 的生命周期。
3. **BlockPool 层**：执行 Block 的分配、复用和释放。

这种设计使不同模型结构可以复用统一的 Block 管理框架，同时通过不同 Spec 和 Manager 实现 Attention、MLA、Mamba、Sliding Window 等 Cache 类型的差异化管理。

用纯文本替换所有公式检查转义字符和乱码统一公式中的变量说明
