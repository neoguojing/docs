
# **vLLM KV Cache 核心类架构**

## **1. 整体类关系**

KV Cache 采用分层管理架构，核心分为管理入口、协调层、类型管理层和物理资源层。

```
                 Scheduler
                     |
                     v
               KVCacheManager
                (统一入口)
                     |
                     v
             KVCacheCoordinator
                (Group协调)
                     |
          +----------+----------+
          |          |          |
          v          v          v
       Manager 0  Manager 1  Manager N
          |          |          |
          +----------+----------+
                     |
             SingleTypeKVCacheManager
                (类型管理抽象)
                     |
                     v
                  BlockPool
                (物理资源池)
                     |
                     v
                KVCacheBlock
                (Block元数据)

KVCacheBlocks：封装各 Group 的 Block 分配结果，返回 Scheduler。
```

## **2. 核心类及作用**

| 类                          | 核心职责                                    | 关键属性                                                                                |
| -------------------------- | --------------------------------------- | ----------------------------------------------------------------------------------- |
| `KVCacheManager`           | KV Cache 总管理入口，服务 Scheduler             | `coordinator`、`block_pool`、`num_kv_cache_groups`、`watermark_blocks`                 |
| `KVCacheCoordinator`       | 协调多个 KV Cache Group，创建对应的类型管理器          | `single_type_managers`、`block_pool`、`group_block_sizes`、`eagle_group_ids`           |
| `SingleTypeKVCacheManager` | 管理一种 KV Cache 类型，负责 Request 与 Block 的映射 | `block_size`、`kv_cache_spec`、`req_to_blocks`、`num_cached_block`、`kv_cache_group_id` |
| `BlockPool`                | 统一管理物理 Block，负责分配、释放、缓存查找和淘汰            | `blocks`、`free_block_queue`、`cached_block_hash_to_block`、`null_block`               |
| `KVCacheBlock`             | 单个物理 Block 的元数据                         | `block_id`、`ref_cnt`、`_block_hash`、`prev_free_block`、`next_free_block`              |
| `KVCacheBlocks`            | 封装多个 Group 的 Block 分配结果，作为 Scheduler 接口 | `blocks`                                                                            |

## **3. 关键类参数**

**① KVCacheManager**

| 参数                     | 含义                        |
| ---------------------- | ------------------------- |
| `kv_cache_config`      | KV Cache 整体配置             |
| `max_model_len`        | 模型最大长度                    |
| `scheduler_block_size` | Scheduler 调度粒度            |
| `hash_block_size`      | Hash 计算粒度                 |
| `enable_caching`       | 是否启用 Prefix Cache         |
| `watermark_blocks`     | Admission 时预留的空闲 Block 数量 |

**② KVCacheCoordinator**

| 参数                     | 含义                            |
| ---------------------- | ----------------------------- |
| `block_pool`           | 共享物理 Block 池                  |
| `single_type_managers` | 每个 KV Cache Group 对应的 Manager |
| `scheduler_block_size` | 统一调度粒度                        |
| `group_block_sizes`    | 各 Group 的 Block Size          |
| `eagle_group_ids`      | EAGLE 对应的 Group ID            |
| `retention_interval`   | Prefix Cache 保留间隔             |

**③ SingleTypeKVCacheManager**

| 参数                  | 含义                           |
| ------------------- | ---------------------------- |
| `kv_cache_spec`     | 当前 Group 的 KV Cache 类型规格     |
| `block_size`        | 当前 Manager 实际 Block Size     |
| `kv_cache_group_id` | 所属 Group ID                  |
| `req_to_blocks`     | Request ID → KVCacheBlock 列表 |
| `num_cached_block`  | 请求已命中的缓存 Block 数             |
| `_null_block`       | 空 Block 占位符                  |

**④ BlockPool / KVCacheBlock**

| 类            | 参数                           | 含义                |
| ------------ | ---------------------------- | ----------------- |
| BlockPool    | `blocks`                     | 所有物理 Block        |
| BlockPool    | `free_block_queue`           | 空闲 Block 双向链表     |
| BlockPool    | `cached_block_hash_to_block` | Hash → Block 索引   |
| BlockPool    | `null_block`                 | 特殊空 Block         |
| KVCacheBlock | `block_id`                   | 物理 Block ID       |
| KVCacheBlock | `ref_cnt`                    | 引用计数              |
| KVCacheBlock | `_block_hash`                | Prefix Cache Hash |
| KVCacheBlock | `pool`                       | 所属 BlockPool      |

## **4. 关键数据结构**

