下面整理的是前面那篇 **《vLLM Core Engine Process Manager：架构、类图与关键流程》**，不是刚才的 Executor / ModelRunner 执行架构文档。

保留原来的核心内容，整理成 GitHub 可直接渲染的 Markdown，包含 Mermaid 类图、流程图、概念表格和源码阅读路线。



# vLLM Core Engine Process Manager：架构、类图与关键流程

## 1. 模块定位

Core Engine Process Manager 是 vLLM V1 中负责 EngineCore 生命周期管理的模块。

它不负责具体的模型推理计算，而是负责：

- 创建和启动 EngineCore 进程。
- 管理 EngineCore 的 DP Rank。
- 分配和映射 GPU 资源。
- 建立 EngineCore 与 Frontend 之间的 ZMQ 通信。
- 通过 HELLO / READY 握手确认 EngineCore 启动状态。
- 监控 EngineCore 进程运行状态。
- 管理 EngineCore 的退出和资源清理。
- 支持 Multiprocessing 和 Ray 两种进程管理方式。

核心代码涉及：

| 类 / 函数                       | 职责                                 |
| ---------------------------- | ---------------------------------- |
| `CoreEngine`                 | 跟踪单个 EngineCore 的启动状态              |
| `CoreEngineProcManager`      | 管理本地 Multiprocessing EngineCore 进程 |
| `CoreEngineActorManager`     | 管理 Ray EngineCore Actor            |
| `EngineZmqAddresses`         | 保存 EngineCore 与 Frontend 的通信地址     |
| `EngineHandshakeMetadata`    | 保存启动握手所需的元数据                       |
| `CoreEngineLaunch`           | 封装 EngineCore 启动资源和状态              |
| `launch_core_engines()`      | 启动 EngineCore 的统一入口                |
| `wait_for_engine_startup()`  | 等待 EngineCore 完成启动握手               |
| `get_engine_zmq_addresses()` | 创建 Frontend 与 EngineCore 的 ZMQ 地址  |

---

## 2. 整体架构

### 2.1 组件关系

```
flowchart TB
    A["Frontend / API Server"]

    B["launch_core_engines()"]

    C{"选择 DP Backend"}

    D["CoreEngineProcManager"]
    E["CoreEngineActorManager"]

    F["EngineCoreProc<br/>Multiprocessing"]
    G["EngineCoreActor<br/>Ray"]

    H["EngineCore"]

    I["ZMQ Handshake<br/>HELLO / READY"]

    J["Frontend 与 EngineCore<br/>建立通信"]

    A --> B
    B --> C
    C -->|"Multiprocessing"| D
    C -->|"Ray"| E

    D --> F
    E --> G

    F --> H
    G --> H

    H --> I
    I --> J
```

### 2.2 两种管理模式

| 对比项  | Multiprocessing                    | Ray                        |
| ---- | ---------------------------------- | -------------------------- |
| 管理类  | `CoreEngineProcManager`            | `CoreEngineActorManager`   |
| 执行实体 | Python Process                     | Ray Actor                  |
| 创建方式 | `context.Process()`                | `ray.remote()`             |
| 资源管理 | 本地进程与 GPU 映射                       | Placement Group            |
| 启动入口 | `EngineCoreProc.run_engine_core()` | `EngineCoreActor`          |
| 监控方式 | Process Sentinel                   | `ray.wait()` / `ray.get()` |
| 适用场景 | 本地进程管理                             | 分布式节点与 Ray 调度              |

注意：Ray 路径与 Multiprocessing 路径的启动机制并不完全相同。不能简单认为两者都执行同一套 HELLO / READY 握手流程。

---

## 3. 类图

### 3.1 核心类关系

```
classDiagram
    class CoreEngineState {
        <<enumeration>>
        NEW
        CONNECTED
        READY
    }

    class CoreEngine {
        +int index
        +bool local
        +bytes identity
        +CoreEngineState state
        +connected()
        +ready()
    }

    class EngineZmqAddresses {
        +list inputs
        +list outputs
        +str coordinator_input
        +str coordinator_output
        +str frontend_stats_publish_address
    }

    class EngineHandshakeMetadata {
        +EngineZmqAddresses addresses
        +dict parallel_config
    }

    class CoreEngineLaunch {
        +engine_manager
        +coordinator
        +addresses
        +tensor_queue
        +watched_frontend_processes
    }

    class CoreEngineProcManager {
        +list processes
        +shutdown()
        +monitor_engine_liveness()
        +finished_procs()
        +sentinels()
    }

    class CoreEngineActorManager {
        +list local_engine_actors
        +list remote_engine_actors
        +shutdown()
        +monitor_engine_liveness()
        +scale_up()
        +scale_down()
    }

    class CoreEngineProc {
        +run_engine_core()
    }

    class EngineCoreActor {
        +wait_for_init()
        +run()
    }

    CoreEngine --> CoreEngineState
    CoreEngineLaunch --> CoreEngineProcManager
    CoreEngineLaunch --> CoreEngineActorManager
    CoreEngineLaunch --> EngineZmqAddresses

    CoreEngineProcManager --> CoreEngineProc
    CoreEngineActorManager --> EngineCoreActor

    EngineHandshakeMetadata --> EngineZmqAddresses
```

