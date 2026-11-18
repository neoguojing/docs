# **vLLM Scheduler 源码分析：架构、核心类与调度流程**

> 基于 vLLM V1 `Scheduler` 实现源码整理。重点分析 Scheduler 的职责、核心数据结构、类之间的关系、调度算法、KV Cache 管理、请求生命周期及关键执行流程。

## **1. Scheduler 的定位**

`Scheduler` 是 vLLM V1 Engine Core 中的核心调度组件，负责决定每个模型执行 Step 应该处理哪些请求、每个请求处理多少 Token，以及如何协调 KV Cache 等资源。

它不仅是一个请求队列管理器，还承担以下职责：

1. **请求调度**：决定 Running 和 Waiting 请求的执行顺序。
2. **Token 调度**：为每个请求分配当前 Step 的 Token 预算。
3. **KV Cache 管理**：申请、释放 KV Block，处理 Prefix Cache。
4. **请求抢占**：KV Cache 不足时抢占低优先级请求。
5. **多模态调度**：管理 Encoder 输入预算和 Encoder Cache。
6. **分布式 KV 管理**：协调远程 KV 加载、保存和传输。
7. **高级调度优化**：支持 Chunked Prefill、Speculative Decoding、DP Prefill Balancing 等。
8. **状态管理**：根据模型执行结果更新请求状态，判断请求是否完成。

**核心定位：Scheduler 负责制定执行计划，不负责执行模型计算。**

### **1.1 Scheduler 在 Engine Core 中的位置**

```
                   Engine Core
                       |
                       |
              +------------------+
              |    Scheduler     |
              |                  |
              | 请求调度          |
              | Token预算管理     |
              | KV Cache管理      |
              | 请求抢占          |
              +------------------+
                       |
                       | SchedulerOutput
                       v
              +------------------+
              |   Model Runner    |
              |                  |
              | Input Batch      |
              | Tensor准备        |
              | Model Forward     |
              | Sampling         |
              +------------------+
                       |
                       | ModelRunnerOutput
                       v
              +------------------+
              |    Scheduler     |
              |                  |
              | 更新请求状态      |
              | 更新Token进度     |
              | 判断请求完成      |
              | 释放资源          |
              +------------------+
                       |
                       v
                EngineCoreOutputs
```

Scheduler 和 Model Runner 形成一个迭代执行闭环：

- Scheduler：决定本轮执行什么。
- Model Runner：执行本轮模型计算。
- Scheduler：消费执行结果，更新请求状态。

## **2. 核心架构与类关系**

### **2.1 核心类关系图**

```
                       SchedulerInterface
                              ^
                              |
                         implements
                              |
                         Scheduler
                              |
          +-------------------+-------------------+
          |                   |                   |
          v                   v                   v
       Request           RequestQueue         KVCacheManager
          |                   |                   |
          |                   |                   |
          |              Waiting Queue            |
          |              Skipped Queue            |
          |                                       |
          |                                  KVCacheBlocks
          |                                       |
          |                                  KVCacheBlock
          |
          +-------------------+
                              |
                              v
                     SchedulerOutput
                              |
                              v
                         Model Runner
                              |
                              v
                      ModelRunnerOutput
                              |
                              v
                     EngineCoreOutputs


       Scheduler
          |
          +--------------------------+
          |                          |
          v                          v
 EncoderCacheManager            KVConnector
          |                          |
          v                          v
 Encoder Cache                Remote KV Transfer

       Scheduler
          |
          +--------------------------+
          |
          v
    StructuredOutputManager
          |
          v
     Grammar / Mask
```

### **2.2 主要类职责**

| 类                         | 职责               | 说明                               |
| ------------------------- | ---------------- | -------------------------------- |
| `SchedulerInterface`      | 抽象接口             | 定义 Scheduler 对外操作                |
| `Scheduler`               | 调度实现             | 负责请求、Token、资源的统一调度               |
| `Request`                 | 请求状态             | 保存 Prompt、Output、Token 进度及状态     |
| `RequestQueue`            | 请求队列             | 管理 Waiting 请求及调度顺序               |
| `KVCacheManager`          | KV 管理            | 管理 KV Block 分配、释放、Prefix Cache   |
| `EncoderCacheManager`     | Encoder Cache 管理 | 管理多模态 Encoder 缓存                 |
| `SchedulerOutput`         | 调度结果             | Scheduler 传递给 Model Runner 的执行计划 |
| `ModelRunnerOutput`       | 模型执行结果           | Model Runner 返回的执行结果             |
| `KVConnectorBase_V1`      | KV 传输接口          | 协调远程 KV 加载、保存和传输                 |
| `ECConnectorBase`         | Encoder Cache 传输 | 协调远程 Encoder Cache 传输            |
| `StructuredOutputManager` | 结构化输出管理          | 管理 Grammar 等约束输出能力               |

需要注意：

- `Scheduler` 继承 `SchedulerInterface`。
- `KVCacheManager`、`RequestQueue`、`EncoderCacheManager` 是 Scheduler 的协作组件，不是其子类。
- `SchedulerOutput` 和 `ModelRunnerOutput` 是两个方向不同的数据结构。

