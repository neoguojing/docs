下面整理成可直接提交到 GitHub 仓库的 `README.md` / 技术笔记格式，使用 GitHub 原生支持的 Markdown、Mermaid 类图和时序图。保留源码中的类名、方法名和关键流程，并明确区分 `EngineCoreClient` 与后端 `Executor` 的职责边界。

# vLLM EngineCoreClient 源码解析

> 源码范围：`vllm/v1/engine/core_client.py`\
> 核心职责：连接 Frontend 与 EngineCore，负责请求发送、结果接收、进程通信、数据并行路由和生命周期管理。

## 1. 整体架构

### 1.1 核心定位

`EngineCoreClient` 是 Frontend 与 EngineCore 之间的通信抽象层。

它不直接执行模型推理，而是负责：

- 将 Frontend 请求发送给 EngineCore。
- 接收 EngineCore 返回的推理结果。
- 管理 EngineCore 的进程和通信资源。
- 支持同步、异步和多进程通信模式。
- 支持 Data Parallel 场景下的请求路由和负载均衡。

整体调用链：

```
AsyncLLM / LLM
      |
      v
EngineCoreClient
      |
      +---------------------------+
      |                           |
      v                           v
InprocClient                   MPClient
                                  |
                    +-------------+-------------+
                    |                           |
                    v                           v
               SyncMPClient                AsyncMPClient
                                                |
                                  +-------------+-------------+
                                  |                           |
                                  v                           v
                            DPAsyncMPClient             DPLBAsyncMPClient
                                  |                           |
                             外部负载均衡                  内部负载均衡
```

多进程模式下的通信链路：

```
Frontend
   |
   v
EngineCoreClient
   |
   | ZMQ ROUTER
   v
EngineCoreProc
   |
   v
EngineCore
   |
   v
Scheduler / Executor
   |
   v
GPU Worker / Model Execution
```

注意：`Scheduler`、`Executor` 和 GPU Worker 属于后端执行链路，不是 `EngineCoreClient` 本身的组成部分。

## 2. 类图

### 2.1 EngineCoreClient 继承关系

```
classDiagram
    class EngineCoreClient {
        <<abstract>>
        +make_client()
        +make_async_mp_client()
        +add_request()
        +get_output()
        +shutdown()
        +get_supported_tasks()
    }

    class InprocClient {
        -engine_core: EngineCore
        +add_request()
        +get_output()
        +get_supported_tasks()
    }

    class MPClient {
        <<abstract>>
        -ctx
        -input_socket
        -output_socket
        -resources
        -encoder
        -decoder
        +_send_input()
        +_send_input_message()
        +call_utility()
        +shutdown()
    }

    class SyncMPClient {
        -outputs_queue
        -output_thread
        +add_request()
        +get_output()
    }

    class AsyncMPClient {
        -outputs_queue
        -output_queue_task
        +add_request_async()
        +get_output_async()
        +call_utility_async()
    }

    class DPAsyncMPClient {
        -stats_update_task
        -first_req_send_socket
        +get_core_engine_for_request()
        +add_request_async()
    }

    class DPLBAsyncMPClient {
        -reqs_in_flight
        -engine_inflight
        -lb_engines
        +get_core_engine_for_request()
        +process_engine_outputs()
    }

    EngineCoreClient <|-- InprocClient
    EngineCoreClient <|-- MPClient
    MPClient <|-- SyncMPClient
    MPClient <|-- AsyncMPClient
    AsyncMPClient <|-- DPAsyncMPClient
    DPAsyncMPClient <|-- DPLBAsyncMPClient
```

### 2.2 关键类职责

| 类                   | 职责           | 关键点                      |
| ------------------- | ------------ | ------------------------ |
| `EngineCoreClient`  | 客户端抽象层       | 负责客户端类型选择                |
| `InprocClient`      | 进程内通信        | 直接调用 `EngineCore`        |
| `MPClient`          | 多进程通信基础类     | ZMQ、进程管理、编解码             |
| `SyncMPClient`      | 同步客户端        | 线程和同步 Queue              |
| `AsyncMPClient`     | 异步客户端        | asyncio Task 和 Queue     |
| `DPAsyncMPClient`   | DP 外部 LB 客户端 | 一个客户端主要路由到自己的 EngineCore |
| `DPLBAsyncMPClient` | DP 内部 LB 客户端 | 在多个 EngineCore 之间进行负载均衡  |

## 3. 客户端创建流程

### 3.1 工厂方法

入口：

```
EngineCoreClient.make_client(
    multiprocess_mode,
    asyncio_mode,
    vllm_config,
    executor_class,
    log_stats,
    renderer,
)
```