### 3.2 类的职责

| 类                         | 核心职责                | 关键理解                                       |
| ------------------------- | ------------------- | ------------------------------------------ |
| `CoreEngine`              | 管理单个 EngineCore 的状态 | 关注单个 EngineCore 是否连接、是否就绪                  |
| `CoreEngineProcManager`   | 管理本地进程              | 负责创建、启动、监控、关闭                              |
| `CoreEngineActorManager`  | 管理 Ray Actor        | 负责 Actor 创建、资源调度、生命周期                      |
| `EngineZmqAddresses`      | 保存通信地址              | 统一管理 Frontend、EngineCore、Coordinator 的通信端点 |
| `EngineHandshakeMetadata` | 保存握手元数据             | 向 EngineCore 传递通信地址和并行配置                   |
| `CoreEngineLaunch`        | 启动资源封装              | 统一管理启动过程中产生的资源                             |

---

## 4. 核心概念

### 4.1 DP Rank

DP（Data Parallel）表示数据并行。

每个 DP Rank 对应一个独立的 EngineCore 实例，各自负责一部分请求。

例如：

```
DP Size = 4

DP Rank 0 -> EngineCore 0
DP Rank 1 -> EngineCore 1
DP Rank 2 -> EngineCore 2
DP Rank 3 -> EngineCore 3
```

几个容易混淆的概念：

| 概念                         | 含义                        |
| -------------------------- | ------------------------- |
| `data_parallel_size`       | 全局 DP 实例数量                |
| `data_parallel_rank`       | 当前 EngineCore 的全局 DP Rank |
| `data_parallel_size_local` | 当前节点上的 DP 实例数量            |
| `data_parallel_rank_local` | 当前节点内的 DP Rank            |

注意：DP Rank 与 TP Rank、PP Rank 是不同维度。

一个 EngineCore 内部还可以通过 TP、PP 等方式管理多个 GPU Worker。

### 4.2 World Size

`world_size` 表示模型执行过程中分布式并行组的规模。

通常与 TP、PP 等并行维度相关，不能简单等同于 TP Size。

例如：

```
TP = 4
PP = 1

World Size = TP × PP = 4
```

如果使用 TP=4、PP=2：

```
World Size = 4 × 2 = 8
```

### 4.3 Placement Group

Placement Group 是 Ray 提供的资源组织和调度机制。

它可以将一组 CPU、GPU 资源作为一个整体进行分配。

vLLM 使用 Placement Group 管理 EngineCore Actor 及其对应的 GPU Worker 资源。

常见调度策略：

| 策略              | 含义                  |
| --------------- | ------------------- |
| `STRICT_PACK`   | 所有 Bundle 必须放在同一个节点 |
| `PACK`          | 尽可能将 Bundle 放在较少节点  |
| `STRICT_SPREAD` | Bundle 必须分布在不同节点    |
| `SPREAD`        | 尽可能将 Bundle 分散到不同节点 |

### 4.4 ZMQ ROUTER Identity

在 Multiprocessing 启动路径中，EngineCore 使用自己的 Rank 构造 ZMQ Identity：

```
self.identity = index.to_bytes(2, "little")
```

作用：

- 区分不同 EngineCore。
- 让 ROUTER Socket 识别消息来源。
- 将握手消息关联到对应 DP Rank。

例如：

```
EngineCore 0 -> Identity: 00 00
EngineCore 1 -> Identity: 01 00
EngineCore 2 -> Identity: 02 00
```

### 4.5 HELLO / READY

启动握手有两个重要状态：

| 状态      | 含义                      |
| ------- | ----------------------- |
| `HELLO` | EngineCore 已连接，开始交换启动信息 |
| `READY` | EngineCore 初始化完成，可以提供服务 |