## **3. Scheduler 初始化**

入口：

```
class Scheduler(SchedulerInterface):    def __init__(        self,        vllm_config: VllmConfig,        kv_cache_config: KVCacheConfig,        structured_output_manager: StructuredOutputManager,        block_size: int,        hash_block_size: int | None = None,        mm_registry: MultiModalRegistry = MULTIMODAL_REGISTRY,        include_finished_set: bool = False,        log_stats: bool = False,    ) -> None:
```

初始化主要完成五件事情：

1. 保存模型、调度、缓存、并行等配置。
2. 初始化请求队列和请求状态容器。
3. 初始化 KV Cache Manager。
4. 初始化 Encoder Cache Manager 和 Connector。
5. 初始化 Speculative Decoding、Mamba、监控等辅助能力。

### **3.1 初始化架构**

```
                  Scheduler.__init__()
                           |
          +----------------+----------------+
          |                |                |
          v                v                v
       配置初始化        请求管理初始化      资源管理初始化
          |                |                |
          |                |                |
      VllmConfig       requests           KVCacheManager
      SchedulerConfig  waiting            EncoderCacheManager
      CacheConfig      skipped_waiting    KVConnector
      ParallelConfig   running            ECConnector
          |                |                |
          +----------------+----------------+
                           |
                           v
                     高级能力初始化
                           |
          +----------------+----------------+
          |                |                |
          v                v                v
       Spec Decode       Mamba           Metrics
       EAGLE / MTP       Cache           KV Events
```

### **3.2 重要配置**

| 配置                     | 作用                         |
| ---------------------- | -------------------------- |
| `scheduler_config`     | 调度策略、最大请求数、Token 预算        |
| `cache_config`         | Prefix Cache、KV Cache 相关配置 |
| `parallel_config`      | DP、PP、DCP 等并行配置            |
| `speculative_config`   | Speculative Decoding 配置    |
| `kv_transfer_config`   | KV 传输配置                    |
| `ec_transfer_config`   | Encoder Cache 传输配置         |
| `observability_config` | Metrics、性能监控配置             |

### **3.3 核心调度约束**

```
self.max_num_running_reqs = self.scheduler_config.max_num_seqsself.max_num_scheduled_tokens = (    self.scheduler_config.max_num_scheduled_tokens    if self.scheduler_config.max_num_scheduled_tokens is not None    else self.scheduler_config.max_num_batched_tokens)self.max_model_len = vllm_config.model_config.max_model_len
```

三个重要限制：

| 变量                         | 含义                     |
| -------------------------- | ---------------------- |
| `max_num_running_reqs`     | 同时运行的最大请求数量            |
| `max_num_scheduled_tokens` | 每个调度 Step 的 Token 预算上限 |
| `max_model_len`            | 模型支持的最大序列长度            |

其中，`max_num_scheduled_tokens` 是 Scheduler 每轮调度的核心约束之一。

## **4. Scheduler 核心数据结构**

Scheduler 内部维护多个请求容器。

### **4.1 核心成员变量**

```
self.requests: dict[str, Request] = {}self.waiting = create_request_queue(self.policy)self.skipped_waiting = create_request_queue(self.policy)self.running: list[Request] = []self.finished_req_ids: set[str] = set()self.reset_preempted_req_ids: set[str] = set()
```

### **4.2 请求容器关系**

```
                       requests
                    dict[req_id, Request]
                            |
             +--------------+--------------+
             |              |              |
             v              v              v
          waiting      skipped_waiting    running
             |              |              |
             |              |              |
          等待调度       暂时不能调度       已进入运行队列
             |              |              |
             +--------------+--------------+
                            |
                            v
                       schedule()
                            |
                            v
                       Model Runner
```

### **4.3 各容器的作用**

| 容器                        | 类型             | 作用                       |
| ------------------------- | -------------- | ------------------------ |
| `requests`                | `dict`         | Scheduler 管理的 Request 索引 |
| `waiting`                 | `RequestQueue` | 等待调度的请求                  |
| `skipped_waiting`         | `RequestQueue` | 因异步依赖或调度约束暂时跳过的请求        |
| `running`                 | `list`         | 已进入运行队列的请求               |
| `finished_req_ids`        | `set`          | 通知执行侧清理已完成请求的状态          |
| `reset_preempted_req_ids` | `set`          | 通知执行侧处理被抢占请求的状态重置        |

注意：

`requests` 不等价于 `waiting + skipped_waiting + running`。

请求完成后，可能因为异步 KV 传输等原因仍然需要保留部分状态，等待后续清理。

### **4.4 为什么需要 skipped_waiting？**

一个请求进入 Waiting 队列，不代表它一定能立即执行。

例如：

- 远程 KV 尚未加载完成。
- Encoder Cache 尚不可用。
- 异步 KV 传输尚未完成。
- 某些调度约束暂时不满足。

如果直接阻塞整个调度过程，会降低其他请求的执行效率。

因此 Scheduler 可以暂时跳过这些请求，将其放入 `skipped_waiting`，继续尝试调度其他请求。

