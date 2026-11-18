以下是完整的 `vllm_executor_architecture.md`。保留 Executor 的架构、核心类、关键接口、初始化流程、RPC、异步执行和分布式机制，所有图统一使用纯文本字符图，**不使用 Mermaid**，可直接复制到 GitHub Markdown 文件中。

# vLLM Executor 核心架构、类图与执行流程

## 1. Executor 整体定位

Executor 是 vLLM 执行管理层，位于 Scheduler 与 Worker 之间。

它不负责决定请求如何调度，而是负责将 Scheduler 生成的执行任务交给 Worker，并协调 Worker 的初始化、方法调用、分布式通信和资源管理。

### 1.1 整体架构

```
+--------------------------------------------------+
|                  API Server                      |
|             OpenAI API / Frontend                |
+-------------------------+------------------------+
                          |
                          v
+--------------------------------------------------+
|                     Engine                       |
|                                                  |
|  +--------------------------------------------+  |
|  |                  Scheduler                 |  |
|  |                                            |  |
|  |  - 请求调度                                 |  |
|  |  - Batch 构建                               |  |
|  |  - Token Budget 管理                        |  |
|  |  - KV Cache Block 分配                      |  |
|  +----------------------+---------------------+  |
|                         |                        |
|                         | SchedulerOutput        |
|                         v                        |
|  +--------------------------------------------+  |
|  |                  Executor                  |  |
|  |                                            |  |
|  |  - Worker 生命周期管理                     |  |
|  |  - RPC 调用分发                            |  |
|  |  - 分布式执行协调                          |  |
|  |  - 设备资源管理                            |  |
|  |  - 模型执行管理                            |  |
|  +----------------------+---------------------+  |
+-------------------------|------------------------+
                          |
                          | RPC / Method Invocation
                          v
+--------------------------------------------------+
|                    Worker                        |
|                                                  |
|  - Device 初始化                                 |
|  - Model 加载                                    |
|  - Distributed 初始化                            |
|  - ModelRunner 管理                              |
+-------------------------+------------------------+
                          |
                          v
+--------------------------------------------------+
|                  ModelRunner                     |
|                                                  |
|  - Input Preparation                             |
|  - Attention / KV Cache                          |
|  - Model Forward                                 |
|  - Sampling                                      |
|  - Output Processing                             |
+-------------------------+------------------------+
                          |
                          v
+--------------------------------------------------+
|                     Model                        |
|                                                  |
|  Transformer / LLM / Multimodal Model            |
+--------------------------------------------------+
```

### 1.2 核心职责

| 职责            | 主要功能                                |
| ------------- | ----------------------------------- |
| Worker 生命周期管理 | 创建 Worker、初始化设备、加载模型                |
| RPC 执行管理      | 向 Worker 发送方法调用，收集执行结果              |
| 分布式协调         | 管理 Rank、Process Group、分布式初始化        |
| KV Cache 管理   | 获取可用显存，协调 KV Cache 容量               |
| 模型运行管理        | Warmup、Profiling、Compile、CUDA Graph |
| 运行时资源管理       | LoRA、多模态缓存、Sleep/Wakeup 等           |

### 1.3 核心分工

| 组件          | 核心问题   | 职责                        |
| ----------- | ------ | ------------------------- |
| Scheduler   | 执行什么   | 决定哪些请求进入当前执行批次            |
| Executor    | 如何组织执行 | 管理 Worker、分发调用、协调执行       |
| Worker      | 在哪里执行  | 管理设备、模型和运行环境              |
| ModelRunner | 具体怎么执行 | 执行 Forward、Sampling 等模型操作 |
| Model       | 如何计算   | 执行模型网络计算                  |

核心理解：

**Scheduler 决定执行内容，Executor 负责组织执行，Worker 提供执行环境，ModelRunner 执行模型计算。**

---

## 2. Executor 核心类关系

### 2.1 类关系图