状态变化：

```
stateDiagram-v2
    [*] --> NEW
    NEW --> CONNECTED: HELLO
    CONNECTED --> READY: READY
```

其中：

- `NEW`：尚未建立连接。
- `CONNECTED`：已经建立连接并完成初始握手。
- `READY`：初始化完成。

---

## 5. 关键流程

### 5.1 EngineCore 启动主流程

核心入口：

```
launch_core_engines(
    vllm_config,
    executor_class,
    log_stats,
    addresses,
)
```

整体流程：

```
flowchart TD
    A["launch_core_engines()"]
    B["读取 ParallelConfig"]
    C["判断是否需要 DP Coordinator"]
    D{"DP Backend"}
    E["创建 CoreEngineProcManager"]
    F["创建 CoreEngineActorManager"]
    G["启动 EngineCore 进程"]
    H["创建 Ray Actor"]
    I["等待 EngineCore 初始化"]
    J["完成启动"]

    A --> B
    B --> C
    C --> D
    D -->|"Multiprocessing"| E
    D -->|"Ray"| F
    E --> G
    F --> H
    G --> I
    H --> I
    I --> J
```

注意：这是概念级流程。Ray 路径通过 Actor 初始化和启动流程管理就绪状态，Multiprocessing 路径则有独立的 ZMQ 握手等待流程。

### 5.2 Multiprocessing 进程创建

核心类：

`CoreEngineProcManager`

核心逻辑：

```
from vllm.v1.engine.core import EngineCoreProc

self.processes.append(
    context.Process(
        target=EngineCoreProc.run_engine_core,
        name=f"EngineCore_DP{global_index}",
        kwargs={
            "vllm_config": vllm_config,
            "executor_class": executor_class,
            "dp_rank": global_index,
            "local_dp_rank": local_index,
        },
    )
)
```

执行过程：

1. 获取 multiprocessing context。
2. 根据 local DP Rank 和 global DP Rank 创建进程。
3. 设置 EngineCore 进程入口。
4. 配置 GPU 映射和 NUMA 环境。
5. 调用 `proc.start()` 启动进程。

进程创建关系：

```
flowchart TD
    A["CoreEngineProcManager"]
    B["计算 local_dp_rank"]
    C["计算 global_dp_rank"]
    D["创建 multiprocessing.Process"]
    E["target = EngineCoreProc.run_engine_core"]
    F["配置 GPU / NUMA"]
    G["proc.start()"]

    A --> B
    B --> C
    C --> D
    D --> E
    E --> F
    F --> G
```

### 5.3 GPU ID 映射

DP 环境下，不同 EngineCore 需要使用不同的 GPU 资源。

例如：

```
总 GPU 数量：8
每个 DP Rank 使用 2 张 GPU

DP Rank 0 -> GPU [0, 1]
DP Rank 1 -> GPU [2, 3]
DP Rank 2 -> GPU [4, 5]
DP Rank 3 -> GPU [6, 7]
```

核心逻辑：

```
start = local_dp_rank * world_size
stop = start + local_world_size

gpu_ids = user_assigned_gpu_ids[start:stop]
```

如果没有显式指定 `--device-ids`，则根据平台设备映射机制计算物理设备 ID。

需要区分：

- EngineCore 自身没有 TP/PP local rank。
- EngineCore 进程使用 `local_rank=0` 配置自身环境。
- 真正的 TP/PP Worker 进程由 Executor 单独绑定自己的 local rank。

### 5.4 ZMQ 地址分配

核心函数：

`get_engine_zmq_addresses()`

主要负责生成 Frontend 与 EngineCore 通信所需的地址。

常见形式：

| 类型  | 地址形式              | 适用场景      |
| --- | ----------------- | --------- |
| IPC | `ipc://...`       | 本地进程通信    |
| TCP | `tcp://host:port` | 跨进程、跨节点通信 |

代码中使用：

```
def _addr():
    if client_local_only:
        return get_open_zmq_ipc_path()

    return get_tcp_uri(
        host,
        0 if defer_api_server_ports else get_open_port(),
    )
```

其中：

- 本地通信优先使用 IPC。
- 需要跨节点通信时使用 TCP。
- `tcp://host:0` 表示由消费者绑定端口，并将实际端口反馈给调用方。

### 5.5 HELLO / READY 握手流程

核心函数：

`wait_for_engine_startup()`

核心作用：

等待所有需要管理的 EngineCore 完成连接和初始化。