## **5. RequestQueue 与调度策略**

Scheduler 支持不同的请求调度策略。

源码中的核心概念：

```
self.policy = SchedulingPolicy(self.scheduler_config.policy)self.waiting = create_request_queue(self.policy)self.skipped_waiting = create_request_queue(self.policy)
```

### **5.1 调度策略**

| 策略         | 含义                      | 特点        |
| ---------- | ----------------------- | --------- |
| `FCFS`     | First Come First Served | 先到先服务     |
| `PRIORITY` | 优先级调度                   | 根据请求优先级排序 |

### **5.2 Waiting 队列选择**

核心方法：

```
_select_waiting_queue_for_scheduling()
```

它的作用是从 `waiting` 和 `skipped_waiting` 中选择下一条需要检查的请求队列。

```
                waiting
                   |
                   |
             +-----+-----+
             |           |
             v           v
         FCFS策略     PRIORITY策略
             |           |
             v           v
       按队列顺序    比较队首请求优先级
             |           |
             +-----+-----+
                   |
                   v
          选择待检查的 Request
```

`skipped_waiting` 并不是错误队列，而是调度过程中暂时无法满足条件的请求集合。

## **6. schedule()：核心调度算法**

`Scheduler.schedule()` 是整个 Scheduler 最重要的方法。

```
def schedule(self, throttle_prefills: bool = False) -> SchedulerOutput:
```

它的核心任务是：

> 在当前 Step 的资源约束下，为 Running 和 Waiting 请求分配 Token、KV Block 和其他必要资源，最终生成 SchedulerOutput。

### **6.1 一个重要设计：没有独立的 Prefill / Decode 调度阶段**

源码注释明确指出：

```
# There's no "decoding phase" nor "prefill phase" in the scheduler.
```

Scheduler 并不是简单地先执行 Prefill、再执行 Decode。

它统一使用：

- `num_computed_tokens`
- `num_tokens_with_spec`

来判断一个请求还需要处理多少 Token。

核心计算：

```
num_new_tokens = (    request.num_tokens_with_spec    + request.num_output_placeholders    - request.num_computed_tokens)
```

可以理解为：

```
本轮待处理 Token
    =
请求当前需要处理的 Token 总数
    -
已经计算完成的 Token 数
```

再受到 Token Budget、Input Budget、模型最大长度等约束。

这种统一的 Token 调度方式，可以覆盖：

- Prefill
- Decode
- Chunked Prefill
- Prefix Caching
- Speculative Decoding

### **6.2 schedule() 整体流程**

```
                    schedule()
                        |
                        v
              current_step += 1
                        |
                        v
              初始化调度预算和结果容器
                        |
                        v
              KVCacheManager.new_step_starts()
                        |
                        v
              +----------------------+
              | 调度 Running Requests |
              +----------------------+
                        |
                        v
              计算每个请求的 Token 数
                        |
                        v
              检查 Encoder / Mamba 等约束
                        |
                        v
              KVCacheManager.allocate_slots()
                        |
                 +------+------+
                 |             |
                 v             v
             分配成功       分配失败
                 |             |
                 |             v
                 |          请求抢占
                 |             |
                 |             v
                 |         重新分配
                 |             |
                 +------+------+
                        |
                        v
              +----------------------+
              | 调度 Waiting Requests|
              +----------------------+
                        |
                        v
              检查请求状态和调度约束
                        |
                        v
              Prefix Cache / Remote KV
                        |
                        v
              KV Block 分配
                        |
                        v
              构建 NewRequestData /
              CachedRequestData
                        |
                        v
              构建 SchedulerOutput
                        |
                        v
              构建 KV / EC Connector Metadata
                        |
                        v
              _update_after_schedule()
                        |
                        v
                return SchedulerOutput
```

### **6.3 第一步：初始化调度预算**

源码：

```
token_budget = self.max_num_scheduled_tokensinput_budget = self.scheduler_config.max_num_batched_tokens
```

两种预算：

| 预算             | 含义                      |
| -------------- | ----------------------- |
| `token_budget` | 当前调度 Step 可以分配的 Token 数 |
| `input_budget` | 当前 Batch 输入 Token 的预算   |

此外还涉及：

- `draft_slots`：Speculative Decoding 预留空间。
- `encoder_compute_budget`：多模态 Encoder 计算预算。
- `max_num_running_reqs`：最大运行请求数。

因此，一个请求即使有待计算 Token，也不代表一定能在当前 Step 执行。

### **6.4 第二步：优先调度 Running Requests**

源码注释：

```
# First, schedule the RUNNING requests.
```

核心循环：

```
req_index = 0while req_index < len(self.running) and token_budget > 0:    request = self.running[req_index]
```

对于每个 Running 请求：

1. 检查是否满足当前 Step 执行条件。
2. 检查 Decode 是否满足调度间隔约束。
3. 检查 Encoder Cache 是否可用。
4. 计算需要处理的 Token 数。
5. 根据预算限制 Token 数。
6. 检查 Mamba 等特殊约束。
7. 调用 KVCacheManager 分配 KV Block。
8. 记录调度结果。