```
                         +----------------------+
                         |       Executor       |
                         |      <<abstract>>    |
                         +----------+-----------+
                                    |
                 +------------------+------------------+
                 |                                     |
                 v                                     v
       +---------------------+             +---------------------+
       |   UniProcExecutor   |             |   Other Executors   |
       +----------+----------+             |                     |
                  |                        | - MultiprocExecutor |
                  |                        | - RayExecutor       |
                  |                        +---------------------+
                  |
                  | inherits
                  v
       +-----------------------------+
       | ExecutorWithExternalLauncher|
       +-----------------------------+


       +---------------------+
       |   UniProcExecutor   |
       +----------+----------+
                  |
                  | manages
                  v
       +---------------------+
       |  WorkerWrapperBase  |
       +---------------------+
                  |
                  | invokes
                  v
       +---------------------+
       |       Worker        |
       +---------------------+
                  |
                  | manages
                  v
       +---------------------+
       |     ModelRunner     |
       +---------------------+


       +---------------------+
       |   UniProcExecutor   |
       +----------+----------+
                  |
                  | creates / adapts
                  v
       +---------------------+
       |  AsyncOutputFuture  |
       +---------------------+
```

注意：

- `Executor` 是抽象接口层。
- `UniProcExecutor` 是单进程执行后端。
- `ExecutorWithExternalLauncher` 继承 `UniProcExecutor`，复用其执行管理能力，并适配外部启动的分布式环境。
- `WorkerWrapperBase` 是 Worker 的包装和调用入口。
- `AsyncOutputFuture` 用于适配异步模型输出。

### 2.2 核心类职责

| 类                              | 职责               | 关键关注点            |
| ------------------------------ | ---------------- | ---------------- |
| `Executor`                     | Executor 抽象接口    | 定义统一执行能力         |
| `UniProcExecutor`              | 单进程执行后端          | Worker 创建、RPC 调用 |
| `ExecutorWithExternalLauncher` | 外部 Launcher 执行后端 | 分布式环境适配          |
| `WorkerWrapperBase`            | Worker 包装层       | Worker 初始化和方法调用  |
| `AsyncOutputFuture`            | 异步输出适配           | 延迟获取模型执行结果       |

---

## 3. Executor 后端选择机制

### 3.1 后端类型

vLLM 支持不同的 Executor 后端，以适配不同的执行部署方式。

| Backend             | Executor                       | 说明                  |
| ------------------- | ------------------------------ | ------------------- |
| `uni`               | `UniProcExecutor`              | 单进程执行               |
| `mp`                | `MultiprocExecutor`            | 多进程执行               |
| `ray`               | `RayExecutor`                  | 使用 Ray 管理分布式 Worker |
| `external_launcher` | `ExecutorWithExternalLauncher` | 外部启动器管理分布式进程        |
| 自定义                 | Custom Executor                | 用户扩展执行后端            |

### 3.2 后端选择流程

```
                 Engine 初始化
                       |
                       v
              Executor.get_class()
                       |
                       v
          读取 Executor Backend 配置
                       |
                       v
              +----------------+
              | Backend 类型？ |
              +----------------+
                 |    |    |
       +---------+    |    +----------------+
       |              |                     |
       v              v                     v
      uni             mp                   ray
       |              |                     |
       v              v                     v
 UniProcExecutor  MultiprocExecutor     RayExecutor
       |
       | external_launcher
       v
 ExecutorWithExternalLauncher

                       |
                       v
                 创建实例
                       |
                       v
                初始化 Executor
```

实际可选后端和配置名称应以当前 vLLM 版本的 `Executor.get_class()` 实现为准。

### 3.3 阅读重点

重点看 `Executor.get_class()`：

1. 如何读取配置中的 backend。
2. 如何映射到具体 Executor 类。
3. 是否支持自定义 Executor。
4. 不同 backend 的初始化入口有什么差异。

---

## 4. Executor 抽象接口

### 4.1 核心接口总览