**Request 与 Block 的映射（Manager 内部）**

```
req_to_blocks

Request A
   |
   +---- Block 10
   +---- Block 11
   +---- Block 12

Request B
   |
   +---- Block 20
   +---- Block 21
```

**BlockPool 物理资源管理**

```
BlockPool
   |
   +---- blocks
   |       +---- Block 0
   |       +---- Block 1
   |       +---- Block 2
   |
   +---- free_block_queue
   |
   +---- cached_block_hash_to_block
```

**KVCacheBlocks 对外数据结构**

```
blocks[group_id][block_index]
```

外层是 KV Cache Group，内层是该 Group 的 Block 列表。

不同 Group 可以拥有不同的 Block Size，因此不采用 Token Block 作为外层维度。

## **5. 架构设计总结**

| 设计层次                     | 解决的问题                 |
| ------------------------ | --------------------- |
| KVCacheManager           | 隔离 Scheduler 与底层实现    |
| KVCacheCoordinator       | 协调多个 KV Cache Group   |
| SingleTypeKVCacheManager | 解耦不同 KV Cache 类型的管理策略 |
| BlockPool                | 统一管理物理 Block 资源       |
| KVCacheBlock             | 描述单个 Block 的元数据       |
| KVCacheBlocks            | 封装分配结果，隔离内部数据结构       |

**核心理解：**

- `KVCacheManager`：总入口。
- `KVCacheCoordinator`：管理多个 Group。
- `SingleTypeKVCacheManager`：管理不同类型的 KV Cache。
- `BlockPool`：管理物理 Block 的生命周期。
- `KVCacheBlock`：物理 Block 元数据。
- `KVCacheBlocks`：Scheduler 与 Manager 之间的数据接口。

本质上是将 **KV Cache 的逻辑管理（Request → Block）与物理资源管理（BlockPool → KVCacheBlock）分离**，同时通过 Coordinator 支持多种 KV Cache 类型共存。

可以。这段 `BlockTables` 代码基本补齐了我们之前缺少的关键环节：**它负责把 Scheduler 分配的逻辑 Block 信息，转换成 GPU 模型执行时可以使用的物理 Block 映射和 Token 写入位置。**

不过要注意：它不是直接操作 KV Cache Tensor 的类，而是负责构建模型访问 KV Cache 所需要的索引数据。

下面结合代码梳理。

# **1. 先明确三个不同的 Block 概念**

| 概念       | 对应对象                       | 作用                                |
| -------- | -------------------------- | --------------------------------- |
| 管理对象     | `KVCacheBlock`             | CPU 侧管理 Block 生命周期                |
| 映射表      | `BlockTables.block_tables` | 记录 Request 逻辑 Block → 物理 Block ID |
| GPU 数据存储 | KV Cache Tensor            | 真正保存 K/V 数据                       |

它们之间的关系：

```
Scheduler
    |
    | 分配 KVCacheBlock
    v
KVCacheBlock(block_id=7)
KVCacheBlock(block_id=12)
    |
    | 提取 block_id
    v
BlockTables.append_block_ids()
    |
    v
block_tables
    |
    | Gather
    v
input_block_tables
    |
    | 计算 Token 写入位置
    v
slot_mappings
    |
    v
Attention / KV Cache Kernel
    |
    v
GPU KV Cache Tensor
```

这里最重要的是：`block_id` 是贯穿管理层和执行层的桥梁。

# **2. BlockTables 内部的数据结构**

## **2.1 block_tables：维护逻辑 Block 到物理 Block 的映射**

初始化代码：

```
self.block_tables: list[StagedWriteTensor] = []for i in range(self.num_kv_cache_groups):    max_num_blocks = (        max_num_blocks_per_group[i]        * self.blocks_per_kv_block[i]    )    block_table = StagedWriteTensor(        (self.max_num_reqs, max_num_blocks),        dtype=torch.int32,        device=device    )    self.block_tables.append(block_table)
```

其数据结构是：

```
block_tables[group_id]

       Request 0    Request 1    Request 2
       ----------------------------------
       [7, 12, 15]  [3, 8]       [9, 10]
```

含义：

```
Request 0:
    逻辑 Block 0 -> 物理 Block 7
    逻辑 Block 1 -> 物理 Block 12
    逻辑 Block 2 -> 物理 Block 15
```

这里要注意：

`block_tables` 是按 KV Cache Group 分组的，每个 Group 有自己的 Block Table。

## **2.2 append_block_ids：把 Scheduler 分配结果写入 Block Table**