Token 数计算：

```
num_new_tokens = (    request.num_tokens_with_spec    + request.num_output_placeholders    - request.num_computed_tokens)
```

随后限制：

```
num_new_tokens = min(    num_new_tokens,    token_budget,    input_budget - draft_slots,)
```

同时不能超过模型最大长度：

```
num_new_tokens = min(    num_new_tokens,    self.max_model_len    - request.num_computed_tokens    - self.num_sampled_tokens_per_step,)
```

### **6.5 第三步：KV Block 分配**

核心方法：

```
new_blocks = self.kv_cache_manager.allocate_slots(    request,    num_new_tokens,    num_lookahead_tokens=self.num_lookahead_tokens,)
```

这里需要区分两个角色：

| 组件             | 职责                        |
| -------------- | ------------------------- |
| Scheduler      | 决定请求需要多少 KV 空间，何时分配、何时抢占  |
| KVCacheManager | 实际管理 KV Block 的分配、释放及缓存状态 |

分配成功：

```
Request
   |
   v
allocate_slots()
   |
   v
获得 KVCacheBlocks
   |
   v
记录到 req_to_new_blocks
   |
   v
加入本轮调度结果
```

分配失败：

```
allocate_slots()
      |
      v
返回 None
      |
      v
选择抢占对象
      |
      v
_preempt_request()
      |
      v
释放可释放的 KV Block
      |
      v
重新尝试分配
```

### **6.6 第四步：请求抢占 Preemption**

当 KV Cache 不足以满足新的分配请求时，Scheduler 可以抢占已有请求。

核心方法：

```
_preempt_request()
```

源码中不同策略选择不同的抢占对象：

```
if self.policy == SchedulingPolicy.PRIORITY:    preempted_req = max(        self.running,        key=lambda r: (r.priority, r.arrival_time),    )else:    preempted_req = self.running[-1]
```

也就是说：

- `PRIORITY`：根据请求优先级和到达时间选择抢占对象。
- `FCFS`：选择 Running 列表末尾的请求。

抢占过程：

```
             Running Request
                    |
                    v
              需要抢占
                    |
                    v
             _preempt_request()
                    |
          +---------+---------+
          |                   |
          v                   v
       释放 KV             清理执行状态
          |                   |
          +---------+---------+
                    |
                    v
             Request 状态变更
                    |
                    v
              重新进入 Waiting
                    |
                    v
              后续重新调度
```

**注意：Preemption 不等于请求终止。**

被抢占请求仍然可以重新进入调度流程，后续可能需要重新计算之前的 Prompt Token。

### **6.7 第五步：调度 Waiting Requests**

源码注释：

```
# Next, schedule the WAITING requests.
```

Waiting 请求需要经过更多检查：

1. 是否允许继续调度。
2. 当前 Running 数量是否达到上限。
3. 请求是否处于被阻塞状态。
4. 是否有尚未交付的异步输出。
5. LoRA 数量是否超出限制。
6. 是否命中本地 Prefix Cache。
7. 是否命中远程 KV Cache。
8. Encoder 输入是否可以调度。
9. 是否可以分配 KV Block。

其中，Prefix Cache 的处理非常重要。

```
_get_local_prefix_cache_hit()
```

先检查本地 Prefix Cache，再根据 Connector 配置检查远程 KV Cache：

```
self.connector.get_num_new_matched_tokens(    request, block_aligned_local)
```

这意味着新请求并不一定需要从头计算 Prompt。

如果有可复用的 KV Cache，可以跳过已经计算过的 Token。

### **6.8 第六步：构建 SchedulerOutput**

最终 Scheduler 会将调度结果组织成 `SchedulerOutput`。

源码：

```
scheduler_output = SchedulerOutput(    scheduled_new_reqs=new_reqs_data,    scheduled_cached_reqs=cached_reqs_data,    num_scheduled_tokens=num_scheduled_tokens,    total_num_scheduled_tokens=total_num_scheduled_tokens,    scheduled_spec_decode_tokens=scheduled_spec_decode_tokens,    scheduled_encoder_inputs=scheduled_encoder_inputs,    scheduled_encoder_input_stats=scheduled_encoder_input_stats,    num_common_prefix_blocks=num_common_prefix_blocks,    preempted_req_ids=self.reset_preempted_req_ids,    finished_req_ids=self.finished_req_ids,    free_encoder_mm_hashes=self.encoder_cache_manager.get_freed_mm_hashes(),    new_block_ids_to_zero=self._get_new_block_ids_to_zero(),    has_sync_kv_loads=has_sync_kv_loads,    kv_cache_block_copies=pending_kv_cache_block_copies,    kv_connector_block_state=kv_connector_block_state,    num_spec_tokens_to_schedule=num_spec_tokens_to_schedule,    ec_manager_metadata=self.encoder_cache_manager.get_manager_metadata(),)
```

主要字段：