| 方法                             | 作用                   |
| ------------------------------ | -------------------- |
| `get_class()`                  | 根据配置选择 Executor 实现   |
| `collective_rpc()`             | 向 Worker 分发 RPC 方法调用 |
| `execute_model()`              | 执行模型 Forward         |
| `sample_tokens()`              | 执行 Token Sampling    |
| `determine_available_memory()` | 确定可用设备内存             |
| `check_health()`               | 检查 Executor 健康状态     |
| `shutdown()`                   | 关闭执行后端               |
| `initialize_from_config()`     | 根据配置初始化执行后端          |
| `profile()`                    | 执行性能分析               |
| `sleep()`                      | 控制执行后端休眠             |
| `wake_up()`                    | 恢复执行后端               |

具体方法的抽象实现、默认实现和子类覆盖情况，需要结合源码分别判断。

### 4.2 collective_rpc：核心分发接口

`collective_rpc()` 是 Executor 最重要的接口之一。

主要作用：

- 指定需要调用的 Worker 方法。
- 传递位置参数和关键字参数。
- 指定目标 Rank（具体能力取决于实现）。
- 控制同步或非阻塞调用。
- 收集 Worker 执行结果。

调用示例：

```
executor.collective_rpc(
    "some_worker_method",
    args=(),
    kwargs={},
)
```

可以将其理解为：

```
             Executor
                 |
                 | collective_rpc(method, args, kwargs)
                 |
                 v
          Worker Wrapper
                 |
                 | execute_method()
                 |
                 v
              Worker
                 |
                 | 执行目标方法
                 |
                 v
             Result
                 |
                 v
             Executor
```

**设计原则：RPC 主要传递控制消息和方法调用，大规模数据传输应该考虑独立的数据通信通道。**

### 4.3 execute_model()

`execute_model()` 负责将 Scheduler 产生的执行任务交给 Worker 执行。

典型输入：

- `SchedulerOutput`
- 当前执行批次相关信息

典型输出：

- 模型执行结果
- ModelRunner 输出
- 异步输出对象（取决于执行路径）

调用关系：

```
Scheduler
    |
    | execute_model(SchedulerOutput)
    v
Executor
    |
    | collective_rpc()
    v
Worker
    |
    | execute_model()
    v
ModelRunner
    |
    | Model Forward
    v
Model
    |
    | Forward Result
    v
ModelRunner
    |
    v
Worker
    |
    v
Executor
    |
    v
Scheduler
```

### 4.4 sample_tokens()

Sampling 是模型执行流程中的重要环节。

在相应执行路径中，Executor 通过 `sample_tokens()` 将 Sampling 操作交给 Worker，再由 Worker/ModelRunner 完成具体操作。

需要区分：

- Forward：模型计算，生成 Logits 等结果。
- Sampling：根据 Logits 和采样参数选择 Token。
- Executor：负责组织和分发调用，不负责具体采样算法。

### 4.5 execute_model 与 sample_tokens 调用关系

```
+-------------------+
|     Scheduler     |
+---------+---------+
          |
          | execute_model()
          v
+-------------------+
|     Executor      |
+---------+---------+
          |
          | collective_rpc()
          v
+-------------------+
|      Worker       |
+---------+---------+
          |
          | execute_model()
          v
+-------------------+
|    ModelRunner    |
+---------+---------+
          |
          | Forward
          v
+-------------------+
|       Model       |
+---------+---------+
          |
          | Logits / Model Output
          v
+-------------------+
|    ModelRunner    |
+---------+---------+
          |
          | Sampling（对应执行路径）
          v
+-------------------+
|      Worker       |
+---------+---------+
          |
          v
+-------------------+
|     Executor      |
+---------+---------+
          |
          v
+-------------------+
|     Scheduler     |
+-------------------+
```

注意：Forward 与 Sampling 是否分开调用、由哪个组件触发，需要结合具体模型执行路径和当前版本实现分析。

---

## 5. UniProcExecutor 初始化流程

### 5.1 初始化整体流程

`UniProcExecutor` 负责在当前进程内创建和管理 Worker。

```
                 创建 UniProcExecutor
                           |
                           v
                创建 WorkerWrapperBase
                           |
                           v
               准备 Distributed 参数
                           |
                           v
                 设置 Worker Device
                           |
                           v
                    init_worker()
                           |
                           v
                    init_device()
                           |
                           v
                     load_model()
                           |
                           v
               更新 Block Size 等信息
                           |
                           v
                    初始化完成
```