这是连接 `KVCacheBlock` 和 `BlockTables` 的关键函数。

```
def append_block_ids(    self,    req_index: int,    new_block_ids: tuple[list[int], ...],    overwrite: bool,):    for i in range(self.num_kv_cache_groups):        start = self.num_blocks.np[i, req_index] if not overwrite else 0        block_ids = new_block_ids[i]        bpk = self.blocks_per_kv_block[i]        if bpk > 1:            block_ids = [                b * bpk + k                for b in block_ids                for k in range(bpk)            ]        end = start + len(block_ids)        self.block_tables[i].stage_write(            req_index, start, block_ids        )        self.num_blocks.np[i, req_index] = end
```

这里做了三件事情：

1. 确定当前 Request 的 Block Table 写入位置。
2. 将新分配的物理 Block ID 追加进去。
3. 更新当前 Request 的 Block 数量。

举例：

```
new_block_ids = (    [7, 12],)
```

执行后：

```
Request 0

block_tables[0][0]

[7, 12]
```

### **blocks_per_kv_block 是什么？**

```
self.blocks_per_kv_block = [    bs // kbs    for bs, kbs in zip(block_sizes, kernel_block_sizes)]
```

它表示：

`一个 Scheduler KV Block 对应多少个 Kernel Block`。

例如：

```
scheduler block_size = 32
kernel block_size    = 16

blocks_per_kv_block = 2
```

那么一个 Scheduler Block ID 为 7 时，会展开成：

```
7 * 2 + 0 = 14
7 * 2 + 1 = 15
```

即：

```
Scheduler Block 7
       |
       +---- Kernel Block 14
       +---- Kernel Block 15
```

这是为了兼容 Scheduler 分配粒度和 Kernel 实际访问粒度不同的情况。

# **3. apply_staged_writes：同步到 GPU**

注意，`append_block_ids()` 并没有立即修改 GPU Tensor。

它调用的是：

```
self.block_tables[i].stage_write(...)
```

真正提交在：

```
def apply_staged_writes(self):    if self.num_kv_cache_groups == 1:        self.block_tables[0].apply_write()    elif self.num_kv_cache_groups > 1:        self.fused_writer.apply(            self.block_tables,            self.block_table_ptrs,            self.block_table_strides        )    self.num_blocks.copy_to_uva()
```

这里体现了一个性能优化：

**先在 CPU 侧暂存多个 Block Table 更新，再批量提交到 GPU。**

`StagedWriteTensor` 的数据结构：

```
CPU
 |
 +-- _staged_write_indices
 +-- _staged_write_starts
 +-- _staged_write_contents
 +-- _staged_write_cu_lens
          |
          v
     apply_write()
          |
          v
   _apply_write_kernel
          |
          v
     GPU block_tables
```

因此，`StagedWriteTensor` 是 Block Table 的批量更新工具。

# **4. gather_block_tables：为模型 Forward 准备 Block Table**

这里有两个容易混淆的对象：

- `block_tables`：维护所有 Request 的 Block 映射。
- `input_block_tables`：本次模型 Forward 实际使用的 Block Table。

初始化：

```
self.input_block_tables = [    torch.zeros_like(b.gpu)    for b in self.block_tables]
```

Gather：

```
_gather_block_tables_kernel[(self.num_kv_cache_groups, num_reqs_padded)](    idx_mapping,    self.block_table_ptrs,    out_ptrs,    self.block_table_strides,    self.num_blocks.gpu,    ...)
```

为什么需要 Gather？

因为 Scheduler 管理的 Request 顺序和本次模型执行的 Request 顺序可能不同。

例如：

```
全局 Block Table：

Request A -> [7, 12]
Request B -> [3, 8]
Request C -> [9, 10]
```

本次 Batch：

```
idx_mapping = [2, 0]

实际 Forward 顺序：

Batch 0 -> Request C -> [9, 10]
Batch 1 -> Request A -> [7, 12]
```

Gather 后：

```
input_block_tables

Batch 0 -> [9, 10]
Batch 1 -> [7, 12]
```

这样模型就可以按照当前 Batch 的 Request 顺序访问 Block。

# **5. compute_slot_mappings：计算 Token 的物理写入位置**

这是整个链路中最重要的函数之一。