| 字段                             | 作用                            |
| ------------------------------ | ----------------------------- |
| `scheduled_new_reqs`           | 本轮新调度或重新恢复的请求信息               |
| `scheduled_cached_reqs`        | 本轮继续执行的已有请求信息                 |
| `num_scheduled_tokens`         | 每个请求本轮调度的 Token 数             |
| `total_num_scheduled_tokens`   | 本轮总调度 Token 数                 |
| `scheduled_spec_decode_tokens` | Speculative Decoding 相关 Token |
| `scheduled_encoder_inputs`     | 本轮调度的 Encoder 输入              |
| `num_common_prefix_blocks`     | 公共 Prefix Cache Block 信息      |
| `preempted_req_ids`            | 被抢占请求 ID                      |
| `finished_req_ids`             | 已完成请求 ID                      |
| `new_block_ids_to_zero`        | 需要清零的 KV Block ID             |
| `kv_connector_metadata`        | KV Connector 元数据              |
| `ec_connector_metadata`        | Encoder Cache Connector 元数据   |

最终执行：

```
self._update_after_schedule(scheduler_output)return scheduler_output
```

**SchedulerOutput 是调度计划，不是模型执行结果。**

## **7. SchedulerOutput 与 ModelRunnerOutput**

这两个结构体是 Scheduler 和 Model Runner 之间最重要的数据交换对象。

### **7.1 数据流向**

```
                 Scheduler
                     |
                     |
                     v
              SchedulerOutput
                     |
                     |
                     v
                Model Runner
                     |
           +---------+---------+
           |         |         |
           v         v         v
        Prepare   Forward   Sampling
        Inputs    Model
           |         |         |
           +---------+---------+
                     |
                     v
              ModelRunnerOutput
                     |
                     |
                     v
                 Scheduler
```

### **7.2 两种数据结构的区别**

| 维度   | SchedulerOutput      | ModelRunnerOutput      |
| ---- | -------------------- | ---------------------- |
| 产生方  | Scheduler            | Model Runner           |
| 消费方  | Model Runner         | Scheduler              |
| 本质   | 调度计划                 | 模型执行结果                 |
| 主要内容 | 请求、Token、Block、调度元数据 | 采样 Token、Logprobs、执行结果 |
| 作用   | 告诉执行侧本轮做什么           | 告诉调度侧本轮执行结果            |

### **7.3 NewRequestData 和 CachedRequestData**

SchedulerOutput 中需要区分新请求和已有请求。

```
                SchedulerOutput
                       |
             +---------+---------+
             |                   |
             v                   v
       NewRequestData      CachedRequestData
             |                   |
             v                   v
       新请求初始化          已有请求更新
       或请求恢复            增量更新状态
             |                   |
             +---------+---------+
                       |
                       v
                  Model Runner
```

为什么需要区分？

因为 Model Runner 维护自己的执行侧请求状态。

- 新请求需要建立执行状态。
- 已有请求只需要更新 Token、Block 等变化。

这样可以避免每个 Step 都重新初始化整个 Batch。

## **8. update_from_output()：执行结果处理**

`update_from_output()` 是 Scheduler 的另一个核心方法。

它负责消费 Model Runner 的输出，将执行结果同步回 Scheduler 内部的 Request 状态。

### **8.1 核心职责**

1. 接收 ModelRunnerOutput。
2. 获取本轮请求的生成 Token。
3. 更新 Request 的 Token 状态。
4. 处理 Speculative Decoding 结果。
5. 判断请求是否完成。
6. 释放请求占用的资源。
7. 生成 EngineCoreOutputs 返回上层。

### **8.2 核心流程**

```
             ModelRunnerOutput
                     |
                     v
              update_from_output()
                     |
                     v
             遍历本轮调度请求
                     |
                     v
            获取请求对应的输出索引
                     |
                     v
              更新 Request 状态
                     |
          +----------+----------+
          |                     |
          v                     v
      生成 Token             更新计算进度
          |                     |
          +----------+----------+
                     |
                     v
              check_stop()
                     |
              +------+------+
              |             |
              v             v
          请求未完成       请求完成
              |             |
              v             v
          保留运行状态    _free_request()
              |             |
              |             v
              |         释放相关资源
              |             |
              +------+------+
                     |
                     v
              EngineCoreOutputs
                     |
                     v
                 Engine Core
```

### **8.3 请求完成判断**

Scheduler 会检查请求是否达到停止条件，例如：

- 生成 Token 数达到 `max_tokens`。
- 生成 EOS Token。
- 命中 Stop 条件。
- 请求被取消或终止。
- 发生需要终止请求的错误。

核心逻辑涉及：

```
check_stop(...)
```

### **8.4 \_free_request()**

请求完成后，Scheduler 需要清理对应状态。

主要涉及：

- 从请求管理结构中移除请求。
- 释放 Encoder Cache。
- 处理 KV Connector 的请求完成通知。
- 将请求 ID 加入 `finished_req_ids`。
- 释放 KV Block，必要时延迟释放。

注意：请求完成和 KV Block 立即可复用并不总是同一时刻。

如果存在异步 KV 传输或多个 In-flight Batch，需要等待相关执行完成后再释放。

## **9. KVCacheManager：KV Cache 管理核心**

`KVCacheManager` 是 Scheduler 最重要的协作组件之一。