### 5.2 初始化阶段说明

| 阶段                | 主要作用                |
| ----------------- | ------------------- |
| 创建 Worker Wrapper | 创建 Worker 包装对象      |
| 准备 Distributed 参数 | 准备分布式初始化需要的信息       |
| 设置 Device         | 确定 Worker 使用的设备     |
| `init_worker()`   | 初始化 Worker 基础信息     |
| `init_device()`   | 初始化 CUDA 等设备环境      |
| `load_model()`    | 加载模型和运行环境           |
| 更新 Block Size     | 同步模型及 KV Cache 相关信息 |

### 5.3 Executor 和 Worker 的初始化边界

```
              UniProcExecutor
                     |
                     | 创建
                     v
              WorkerWrapperBase
                     |
                     | 初始化调用
                     v
                   Worker
                     |
          +----------+----------+
          |          |          |
          v          v          v
      init_worker init_device load_model
          |          |          |
          +----------+----------+
                     |
                     v
                Worker 就绪
```

需要注意：

Executor 负责组织初始化流程，具体设备初始化和模型加载由 Worker 完成。

不要把 Worker 内部所有初始化细节都理解为 Executor 自己执行。

---

## 6. WorkerWrapperBase 与 RPC 调用

### 6.1 WorkerWrapperBase 的定位

`WorkerWrapperBase` 是 Executor 与 Worker 之间的重要包装层。

主要作用：

1. 包装 Worker 对象。
2. 提供 Worker 初始化入口。
3. 提供 Worker 方法调用入口。
4. 处理调用参数和执行上下文。
5. 将 Worker 方法执行结果返回给 Executor。

### 6.2 调用关系

```
+------------------------+
|     UniProcExecutor    |
+-----------+------------+
            |
            | collective_rpc()
            v
+------------------------+
|   WorkerWrapperBase    |
|                        |
|   execute_method()     |
+-----------+------------+
            |
            | 调用指定方法
            v
+------------------------+
|         Worker         |
|                        |
|   execute_model()      |
|   init_device()        |
|   load_model()         |
+-----------+------------+
            |
            v
+------------------------+
|      ModelRunner       |
+------------------------+
```

### 6.3 RPC 调用时序

```
 Scheduler       Executor       WorkerWrapper       Worker
     |               |                |                |
     | collective    |                |                |
     | operation     |                |                |
     |-------------->|                |                |
     |               | execute_method |                |
     |               |--------------->|                |
     |               |                | invoke method  |
     |               |                |--------------->|
     |               |                |                |
     |               |                |    execute     |
     |               |                |<---------------|
     |               |     Result     |                |
     |               |<---------------|                |
     |    Result     |                |                |
     |<--------------|                |                |
```

---

## 7. 同步执行与异步执行

### 7.1 同步调用

同步调用需要等待 Worker 返回结果。

```
        Executor
            |
            | collective_rpc()
            v
          Worker
            |
            | 执行方法
            |
            v
          Result
            |
            v
        Executor
            |
            | 返回结果
            v
        Scheduler
```

### 7.2 非阻塞调用

`UniProcExecutor` 支持 `non_block=True`。

当 Worker 返回 `AsyncModelRunnerOutput` 时，Executor 会将其包装成 `AsyncOutputFuture`。

```
              Executor
                  |
                  | collective_rpc(non_block=True)
                  v
                Worker
                  |
                  | execute_model()
                  v
         AsyncModelRunnerOutput
                  |
                  v
              Executor
                  |
                  | 包装异步输出
                  v
          AsyncOutputFuture
                  |
                  | 返回 Future
                  v
              上层调用者
                  |
                  | get_output()
                  v
             最终输出结果
```

### 7.3 AsyncOutputFuture

`AsyncOutputFuture` 的作用是将异步模型输出适配为可以延迟获取结果的 Future 封装。