创建逻辑：

```
flowchart TD
    A["make_client()"] --> B{"multiprocess_mode?"}
    B -->|False| C["InprocClient"]
    B -->|True| D{"asyncio_mode?"}
    D -->|False| E["SyncMPClient"]
    D -->|True| F["make_async_mp_client()"]
    F --> G{"data_parallel_size > 1?"}
    G -->|False| H["AsyncMPClient"]
    G -->|True| I{"data_parallel_external_lb?"}
    I -->|True| J["DPAsyncMPClient"]
    I -->|False| K["DPLBAsyncMPClient"]
```

### 3.2 创建模式对比

| 模式       | 客户端                 | 通信方式                 | 适用场景                 |
| -------- | ------------------- | -------------------- | -------------------- |
| 进程内同步    | `InprocClient`      | 直接函数调用               | EngineCore 与调用方在同一进程 |
| 多进程同步    | `SyncMPClient`      | ZMQ + 线程             | 同步 LLM               |
| 多进程异步    | `AsyncMPClient`     | ZMQ + asyncio        | AsyncLLM             |
| DP 外部 LB | `DPAsyncMPClient`   | ZMQ + DP Coordinator | 外部负责 DP 请求分发         |
| DP 内部 LB | `DPLBAsyncMPClient` | ZMQ + 内部负载均衡         | 客户端负责选择 EngineCore   |

注意：当前实现不支持 `asyncio_mode=True` 且 `multiprocess_mode=False` 的组合。

## 4. MPClient 初始化流程

`MPClient` 是多进程客户端的基础类，主要完成通信设施和 EngineCore 子进程的初始化。

### 4.1 初始化主要步骤

```
flowchart TD
    A["MPClient.__init__()"] --> B["创建 ZMQ Context"]
    B --> C["创建 BackgroundResources"]
    C --> D["初始化 DP 配置"]
    D --> E["创建 ZMQ Socket"]
    E --> F["准备 EngineCore 地址"]
    F --> G["启动 EngineCore 进程"]
    G --> H["等待 EngineCore Ready"]
    H --> I["处理 Ready Response"]
    I --> J["初始化 Encoder / Decoder"]
    J --> K["启动进程存活监控"]
    K --> L["初始化完成"]
```

### 4.2 关键组件

| 组件                       | 作用                           |
| ------------------------ | ---------------------------- |
| `zmq.Context`            | 管理 ZMQ 通信上下文                 |
| `ROUTER`                 | Frontend 向指定 EngineCore 发送请求 |
| `PULL`                   | Frontend 接收 EngineCore 返回的数据 |
| `MsgpackEncoder`         | 将请求序列化为消息帧                   |
| `MsgpackDecoder`         | 将返回消息反序列化                    |
| `BackgroundResources`    | 管理 Socket、Task 等后台资源         |
| `CoreEngineProcManager`  | 管理 EngineCore 子进程            |
| `CoreEngineActorManager` | 管理 EngineCore Actor          |
| `weakref.finalize`       | 在对象生命周期结束时触发资源清理             |

### 4.3 ROUTER 与 PULL

多进程通信的核心是两个方向：

```
Frontend                          EngineCoreProc
   |                                    |
   |---------- ROUTER ----------------->|
   |           Request                  |
   |                                    |
   |<----------- PULL ------------------|
   |           Output                   |
```

其中：

- `ROUTER`：请求发送通道，可以通过 EngineIdentity 指定目标 EngineCore。
- `PULL`：结果接收通道，负责接收 EngineCore 返回的消息。
- `EngineIdentity`：EngineCore 在 ZMQ 路由中的身份标识。

请求发送时，消息结构可以理解为：

```
(engine_identity, request_type, payload...)
```

其中 `request_type` 用于区分 ADD、ABORT、UTILITY 等请求类型。

## 5. 请求发送与结果接收流程

### 5.1 请求发送

核心方法：

```
_send_input()
_send_input_message()
add_request()
add_request_async()
```

异步请求流程：

```
sequenceDiagram
    participant F as Frontend
    participant C as AsyncMPClient
    participant Z as ZMQ ROUTER
    participant P as EngineCoreProc
    participant E as EngineCore

    F->>C: add_request_async(request)
    C->>C: _ensure_stats_update_task()
    C->>C: 设置 current_wave / client_index
    C->>C: get_core_engine_for_request()
    C->>C: Encoder.encode(request)
    C->>Z: send_multipart(engine_identity, message)
    Z->>P: 发送请求
    P->>E: 分发请求
    C->>C: 等待发送完成
    C->>C: _ensure_output_queue_task()
```