### **9.1 Scheduler 与 KVCacheManager 的职责边界**

```
                  Scheduler
                      |
                      |
             决定是否需要 KV
             决定分配多少 KV
             决定是否需要抢占
                      |
                      v
                KVCacheManager
                      |
            +---------+---------+
            |         |         |
            v         v         v
         Allocate    Free    Prefix Cache
            |         |         |
            +---------+---------+
                      |
                      v
                 KVCacheBlocks
                      |
                      v
                 KVCacheBlock
```

Scheduler 决定资源调度策略，KVCacheManager 负责具体的 Block 管理。

### **9.2 重要操作**

| 方法                               | 作用                   |
| -------------------------------- | -------------------- |
| `new_step_starts()`              | 开始新的调度 Step          |
| `allocate_slots()`               | 为请求分配 KV Block       |
| `free()`                         | 释放请求使用的 KV Block     |
| `get_computed_blocks()`          | 获取已经计算的缓存 Block      |
| `get_num_common_prefix_blocks()` | 获取公共 Prefix Block 信息 |
| `reset_prefix_cache()`           | 重置 Prefix Cache      |
| `take_new_block_ids()`           | 获取新分配的 Block ID      |
| `take_kv_cache_block_copies()`   | 获取待执行的 Block Copy 信息 |

### **9.3 Prefix Cache 的作用**

假设两个请求：

```
Request A:
Hello, I am a software engineer. Tell me about Go.

Request B:
Hello, I am a software engineer. Tell me about Python.
```

二者具有相同 Prompt 前缀：

```
Hello, I am a software engineer.
```

如果 Prefix Cache 命中：

```
              Request A
                  |
                  v
           计算 Prompt Prefix
                  |
                  v
              KV Cache
                  ^
                  |
              Request B
                  |
                  v
           查询 Prefix Cache
                  |
                  v
              命中缓存
                  |
                  v
          跳过已计算的 Prefix
                  |
                  v
           只计算剩余 Token
```

这样可以减少重复 Prefill 计算。

### **9.4 延迟释放 KV Block**

源码中存在：

```
self.sched_step_seq = 0self.processed_step_seq = 0self.deferred_frees: deque[tuple[int, list[KVCacheBlock]]] = deque()
```

其核心目的是解决异步执行下的资源竞争。

例如：

```
Step N:
GPU 正在使用 KV Block 10
        |
        v
Step N+1:
Scheduler 认为请求已经结束
        |
        v
不能立即复用 Block 10
        |
        v
等待 Step N 的执行结果处理完成
        |
        v
满足释放条件
        |
        v
释放 Block 10
```

核心思想：

**CPU 调度侧认为请求完成，不代表 GPU 或异步传输侧已经不再使用对应 KV Block。**

因此需要通过 Step 序号和延迟释放队列保证资源安全。

## **10. EncoderCacheManager：多模态缓存管理**

除了 KV Cache，Scheduler 还管理多模态 Encoder Cache。

### **10.1 两种缓存的区别**

| 缓存            | 保存内容                    | 主要作用                          |
| ------------- | ----------------------- | ----------------------------- |
| KV Cache      | Attention 的 Key / Value | 避免重复执行已有 Token 的 Attention 计算 |
| Encoder Cache | 多模态 Encoder 输出          | 避免重复执行图像、音频等输入的 Encoder 计算    |

### **10.2 调度过程**

```
                Request
                   |
                   v
             检查 Encoder 输入
                   |
                   v
        _try_schedule_encoder_inputs()
                   |
           +-------+-------+
           |               |
           v               v
       可以调度         暂时不能调度
           |               |
           v               v
      分配 Encoder       跳过请求
         Cache
           |
           v
      记录调度信息
           |
           v
      SchedulerOutput
           |
           v
       Model Runner
```

相关组件：

- `EncoderCacheManager`
- `EncoderDecoderCacheManager`
- `MultiModalBudget`
- `ECConnector`

## **11. KVConnector：远程 KV 传输**

KVConnector 用于协调不同实例之间的 KV Cache 传输。

典型场景：

- Prefill / Decode 分离部署。
- KV Cache Offloading。
- 远程 KV Cache 加载。
- 分布式 KV Cache 共享。

### **11.1 Scheduler 和 Worker 各有 Connector**

源码注释明确指出：

```
# Create KVConnector for the Scheduler.# Note that each Worker will have a corresponding KVConnector# with Role=WORKER.
```

架构：

```
                   Scheduler
                       |
                       v
              KVConnector
             Role=SCHEDULER
                       |
                       |
                 Metadata
                       |
                       v
                  Model Runner
                       |
                       v
                    Worker
                       |
                       v
                KVConnector
                Role=WORKER
                       |
                       v
                Remote KV Storage
```

Scheduler 侧主要负责调度协调和元数据构建，Worker 侧配合实际 KV 数据加载、保存和传输。

### **11.2 重要方法**