| 项目   | 说明                          |
| ---- | --------------------------- |
| 输入   | `AsyncModelRunnerOutput`    |
| 输出   | `AsyncOutputFuture`         |
| 作用   | 延迟获取异步执行结果                  |
| 典型接口 | `get_output()`、`is_ready()` |
| 使用场景 | 非阻塞模型执行                     |

### 7.4 非阻塞调用的关键区别

需要特别注意：

1. `non_block=True` 不等于 Executor 一定创建了后台线程。
2. 只有 Worker 返回 `AsyncModelRunnerOutput` 时，才会使用 `AsyncOutputFuture` 适配。
3. 如果返回普通结果，可以包装成已经完成的 Future。
4. `supports_async_scheduling=True` 是执行能力声明，不代表存在后台线程。

### 7.5 UniProcExecutor 的返回值差异

抽象接口与具体实现的返回值需要分开理解。

| 位置                                             | 返回形式                            |
| ---------------------------------------------- | ------------------------------- |
| 抽象 `Executor.collective_rpc()`                 | 通常以多个 Worker 结果列表为抽象形式          |
| `Executor.execute_model()` / `sample_tokens()` | 抽象实现通常从 RPC 结果列表中取对应结果          |
| `UniProcExecutor.collective_rpc()`             | 可以通过 `single_value=True` 返回单个结果 |
| `UniProcExecutor` 非阻塞路径                        | 可能返回 `AsyncOutputFuture`        |

因此，不能简单认为所有 Executor 的 RPC 返回形式完全相同。

---

## 8. ExecutorWithExternalLauncher

### 8.1 设计目的

`ExecutorWithExternalLauncher` 用于适配外部启动器管理分布式进程的场景。

例如使用 `torchrun` 启动多个进程：

```
torchrun \
    --nproc-per-node=2 \
    --master-addr=127.0.0.1 \
    --master-port=29500 \
    ...
```

与 vLLM 自己管理多进程生命周期不同，外部 Launcher 已经负责启动各个进程。

Executor 需要适配这些已经存在的进程和分布式环境。

### 8.2 类继承关系

```
              +-------------------+
              |     Executor      |
              |    Abstract       |
              +---------+---------+
                        |
                        v
              +-------------------+
              |  UniProcExecutor  |
              +---------+---------+
                        |
                        | inherits
                        v
       +----------------------------------+
       | ExecutorWithExternalLauncher     |
       +----------------------------------+
```

### 8.3 进程架构

```
                 External Launcher
                     (torchrun)
                          |
             +------------+------------+
             |            |            |
             v            v            v
        Process 0     Process 1     Process 2
         Rank 0        Rank 1        Rank 2
             |            |            |
             v            v            v
        +---------+  +---------+  +---------+
        | Engine  |  | Engine  |  | Engine  |
        +----+----+  +----+----+  +----+----+
             |            |            |
             v            v            v
        +---------+  +---------+  +---------+
        |Executor |  |Executor |  |Executor |
        +----+----+  +----+----+  +----+----+
             |            |            |
             v            v            v
        +---------+  +---------+  +---------+
        | Worker  |  | Worker  |  | Worker  |
        +----+----+  +----+----+  +----+----+
             |            |            |
             +------------+------------+
                          |
                 Distributed Group
                   NCCL / Gloo
```

这里表达的是外部启动器启动多个进程、每个进程内运行对应执行组件的基本关系。实际 Engine、Executor 和 Worker 的组合仍需结合具体运行模式判断。

### 8.4 分布式环境变量

外部 Launcher 通常为各进程设置分布式环境变量。

| 环境变量          | 作用           |
| ------------- | ------------ |
| `RANK`        | 当前进程的全局 Rank |
| `LOCAL_RANK`  | 当前节点内部的 Rank |
| `MASTER_ADDR` | 分布式主节点地址     |
| `MASTER_PORT` | 分布式主节点端口     |
| `WORLD_SIZE`  | 全局进程数量       |

使用 `env://` 初始化分布式通信时，进程通过这些环境变量获取初始化信息。

### 8.5 为什么使用 External Launcher？

主要原因：