主要有两个计数器：

```
conn_pending = [local_count, remote_count]
start_pending = [0, 0]
```

含义：

| 计数器             | 含义                         |
| --------------- | -------------------------- |
| `conn_pending`  | 尚未完成连接握手的 EngineCore 数量    |
| `start_pending` | 已连接但尚未完成初始化的 EngineCore 数量 |

流程：

```
sequenceDiagram
    participant F as Frontend / Manager
    participant E as EngineCore 0
    participant E2 as EngineCore 1

    F->>E: 建立 ZMQ 连接
    F->>E2: 建立 ZMQ 连接

    E->>F: HELLO
    F->>F: conn_pending - 1
    F->>F: start_pending + 1

    E2->>F: HELLO
    F->>F: conn_pending - 1
    F->>F: start_pending + 1

    E->>F: READY
    F->>F: start_pending - 1

    E2->>F: READY
    F->>F: start_pending - 1

    F->>F: 所有计数器归零
    Note over F: EngineCore 启动完成
```

当以下条件满足时，启动握手结束：

```
while any(conn_pending) or any(start_pending):
    ...
```

也就是：

```
所有 EngineCore 完成连接
并且
所有 EngineCore 完成初始化
```

此外，等待期间还会监控 EngineCore、Coordinator 和相关 Frontend 进程是否异常退出。

### 5.6 DP Coordinator

DP Coordinator 不是每种场景都需要启动。

代码中的判断条件：

```
run_coordinator = (
    vllm_config.needs_dp_coordinator
    and not offline_mode
    and dp_rank == 0
)
```

主要作用：

1. 内部或混合负载均衡：收集并发布 EngineCore 队列状态。
2. MoE 模型：提供额外的 Wave Coordination 协调能力。

可以理解为：

```
flowchart TD
    A["DP Rank 0"]
    B{"needs_dp_coordinator<br/>且非 offline mode"}
    C["启动 DP Coordinator"]
    D["不启动 Coordinator"]
    E["收集 Queue Stats"]
    F["负载均衡 / MoE Wave Coordination"]

    A --> B
    B -->|"Yes"| C
    B -->|"No"| D
    C --> E
    E --> F
```

### 5.7 EngineCore 进程监控

Multiprocessing 路径：

```
connection.wait(sentinels, timeout=1)
```

通过 Process Sentinel 检测进程是否退出。

Ray 路径：

```
ray.wait(...)
ray.get(...)
```

通过 Ray ObjectRef 检测 Actor 运行状态，并处理 `RayActorError` 等异常。

两种方式的差异：

| 管理方式            | 监控机制                       |
| --------------- | -------------------------- |
| Multiprocessing | Process Sentinel           |
| Ray             | `ray.wait()` / `ray.get()` |

### 5.8 进程关闭

`CoreEngineProcManager.shutdown()` 主要负责：

1. 设置停止标志。
2. 解除 Finalizer。
3. 根据配置计算关闭超时时间。
4. 调用统一的进程关闭逻辑。

ROCm 环境下，在特定超时配置下可能额外给予清理时间，以便 GPU 资源正常释放。

---

## 6. Ray Placement Group 与 EngineCore Actor

Ray 模式下，主要由：

`CoreEngineActorManager`

负责管理 EngineCore Actor。

### 6.1 Actor 创建

核心逻辑：

```
from vllm.v1.engine.core import (
    DPMoEEngineCoreActor,
    EngineCoreActor,
)

actor_class = (
    DPMoEEngineCoreActor
    if dp_size > 1 and vllm_config.model_config.is_moe
    else EngineCoreActor
)
```

根据 DP Size 和模型是否为 MoE，选择对应的 Actor 类型。

### 6.2 Placement Group 资源组织

一个 Placement Group 可以包含多个 GPU Bundle 和一个 CPU-only Control Bundle。

概念示例：

```
Placement Group
    |
    +-- Bundle 0: GPU
    +-- Bundle 1: GPU
    +-- Bundle 2: GPU
    +-- Bundle 3: GPU
    +-- Bundle 4: CPU (EngineCore Actor)
```

在对应的资源组织方式中：

```
placement_group_bundle_index = world_size
```

表示 EngineCore Actor 放置在最后的 CPU-only Bundle。

这样可以将 EngineCore Actor 与其对应的 GPU Worker 资源保持在预期的节点布局中。

### 6.3 Ray 启动流程