| 方法                             | 作用                  |
| ------------------------------ | ------------------- |
| `get_num_new_matched_tokens()` | 查询远程可复用的 KV Token 数 |
| `update_state_after_alloc()`   | KV 分配后的状态更新         |
| `build_connector_meta()`       | 构建传输元数据             |
| `request_finished()`           | 通知请求完成              |
| `has_pending_push_work()`      | 检查是否存在待处理的 KV Push  |

具体的数据读写由 Connector 与 Worker 侧实现协同完成。

## **12. Scheduler 中的高级调度机制**

Scheduler 不只是简单的 FCFS 队列，还集成了多种优化机制。

| 机制                    | 解决的问题                   | 核心作用                  |
| --------------------- | ----------------------- | --------------------- |
| Continuous Batching   | 请求动态进入和退出               | 每轮重新调度请求              |
| Chunked Prefill       | 长 Prompt 占用过多计算资源       | 将 Prefill 拆分成多个 Chunk |
| Prefix Caching        | 重复 Prompt 计算            | 复用已有 KV Cache         |
| Speculative Decoding  | Decode 速度               | 调度 Draft Token        |
| EAGLE / MTP           | 推测解码优化                  | 配合 Draft Token 调度     |
| DP Prefill Balancing  | DP Rank 之间的 Prefill 不均衡 | 调整 Prefill 调度时机       |
| Mamba Cache Alignment | Mamba 状态缓存边界            | 对齐 Prefill 分块边界       |
| LoRA Scheduling       | Batch 中 LoRA 数量约束       | 限制同一 Batch 的 LoRA 种类  |
| Structured Output     | 结构化生成约束                 | 配合 Grammar 和 Bitmask  |

这些机制是否启用，取决于模型类型、配置和部署模式，并非所有机制都会在每次调度中生效。

### **12.1 Chunked Prefill**

长 Prompt 可以拆分为多个计算块：

```
Prompt:
[T1 T2 T3 T4 T5 T6 T7 T8 T9 T10]

Step 1:
[T1 T2 T3 T4]

Step 2:
[T5 T6 T7]

Step 3:
[T8 T9 T10]

之后进入 Decode
```

作用：

- 避免长 Prompt 一次性占用大量 Token Budget。
- 让 Decode 请求有机会与 Prefill 请求共同执行。
- 提升 GPU 执行资源利用率。

### **12.2 Continuous Batching**

假设同时有三个请求：

```
Step 1:
Request A: Prefill
Request B: Prefill

Step 2:
Request A: Decode
Request B: Decode
Request C: Prefill

Step 3:
Request A: Decode
Request B: Finished
Request C: Decode
Request D: Prefill
```

请求可以在每个 Step 动态进入或退出 Batch，而不是固定等待整个 Batch 执行完毕。

### **12.3 Speculative Decoding**

Scheduler 需要处理 Draft Token：

```
             Draft Tokens
                  |
                  v
          Scheduler 调度预算
                  |
                  v
          scheduled_spec_decode_tokens
                  |
                  v
              Model Runner
                  |
                  v
           验证 Draft Tokens
                  |
                  v
          ModelRunnerOutput
                  |
                  v
          更新 Request Token 状态
```

Scheduler 不负责执行 Draft Token 的模型计算，但需要管理对应的 Token 进度和调度资源。

## **13. Pause / Resume：调度控制**

Scheduler 支持暂停和恢复调度。

`PauseState`：

```
class PauseState(Enum):    UNPAUSED = 0    PAUSED_NEW = 1    PAUSED_ALL = 2
```

| 状态           | 作用       |
| ------------ | -------- |
| `UNPAUSED`   | 正常调度     |
| `PAUSED_NEW` | 暂停新请求调度  |
| `PAUSED_ALL` | 暂停所有请求调度 |

在 `schedule()` 中：

```
if self._pause_state == PauseState.PAUSED_ALL:    token_budget = 0
```

暂停机制主要用于控制调度行为，而不是直接终止正在执行的模型计算。

## **14. Scheduler 的完整请求生命周期**

把前面各个模块串起来，可以得到完整请求生命周期。

```
                   add_request()
                        |
                        v
                  创建 Request
                        |
                        v
                   waiting
                        |
                        v
                    schedule()
                        |
                        v
              检查 Prefix Cache
                        |
                        v
               分配 KV Cache Block
                        |
              +---------+---------+
              |                   |
              v                   v
           成功分配             分配失败
              |                   |
              |                   v
              |                Preemption
              |                   |
              |                   v
              |              重新进入 Waiting
              |
              v
         SchedulerOutput
              |
              v
          Model Runner
              |
              v
        ModelRunnerOutput
              |
              v
      update_from_output()
              |
              v
        更新 Request 状态
              |
         +----+----+
         |         |
         v         v
      未完成     已完成
         |         |
         v         v
      running   _free_request()
         |         |
         |         v
         |      释放 KV Cache
         |      释放 Encoder Cache
         |      通知 Worker 清理状态
         |         |
         +---------+
              |
              v
             结束
```

## **15. 重点源码阅读路线**

Scheduler 代码量较大，不建议按照文件顺序逐行阅读。建议以请求生命周期为主线。

### **15.1 P0：必须掌握**