- 复用外部集群和进程管理系统。
- 兼容 torchrun 等已有分布式启动方式。
- 避免 Executor 重复管理进程生命周期。
- 适配外部环境中的 Rank 和通信初始化。

---

## 9. 分布式协调与 Rank

### 9.1 Rank、Local Rank、RPC Rank

这几个概念容易混淆。

| 名称           | 含义                  | 典型用途         |
| ------------ | ------------------- | ------------ |
| `rank`       | 分布式进程组中的全局 Rank     | 标识全局进程       |
| `local_rank` | 当前节点内部的 Rank        | 选择本地 GPU     |
| `rpc_rank`   | RPC 调用上下文中的 Rank 概念 | 选择或标识 RPC 目标 |

示例：

```
                 Distributed World
                       |
          +------------+------------+
          |                         |
          v                         v
       Node 0                    Node 1
          |                         |
     +----+----+               +----+----+
     |         |               |         |
     v         v               v         v
  Process 0 Process 1        Process 2 Process 3
  rank=0    rank=1           rank=2    rank=3
  local=0   local=1          local=0   local=1
```

注意：

- `rank` 是全局编号。
- `local_rank` 是节点内编号。
- `rpc_rank` 不能直接等同于分布式 `rank`。
- 在 `UniProcExecutor` 中，默认 Rank 和 Local Rank 的处理方式需要结合具体初始化代码分析。

### 9.2 Distributed Process Group

分布式执行时，多个进程需要建立通信组。

```
             Distributed Process Group
                         |
        +----------------+----------------+
        |                |                |
        v                v                v
     Rank 0           Rank 1           Rank 2
        |                |                |
        +----------------+----------------+
                         |
                  Collective Ops
                         |
             +-----------+-----------+
             |           |           |
             v           v           v
          AllReduce   AllGather   Broadcast
```

常见通信操作：

| 操作        | 作用                     |
| --------- | ---------------------- |
| AllReduce | 多个 Rank 规约计算，并同步结果     |
| AllGather | 收集多个 Rank 的数据          |
| Broadcast | 将一个 Rank 的数据广播到其他 Rank |
| Reduce    | 将多个 Rank 的数据规约到指定 Rank |

具体使用哪些通信操作，应结合对应执行路径和 Process Group 实现确认。

---

## 10. TCPStore 与 FileStore

分布式初始化需要在多个进程之间交换初始化信息。

常见 Store：

| 类型        | 作用               |
| --------- | ---------------- |
| TCPStore  | 通过 TCP 提供分布式键值同步 |
| FileStore | 通过共享文件进行初始化信息交换  |

它们主要用于分布式初始化和进程间协调，不是模型推理的大规模数据传输通道。

可以理解为：

```
       Process 0       Process 1       Process 2
           |               |               |
           +---------------+---------------+
                           |
                           v
                    Distributed Store
                           |
                  +--------+--------+
                  |                 |
                  v                 v
                TCPStore         FileStore
```

---

## 11. determine_available_memory：显存协调

### 11.1 为什么需要协调显存？

分布式模型执行中，不同 Rank 所在 GPU 的可用显存可能不同。

例如：

```
Rank 0:  可用显存 18 GB
Rank 1:  可用显存 16 GB
Rank 2:  可用显存 20 GB
Rank 3:  可用显存 17 GB
```

如果每个 Rank 独立按照本地最大可用显存规划 KV Cache，可能导致不同 Rank 的资源规划不一致。

因此需要获得所有 Rank 中最小的可用显存。

### 11.2 执行流程

```
          Rank 0       Rank 1       Rank 2       Rank 3
             |            |            |            |
             v            v            v            v
         查询显存     查询显存     查询显存     查询显存
             |            |            |            |
             v            v            v            v
          18 GB        16 GB        20 GB        17 GB
             |            |            |            |
             +------------+------------+------------+
                          |
                          v
                CPU Process Group
                          |
                          v
                   AllReduce MIN
                          |
                          v
                      16 GB
                          |
             +------------+------------+
             |            |            |
             v            v            v
          Rank 0       Rank 1       Rank 2 ...
          16 GB        16 GB        16 GB
```