核心代码逻辑：

```
async def add_request_async(self, request):
    self._ensure_stats_update_task()

    request.current_wave = self.current_wave
    request.client_index = self.client_index

    chosen_engine = self.get_core_engine_for_request(request)

    to_await = self._send_input(
        EngineCoreRequestType.ADD,
        request,
        chosen_engine,
    )

    await to_await

    self._ensure_output_queue_task()
```

### 5.2 请求结果接收

`AsyncMPClient` 使用后台 asyncio Task 持续读取输出 Socket。

```
sequenceDiagram
    participant E as EngineCoreProc
    participant Z as ZMQ PULL
    participant T as OutputQueueTask
    participant Q as asyncio.Queue
    participant F as Frontend

    E->>Z: 返回 EngineCoreOutputs
    Z->>T: recv_multipart()
    T->>T: validate_alive()
    T->>T: MsgpackDecoder.decode()
    T->>T: 判断 UtilityOutput
    alt 普通推理输出
        T->>Q: put_nowait(outputs)
    else Utility RPC
        T->>T: 根据 call_id 匹配 Future
    end
    F->>Q: get_output_async()
    Q-->>F: EngineCoreOutputs
```

关键方法：

```
_ensure_output_queue_task()
process_outputs_socket()
get_output_async()
```

其中：

- `process_outputs_socket()`：后台读取 ZMQ 输出。
- `outputs_queue`：保存推理输出。
- `get_output_async()`：Frontend 获取结果。
- `validate_alive()`：检查 EngineCore 是否异常退出。

## 6. Utility RPC 机制

除普通推理请求外，客户端还需要向 EngineCore 发送管理命令。

例如：

- `sleep()` / `wake_up()`
- `profile()`
- `reset_prefix_cache()`
- `reset_encoder_cache()`
- `add_lora()` / `remove_lora()`
- `collective_rpc()`
- `get_supported_tasks()`

### 6.1 请求与返回关联

通过 `call_id` 将异步返回结果和原始调用关联起来。

```
sequenceDiagram
    participant F as Frontend
    participant C as MPClient
    participant E as EngineCore
    participant M as Output Handler

    F->>C: call_utility(method, args)
    C->>C: 生成 call_id
    C->>C: utility_results[call_id] = Future
    C->>E: 发送 UTILITY 请求
    E->>M: 返回 UtilityOutput(call_id)
    M->>M: 查找 utility_results[call_id]
    M->>M: Future.set_result()
    M-->>F: 返回结果
```

核心数据结构：

```
utility_results: dict[int, AnyFuture]
```

作用：

1. 每个 Utility RPC 生成唯一 `call_id`。
2. 创建对应的 Future。
3. 将 Future 存入字典。
4. EngineCore 返回结果时，根据 `call_id` 查找 Future。
5. 调用 `set_result()` 或 `set_exception()`。

这样即使多个 RPC 并发执行、返回顺序不同，也能正确匹配调用方。

## 7. Sync 与 Async 的区别

| 对比项  | SyncMPClient    | AsyncMPClient         |
| ---- | --------------- | --------------------- |
| 执行模型 | 同步              | asyncio               |
| 输出读取 | 后台线程            | 后台 asyncio Task       |
| 消息队列 | `queue.Queue`   | `asyncio.Queue`       |
| 获取结果 | `get_output()`  | `get_output_async()`  |
| 发送请求 | `add_request()` | `add_request_async()` |
| 适用接口 | LLM             | AsyncLLM              |

核心区别不是通信协议，而是 Frontend 对 IO 的调度方式。

两者都复用 `MPClient` 的 ZMQ 通信基础设施。

## 8. Data Parallel 请求路由

当 `data_parallel_size > 1` 时，需要决定请求发往哪个 EngineCore。

这里有两种模式：

### 8.1 外部负载均衡：DPAsyncMPClient

外部 LB 模式下，Frontend 客户端通常只负责向自己管理的 EngineCore 发送请求。

```
External Load Balancer
          |
          +----------------+
          |                |
          v                v
       Client 0         Client 1
          |                |
          v                v
       Engine 0         Engine 1
```

核心特点：

- 由外部负载均衡器决定请求分发。
- `DPAsyncMPClient.get_core_engine_for_request()` 返回自己的 `core_engine`。
- 通过 Coordinator 交换 DP 统计信息。
- 使用 `FIRST_REQ` 通知 Coordinator 有新的请求进入。

### 8.2 内部负载均衡：DPLBAsyncMPClient

内部 LB 模式下，一个客户端可以在多个 EngineCore 之间选择目标。

