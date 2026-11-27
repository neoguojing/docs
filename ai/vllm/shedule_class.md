# **vLLM Scheduler：Schedule 依赖类、关键变量与核心接口**

基于提供的 `Scheduler` 源码，本文只分析 `schedule()` 的直接依赖、关键内部变量和核心接口，不展开调度算法细节及 KV Cache 内部实现。

## **1. 核心类依赖关系**

`Scheduler` 继承 `SchedulerInterface`，主要依赖以下几组类：

```
Scheduler(SchedulerInterface)
│
├── 1. 请求管理
│   ├── Request
│   ├── RequestStatus
│   ├── RequestQueue
│   ├── SchedulingPolicy
│   └── create_request_queue()
│
├── 2. KV Cache 管理
│   ├── KVCacheManager
│   ├── KVCacheBlocks
│   └── KVCacheBlock
│
├── 3. Encoder / 多模态管理
│   ├── EncoderCacheManager
│   ├── EncoderDecoderCacheManager
│   ├── MultiModalRegistry
│   └── MultiModalBudget
│
├── 4. KV 传输
│   ├── KVConnectorBase_V1
│   ├── KVConnectorFactory
│   ├── ECConnectorBase
│   └── ECConnectorFactory
│
├── 5. 调度结果
│   ├── SchedulerOutput
│   ├── NewRequestData
│   ├── CachedRequestData
│   └── KVConnectorBlockState
│
└── 6. 配置与其他辅助
    ├── VllmConfig
    ├── KVCacheConfig
    ├── StructuredOutputManager
    └── RoutedExpertsManager
```

## **2. 关键依赖类及作用**

| 类                     | 核心作用                           | Scheduler 中的成员              |
| --------------------- | ------------------------------ | --------------------------- |
| `Request`             | 请求实体，保存 Token、状态、计算进度等         | `requests`、`running`        |
| `RequestQueue`        | 管理等待请求队列                       | `waiting`、`skipped_waiting` |
| `SchedulingPolicy`    | 决定请求调度策略                       | `policy`                    |
| `KVCacheManager`      | 管理 KV Block 分配、释放、Prefix Cache | `kv_cache_manager`          |
| `KVCacheBlocks`       | 表示请求关联的 KV Blocks              | `req_to_new_blocks`         |
| `EncoderCacheManager` | 管理多模态 Encoder 缓存               | `encoder_cache_manager`     |
| `KVConnectorBase_V1`  | 管理远程 KV 加载、传输等                 | `connector`                 |
| `ECConnectorBase`     | 管理 Encoder Cache 传输            | `ec_connector`              |
| `SchedulerOutput`     | 封装当前 Step 的调度结果                | `scheduler_output`          |
| `NewRequestData`      | 封装新加入调度批次的请求数据                 | `new_reqs_data`             |
| `CachedRequestData`   | 封装已有请求的更新数据                    | `cached_reqs_data`          |

其中，最值得优先理解的是：

- `Request`：Scheduler 调度的基本对象。
- `RequestQueue`：等待请求的管理结构。
- `KVCacheManager`：决定请求能否获得 KV Cache。
- `SchedulerOutput`：Scheduler 与后续执行阶段之间的数据接口。

## **3. Scheduler 关键内部变量**

### **3.1 请求管理**

| 变量                        | 类型                   | 作用                    |
| ------------------------- | -------------------- | --------------------- |
| `requests`                | `dict[str, Request]` | 所有请求，req_id → Request |
| `waiting`                 | `RequestQueue`       | 等待调度的请求               |
| `skipped_waiting`         | `RequestQueue`       | 本轮因条件限制而跳过的请求         |
| `running`                 | `list[Request]`      | 已进入运行状态的请求            |
| `finished_req_ids`        | `set[str]`           | 已完成请求 ID              |
| `reset_preempted_req_ids` | `set[str]`           | 最近被抢占的请求 ID           |

### **3.2 调度资源约束**

| 变量                         | 作用                           |
| -------------------------- | ---------------------------- |
| `max_num_running_reqs`     | 最大运行请求数                      |
| `max_num_scheduled_tokens` | 单个调度 Step 最大 Token 数         |
| `max_model_len`            | 模型最大上下文长度                    |
| `block_size`               | KV Cache Block 大小            |
| `num_lookahead_tokens`     | Lookahead Token 数            |
| `num_spec_tokens`          | Speculative Decoding Token 数 |

### **3.3 Schedule 局部变量**

这些变量直接参与一次 `schedule()` 调用：

| 变量                             | 类型                         | 作用                    |
| ------------------------------ | -------------------------- | --------------------- |
| `token_budget`                 | int                        | 当前 Step 剩余调度 Token 预算 |
| `input_budget`                 | int                        | 当前 Step 输入 Token 预算   |
| `scheduled_new_reqs`           | `list[Request]`            | 新调度请求                 |
| `scheduled_resumed_reqs`       | `list[Request]`            | 抢占后恢复的请求              |
| `scheduled_running_reqs`       | `list[Request]`            | 本轮继续调度的运行请求           |
| `preempted_reqs`               | `list[Request]`            | 本轮被抢占的请求              |
| `req_to_new_blocks`            | `dict[str, KVCacheBlocks]` | 请求对应的 KV Blocks       |
| `num_scheduled_tokens`         | `dict[str, int]`           | 每个请求本轮调度的 Token 数     |
| `scheduled_spec_decode_tokens` | `dict[str, list[int]]`     | 本轮调度的 Draft Tokens    |
| `scheduled_encoder_inputs`     | `dict[str, list[int]]`     | 本轮调度的 Encoder 输入      |
| `prefill_scheduled`            | bool                       | 当前 Batch 是否包含 Prefill |
| `has_sync_kv_loads`            | bool                       | 是否存在同步 KV 加载          |