```
def compute_slot_mappings(    self,    idx_mapping,    query_start_loc,    positions,    num_tokens_padded,    out=None,):    ...    _compute_slot_mappings_kernel[(num_groups, num_reqs + 1)](        slot_mappings.shape[1],        idx_mapping,        query_start_loc,        positions,        self.block_table_ptrs,        self.block_table_strides,        self.block_sizes_tensor,        self.kernel_block_sizes_tensor,        self.slot_mapping_enabled,        slot_mappings,        ...    )    return slot_mappings[:, :num_tokens_padded]
```

它的输入包括：

| 参数                          | 含义                            |
| --------------------------- | ----------------------------- |
| `idx_mapping`               | Batch Request 到全局 Request 的映射 |
| `query_start_loc`           | 每个 Request 的 Query Token 起始位置 |
| `positions`                 | Token 在序列中的位置                 |
| `block_table_ptrs`          | Block Table 的 GPU 地址          |
| `block_sizes_tensor`        | 各 Group 的 Block Size          |
| `kernel_block_sizes_tensor` | Kernel Block Size             |
| `slot_mappings`             | 输出的物理 Slot 映射                 |

举例：

假设：

```
Block Size = 16

Request A:
    Block Table = [7, 12]

当前 Token Position = 18
```

计算：

```
逻辑 Block Index = 18 // 16 = 1

物理 Block ID = block_table[1] = 12

Block 内偏移 = 18 % 16 = 2

物理 Slot = 12 * 16 + 2 = 194
```

因此：

```
slot_mapping = 194
```

这里的计算是概念示意，实际 `_compute_slot_mappings_kernel` 还要考虑不同 Group、Kernel Block Size 和 Context Parallel 等因素。

# **6. 最终如何写入 KV Cache Tensor？**

到这里，我们已经建立了完整的映射：

```
KVCacheBlock
    |
    | block_id = 12
    v
BlockTables
    |
    | Request A -> [7, 12]
    v
compute_slot_mappings()
    |
    | Token 18 -> Slot 194
    v
slot_mappings
    |
    v
Attention Backend
    |
    | 根据 slot_mapping 写入 K/V
    v
GPU KV Cache Tensor
```

Attention Backend 拿到当前 Token 的 K/V 和 Slot Mapping 后，通过对应的 KV Cache 写入算子，将数据写入指定位置。

例如逻辑上可以理解为：

```
# 仅为概念示意，不是 vLLM 实际源码kv_cache[194] = current_key_value
```

实际情况是 K 和 V 分别写入对应的 Cache 存储位置，具体取决于 KV Cache Layout 和 Backend。

# **7. 最终总结：这几个类如何关联？**

| 类 / 数据结构                   | 所处阶段    | 核心作用                     |
| -------------------------- | ------- | ------------------------ |
| `KVCacheBlock`             | 资源管理    | 管理物理 Block 元数据           |
| `BlockPool`                | 资源管理    | 分配、释放和复用 Block           |
| `SingleTypeKVCacheManager` | 请求管理    | 维护 Request → Block       |
| `BlockTables`              | 执行准备    | 构建 Request → 物理 Block ID |
| `StagedWriteTensor`        | 执行准备    | 批量更新 Block Table         |
| `input_block_tables`       | Forward | 提供当前 Batch 的 Block 映射    |
| `slot_mappings`            | Forward | 提供 Token 级物理写入位置         |
| Attention Backend          | GPU 执行  | 实际读写 KV Cache Tensor     |

**整个过程可以浓缩成这张图：**

```
【初始化】
GPUModelRunner
      |
      v
GPU KV Cache Tensor
      |
      +--------------------------+
                                 |
【请求分配】                     |
Scheduler                        |
      |                          |
      v                          |
KVCacheManager                   |
      |                          |
      v                          |
SingleTypeKVCacheManager         |
      |                          |
      v                          |
BlockPool                        |
      |                          |
      v                          |
KVCacheBlock                     |
      |                          |
      | block_id                 |
      v                          |
BlockTables                      |
      |                          |
      v                          |
input_block_tables               |
      |                          |
      v                          |
slot_mappings                    |
      |                          |
      +------------+-------------+
                   |
                   v
             Attention Backend
                   |
                   v
             写入 K / V Tensor
```

**我的建议：下一步直接跟踪&#x20;**`_compute_slot_mappings_kernel`**&#x20;和 Attention Backend 中的&#x20;**`reshape_and_cache`**（或对应写入算子）。**

这两个位置分别回答最后两个问题：

1. Block Table 如何精确转换成 Slot ID？
2. Slot ID 如何最终转换成 GPU Tensor 的实际内存写入地址？

到这里，你就真正打通了 vLLM 从 KV Cache 初始化、Block 分配到 GPU KV 数据写入的整个链路。