```
flowchart TD
    A["CoreEngineActorManager"]
    B["初始化 Ray"]
    C["创建 Placement Groups"]
    D["创建 EngineCore Actors"]
    E["等待 wait_for_init.remote()"]
    F["调用 actor.run.remote()"]
    G["监控 Actor"]
    H["关闭 Actor"]

    A --> B
    B --> C
    C --> D
    D --> E
    E --> F
    F --> G
    G --> H
```

---

## 7. 关键方法索引

| 方法                                                 | 所属类 / 模块                 | 作用               |
| -------------------------------------------------- | ------------------------ | ---------------- |
| `launch_core_engines()`                            | 模块函数                     | 统一启动入口           |
| `wait_for_engine_startup()`                        | 模块函数                     | 等待启动握手           |
| `get_engine_zmq_addresses()`                       | 模块函数                     | 分配通信地址           |
| `CoreEngine.connected()`                           | `CoreEngine`             | 更新连接状态           |
| `CoreEngine.ready()`                               | `CoreEngine`             | 更新就绪状态           |
| `CoreEngineProcManager.__init__()`                 | `CoreEngineProcManager`  | 创建 EngineCore 进程 |
| `CoreEngineProcManager.shutdown()`                 | `CoreEngineProcManager`  | 关闭进程             |
| `CoreEngineProcManager.monitor_engine_liveness()`  | `CoreEngineProcManager`  | 监控进程状态           |
| `CoreEngineActorManager.__init__()`                | `CoreEngineActorManager` | 创建 Ray Actor     |
| `CoreEngineActorManager.shutdown()`                | `CoreEngineActorManager` | 关闭 Actor         |
| `CoreEngineActorManager.monitor_engine_liveness()` | `CoreEngineActorManager` | 监控 Actor 状态      |

---

## 8. VS Code 断点阅读建议

建议按照启动链路设置断点，而不是按文件顺序逐行阅读。

| 顺序 | 断点位置                               | 观察重点                       |
| -- | ---------------------------------- | -------------------------- |
| 1  | `launch_core_engines()`            | 如何选择 Multiprocessing / Ray |
| 2  | `CoreEngineProcManager.__init__()` | 如何创建 EngineCore 进程         |
| 3  | `EngineCoreProc.run_engine_core()` | 子进程如何进入 EngineCore         |
| 4  | GPU 映射函数                           | DP Rank 如何映射到物理 GPU        |
| 5  | `get_engine_zmq_addresses()`       | IPC / TCP 地址如何生成           |
| 6  | `wait_for_engine_startup()`        | HELLO / READY 如何更新状态       |
| 7  | `CoreEngine.connected()`           | 连接状态如何变化                   |
| 8  | `CoreEngine.ready()`               | 就绪状态如何变化                   |
| 9  | `monitor_engine_liveness()`        | 如何检测进程异常                   |
| 10 | `shutdown()`                       | 进程如何安全退出                   |

---

## 9. 本模块核心总结

Core Engine Process Manager 主要解决四个问题：

| 问题                   | 解决机制                                |
| -------------------- | ----------------------------------- |
| EngineCore 如何创建？     | Multiprocessing Process / Ray Actor |
| EngineCore 如何分配 GPU？ | DP Rank 与物理 GPU 映射                  |
| EngineCore 如何建立连接？   | ZMQ IPC / TCP                       |
| EngineCore 如何确认启动完成？ | HELLO / READY 握手                    |

整体可以总结为：

```
创建 EngineCore
    |
分配 DP Rank 和 GPU
    |
启动进程或 Ray Actor
    |
建立 ZMQ 通信
    |
HELLO：连接成功
    |
READY：初始化完成
    |
监控进程生命周期
    |
安全关闭
```

## 10. 下一步源码阅读方向

当前模块解决的是 EngineCore 的进程管理和启动问题。

下一步建议进入：

`vllm/v1/engine/core.py`

重点阅读：

1. `EngineCoreProc.run_engine_core()`：EngineCore 进程入口。
2. `EngineCore`：推理引擎核心对象。
3. `EngineCore.step()`：调度与执行主流程。
4. `Scheduler`：请求调度和 KV Cache 管理。
5. `Executor`：执行任务分发。
6. Worker / ModelRunner：GPU 模型执行。

最终目标是将进程管理层与推理执行层连接起来，理解从 EngineCore 启动到一次模型推理的完整过程。



建议文件名：`vllm-core-engine-process-manager.md`。Mermaid 图表使用 GitHub 原生支持的语法，可以直接在仓库 README 或 Markdown 文档中渲染。