## **4. Schedule 直接调用的核心接口**

这里只列出源码中与调度过程直接相关的方法。

| 所属类                   | 核心接口                             | 作用                     |
| --------------------- | -------------------------------- | ---------------------- |
| `RequestQueue`        | `peek_request()`                 | 查看队首请求                 |
| `RequestQueue`        | `pop_request()`                  | 取出请求                   |
| `RequestQueue`        | `prepend_request()`              | 将请求放回队首                |
| `RequestQueue`        | `prepend_requests()`             | 批量放回请求                 |
| `KVCacheManager`      | `new_step_starts()`              | 开始新的 KV Cache 调度 Step  |
| `KVCacheManager`      | `get_computed_blocks()`          | 查找本地 Prefix Cache 命中   |
| `KVCacheManager`      | `allocate_slots()`               | 为请求分配 KV Cache Slots   |
| `KVCacheManager`      | `get_blocks()`                   | 获取请求对应的 KV Blocks      |
| `KVCacheManager`      | `get_num_common_prefix_blocks()` | 获取公共前缀 Block 数         |
| `KVCacheManager`      | `take_kv_cache_block_copies()`   | 获取待执行的 KV Block 拷贝     |
| `KVConnectorBase_V1`  | `get_num_new_matched_tokens()`   | 查询远端 KV 命中 Token 数     |
| `KVConnectorBase_V1`  | `update_state_after_alloc()`     | 更新 KV Connector 分配状态   |
| `KVConnectorBase_V1`  | `build_connector_meta()`         | 构建 KV 传输元数据            |
| `EncoderCacheManager` | `can_allocate()`                 | 判断 Encoder Cache 是否可分配 |
| `EncoderCacheManager` | `allocate()`                     | 分配 Encoder Cache       |
| `EncoderCacheManager` | `get_manager_metadata()`         | 获取 Encoder Cache 元数据   |

### **Scheduler 自身的关键辅助接口**

| 方法                               | 作用                         |
| -------------------------------- | -------------------------- |
| `_get_local_prefix_cache_hit()`  | 获取本地 Prefix Cache 命中       |
| `_try_schedule_encoder_inputs()` | 决定本轮调度哪些 Encoder 输入        |
| `_reserve_prefill_lookahead()`   | 预留 Prefill Lookahead Token |
| `_mamba_block_aligned_split()`   | 按 Mamba Block 边界切分 Prefill |
| `_preempt_request()`             | 抢占请求并放回等待队列                |
| `_make_cached_request_data()`    | 构建已有请求的调度数据                |
| `_build_kv_connector_meta()`     | 构建 KV Connector 元数据        |
| `_update_after_schedule()`       | 更新调度后的请求状态                 |

## **5. 核心数据关系**

理解以下数据关系，就能掌握 `schedule()` 的主要依赖结构：

```
                    Scheduler
                        │
          ┌─────────────┼──────────────┐
          │             │              │
      Request        RequestQueue   KVCacheManager
          │             │              │
          │         waiting/running    │
          │                            │
          └────────────┬───────────────┘
                       │
                 schedule()
                       │
           ┌───────────┼────────────┐
           │           │            │
       请求调度     KV Block分配   资源预算控制
           │           │            │
           └───────────┼────────────┘
                       │
                SchedulerOutput
                       │
           ┌───────────┼────────────┐
           │           │            │
      NewRequestData CachedRequestData
           │           │            │
           └───────────┴────────────┘
                       │
                 后续执行阶段
```

## **6. 建议优先阅读顺序**

如果只研究 Scheduler，不建议一开始就阅读所有辅助模块。

1. `Request`：理解请求的数据结构和状态。
2. `RequestQueue`：理解请求如何进入、离开等待队列。
3. `KVCacheManager`：理解 `allocate_slots()` 和 `get_computed_blocks()`。
4. `Scheduler.schedule()`：理解资源预算、请求调度和抢占。
5. `SchedulerOutput`：理解调度结果如何交给后续执行阶段。

**核心认识：**

Scheduler 本身不执行模型推理。它主要协调三个对象：

- `Request`：决定调度谁。
- `RequestQueue`：决定从哪里取请求。
- `KVCacheManager`：决定请求能否获得 KV Cache。

最后通过 `SchedulerOutput` 将本轮调度决策转换成后续执行阶段需要的数据。

说明：以上仅整理所提供源码中可确认的类、成员和接口，没有展开未提供的依赖类内部实现。&#x20;

粘贴的文本 (1)(20261127-134921).txt