### 11.3 数学表达

[\
M\_{\text{global}} = \min(M_0,M_1,\ldots,M\_{N-1})\
]

其中：

- (M_i)：第 (i) 个 Rank 的可用显存。
- (N)：分布式进程数量。
- (M\_{\text{global}})：统一使用的显存规划上限。

当前实现中，该协调过程使用 CPU Process Group 执行 MIN AllReduce。

核心目的是保证所有 Rank 的资源规划一致，避免某个 Rank 成为显存瓶颈。

---

## 12. Executor 与 Worker 的职责边界

```
+--------------------------------------------------+
|                    Executor                      |
|                                                  |
|  1. Worker 生命周期管理                         |
|  2. RPC 调用分发                                |
|  3. 分布式执行协调                              |
|  4. 资源规划和管理                              |
|  5. 执行后端管理                                |
+-------------------------+------------------------+
                          |
                          | RPC / Method Invocation
                          v
+--------------------------------------------------+
|                     Worker                       |
|                                                  |
|  1. Device 初始化                               |
|  2. Distributed 环境初始化                      |
|  3. Model 加载                                  |
|  4. ModelRunner 管理                            |
+-------------------------+------------------------+
                          |
                          v
+--------------------------------------------------+
|                   ModelRunner                    |
|                                                  |
|  1. Input Preparation                            |
|  2. Model Forward                                |
|  3. Sampling                                     |
|  4. Output Processing                            |
+-------------------------+------------------------+
                          |
                          v
+--------------------------------------------------+
|                      Model                       |
+--------------------------------------------------+
```

### 12.1 职责对比

| 对比维度      | Executor | Worker | ModelRunner |
| --------- | -------- | ------ | ----------- |
| 核心定位      | 执行管理     | 执行环境   | 模型运行        |
| Worker 创建 | 负责组织     | 不负责    | 不负责         |
| 设备初始化     | 组织调用     | 执行     | 使用设备        |
| 模型加载      | 组织调用     | 执行或管理  | 使用模型        |
| Forward   | 发起调用     | 转发或组织  | 执行          |
| Sampling  | 发起调用     | 转发或组织  | 执行          |
| 分布式协调     | 负责整体协调   | 参与通信   | 通常不负责整体协调   |

---

## 13. 关键方法速查

| 方法                             | 所属组件              | 主要作用              |
| ------------------------------ | ----------------- | ----------------- |
| `get_class()`                  | Executor          | 选择执行后端            |
| `collective_rpc()`             | Executor          | 分发 Worker 方法调用    |
| `execute_model()`              | Executor / Worker | 执行模型 Forward      |
| `sample_tokens()`              | Executor / Worker | 执行 Token Sampling |
| `determine_available_memory()` | Executor          | 确定可用显存            |
| `check_health()`               | Executor          | 健康检查              |
| `shutdown()`                   | Executor          | 关闭执行后端            |
| `init_worker()`                | Worker            | 初始化 Worker        |
| `init_device()`                | Worker            | 初始化计算设备           |
| `load_model()`                 | Worker            | 加载模型              |
| `get_output()`                 | AsyncOutputFuture | 获取异步输出            |
| `is_ready()`                   | AsyncOutputFuture | 检查异步输出是否就绪        |

---

## 14. 源码阅读路线与断点建议

### 14.1 推荐阅读顺序

```
       abstract.py
           |
           | 理解抽象接口
           v
       get_class()
           |
           | 理解后端选择
           v
    UniProcExecutor.__init__()
           |
           | 理解 Worker 创建
           v
    WorkerWrapperBase
           |
           | 理解 Worker 包装和调用
           v
      collective_rpc()
           |
           | 理解 RPC 分发
           v
      execute_model()
           |
           | 理解模型执行调用
           v
    AsyncOutputFuture
           |
           | 理解异步结果适配
           v
ExecutorWithExternalLauncher
           |
           | 理解外部启动器模式
           v
MultiprocExecutor / RayExecutor
             |
             v
       对比不同后端
```

