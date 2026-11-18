下面整理为精简版 GitHub Markdown 文档，保留核心架构、类关系、关键接口、初始化流程和推理执行流程。所有图均使用 GitHub 支持的 Mermaid。

# vLLM WorkerBase / WorkerWrapperBase 源码分析

## 1. 核心定位

`WorkerBase` 和 `WorkerWrapperBase` 是 vLLM V1 执行层的重要组成部分，负责统一 Worker 接口、管理 Worker 生命周期，以及将 Executor 的执行请求转发给具体 Worker。

核心职责：

| 类                   | 职责                            |
| ------------------- | ----------------------------- |
| `WorkerBase`        | 定义统一 Worker 接口，屏蔽不同硬件实现差异     |
| `WorkerWrapperBase` | 管理 Worker 初始化、环境注入、多模态缓存和调用代理 |
| 具体 Worker           | 实现设备初始化、模型加载和推理执行             |
| `ModelRunner`       | 负责模型前向计算和推理执行                 |

核心关系：

- Scheduler：决定本轮执行哪些请求。
- Executor：组织和分发执行任务。
- Worker：管理设备和模型执行环境。
- ModelRunner：执行具体模型计算。

## 2. 整体架构

```
                  EngineCore
                      |
                  Scheduler
                      |
                SchedulerOutput
                      |
                   Executor
                      |
               WorkerWrapperBase
                      |
                  WorkerBase
                      |
                具体 Worker
                      |
                  ModelRunner
                      |
                 Model / GPU
```

## 3. 核心类关系

```
classDiagram
    class WorkerBase {
        +VllmConfig vllm_config
        +torch.device device
        +nn.Module model_runner
        +get_kv_cache_spec()
        +get_supported_kv_cache_layouts()
        +set_kv_cache_layout()
        +init_device()
        +load_model()
        +compile_or_warm_up_model()
        +execute_model()
        +sample_tokens()
        +get_model()
        +add_lora()
        +remove_lora()
        +shutdown()
    }

    class WorkerWrapperBase {
        +int rpc_rank
        +int global_rank
        +WorkerBase worker
        +update_environment_variables()
        +init_worker()
        +initialize_from_config()
        +init_device()
        +execute_model()
        +reset_mm_cache()
        +shutdown()
        +__getattr__()
    }

    WorkerWrapperBase --> WorkerBase : 包装和代理
    WorkerBase --> ModelRunner : 使用
```

## 4. WorkerBase：统一 Worker 接口

### 4.1 初始化

```
def __init__(
    self,
    vllm_config,
    local_rank,
    rank,
    distributed_init_method,
    is_driver_worker=False,
):
```

保存模型、设备、调度、KV Cache、并行等配置。

| 参数                        | 含义             |
| ------------------------- | -------------- |
| `vllm_config`             | 完整运行配置         |
| `local_rank`              | 本机设备逻辑编号       |
| `rank`                    | 分布式通信组 Rank    |
| `distributed_init_method` | 分布式初始化方式       |
| `is_driver_worker`        | 是否承担 Driver 职责 |

注意区分：

- `local_rank`：本机设备编号。
- `rank`：分布式通信组编号。
- `rpc_rank`：Executor 内部 Worker 编号。

### 4.2 KV Cache 接口

| 方法                                 | 作用                              |
| ---------------------------------- | ------------------------------- |
| `get_kv_cache_spec()`              | 获取 KV Cache 规格                  |
| `get_supported_kv_cache_layouts()` | 获取 Attention Backend 支持的 Layout |
| `set_kv_cache_layout()`            | 接收 Engine Core 确定的 Layout       |
| `get_cache_block_size_bytes()`     | 获取单个 KV Cache Block 大小          |

KV Cache 配置流程：

```
Worker
  |
  +-- get_kv_cache_spec()
  |
  +-- get_supported_kv_cache_layouts()
            |
            v
        Engine Core
            |
       规划 KV Cache
            |
       选择 Layout
            |
            v
Worker.set_kv_cache_layout()
            |
            v
initialize_from_config()
```

核心设计：Worker 提供规格和能力，Engine Core 负责协调规划，Worker 根据最终配置初始化。

### 4.3 模型生命周期接口

| 方法                           | 作用       |
| ---------------------------- | -------- |
| `init_device()`              | 初始化设备环境  |
| `load_model()`               | 加载模型权重   |
| `compile_or_warm_up_model()` | 编译或预热模型  |
| `get_model()`                | 获取模型实例   |
| `check_health()`             | 健康检查     |
| `synchronize_device()`       | 等待设备任务完成 |
| `shutdown()`                 | 释放资源     |