```
                 Frontend
                    |
                    v
             DPLBAsyncMPClient
                    |
          +---------+---------+
          |         |         |
          v         v         v
       Engine 0  Engine 1  Engine 2
```

核心方法：

```
get_core_engine_for_request()
process_engine_outputs()
```

### 8.3 负载均衡状态

主要数据结构：

| 字段                     | 含义                                 |
| ---------------------- | ---------------------------------- |
| `lb_engines`           | 从 Coordinator 获取的各 EngineCore 负载快照 |
| `engine_inflight`      | 当前客户端各 EngineCore 的未完成请求数          |
| `reqs_in_flight`       | Request ID 到 EngineIdentity 的映射    |
| `current_wave`         | 当前 DP 调度周期                         |
| `engine_ranks_managed` | 当前客户端管理的 Engine Rank               |

Coordinator 提供全局负载信息，客户端维护本地请求状态。

### 8.4 负载评分公式

源码中的核心评分逻辑：

[\
S_i =\
\max(C \cdot I_i,\ W_i + R_i)\
+\
6W_i\max(0,K_i-0.5)\
]

其中：

| 符号    | 含义                         |
| ----- | -------------------------- |
| (S_i) | Engine (i) 的负载评分           |
| (C)   | `client_count`，客户端数量       |
| (I_i) | 当前客户端在 Engine (i) 上的未完成请求数 |
| (W_i) | Engine (i) 的 waiting 请求数   |
| (R_i) | Engine (i) 的 running 请求数   |
| (K_i) | Engine (i) 的 KV Cache 使用率  |

选择评分最低的 EngineCore：

[\
engine = \arg\min_i S_i\
]

解释：

1. `client_count * inflight`：估计所有客户端在该 Engine 上的请求压力。
2. `waiting + running`：Coordinator 提供的全局请求负载。
3. `max()`：避免统计快照延迟导致本地刚发送的请求被忽略。
4. KV Cache 惩罚：当 KV Cache 使用率超过 50% 时，提高 waiting 请求的惩罚。
5. 最终选择负载评分最低的 EngineCore。

注意：这是基于请求数量和 KV Cache 使用率的负载估算，不是直接测量 GPU 利用率。

### 8.5 为什么需要本地 inflight 计数？

Coordinator 的统计信息并非实时同步，源码注释指出统计更新存在约 100ms 的间隔。

如果仅依赖 Coordinator 快照：

```
Client 发送请求 A
       |
       v
Coordinator 统计尚未更新
       |
       v
Client 发送请求 B
       |
       v
仍然认为 Engine 负载较低
```

本地 `engine_inflight` 能够立即反映当前客户端已发送但尚未完成的请求，降低统计延迟带来的路由偏差。

## 9. 请求结束与 ABORT 路由

`DPLBAsyncMPClient` 维护：

```
reqs_in_flight: dict[str, EngineIdentity]
engine_inflight: Counter[EngineIdentity]
```

其作用是：

- 请求发送时记录 Request ID 对应的 Engine。
- 增加对应 Engine 的 inflight 计数。
- 请求结束时清理映射并减少计数。
- ABORT 请求根据 Request ID 找到原始 EngineCore。
- 将 ABORT 请求发送给正确的 EngineCore。

为什么需要记录 Request ID 到 Engine 的映射？

因为请求可能被路由到不同的 EngineCore，取消请求时必须找到真正执行该请求的 EngineCore。

## 10. Elastic EP 动态扩缩容

`DPLBAsyncMPClient` 还支持 Elastic EP 相关协调逻辑。

涉及：

- `ElasticScalingCache`
- `EEPNotificationType`
- Prepare / Commit
- EngineCore 通知
- Engine Rank 调整

简化流程：

```
sequenceDiagram
    participant C as DPLBAsyncMPClient
    participant E1 as Existing EngineCore
    participant E2 as New EngineCore
    participant D as DP Coordinator

    C->>D: Prepare scaling
    D->>E1: 准备扩缩容
    D->>E2: 初始化新增 Engine
    E1-->>C: Prepare 通知
    E2-->>C: Prepare 通知
    C->>C: 更新扩缩容缓存
    C->>D: Commit scaling
    D->>E1: 提交配置
    D->>E2: 提交配置
    D-->>C: 完成通知
    C->>C: 更新 Engine 路由状态
```

核心设计思想：

- 扩缩容不是简单地修改 Engine 数量。
- 需要协调 EngineCore、Coordinator 和 Frontend。
- 移除 Engine 前，需要停止向待移除 Engine 分配新请求。
- 通过 Prepare / Commit 保证各参与者协调完成。

