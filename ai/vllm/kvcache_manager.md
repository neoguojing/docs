
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