三个关键阶段：

```
init_device()
     |
     v
load_model()
     |
     v
compile_or_warm_up_model()
```

具体执行顺序以实际 Worker 启动流程为准。

### 4.4 模型执行接口

```
execute_model(scheduler_output)
sample_tokens(grammar_output)
```

`execute_model()` 接收 SchedulerOutput，执行模型计算。

返回值：

| 返回值                      | 含义                        |
| ------------------------ | ------------------------- |
| `ModelRunnerOutput`      | 同步执行结果                    |
| `AsyncModelRunnerOutput` | 异步执行结果                    |
| `None`                   | 需要紧接着调用 `sample_tokens()` |

执行流程：

```
SchedulerOutput
      |
      v
execute_model()
      |
      v
模型前向计算
      |
      v
返回结果判断
      |
      +---- ModelRunnerOutput
      |
      +---- AsyncModelRunnerOutput
      |
      +---- None
               |
               v
          sample_tokens()
               |
               v
           最终输出
```

### 4.5 其他接口

| 接口                                | 作用                      |
| --------------------------------- | ----------------------- |
| `add_lora()`                      | 加载 LoRA                 |
| `remove_lora()`                   | 卸载 LoRA                 |
| `pin_lora()`                      | 固定 LoRA，避免驱逐            |
| `list_loras()`                    | 查询 LoRA                 |
| `reset_mm_cache()`                | 重置多模态缓存                 |
| `supports_draft_weight_updates()` | 判断是否支持 Draft Model 权重更新 |

## 5. WorkerWrapperBase：生命周期管理与代理

### 5.1 Rank 设计

```
self.rpc_rank = rpc_rank
self.global_rank = (
    self.rpc_rank if global_rank is None else global_rank
)
```

RPC Rank 和 Global Rank 不一定相同。

例如两个 Executor 协同执行 TP=2：

| Executor   | RPC Rank | Global Rank |
| ---------- | -------- | ----------- |
| Executor 0 | 0        | 0           |
| Executor 1 | 0        | 1           |

RPC Rank 用于定位当前 Executor 内的 Worker；Global Rank 用于分布式 Worker 配置索引。

### 5.2 init_worker：延迟初始化

核心代码：

```
def init_worker(self, all_kwargs):
    kwargs = all_kwargs[self.rpc_rank]

    self.vllm_config = kwargs["vllm_config"]

    load_general_plugins()

    worker_class = resolve_obj_by_qualname(
        self.vllm_config.parallel_config.worker_cls
    )

    # 动态注入 Worker Extension
    # 设置物理 GPU ID
    # 初始化多模态 Receiver Cache

    with set_current_vllm_config(self.vllm_config):
        self.worker = worker_class(**kwargs)
```

主要步骤：

1. 根据 RPC Rank 获取 Worker 初始化参数。
2. 设置当前 VllmConfig。
3. 加载通用插件。
4. 根据类限定名动态加载 Worker。
5. 动态注入 Worker Extension。
6. 设置物理 GPU 分配信息。
7. 初始化多模态 Receiver Cache。
8. 实例化具体 Worker。

动态扩展机制：

```
worker_class.__bases__ = (
    worker_class.__bases__ + (worker_extension_cls,)
)
```

通过修改类的继承关系，将扩展类动态注入 Worker，使其具备额外的 RPC 扩展方法。

### 5.3 设备与配置初始化

```
def initialize_from_config(self, kv_cache_configs):
    kv_cache_config = kv_cache_configs[self.global_rank]
    self.worker.initialize_from_config(kv_cache_config)

def init_device(self):
    self.worker.init_device()
```

Wrapper 主要负责：

- 根据 Global Rank 选择 KV Cache 配置。
- 设置当前配置上下文。
- 将初始化任务委托给具体 Worker。

注意：`initialize_from_config()` 没有在当前展示的 WorkerBase 中声明，需要结合具体 Worker 实现理解。

### 5.4 多模态缓存处理

```
def _apply_mm_cache(self, scheduler_output):
    mm_cache = self.mm_receiver_cache
    if mm_cache is None:
        return

    for req_data in scheduler_output.scheduled_new_reqs:
        req_data.mm_features = (
            mm_cache.get_and_update_features(
                req_data.mm_features
            )
        )
```

作用：

- 遍历本轮新调度请求。
- 获取请求的多模态特征。
- 从 Receiver Cache 获取或更新特征。
- 将处理后的特征写回请求。