### 14.2 推荐断点

| 断点位置                           | 观察重点                        |
| ------------------------------ | --------------------------- |
| `Executor.get_class()`         | 如何选择 Executor 后端            |
| `UniProcExecutor.__init__()`   | Worker 如何创建                 |
| `init_worker()`                | Worker 如何初始化                |
| `init_device()`                | GPU 如何初始化                   |
| `load_model()`                 | 模型如何加载                      |
| `collective_rpc()`             | RPC 如何调用和返回                 |
| `execute_model()`              | SchedulerOutput 如何传给 Worker |
| `sample_tokens()`              | Sampling 如何触发               |
| `AsyncOutputFuture`            | 异步结果如何适配                    |
| `determine_available_memory()` | 多 Rank 显存如何协调               |

### 14.3 建议重点跟踪的变量

| 变量                        | 观察目的                  |
| ------------------------- | --------------------- |
| `workers`                 | Executor 管理了哪些 Worker |
| `rank`                    | 当前进程的分布式 Rank         |
| `local_rank`              | 当前进程对应的本地设备编号         |
| `distributed_init_method` | 分布式初始化地址              |
| `device_config`           | Worker 设备配置           |
| `non_block`               | 当前 RPC 是否采用非阻塞路径      |
| `single_value`            | 是否以单值形式返回结果           |

---

## 15. 阅读 Executor 后需要掌握的问题

建议通过源码断点回答以下问题，作为源码学习的检查清单：

### 架构与类

- Executor 在 Engine、Scheduler、Worker 之间处于什么位置？
- Executor 与 Worker 的职责边界是什么？
- `Executor` 为什么设计为抽象类？
- `UniProcExecutor` 和 `ExecutorWithExternalLauncher` 有什么关系？
- 不同 Executor 后端分别适用于什么场景？

### 初始化与执行

- Executor 如何创建 Worker？
- `init_worker()`、`init_device()`、`load_model()` 分别做什么？
- `collective_rpc()` 如何找到并调用 Worker 方法？
- `execute_model()` 如何把 SchedulerOutput 传给 Worker？
- Worker 如何将执行结果返回给 Executor？

### 异步执行

- `non_block=True` 改变了什么？
- `AsyncModelRunnerOutput` 为什么需要适配？
- `AsyncOutputFuture` 如何获取最终结果？
- 非阻塞调用是否一定创建后台线程？

### 分布式执行

- `rank` 和 `local_rank` 有什么区别？
- 外部 Launcher 如何提供分布式环境？
- TCPStore 和 FileStore 有什么作用？
- 为什么可用显存需要跨 Rank 取 MIN？
- Executor 如何适配多进程执行？

---

## 16. 核心总结

Executor 本质上是 vLLM 的执行后端抽象层。

它解决的核心问题不是“如何调度请求”，而是：

**如何把 Scheduler 的调度结果交给一个或多个 Worker 执行，并管理执行过程中的进程、设备、通信和资源。**

需要重点掌握四条主线：

| 主线          | 核心内容                                                   |
| ----------- | ------------------------------------------------------ |
| 后端选择        | `get_class()`、UniProc、MP、Ray、External Launcher         |
| Worker 生命周期 | Worker 创建、设备初始化、模型加载                                   |
| 执行调用        | `collective_rpc()`、`execute_model()`、`sample_tokens()` |
| 分布式协调       | Rank、Process Group、Store、AllReduce                     |

最终形成以下认知：

```
                 Scheduler
                     |
                     | 决定执行什么
                     v
                  Executor
                     |
                     | 组织和分发执行
                     v
                   Worker
                     |
                     | 提供设备和模型运行环境
                     v
                ModelRunner
                     |
                     | 执行 Forward / Sampling
                     v
                   Model
                     |
                     v
                 Execution Result
                     |
                     v
                  Scheduler
```

**一句话记忆：Scheduler 管调度，Executor 管执行组织，Worker 管运行环境，ModelRunner 管模型计算。**

校正 Executor 类图和职责统一 collective_rpc 返回语义重画真正的多进程通信架构