| 方法                                | 重点理解                  |
| --------------------------------- | --------------------- |
| `__init__()`                      | Scheduler 如何初始化各种管理组件 |
| `schedule()`                      | 整体调度算法                |
| `update_from_output()`            | 如何消费模型执行结果            |
| `_preempt_request()`              | 请求抢占机制                |
| `_free_request()`                 | 请求完成后的资源释放            |
| `_update_after_schedule()`        | 调度完成后的状态更新            |
| `KVCacheManager.allocate_slots()` | KV Block 分配           |

### **15.2 P1：深入理解**

| 方法                                       | 重点理解            |
| ---------------------------------------- | --------------- |
| `_select_waiting_queue_for_scheduling()` | Waiting 队列选择    |
| `_get_local_prefix_cache_hit()`          | 本地 Prefix Cache |
| `_try_schedule_encoder_inputs()`         | 多模态输入调度         |
| `_update_waiting_for_remote_kv()`        | 远程 KV 等待处理      |
| `_build_kv_connector_meta()`             | KV 传输元数据        |
| `_drain_deferred_frees()`                | 延迟释放机制          |

### **15.3 P2：高级功能**

| 方法或模块                   | 重点理解            |
| ----------------------- | --------------- |
| `get_grammar_bitmask()` | Grammar 约束      |
| `make_stats()`          | 调度统计            |
| `reset_prefix_cache()`  | Prefix Cache 重置 |
| Speculative Decoding    | Draft Token 调度  |
| Mamba Cache Alignment   | Mamba 缓存边界      |
| DP Prefill Balancing    | DP 调度均衡         |

## **16. 建议断点调试路线**

建议从一个最简单的请求开始：

```
Prompt = "Hello"
max_tokens = 5
```

沿着以下调用顺序断点：

```
1. add_request()
       |
       v
2. _enqueue_waiting_request()
       |
       v
3. schedule()
       |
       v
4. _select_waiting_queue_for_scheduling()
       |
       v
5. _get_local_prefix_cache_hit()
       |
       v
6. KVCacheManager.allocate_slots()
       |
       v
7. NewRequestData.from_request()
       |
       v
8. _update_after_schedule()
       |
       v
9. Model Runner 执行
       |
       v
10. update_from_output()
       |
       v
11. check_stop()
       |
       v
12. _free_request()
```

### **建议重点观察的变量**

| 变量                             | 观察目的                |
| ------------------------------ | ------------------- |
| `request.status`               | 请求状态如何变化            |
| `request.num_computed_tokens`  | 已计算 Token 数如何变化     |
| `request.num_tokens_with_spec` | 当前需要处理的 Token 数     |
| `num_new_tokens`               | 本轮给请求分配多少 Token     |
| `token_budget`                 | 当前 Step 剩余 Token 预算 |
| `input_budget`                 | 当前 Batch 剩余输入预算     |
| `new_blocks`                   | 本轮分配的 KV Block      |
| `self.running`                 | Running 队列变化        |
| `self.waiting`                 | Waiting 队列变化        |
| `scheduler_output`             | 调度结果如何组织            |

### **进一步实验**

| 实验                      | 观察目标                |
| ----------------------- | ------------------- |
| 同时提交多个短请求               | Continuous Batching |
| 提交一个长 Prompt            | Chunked Prefill     |
| 提交相同前缀的两个请求             | Prefix Cache        |
| 限制 KV Block 数量          | Preemption          |
| 开启 Speculative Decoding | Draft Token 调度      |
| 开启多模态模型                 | Encoder Cache 调度    |

## **17. 最终总结：Scheduler 应该掌握的五个核心问题**

| 核心问题                 | 对应模块                          |
| -------------------- | ----------------------------- |
| 请求什么时候执行？            | `RequestQueue`、`schedule()`   |
| 每个请求本轮执行多少 Token？    | Token Budget、`num_new_tokens` |
| 请求需要的 KV Cache 如何管理？ | `KVCacheManager`              |
| KV Cache 不够怎么办？      | `_preempt_request()`          |
| 模型执行完之后如何更新请求？       | `update_from_output()`        |

**理解 Scheduler，最重要的是掌握两个核心闭环：**

```
调度闭环：

Request
   |
   v
Scheduler.schedule()
   |
   v
SchedulerOutput
   |
   v
Model Runner
   |
   v
ModelRunnerOutput
   |
   v
Scheduler.update_from_output()
   |
   v
Request 状态更新
```

```
资源管理闭环：

Request 需要 KV
      |
      v
KVCacheManager.allocate_slots()
      |
      v
获得 KV Block
      |
      v
Model Runner 使用 KV
      |
      v
Request 完成 / Preemption
      |
      v
释放或延迟释放 KV Block
      |
      v
KV Block 回到可分配资源池
```

从架构角度看，Scheduler 是连接请求生命周期、Token 调度、KV Cache 管理和模型执行的核心控制平面。理解它，就能把 vLLM 从请求进入 Engine Core 到模型执行、结果返回的关键机制串联起来。

源码依据：用户提供的 Scheduler 实现文件。上述方法和字段以该文件版本为准。&#x20;

粘贴的文本 (1)(20261117-153619).txt