用于多模态数据的缓存复用和跨 Worker 数据传递。

### 5.5 execute_model：执行代理

```
def execute_model(self, scheduler_output):
    self._apply_mm_cache(scheduler_output)
    return self.worker.execute_model(scheduler_output)
```

Wrapper 不执行模型计算，只在执行前增加多模态缓存处理，然后调用具体 Worker。

### 5.6 **getattr**：透明代理

```
def __getattr__(self, attr):
    return getattr(self.worker, attr)
```

当 Wrapper 没有定义某个方法时，自动转发到内部 Worker。

例如：

```
wrapper.add_lora(request)
wrapper.get_model()
wrapper.check_health()
```

无需在 Wrapper 中重复实现这些接口。

注意：Wrapper 显式定义的方法优先执行自身逻辑，不会通过 `__getattr__` 转发。

### 5.7 资源管理

```
def reset_mm_cache(self):
    if self.mm_receiver_cache is not None:
        self.mm_receiver_cache.clear_cache()

    self.worker.reset_mm_cache()
```

清理两处多模态缓存：

1. Wrapper 的 Receiver Cache。
2. ModelRunner 内部的 MM Cache。

```
def shutdown(self):
    if self.worker is not None:
        self.worker.shutdown()
```

最终委托具体 Worker 释放资源。

## 6. 完整工作流程

### 6.1 Worker 初始化

```
flowchart TD
    A["创建 WorkerWrapperBase"] --> B["设置 Rank"]
    B --> C["更新环境变量"]
    C --> D["init_worker"]
    D --> E["加载插件和 Worker 类"]
    E --> F["注入 Worker Extension"]
    F --> G["设置 GPU 映射"]
    G --> H["初始化 MM Receiver Cache"]
    H --> I["实例化 Worker"]
    I --> J["init_device"]
    J --> K["Worker 设备初始化完成"]
```

### 6.2 推理执行

```
sequenceDiagram
    participant S as Scheduler
    participant E as Executor
    participant W as WorkerWrapperBase
    participant R as Worker
    participant M as ModelRunner

    S->>E: SchedulerOutput
    E->>W: execute_model()
    W->>W: _apply_mm_cache()
    W->>R: execute_model()
    R->>M: 模型计算
    M-->>R: 执行结果
    R-->>W: ModelRunnerOutput
    W-->>E: 返回结果
```

## 7. 核心设计总结

| 设计思想              | 解决的问题                  |
| ----------------- | ---------------------- |
| WorkerBase 抽象     | 统一不同硬件的执行接口            |
| WorkerWrapperBase | 管理初始化和公共执行逻辑           |
| 延迟实例化             | 先完成环境准备，再创建 Worker     |
| 动态类加载             | 支持不同 Worker 实现         |
| 动态继承扩展            | 增加 Worker 扩展能力         |
| Rank 解耦           | 区分 Executor 内部编号与分布式编号 |
| 配置上下文             | 保证初始化期间能够访问 VllmConfig |
| 多模态缓存             | 减少重复处理和支持数据复用          |
| 代理转发              | 减少 Wrapper 重复实现        |

## 8. 后续源码阅读

建议按照以下顺序继续阅读：

| 优先级 | 模块                                     | 重点                       |
| --- | -------------------------------------- | ------------------------ |
| P0  | `vllm/v1/executor/uniproc_executor.py` | 单进程 Executor 如何调用 Worker |
| P0  | `vllm/v1/worker/gpu_worker.py`         | GPU 初始化、模型加载             |
| P0  | `vllm/v1/worker/gpu_model_runner.py`   | 模型执行、KV Cache、Sampling   |
| P1  | `vllm/v1/core/sched/output.py`         | SchedulerOutput 数据结构     |
| P1  | `vllm/v1/kv_cache_interface.py`        | KVCacheSpec 与缓存配置        |

**最重要的下一条调用链：**

```
Scheduler
    |
    v
SchedulerOutput
    |
    v
Executor
    |
    v
WorkerWrapperBase.execute_model()
    |
    v
Worker.execute_model()
    |
    v
ModelRunner.execute_model()
    |
    v
Model Forward
    |
    v
ModelRunnerOutput
```

通过这条链路，可以串联 Scheduler、Executor、Worker、ModelRunner 四个核心模块，理解 vLLM 从请求调度到模型执行的完整过程。

补充 Worker 与 ModelRunner 边界修正初始化与执行流程关系