## 11. 进程监控与资源回收

### 11.1 EngineCore 进程监控

`MPClient` 会启动后台监控线程，检查 EngineCore 进程是否存活。

```
MPClient
   |
   v
Engine Monitor Thread
   |
   v
monitor_engine_liveness()
   |
   +---- Engine 正常：继续监控
   |
   +---- Engine 异常退出
              |
              v
       engine_dead = True
              |
              v
          shutdown()
```

### 11.2 BackgroundResources

`BackgroundResources` 统一管理后台资源：

- ZMQ Socket
- ZMQ Context
- 输出处理 Task
- DP 统计 Task
- 相关后台通信资源

使用：

```
weakref.finalize(self, self.resources)
```

在客户端对象生命周期结束时触发资源清理。

这样可以避免客户端初始化失败、异常退出或对象被回收时遗留通信资源。

## 12. 核心概念速查

| 概念               | 解释                            |
| ---------------- | ----------------------------- |
| EngineCoreClient | Frontend 与 EngineCore 之间的通信抽象 |
| Inproc           | 同一进程内直接调用                     |
| Multiprocess     | EngineCore 在独立进程中运行           |
| ZMQ              | 进程间消息通信机制                     |
| ROUTER           | 请求发送端，可按身份路由                  |
| PULL             | 接收 EngineCore 输出              |
| EngineIdentity   | EngineCore 的通信身份              |
| Msgpack          | 请求与结果的序列化格式                   |
| Utility RPC      | 对 EngineCore 的管理类远程调用         |
| Future           | 异步结果占位对象                      |
| asyncio.Queue    | 异步输出队列                        |
| DP               | Data Parallel，数据并行            |
| Coordinator      | DP 场景下协调各 EngineCore 的组件      |
| External LB      | 外部负载均衡                        |
| Internal LB      | 客户端内部负载均衡                     |
| inflight         | 已发送但尚未完成的请求                   |
| KV Cache Usage   | KV Cache 使用率                  |
| Elastic EP       | 支持动态扩缩容的 Expert Parallel 相关机制 |

## 13. 源码断点阅读路线

建议按以下顺序阅读，不要从文件第一行开始逐行阅读。

| 顺序 | 类 / 方法                                            | 阅读目标                 |
| -- | ------------------------------------------------- | -------------------- |
| 1  | `EngineCoreClient.make_client()`                  | 理解客户端创建和模式选择         |
| 2  | `make_async_mp_client()`                          | 理解 DP 模式分支           |
| 3  | `MPClient.__init__()`                             | 理解 ZMQ、进程启动、Ready 握手 |
| 4  | `MPClient._send_input()`                          | 理解请求序列化与发送           |
| 5  | `AsyncMPClient.add_request_async()`               | 理解异步请求入口             |
| 6  | `process_outputs_socket()`                        | 理解后台输出处理             |
| 7  | `get_output_async()`                              | 理解 Frontend 获取结果     |
| 8  | `call_utility_async()`                            | 理解 Utility RPC       |
| 9  | `DPAsyncMPClient`                                 | 理解 DP Coordinator 通信 |
| 10 | `DPLBAsyncMPClient.get_core_engine_for_request()` | 理解负载均衡               |
| 11 | `process_engine_outputs()`                        | 理解请求结束和负载状态维护        |
| 12 | Elastic EP 相关方法                                   | 理解动态扩缩容              |

## 14. 下一步：进入 EngineCore 执行链路

本文件主要解决的是：

> Frontend 如何把请求发送给 EngineCore，以及如何接收 EngineCore 的返回结果？

下一步应该继续阅读：

```
EngineCoreProc
     |
     v
EngineCore
     |
     v
Scheduler
     |
     v
Executor
     |
     v
GPU Worker
     |
     v
Model Runner
     |
     v
GPU Model Execution
```

重点需要回答：

1. `EngineCoreProc` 如何启动和运行 EngineCore？
2. `EngineCore` 如何接收请求并调用 Scheduler？
3. Scheduler 如何决定请求进入 Prefill 还是 Decode？
4. Executor 如何调度 GPU Worker？
5. Model Runner 如何执行模型计算？
6. KV Cache 如何分配、管理和回收？
7. 推理结果如何从 EngineCore 返回 Frontend？

这条链路才是理解 vLLM 推理执行机制的核心。

这版可以直接保存为 GitHub 仓库中的 `docs/vllm-engine-core-client.md`。其中 Mermaid 图在 GitHub Markdown 页面中可直接渲染。下一步建议沿用同样的结构，继续整理 `EngineCore → Scheduler → Executor → ModelRunner`。
