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

# **vLLM GPU Worker 源码解析**

源码：`vllm/v1/worker/gpu_worker.py`

## **1. 核心定位**

`Worker` 是 vLLM 执行层的 GPU 工作进程，负责 GPU 设备管理、分布式初始化、模型加载、显存管理、KV Cache 初始化以及模型执行。

**核心职责：**

| 职责           | 主要方法                           | 作用                      |
| ------------ | ------------------------------ | ----------------------- |
| 设备初始化        | `init_device()`                | 初始化 CUDA、设置 GPU、建立分布式环境 |
| 模型加载         | `load_model()`                 | 创建并加载模型                 |
| 显存分析         | `determine_available_memory()` | 计算 KV Cache 可使用的显存      |
| KV Cache 初始化 | `initialize_from_config()`     | 根据配置初始化 KV Cache        |
| 模型预热         | `compile_or_warm_up_model()`   | 编译模型、预热算子和 CUDA Graph   |
| 模型执行         | `execute_model()`              | 调用 ModelRunner 执行推理     |
| Token 采样     | `sample_tokens()`              | 执行 Token 采样             |
| 资源管理         | `sleep()` / `wake_up()`        | 管理 GPU 资源和模型状态          |

## **2. 核心架构**

\#chatgpt-mermaid-\_r_6vc\_{font-family:-apple-system-body,ui-sans-serif,-apple-system,system-ui,"Segoe UI",Helvetica,"Apple Color Emoji",Arial,sans-serif,"Segoe UI Emoji","Segoe UI Symbol";font-size:16px;fill:rgb(13, 13, 13);}@keyframes edge-animation-frame{from{stroke-dashoffset:0;}}@keyframes dash{to{stroke-dashoffset:0;}}#chatgpt-mermaid-\_r_6vc\_ .edge-animation-slow{stroke-dasharray:9,5!important;stroke-dashoffset:900;animation:dash 50s linear infinite;stroke-linecap:round;}#chatgpt-mermaid-\_r_6vc\_ .edge-animation-fast{stroke-dasharray:9,5!important;stroke-dashoffset:900;animation:dash 20s linear infinite;stroke-linecap:round;}#chatgpt-mermaid-\_r_6vc\_ .error-icon{fill:rgb(243, 243, 243);}#chatgpt-mermaid-\_r_6vc\_ .error-text{fill:rgb(13, 13, 13);stroke:rgb(13, 13, 13);}#chatgpt-mermaid-\_r_6vc\_ .edge-thickness-normal{stroke-width:1px;}#chatgpt-mermaid-\_r_6vc\_ .edge-thickness-thick{stroke-width:3.5px;}#chatgpt-mermaid-\_r_6vc\_ .edge-pattern-solid{stroke-dasharray:0;}#chatgpt-mermaid-\_r_6vc\_ .edge-thickness-invisible{stroke-width:0;fill:none;}#chatgpt-mermaid-\_r_6vc\_ .edge-pattern-dashed{stroke-dasharray:3;}#chatgpt-mermaid-\_r_6vc\_ .edge-pattern-dotted{stroke-dasharray:2;}#chatgpt-mermaid-\_r_6vc\_ .marker{fill:rgb(143, 143, 143);stroke:rgb(143, 143, 143);}#chatgpt-mermaid-\_r_6vc\_ .marker.cross{stroke:rgb(143, 143, 143);}#chatgpt-mermaid-\_r_6vc\_ svg{font-family:-apple-system-body,ui-sans-serif,-apple-system,system-ui,"Segoe UI",Helvetica,"Apple Color Emoji",Arial,sans-serif,"Segoe UI Emoji","Segoe UI Symbol";font-size:16px;}#chatgpt-mermaid-\_r_6vc\_ p{margin:0;}#chatgpt-mermaid-\_r_6vc\_ g.classGroup text{fill:rgb(83, 154, 248);stroke:none;font-family:-apple-system-body,ui-sans-serif,-apple-system,system-ui,"Segoe UI",Helvetica,"Apple Color Emoji",Arial,sans-serif,"Segoe UI Emoji","Segoe UI Symbol";font-size:10px;}#chatgpt-mermaid-\_r_6vc\_ g.classGroup text .title{font-weight:bolder;}#chatgpt-mermaid-\_r_6vc\_ .cluster-label text{fill:rgb(13, 13, 13);}#chatgpt-mermaid-\_r_6vc\_ .cluster-label span{color:rgb(13, 13, 13);}#chatgpt-mermaid-\_r_6vc\_ .cluster-label span p{background-color:transparent;}#chatgpt-mermaid-\_r_6vc\_ .cluster rect{fill:rgb(243, 243, 243);stroke:rgba(0, 0, 0, 0.1);stroke-width:1px;}#chatgpt-mermaid-\_r_6vc\_ .cluster text{fill:rgb(13, 13, 13);}#chatgpt-mermaid-\_r_6vc\_ .cluster span{color:rgb(13, 13, 13);}#chatgpt-mermaid-\_r_6vc\_ .nodeLabel,#chatgpt-mermaid-\_r_6vc\_ .edgeLabel{color:rgb(13, 13, 13);}#chatgpt-mermaid-\_r_6vc\_ .noteLabel .nodeLabel,#chatgpt-mermaid-\_r_6vc\_ .noteLabel .edgeLabel{color:rgb(13, 13, 13);}#chatgpt-mermaid-\_r_6vc\_ .edgeLabel .label rect{fill:rgb(222, 234, 251);}#chatgpt-mermaid-\_r_6vc\_ .label text{fill:rgb(13, 13, 13);}#chatgpt-mermaid-\_r_6vc\_ .labelBkg{background:rgb(222, 234, 251);}#chatgpt-mermaid-\_r_6vc\_ .edgeLabel .label span{background:rgb(222, 234, 251);}#chatgpt-mermaid-\_r_6vc\_ .classTitle{font-weight:bolder;}#chatgpt-mermaid-\_r_6vc\_ .node rect,#chatgpt-mermaid-\_r_6vc\_ .node circle,#chatgpt-mermaid-\_r_6vc\_ .node ellipse,#chatgpt-mermaid-\_r_6vc\_ .node polygon,#chatgpt-mermaid-\_r_6vc\_ .node path{fill:rgb(222, 234, 251);stroke:rgb(83, 154, 248);stroke-width:1;}#chatgpt-mermaid-\_r_6vc\_ .divider{stroke:rgb(83, 154, 248);stroke-width:1;}#chatgpt-mermaid-\_r_6vc\_ g.clickable{cursor:pointer;}#chatgpt-mermaid-\_r_6vc\_ g.classGroup rect{fill:rgb(222, 234, 251);stroke:rgb(83, 154, 248);}#chatgpt-mermaid-\_r_6vc\_ g.classGroup line{stroke:rgb(83, 154, 248);stroke-width:1;}#chatgpt-mermaid-\_r_6vc\_ .classLabel .box{stroke:none;stroke-width:0;fill:rgb(222, 234, 251);opacity:0.5;}#chatgpt-mermaid-\_r_6vc\_ .classLabel .label{fill:rgb(83, 154, 248);font-size:10px;}#chatgpt-mermaid-\_r_6vc\_ .relation{stroke:rgb(143, 143, 143);stroke-width:1;fill:none;}#chatgpt-mermaid-\_r_6vc\_ .dashed-line{stroke-dasharray:3;}#chatgpt-mermaid-\_r_6vc\_ .dotted-line{stroke-dasharray:1 2;}#chatgpt-mermaid-\_r_6vc\_ [id$="-compositionStart"],#chatgpt-mermaid-\_r_6vc\_ .composition{fill:rgb(143, 143, 143)!important;stroke:rgb(143, 143, 143)!important;stroke-width:1;}#chatgpt-mermaid-\_r_6vc\_ [id$="-compositionEnd"],#chatgpt-mermaid-\_r_6vc\_ .composition{fill:rgb(143, 143, 143)!important;stroke:rgb(143, 143, 143)!important;stroke-width:1;}#chatgpt-mermaid-\_r_6vc\_ [id$="-dependencyStart"],#chatgpt-mermaid-\_r_6vc\_ .dependency{fill:rgb(143, 143, 143)!important;stroke:rgb(143, 143, 143)!important;stroke-width:1;}#chatgpt-mermaid-\_r_6vc\_ [id$="-dependencyEnd"],#chatgpt-mermaid-\_r_6vc\_ .dependency{fill:rgb(143, 143, 143)!important;stroke:rgb(143, 143, 143)!important;stroke-width:1;}#chatgpt-mermaid-\_r_6vc\_ [id$="-extensionStart"],#chatgpt-mermaid-\_r_6vc\_ .extension{fill:transparent!important;stroke:rgb(143, 143, 143)!important;stroke-width:1;}#chatgpt-mermaid-\_r_6vc\_ [id$="-extensionEnd"],#chatgpt-mermaid-\_r_6vc\_ .extension{fill:transparent!important;stroke:rgb(143, 143, 143)!important;stroke-width:1;}#chatgpt-mermaid-\_r_6vc\_ [id$="-aggregationStart"],#chatgpt-mermaid-\_r_6vc\_ .aggregation{fill:transparent!important;stroke:rgb(143, 143, 143)!important;stroke-width:1;}#chatgpt-mermaid-\_r_6vc\_ [id$="-aggregationEnd"],#chatgpt-mermaid-\_r_6vc\_ .aggregation{fill:transparent!important;stroke:rgb(143, 143, 143)!important;stroke-width:1;}#chatgpt-mermaid-\_r_6vc\_ [id$="-lollipopStart"],#chatgpt-mermaid-\_r_6vc\_ .lollipop{fill:rgb(222, 234, 251)!important;stroke:rgb(143, 143, 143)!important;stroke-width:1;}#chatgpt-mermaid-\_r_6vc\_ [id$="-lollipopEnd"],#chatgpt-mermaid-\_r_6vc\_ .lollipop{fill:rgb(222, 234, 251)!important;stroke:rgb(143, 143, 143)!important;stroke-width:1;}#chatgpt-mermaid-\_r_6vc\_ .edgeTerminals{font-size:11px;line-height:initial;}#chatgpt-mermaid-\_r_6vc\_ .classTitleText{text-anchor:middle;font-size:18px;fill:rgb(13, 13, 13);}#chatgpt-mermaid-\_r_6vc\_ .edgeLabel[data-look="neo"]{background-color:rgb(252, 252, 252);text-align:center;}#chatgpt-mermaid-\_r_6vc\_ .edgeLabel[data-look="neo"] p{background-color:rgb(252, 252, 252);}#chatgpt-mermaid-\_r_6vc\_ .edgeLabel[data-look="neo"] rect{opacity:0.5;background-color:rgb(252, 252, 252);fill:rgb(252, 252, 252);}#chatgpt-mermaid-\_r_6vc\_ .label-icon{display:inline-block;height:1em;overflow:visible;vertical-align:-0.125em;}#chatgpt-mermaid-\_r_6vc\_ .node .label-icon path{fill:currentColor;stroke:revert;stroke-width:revert;}#chatgpt-mermaid-\_r_6vc\_ .node .neo-node{stroke:rgb(83, 154, 248);}#chatgpt-mermaid-\_r_6vc\_ [data-look="neo"].node rect,#chatgpt-mermaid-\_r_6vc\_ [data-look="neo"].cluster rect,#chatgpt-mermaid-\_r_6vc\_ [data-look="neo"].node polygon{stroke:url(#chatgpt-mermaid-\_r_6vc\_-gradient);filter:drop-shadow( 1px 2px 2px rgba(185,185,185,1));}#chatgpt-mermaid-\_r_6vc\_ [data-look="neo"].swimlane.cluster rect{filter:none;}#chatgpt-mermaid-\_r_6vc\_ [data-look="neo"].node path{stroke:url(#chatgpt-mermaid-\_r_6vc\_-gradient);stroke-width:1px;}#chatgpt-mermaid-\_r_6vc\_ [data-look="neo"].node .outer-path{filter:drop-shadow( 1px 2px 2px rgba(185,185,185,1));}#chatgpt-mermaid-\_r_6vc\_ [data-look="neo"].node .neo-line path{stroke:rgb(83, 154, 248);filter:none;}#chatgpt-mermaid-\_r_6vc\_ [data-look="neo"].node circle{stroke:url(#chatgpt-mermaid-\_r_6vc\_-gradient);filter:drop-shadow( 1px 2px 2px rgba(185,185,185,1));}#chatgpt-mermaid-\_r_6vc\_ [data-look="neo"].node circle .state-start{fill:#000000;}#chatgpt-mermaid-\_r_6vc\_ [data-look="neo"].icon-shape .icon{fill:url(#chatgpt-mermaid-\_r_6vc\_-gradient);filter:drop-shadow( 1px 2px 2px rgba(185,185,185,1));}#chatgpt-mermaid-\_r_6vc\_ [data-look="neo"].icon-shape .icon-neo path{stroke:url(#chatgpt-mermaid-\_r_6vc\_-gradient);filter:drop-shadow( 1px 2px 2px rgba(185,185,185,1));}#chatgpt-mermaid-\_r_6vc\_ :root{--mermaid-font-family:-apple-system-body,ui-sans-serif,-apple-system,system-ui,"Segoe UI",Helvetica,"Apple Color Emoji",Arial,sans-serif,"Segoe UI Emoji","Segoe UI Symbol";}WorkerBase+init_device()+load_model()+execute_model()Worker-device-model_runner-init_device()-load_model()-determine_available_memory()-initialize_from_config()-compile_or_warm_up_model()+execute_model()+sample_tokens()+sleep()+wake_up()GPUModelRunner+load_model()+initialize_kv_cache()+execute_model()+sample_tokens()+profile_run()AsyncIntermediateTensors+tensorsWeightTransferEngine+transfer_weights()delegates executionweight transferpipeline data

**核心关系：**

- `Worker`：GPU 执行进程的管理入口。
- `GPUModelRunner`：真正负责模型加载、Forward、KV Cache 访问和采样的执行组件。
- `AsyncIntermediateTensors`：Pipeline Parallel 场景下用于传递中间计算结果。
- `WeightTransferEngine`：负责模型权重传输相关功能。

## **3. Worker 初始化流程**

\#chatgpt-mermaid-\_r_6vj\_{font-family:-apple-system-body,ui-sans-serif,-apple-system,system-ui,"Segoe UI",Helvetica,"Apple Color Emoji",Arial,sans-serif,"Segoe UI Emoji","Segoe UI Symbol";font-size:16px;fill:rgb(13, 13, 13);}@keyframes edge-animation-frame{from{stroke-dashoffset:0;}}@keyframes dash{to{stroke-dashoffset:0;}}#chatgpt-mermaid-\_r_6vj\_ .edge-animation-slow{stroke-dasharray:9,5!important;stroke-dashoffset:900;animation:dash 50s linear infinite;stroke-linecap:round;}#chatgpt-mermaid-\_r_6vj\_ .edge-animation-fast{stroke-dasharray:9,5!important;stroke-dashoffset:900;animation:dash 20s linear infinite;stroke-linecap:round;}#chatgpt-mermaid-\_r_6vj\_ .error-icon{fill:rgb(243, 243, 243);}#chatgpt-mermaid-\_r_6vj\_ .error-text{fill:rgb(13, 13, 13);stroke:rgb(13, 13, 13);}#chatgpt-mermaid-\_r_6vj\_ .edge-thickness-normal{stroke-width:1px;}#chatgpt-mermaid-\_r_6vj\_ .edge-thickness-thick{stroke-width:3.5px;}#chatgpt-mermaid-\_r_6vj\_ .edge-pattern-solid{stroke-dasharray:0;}#chatgpt-mermaid-\_r_6vj\_ .edge-thickness-invisible{stroke-width:0;fill:none;}#chatgpt-mermaid-\_r_6vj\_ .edge-pattern-dashed{stroke-dasharray:3;}#chatgpt-mermaid-\_r_6vj\_ .edge-pattern-dotted{stroke-dasharray:2;}#chatgpt-mermaid-\_r_6vj\_ .marker{fill:rgb(143, 143, 143);stroke:rgb(143, 143, 143);}#chatgpt-mermaid-\_r_6vj\_ .marker.cross{stroke:rgb(143, 143, 143);}#chatgpt-mermaid-\_r_6vj\_ svg{font-family:-apple-system-body,ui-sans-serif,-apple-system,system-ui,"Segoe UI",Helvetica,"Apple Color Emoji",Arial,sans-serif,"Segoe UI Emoji","Segoe UI Symbol";font-size:16px;}#chatgpt-mermaid-\_r_6vj\_ p{margin:0;}#chatgpt-mermaid-\_r_6vj\_ .label{font-family:-apple-system-body,ui-sans-serif,-apple-system,system-ui,"Segoe UI",Helvetica,"Apple Color Emoji",Arial,sans-serif,"Segoe UI Emoji","Segoe UI Symbol";color:rgb(13, 13, 13);}#chatgpt-mermaid-\_r_6vj\_ .cluster-label text{fill:rgb(13, 13, 13);}#chatgpt-mermaid-\_r_6vj\_ .cluster-label span{color:rgb(13, 13, 13);}#chatgpt-mermaid-\_r_6vj\_ .cluster-label span p{background-color:transparent;}#chatgpt-mermaid-\_r_6vj\_ .label text,#chatgpt-mermaid-\_r_6vj\_ span{fill:rgb(13, 13, 13);color:rgb(13, 13, 13);}#chatgpt-mermaid-\_r_6vj\_ .node rect,#chatgpt-mermaid-\_r_6vj\_ .node circle,#chatgpt-mermaid-\_r_6vj\_ .node ellipse,#chatgpt-mermaid-\_r_6vj\_ .node polygon,#chatgpt-mermaid-\_r_6vj\_ .node path{fill:rgb(222, 234, 251);stroke:rgb(83, 154, 248);stroke-width:1px;}#chatgpt-mermaid-\_r_6vj\_ .rough-node .label text,#chatgpt-mermaid-\_r_6vj\_ .node .label text,#chatgpt-mermaid-\_r_6vj\_ .image-shape .label,#chatgpt-mermaid-\_r_6vj\_ .icon-shape .label{text-anchor:middle;}#chatgpt-mermaid-\_r_6vj\_ .node .katex path{fill:#000;stroke:#000;stroke-width:1px;}#chatgpt-mermaid-\_r_6vj\_ .rough-node .label,#chatgpt-mermaid-\_r_6vj\_ .node .label,#chatgpt-mermaid-\_r_6vj\_ .image-shape .label,#chatgpt-mermaid-\_r_6vj\_ .icon-shape .label{text-align:center;}#chatgpt-mermaid-\_r_6vj\_ .node.clickable{cursor:pointer;}#chatgpt-mermaid-\_r_6vj\_ .root .anchor path{fill:rgb(143, 143, 143)!important;stroke-width:0;stroke:rgb(143, 143, 143);}#chatgpt-mermaid-\_r_6vj\_ .arrowheadPath{fill:rgb(143, 143, 143);}#chatgpt-mermaid-\_r_6vj\_ .edgePath .path{stroke:rgb(143, 143, 143);stroke-width:1px;}#chatgpt-mermaid-\_r_6vj\_ .flowchart-link{stroke:rgb(143, 143, 143);fill:none;}#chatgpt-mermaid-\_r_6vj\_ .edgeLabel{background-color:rgb(252, 252, 252);text-align:center;}#chatgpt-mermaid-\_r_6vj\_ .edgeLabel p{background-color:rgb(252, 252, 252);}#chatgpt-mermaid-\_r_6vj\_ .edgeLabel rect{opacity:0.5;background-color:rgb(252, 252, 252);fill:rgb(252, 252, 252);}#chatgpt-mermaid-\_r_6vj\_ .labelBkg{background-color:rgba(252, 252, 252, 0.5);}#chatgpt-mermaid-\_r_6vj\_ .cluster rect{fill:rgb(243, 243, 243);stroke:rgba(0, 0, 0, 0.1);stroke-width:1px;}#chatgpt-mermaid-\_r_6vj\_ .cluster text{fill:rgb(13, 13, 13);}#chatgpt-mermaid-\_r_6vj\_ .cluster span{color:rgb(13, 13, 13);}#chatgpt-mermaid-\_r_6vj\_ div.mermaidTooltip{position:absolute;text-align:center;max-width:200px;padding:2px;font-family:-apple-system-body,ui-sans-serif,-apple-system,system-ui,"Segoe UI",Helvetica,"Apple Color Emoji",Arial,sans-serif,"Segoe UI Emoji","Segoe UI Symbol";font-size:12px;background:rgb(243, 243, 243);border:1px solid rgba(0, 0, 0, 0.1);border-radius:2px;pointer-events:none;z-index:100;}#chatgpt-mermaid-\_r_6vj\_ .flowchartTitleText{text-anchor:middle;font-size:18px;fill:rgb(13, 13, 13);}#chatgpt-mermaid-\_r_6vj\_ rect.text{fill:none;stroke-width:0;}#chatgpt-mermaid-\_r_6vj\_ .icon-shape,#chatgpt-mermaid-\_r_6vj\_ .image-shape{background-color:rgb(252, 252, 252);text-align:center;}#chatgpt-mermaid-\_r_6vj\_ .icon-shape p,#chatgpt-mermaid-\_r_6vj\_ .image-shape p{background-color:rgb(252, 252, 252);padding:2px;}#chatgpt-mermaid-\_r_6vj\_ .icon-shape .label rect,#chatgpt-mermaid-\_r_6vj\_ .image-shape .label rect{opacity:0.5;background-color:rgb(252, 252, 252);fill:rgb(252, 252, 252);}#chatgpt-mermaid-\_r_6vj\_ .label-icon{display:inline-block;height:1em;overflow:visible;vertical-align:-0.125em;}#chatgpt-mermaid-\_r_6vj\_ .node .label-icon path{fill:currentColor;stroke:revert;stroke-width:revert;}#chatgpt-mermaid-\_r_6vj\_ .node .neo-node{stroke:rgb(83, 154, 248);}#chatgpt-mermaid-\_r_6vj\_ [data-look="neo"].node rect,#chatgpt-mermaid-\_r_6vj\_ [data-look="neo"].cluster rect,#chatgpt-mermaid-\_r_6vj\_ [data-look="neo"].node polygon{stroke:url(#chatgpt-mermaid-\_r_6vj\_-gradient);filter:drop-shadow( 1px 2px 2px rgba(185,185,185,1));}#chatgpt-mermaid-\_r_6vj\_ [data-look="neo"].swimlane.cluster rect{filter:none;}#chatgpt-mermaid-\_r_6vj\_ [data-look="neo"].node path{stroke:url(#chatgpt-mermaid-\_r_6vj\_-gradient);stroke-width:1px;}#chatgpt-mermaid-\_r_6vj\_ [data-look="neo"].node .outer-path{filter:drop-shadow( 1px 2px 2px rgba(185,185,185,1));}#chatgpt-mermaid-\_r_6vj\_ [data-look="neo"].node .neo-line path{stroke:rgb(83, 154, 248);filter:none;}#chatgpt-mermaid-\_r_6vj\_ [data-look="neo"].node circle{stroke:url(#chatgpt-mermaid-\_r_6vj\_-gradient);filter:drop-shadow( 1px 2px 2px rgba(185,185,185,1));}#chatgpt-mermaid-\_r_6vj\_ [data-look="neo"].node circle .state-start{fill:#000000;}#chatgpt-mermaid-\_r_6vj\_ [data-look="neo"].icon-shape .icon{fill:url(#chatgpt-mermaid-\_r_6vj\_-gradient);filter:drop-shadow( 1px 2px 2px rgba(185,185,185,1));}#chatgpt-mermaid-\_r_6vj\_ [data-look="neo"].icon-shape .icon-neo path{stroke:url(#chatgpt-mermaid-\_r_6vj\_-gradient);filter:drop-shadow( 1px 2px 2px rgba(185,185,185,1));}#chatgpt-mermaid-\_r_6vj\_ .node text{font-size:14px;font-weight:600;letter-spacing:normal;fill:rgb(0, 79, 153);}#chatgpt-mermaid-\_r_6vj\_ .edgeLabels text{font-size:13px;font-weight:600;letter-spacing:-0.08px;fill:rgb(0, 79, 153);}#chatgpt-mermaid-\_r_6vj\_ .node tspan[font-weight="normal"],#chatgpt-mermaid-\_r_6vj\_ .edgeLabels tspan[font-weight="normal"]{font-weight:600;}#chatgpt-mermaid-\_r_6vj\_ .edgeLabel .label rect{opacity:1;rx:13px;ry:13px;fill:rgb(245, 250, 255);stroke:rgb(206, 219, 229);stroke-width:1px;}#chatgpt-mermaid-\_r_6vj\_ .node rect,#chatgpt-mermaid-\_r_6vj\_ .node circle,#chatgpt-mermaid-\_r_6vj\_ .node ellipse,#chatgpt-mermaid-\_r_6vj\_ .node polygon,#chatgpt-mermaid-\_r_6vj\_ .node path{fill:rgb(229, 243, 255);stroke:rgba(0, 0, 0, 0.1);stroke-width:1px;}#chatgpt-mermaid-\_r_6vj\_ .node rect{rx:16px;ry:16px;}#chatgpt-mermaid-\_r_6vj\_ .mermaid-decision .label-container{fill:rgb(245, 250, 255);stroke:rgb(206, 219, 229);stroke-dasharray:2,2;}#chatgpt-mermaid-\_r_6vj\_ .edgePaths .flowchart-link{stroke:rgb(143, 143, 143);stroke-width:1px;stroke-linecap:round;stroke-linejoin:round;}#chatgpt-mermaid-\_r_6vj\_ .marker{fill:rgb(143, 143, 143);stroke:rgb(143, 143, 143);}#chatgpt-mermaid-\_r_6vj\_ :root{--mermaid-font-family:-apple-system-body,ui-sans-serif,-apple-system,system-ui,"Segoe UI",Helvetica,"Apple Color Emoji",Arial,sans-serif,"Segoe UI Emoji","Segoe UI Symbol";}创建 Workerinit_device()设置 CUDA Device初始化分布式环境load_model()创建 GPUModelRunner加载模型权重determine_available_memory()计算 KV Cache 可用显存initialize_from_config()初始化 KV Cachecompile_or_warm_up_model()模型预热完成等待推理请求

### **3.1&#x20;**`init_device()`

初始化 GPU 设备和分布式运行环境。

主要工作：

1. 检查 CUDA 环境。
2. 根据 `local_rank` 选择 GPU。
3. 设置当前 CUDA Device。
4. 设置计算精度及相关运行环境。
5. 初始化分布式通信环境。

**重要概念：**

| 概念           | 含义                      |
| ------------ | ----------------------- |
| `rank`       | 当前进程在分布式通信组中的全局编号       |
| `local_rank` | 当前节点内进程的编号，通常用于选择 GPU   |
| `world_size` | 分布式通信组中的总进程数            |
| `TP`         | Tensor Parallel，张量并行    |
| `PP`         | Pipeline Parallel，流水线并行 |
| `DP`         | Data Parallel，数据并行      |

注意：`rank`、`local_rank`、`world_size` 是不同维度的进程标识信息，不能简单等同。

### **3.2&#x20;**`load_model()`

负责模型加载。

主要流程：

\#chatgpt-mermaid-\_r_72n\_{font-family:-apple-system-body,ui-sans-serif,-apple-system,system-ui,"Segoe UI",Helvetica,"Apple Color Emoji",Arial,sans-serif,"Segoe UI Emoji","Segoe UI Symbol";font-size:16px;fill:rgb(13, 13, 13);}@keyframes edge-animation-frame{from{stroke-dashoffset:0;}}@keyframes dash{to{stroke-dashoffset:0;}}#chatgpt-mermaid-\_r_72n\_ .edge-animation-slow{stroke-dasharray:9,5!important;stroke-dashoffset:900;animation:dash 50s linear infinite;stroke-linecap:round;}#chatgpt-mermaid-\_r_72n\_ .edge-animation-fast{stroke-dasharray:9,5!important;stroke-dashoffset:900;animation:dash 20s linear infinite;stroke-linecap:round;}#chatgpt-mermaid-\_r_72n\_ .error-icon{fill:rgb(243, 243, 243);}#chatgpt-mermaid-\_r_72n\_ .error-text{fill:rgb(13, 13, 13);stroke:rgb(13, 13, 13);}#chatgpt-mermaid-\_r_72n\_ .edge-thickness-normal{stroke-width:1px;}#chatgpt-mermaid-\_r_72n\_ .edge-thickness-thick{stroke-width:3.5px;}#chatgpt-mermaid-\_r_72n\_ .edge-pattern-solid{stroke-dasharray:0;}#chatgpt-mermaid-\_r_72n\_ .edge-thickness-invisible{stroke-width:0;fill:none;}#chatgpt-mermaid-\_r_72n\_ .edge-pattern-dashed{stroke-dasharray:3;}#chatgpt-mermaid-\_r_72n\_ .edge-pattern-dotted{stroke-dasharray:2;}#chatgpt-mermaid-\_r_72n\_ .marker{fill:rgb(143, 143, 143);stroke:rgb(143, 143, 143);}#chatgpt-mermaid-\_r_72n\_ .marker.cross{stroke:rgb(143, 143, 143);}#chatgpt-mermaid-\_r_72n\_ svg{font-family:-apple-system-body,ui-sans-serif,-apple-system,system-ui,"Segoe UI",Helvetica,"Apple Color Emoji",Arial,sans-serif,"Segoe UI Emoji","Segoe UI Symbol";font-size:16px;}#chatgpt-mermaid-\_r_72n\_ p{margin:0;}#chatgpt-mermaid-\_r_72n\_ .label{font-family:-apple-system-body,ui-sans-serif,-apple-system,system-ui,"Segoe UI",Helvetica,"Apple Color Emoji",Arial,sans-serif,"Segoe UI Emoji","Segoe UI Symbol";color:rgb(13, 13, 13);}#chatgpt-mermaid-\_r_72n\_ .cluster-label text{fill:rgb(13, 13, 13);}#chatgpt-mermaid-\_r_72n\_ .cluster-label span{color:rgb(13, 13, 13);}#chatgpt-mermaid-\_r_72n\_ .cluster-label span p{background-color:transparent;}#chatgpt-mermaid-\_r_72n\_ .label text,#chatgpt-mermaid-\_r_72n\_ span{fill:rgb(13, 13, 13);color:rgb(13, 13, 13);}#chatgpt-mermaid-\_r_72n\_ .node rect,#chatgpt-mermaid-\_r_72n\_ .node circle,#chatgpt-mermaid-\_r_72n\_ .node ellipse,#chatgpt-mermaid-\_r_72n\_ .node polygon,#chatgpt-mermaid-\_r_72n\_ .node path{fill:rgb(222, 234, 251);stroke:rgb(83, 154, 248);stroke-width:1px;}#chatgpt-mermaid-\_r_72n\_ .rough-node .label text,#chatgpt-mermaid-\_r_72n\_ .node .label text,#chatgpt-mermaid-\_r_72n\_ .image-shape .label,#chatgpt-mermaid-\_r_72n\_ .icon-shape .label{text-anchor:middle;}#chatgpt-mermaid-\_r_72n\_ .node .katex path{fill:#000;stroke:#000;stroke-width:1px;}#chatgpt-mermaid-\_r_72n\_ .rough-node .label,#chatgpt-mermaid-\_r_72n\_ .node .label,#chatgpt-mermaid-\_r_72n\_ .image-shape .label,#chatgpt-mermaid-\_r_72n\_ .icon-shape .label{text-align:center;}#chatgpt-mermaid-\_r_72n\_ .node.clickable{cursor:pointer;}#chatgpt-mermaid-\_r_72n\_ .root .anchor path{fill:rgb(143, 143, 143)!important;stroke-width:0;stroke:rgb(143, 143, 143);}#chatgpt-mermaid-\_r_72n\_ .arrowheadPath{fill:rgb(143, 143, 143);}#chatgpt-mermaid-\_r_72n\_ .edgePath .path{stroke:rgb(143, 143, 143);stroke-width:1px;}#chatgpt-mermaid-\_r_72n\_ .flowchart-link{stroke:rgb(143, 143, 143);fill:none;}#chatgpt-mermaid-\_r_72n\_ .edgeLabel{background-color:rgb(252, 252, 252);text-align:center;}#chatgpt-mermaid-\_r_72n\_ .edgeLabel p{background-color:rgb(252, 252, 252);}#chatgpt-mermaid-\_r_72n\_ .edgeLabel rect{opacity:0.5;background-color:rgb(252, 252, 252);fill:rgb(252, 252, 252);}#chatgpt-mermaid-\_r_72n\_ .labelBkg{background-color:rgba(252, 252, 252, 0.5);}#chatgpt-mermaid-\_r_72n\_ .cluster rect{fill:rgb(243, 243, 243);stroke:rgba(0, 0, 0, 0.1);stroke-width:1px;}#chatgpt-mermaid-\_r_72n\_ .cluster text{fill:rgb(13, 13, 13);}#chatgpt-mermaid-\_r_72n\_ .cluster span{color:rgb(13, 13, 13);}#chatgpt-mermaid-\_r_72n\_ div.mermaidTooltip{position:absolute;text-align:center;max-width:200px;padding:2px;font-family:-apple-system-body,ui-sans-serif,-apple-system,system-ui,"Segoe UI",Helvetica,"Apple Color Emoji",Arial,sans-serif,"Segoe UI Emoji","Segoe UI Symbol";font-size:12px;background:rgb(243, 243, 243);border:1px solid rgba(0, 0, 0, 0.1);border-radius:2px;pointer-events:none;z-index:100;}#chatgpt-mermaid-\_r_72n\_ .flowchartTitleText{text-anchor:middle;font-size:18px;fill:rgb(13, 13, 13);}#chatgpt-mermaid-\_r_72n\_ rect.text{fill:none;stroke-width:0;}#chatgpt-mermaid-\_r_72n\_ .icon-shape,#chatgpt-mermaid-\_r_72n\_ .image-shape{background-color:rgb(252, 252, 252);text-align:center;}#chatgpt-mermaid-\_r_72n\_ .icon-shape p,#chatgpt-mermaid-\_r_72n\_ .image-shape p{background-color:rgb(252, 252, 252);padding:2px;}#chatgpt-mermaid-\_r_72n\_ .icon-shape .label rect,#chatgpt-mermaid-\_r_72n\_ .image-shape .label rect{opacity:0.5;background-color:rgb(252, 252, 252);fill:rgb(252, 252, 252);}#chatgpt-mermaid-\_r_72n\_ .label-icon{display:inline-block;height:1em;overflow:visible;vertical-align:-0.125em;}#chatgpt-mermaid-\_r_72n\_ .node .label-icon path{fill:currentColor;stroke:revert;stroke-width:revert;}#chatgpt-mermaid-\_r_72n\_ .node .neo-node{stroke:rgb(83, 154, 248);}#chatgpt-mermaid-\_r_72n\_ [data-look="neo"].node rect,#chatgpt-mermaid-\_r_72n\_ [data-look="neo"].cluster rect,#chatgpt-mermaid-\_r_72n\_ [data-look="neo"].node polygon{stroke:url(#chatgpt-mermaid-\_r_72n\_-gradient);filter:drop-shadow( 1px 2px 2px rgba(185,185,185,1));}#chatgpt-mermaid-\_r_72n\_ [data-look="neo"].swimlane.cluster rect{filter:none;}#chatgpt-mermaid-\_r_72n\_ [data-look="neo"].node path{stroke:url(#chatgpt-mermaid-\_r_72n\_-gradient);stroke-width:1px;}#chatgpt-mermaid-\_r_72n\_ [data-look="neo"].node .outer-path{filter:drop-shadow( 1px 2px 2px rgba(185,185,185,1));}#chatgpt-mermaid-\_r_72n\_ [data-look="neo"].node .neo-line path{stroke:rgb(83, 154, 248);filter:none;}#chatgpt-mermaid-\_r_72n\_ [data-look="neo"].node circle{stroke:url(#chatgpt-mermaid-\_r_72n\_-gradient);filter:drop-shadow( 1px 2px 2px rgba(185,185,185,1));}#chatgpt-mermaid-\_r_72n\_ [data-look="neo"].node circle .state-start{fill:#000000;}#chatgpt-mermaid-\_r_72n\_ [data-look="neo"].icon-shape .icon{fill:url(#chatgpt-mermaid-\_r_72n\_-gradient);filter:drop-shadow( 1px 2px 2px rgba(185,185,185,1));}#chatgpt-mermaid-\_r_72n\_ [data-look="neo"].icon-shape .icon-neo path{stroke:url(#chatgpt-mermaid-\_r_72n\_-gradient);filter:drop-shadow( 1px 2px 2px rgba(185,185,185,1));}#chatgpt-mermaid-\_r_72n\_ .node text{font-size:14px;font-weight:600;letter-spacing:normal;fill:rgb(0, 79, 153);}#chatgpt-mermaid-\_r_72n\_ .edgeLabels text{font-size:13px;font-weight:600;letter-spacing:-0.08px;fill:rgb(0, 79, 153);}#chatgpt-mermaid-\_r_72n\_ .node tspan[font-weight="normal"],#chatgpt-mermaid-\_r_72n\_ .edgeLabels tspan[font-weight="normal"]{font-weight:600;}#chatgpt-mermaid-\_r_72n\_ .edgeLabel .label rect{opacity:1;rx:13px;ry:13px;fill:rgb(245, 250, 255);stroke:rgb(206, 219, 229);stroke-width:1px;}#chatgpt-mermaid-\_r_72n\_ .node rect,#chatgpt-mermaid-\_r_72n\_ .node circle,#chatgpt-mermaid-\_r_72n\_ .node ellipse,#chatgpt-mermaid-\_r_72n\_ .node polygon,#chatgpt-mermaid-\_r_72n\_ .node path{fill:rgb(229, 243, 255);stroke:rgba(0, 0, 0, 0.1);stroke-width:1px;}#chatgpt-mermaid-\_r_72n\_ .node rect{rx:16px;ry:16px;}#chatgpt-mermaid-\_r_72n\_ .mermaid-decision .label-container{fill:rgb(245, 250, 255);stroke:rgb(206, 219, 229);stroke-dasharray:2,2;}#chatgpt-mermaid-\_r_72n\_ .edgePaths .flowchart-link{stroke:rgb(143, 143, 143);stroke-width:1px;stroke-linecap:round;stroke-linejoin:round;}#chatgpt-mermaid-\_r_72n\_ .marker{fill:rgb(143, 143, 143);stroke:rgb(143, 143, 143);}#chatgpt-mermaid-\_r_72n\_ :root{--mermaid-font-family:-apple-system-body,ui-sans-serif,-apple-system,system-ui,"Segoe UI",Helvetica,"Apple Color Emoji",Arial,sans-serif,"Segoe UI Emoji","Segoe UI Symbol";}load_model()创建 GPUModelRunner加载模型结构加载模型权重模型加载完成

Worker 将具体模型加载工作委托给 `GPUModelRunner`。

### **3.3&#x20;**`determine_available_memory()`

确定 KV Cache 可以使用的 GPU 显存。

核心目标：

**在模型权重、运行时开销已经占用显存的情况下，确定剩余显存能够容纳多少 KV Cache。**

关键概念：

| 概念                          | 含义                  |
| --------------------------- | ------------------- |
| `gpu_memory_utilization`    | vLLM 允许使用的 GPU 显存比例 |
| `memory_profiling`          | 统计模型运行期间的显存使用情况     |
| `available_kv_cache_memory` | 可分配给 KV Cache 的显存   |

可以用下面的简化模型理解：

\\[ M\_{KV}=M\_{目标预算}-M\_{非KV开销} \\]

其中非 KV 开销包括模型权重及 profiling 测得的运行时显存占用等。

实际计算以源码中的显存 profiling 和相关配置为准。

## **4. KV Cache 初始化**

入口：`initialize_from_config()`

主要工作：

1. 接收 KV Cache 配置。
2. 根据模型结构和并行配置确定 KV Cache 布局。
3. 调用 ModelRunner 初始化 KV Cache。
4. 准备后续 Attention 推理所需的缓存空间。

KV Cache 的作用：

- 保存历史 Token 的 Key 和 Value。
- 避免每生成一个新 Token 都重新计算历史 Token 的 K/V。
- 支持连续批处理和请求间的 KV Block 管理。

KV Cache 并不是简单地为每个请求分配一块连续显存，而是由 vLLM 的 Block 管理机制进行组织。

## **5. 模型编译与预热**

入口：`compile_or_warm_up_model()`

\#chatgpt-mermaid-\_r_72u\_{font-family:-apple-system-body,ui-sans-serif,-apple-system,system-ui,"Segoe UI",Helvetica,"Apple Color Emoji",Arial,sans-serif,"Segoe UI Emoji","Segoe UI Symbol";font-size:16px;fill:rgb(13, 13, 13);}@keyframes edge-animation-frame{from{stroke-dashoffset:0;}}@keyframes dash{to{stroke-dashoffset:0;}}#chatgpt-mermaid-\_r_72u\_ .edge-animation-slow{stroke-dasharray:9,5!important;stroke-dashoffset:900;animation:dash 50s linear infinite;stroke-linecap:round;}#chatgpt-mermaid-\_r_72u\_ .edge-animation-fast{stroke-dasharray:9,5!important;stroke-dashoffset:900;animation:dash 20s linear infinite;stroke-linecap:round;}#chatgpt-mermaid-\_r_72u\_ .error-icon{fill:rgb(243, 243, 243);}#chatgpt-mermaid-\_r_72u\_ .error-text{fill:rgb(13, 13, 13);stroke:rgb(13, 13, 13);}#chatgpt-mermaid-\_r_72u\_ .edge-thickness-normal{stroke-width:1px;}#chatgpt-mermaid-\_r_72u\_ .edge-thickness-thick{stroke-width:3.5px;}#chatgpt-mermaid-\_r_72u\_ .edge-pattern-solid{stroke-dasharray:0;}#chatgpt-mermaid-\_r_72u\_ .edge-thickness-invisible{stroke-width:0;fill:none;}#chatgpt-mermaid-\_r_72u\_ .edge-pattern-dashed{stroke-dasharray:3;}#chatgpt-mermaid-\_r_72u\_ .edge-pattern-dotted{stroke-dasharray:2;}#chatgpt-mermaid-\_r_72u\_ .marker{fill:rgb(143, 143, 143);stroke:rgb(143, 143, 143);}#chatgpt-mermaid-\_r_72u\_ .marker.cross{stroke:rgb(143, 143, 143);}#chatgpt-mermaid-\_r_72u\_ svg{font-family:-apple-system-body,ui-sans-serif,-apple-system,system-ui,"Segoe UI",Helvetica,"Apple Color Emoji",Arial,sans-serif,"Segoe UI Emoji","Segoe UI Symbol";font-size:16px;}#chatgpt-mermaid-\_r_72u\_ p{margin:0;}#chatgpt-mermaid-\_r_72u\_ .label{font-family:-apple-system-body,ui-sans-serif,-apple-system,system-ui,"Segoe UI",Helvetica,"Apple Color Emoji",Arial,sans-serif,"Segoe UI Emoji","Segoe UI Symbol";color:rgb(13, 13, 13);}#chatgpt-mermaid-\_r_72u\_ .cluster-label text{fill:rgb(13, 13, 13);}#chatgpt-mermaid-\_r_72u\_ .cluster-label span{color:rgb(13, 13, 13);}#chatgpt-mermaid-\_r_72u\_ .cluster-label span p{background-color:transparent;}#chatgpt-mermaid-\_r_72u\_ .label text,#chatgpt-mermaid-\_r_72u\_ span{fill:rgb(13, 13, 13);color:rgb(13, 13, 13);}#chatgpt-mermaid-\_r_72u\_ .node rect,#chatgpt-mermaid-\_r_72u\_ .node circle,#chatgpt-mermaid-\_r_72u\_ .node ellipse,#chatgpt-mermaid-\_r_72u\_ .node polygon,#chatgpt-mermaid-\_r_72u\_ .node path{fill:rgb(222, 234, 251);stroke:rgb(83, 154, 248);stroke-width:1px;}#chatgpt-mermaid-\_r_72u\_ .rough-node .label text,#chatgpt-mermaid-\_r_72u\_ .node .label text,#chatgpt-mermaid-\_r_72u\_ .image-shape .label,#chatgpt-mermaid-\_r_72u\_ .icon-shape .label{text-anchor:middle;}#chatgpt-mermaid-\_r_72u\_ .node .katex path{fill:#000;stroke:#000;stroke-width:1px;}#chatgpt-mermaid-\_r_72u\_ .rough-node .label,#chatgpt-mermaid-\_r_72u\_ .node .label,#chatgpt-mermaid-\_r_72u\_ .image-shape .label,#chatgpt-mermaid-\_r_72u\_ .icon-shape .label{text-align:center;}#chatgpt-mermaid-\_r_72u\_ .node.clickable{cursor:pointer;}#chatgpt-mermaid-\_r_72u\_ .root .anchor path{fill:rgb(143, 143, 143)!important;stroke-width:0;stroke:rgb(143, 143, 143);}#chatgpt-mermaid-\_r_72u\_ .arrowheadPath{fill:rgb(143, 143, 143);}#chatgpt-mermaid-\_r_72u\_ .edgePath .path{stroke:rgb(143, 143, 143);stroke-width:1px;}#chatgpt-mermaid-\_r_72u\_ .flowchart-link{stroke:rgb(143, 143, 143);fill:none;}#chatgpt-mermaid-\_r_72u\_ .edgeLabel{background-color:rgb(252, 252, 252);text-align:center;}#chatgpt-mermaid-\_r_72u\_ .edgeLabel p{background-color:rgb(252, 252, 252);}#chatgpt-mermaid-\_r_72u\_ .edgeLabel rect{opacity:0.5;background-color:rgb(252, 252, 252);fill:rgb(252, 252, 252);}#chatgpt-mermaid-\_r_72u\_ .labelBkg{background-color:rgba(252, 252, 252, 0.5);}#chatgpt-mermaid-\_r_72u\_ .cluster rect{fill:rgb(243, 243, 243);stroke:rgba(0, 0, 0, 0.1);stroke-width:1px;}#chatgpt-mermaid-\_r_72u\_ .cluster text{fill:rgb(13, 13, 13);}#chatgpt-mermaid-\_r_72u\_ .cluster span{color:rgb(13, 13, 13);}#chatgpt-mermaid-\_r_72u\_ div.mermaidTooltip{position:absolute;text-align:center;max-width:200px;padding:2px;font-family:-apple-system-body,ui-sans-serif,-apple-system,system-ui,"Segoe UI",Helvetica,"Apple Color Emoji",Arial,sans-serif,"Segoe UI Emoji","Segoe UI Symbol";font-size:12px;background:rgb(243, 243, 243);border:1px solid rgba(0, 0, 0, 0.1);border-radius:2px;pointer-events:none;z-index:100;}#chatgpt-mermaid-\_r_72u\_ .flowchartTitleText{text-anchor:middle;font-size:18px;fill:rgb(13, 13, 13);}#chatgpt-mermaid-\_r_72u\_ rect.text{fill:none;stroke-width:0;}#chatgpt-mermaid-\_r_72u\_ .icon-shape,#chatgpt-mermaid-\_r_72u\_ .image-shape{background-color:rgb(252, 252, 252);text-align:center;}#chatgpt-mermaid-\_r_72u\_ .icon-shape p,#chatgpt-mermaid-\_r_72u\_ .image-shape p{background-color:rgb(252, 252, 252);padding:2px;}#chatgpt-mermaid-\_r_72u\_ .icon-shape .label rect,#chatgpt-mermaid-\_r_72u\_ .image-shape .label rect{opacity:0.5;background-color:rgb(252, 252, 252);fill:rgb(252, 252, 252);}#chatgpt-mermaid-\_r_72u\_ .label-icon{display:inline-block;height:1em;overflow:visible;vertical-align:-0.125em;}#chatgpt-mermaid-\_r_72u\_ .node .label-icon path{fill:currentColor;stroke:revert;stroke-width:revert;}#chatgpt-mermaid-\_r_72u\_ .node .neo-node{stroke:rgb(83, 154, 248);}#chatgpt-mermaid-\_r_72u\_ [data-look="neo"].node rect,#chatgpt-mermaid-\_r_72u\_ [data-look="neo"].cluster rect,#chatgpt-mermaid-\_r_72u\_ [data-look="neo"].node polygon{stroke:url(#chatgpt-mermaid-\_r_72u\_-gradient);filter:drop-shadow( 1px 2px 2px rgba(185,185,185,1));}#chatgpt-mermaid-\_r_72u\_ [data-look="neo"].swimlane.cluster rect{filter:none;}#chatgpt-mermaid-\_r_72u\_ [data-look="neo"].node path{stroke:url(#chatgpt-mermaid-\_r_72u\_-gradient);stroke-width:1px;}#chatgpt-mermaid-\_r_72u\_ [data-look="neo"].node .outer-path{filter:drop-shadow( 1px 2px 2px rgba(185,185,185,1));}#chatgpt-mermaid-\_r_72u\_ [data-look="neo"].node .neo-line path{stroke:rgb(83, 154, 248);filter:none;}#chatgpt-mermaid-\_r_72u\_ [data-look="neo"].node circle{stroke:url(#chatgpt-mermaid-\_r_72u\_-gradient);filter:drop-shadow( 1px 2px 2px rgba(185,185,185,1));}#chatgpt-mermaid-\_r_72u\_ [data-look="neo"].node circle .state-start{fill:#000000;}#chatgpt-mermaid-\_r_72u\_ [data-look="neo"].icon-shape .icon{fill:url(#chatgpt-mermaid-\_r_72u\_-gradient);filter:drop-shadow( 1px 2px 2px rgba(185,185,185,1));}#chatgpt-mermaid-\_r_72u\_ [data-look="neo"].icon-shape .icon-neo path{stroke:url(#chatgpt-mermaid-\_r_72u\_-gradient);filter:drop-shadow( 1px 2px 2px rgba(185,185,185,1));}#chatgpt-mermaid-\_r_72u\_ .node text{font-size:14px;font-weight:600;letter-spacing:normal;fill:rgb(0, 79, 153);}#chatgpt-mermaid-\_r_72u\_ .edgeLabels text{font-size:13px;font-weight:600;letter-spacing:-0.08px;fill:rgb(0, 79, 153);}#chatgpt-mermaid-\_r_72u\_ .node tspan[font-weight="normal"],#chatgpt-mermaid-\_r_72u\_ .edgeLabels tspan[font-weight="normal"]{font-weight:600;}#chatgpt-mermaid-\_r_72u\_ .edgeLabel .label rect{opacity:1;rx:13px;ry:13px;fill:rgb(245, 250, 255);stroke:rgb(206, 219, 229);stroke-width:1px;}#chatgpt-mermaid-\_r_72u\_ .node rect,#chatgpt-mermaid-\_r_72u\_ .node circle,#chatgpt-mermaid-\_r_72u\_ .node ellipse,#chatgpt-mermaid-\_r_72u\_ .node polygon,#chatgpt-mermaid-\_r_72u\_ .node path{fill:rgb(229, 243, 255);stroke:rgba(0, 0, 0, 0.1);stroke-width:1px;}#chatgpt-mermaid-\_r_72u\_ .node rect{rx:16px;ry:16px;}#chatgpt-mermaid-\_r_72u\_ .mermaid-decision .label-container{fill:rgb(245, 250, 255);stroke:rgb(206, 219, 229);stroke-dasharray:2,2;}#chatgpt-mermaid-\_r_72u\_ .edgePaths .flowchart-link{stroke:rgb(143, 143, 143);stroke-width:1px;stroke-linecap:round;stroke-linejoin:round;}#chatgpt-mermaid-\_r_72u\_ .marker{fill:rgb(143, 143, 143);stroke:rgb(143, 143, 143);}#chatgpt-mermaid-\_r_72u\_ :root{--mermaid-font-family:-apple-system-body,ui-sans-serif,-apple-system,system-ui,"Segoe UI",Helvetica,"Apple Color Emoji",Arial,sans-serif,"Segoe UI Emoji","Segoe UI Symbol";}compile_or_warm_up_model()模型编译Dummy RunKernel WarmupCUDA Graph 捕获/预热清理临时资源预热完成

| 概念         | 作用                            |
| ---------- | ----------------------------- |
| Compile    | 编译模型或相关计算图                    |
| Dummy Run  | 使用模拟输入执行模型                    |
| Warmup     | 提前执行算子，完成初始化                  |
| CUDA Graph | 捕获 GPU 操作序列，减少重复执行时的 CPU 调度开销 |
| GC         | 清理临时对象，释放不再使用的资源              |

**为什么需要预热？**

首次运行可能涉及 Kernel 加载、内存分配、编译和 CUDA Graph 捕获等开销。预热能够降低正式推理阶段的额外开销。

## **6. 推理执行流程**

核心入口：`execute_model()`

GPUGPUModelRunnerWorkerEngineCoreGPUGPUModelRunnerWorkerEngineCore#chatgpt-mermaid-\_r_735\_{font-family:-apple-system-body,ui-sans-serif,-apple-system,system-ui,"Segoe UI",Helvetica,"Apple Color Emoji",Arial,sans-serif,"Segoe UI Emoji","Segoe UI Symbol";font-size:16px;fill:rgb(13, 13, 13);}@keyframes edge-animation-frame{from{stroke-dashoffset:0;}}@keyframes dash{to{stroke-dashoffset:0;}}#chatgpt-mermaid-\_r_735\_ .edge-animation-slow{stroke-dasharray:9,5!important;stroke-dashoffset:900;animation:dash 50s linear infinite;stroke-linecap:round;}#chatgpt-mermaid-\_r_735\_ .edge-animation-fast{stroke-dasharray:9,5!important;stroke-dashoffset:900;animation:dash 20s linear infinite;stroke-linecap:round;}#chatgpt-mermaid-\_r_735\_ .error-icon{fill:rgb(243, 243, 243);}#chatgpt-mermaid-\_r_735\_ .error-text{fill:rgb(13, 13, 13);stroke:rgb(13, 13, 13);}#chatgpt-mermaid-\_r_735\_ .edge-thickness-normal{stroke-width:1px;}#chatgpt-mermaid-\_r_735\_ .edge-thickness-thick{stroke-width:3.5px;}#chatgpt-mermaid-\_r_735\_ .edge-pattern-solid{stroke-dasharray:0;}#chatgpt-mermaid-\_r_735\_ .edge-thickness-invisible{stroke-width:0;fill:none;}#chatgpt-mermaid-\_r_735\_ .edge-pattern-dashed{stroke-dasharray:3;}#chatgpt-mermaid-\_r_735\_ .edge-pattern-dotted{stroke-dasharray:2;}#chatgpt-mermaid-\_r_735\_ .marker{fill:rgb(143, 143, 143);stroke:rgb(143, 143, 143);}#chatgpt-mermaid-\_r_735\_ .marker.cross{stroke:rgb(143, 143, 143);}#chatgpt-mermaid-\_r_735\_ svg{font-family:-apple-system-body,ui-sans-serif,-apple-system,system-ui,"Segoe UI",Helvetica,"Apple Color Emoji",Arial,sans-serif,"Segoe UI Emoji","Segoe UI Symbol";font-size:16px;}#chatgpt-mermaid-\_r_735\_ p{margin:0;}#chatgpt-mermaid-\_r_735\_ .actor{stroke:rgb(83, 154, 248);fill:rgb(222, 234, 251);stroke-width:1;}#chatgpt-mermaid-\_r_735\_ rect.actor.outer-path[data-look="neo"]{filter:drop-shadow( 1px 2px 2px rgba(185,185,185,1));}#chatgpt-mermaid-\_r_735\_ rect.note[data-look="neo"]{stroke:rgb(107, 198, 127);fill:rgb(243, 243, 243);filter:drop-shadow( 1px 2px 2px rgba(185,185,185,1));}#chatgpt-mermaid-\_r_735\_ text.actor>tspan{fill:rgb(13, 13, 13);stroke:none;}#chatgpt-mermaid-\_r_735\_ .actor-line{stroke:rgb(143, 143, 143);}#chatgpt-mermaid-\_r_735\_ .innerArc{stroke-width:1.5;stroke-dasharray:none;}#chatgpt-mermaid-\_r_735\_ .messageLine0{stroke-width:1.5;stroke-dasharray:none;stroke:rgb(143, 143, 143);}#chatgpt-mermaid-\_r_735\_ .messageLine1{stroke-width:1.5;stroke-dasharray:2,2;stroke:rgb(143, 143, 143);}#chatgpt-mermaid-\_r_735\_ [id$="-arrowhead"] path{fill:rgb(143, 143, 143);stroke:rgb(143, 143, 143);}#chatgpt-mermaid-\_r_735\_ .sequenceNumber{fill:#707070;}#chatgpt-mermaid-\_r_735\_ [id$="-sequencenumber"]{fill:rgb(143, 143, 143);}#chatgpt-mermaid-\_r_735\_ [id$="-crosshead"] path{fill:rgb(143, 143, 143);stroke:rgb(143, 143, 143);}#chatgpt-mermaid-\_r_735\_ .messageText{fill:rgb(13, 13, 13);stroke:none;}#chatgpt-mermaid-\_r_735\_ .labelBox{stroke:rgba(0, 0, 0, 0.1);fill:rgb(252, 252, 252);filter:none;}#chatgpt-mermaid-\_r_735\_ .labelText,#chatgpt-mermaid-\_r_735\_ .labelText>tspan{fill:rgb(13, 13, 13);stroke:none;}#chatgpt-mermaid-\_r_735\_ .loopText,#chatgpt-mermaid-\_r_735\_ .loopText>tspan{fill:rgb(13, 13, 13);stroke:none;}#chatgpt-mermaid-\_r_735\_ .sectionTitle,#chatgpt-mermaid-\_r_735\_ .sectionTitle>tspan{fill:rgb(13, 13, 13);stroke:none;}#chatgpt-mermaid-\_r_735\_ .loopLine{stroke-width:2px;stroke-dasharray:2,2;stroke:rgba(0, 0, 0, 0.1);fill:rgba(0, 0, 0, 0.1);}#chatgpt-mermaid-\_r_735\_ .note{stroke:rgb(107, 198, 127);fill:rgb(243, 243, 243);}#chatgpt-mermaid-\_r_735\_ .noteText,#chatgpt-mermaid-\_r_735\_ .noteText>tspan{fill:rgb(13, 13, 13);stroke:none;font-weight:normal;}#chatgpt-mermaid-\_r_735\_ .activation0{fill:rgb(243, 243, 243);stroke:hsl(0, 0%, 85.2941176471%);}#chatgpt-mermaid-\_r_735\_ .activation1{fill:rgb(243, 243, 243);stroke:hsl(0, 0%, 85.2941176471%);}#chatgpt-mermaid-\_r_735\_ .activation2{fill:rgb(243, 243, 243);stroke:hsl(0, 0%, 85.2941176471%);}#chatgpt-mermaid-\_r_735\_ .actorPopupMenu{position:absolute;}#chatgpt-mermaid-\_r_735\_ .actorPopupMenuPanel{position:absolute;fill:rgb(222, 234, 251);box-shadow:0px 8px 16px 0px rgba(0,0,0,0.2);filter:drop-shadow(3px 5px 2px rgb(0 0 0 / 0.4));}#chatgpt-mermaid-\_r_735\_ .actor-man circle,#chatgpt-mermaid-\_r_735\_ line{fill:rgb(222, 234, 251);stroke-width:2px;}#chatgpt-mermaid-\_r_735\_ g rect.rect{filter:drop-shadow( 1px 2px 2px rgba(185,185,185,1));stroke:rgb(83, 154, 248);}#chatgpt-mermaid-\_r_735\_ .node .neo-node{stroke:rgb(83, 154, 248);}#chatgpt-mermaid-\_r_735\_ [data-look="neo"].node rect,#chatgpt-mermaid-\_r_735\_ [data-look="neo"].cluster rect,#chatgpt-mermaid-\_r_735\_ [data-look="neo"].node polygon{stroke:url(#chatgpt-mermaid-\_r_735\_-gradient);filter:drop-shadow( 1px 2px 2px rgba(185,185,185,1));}#chatgpt-mermaid-\_r_735\_ [data-look="neo"].swimlane.cluster rect{filter:none;}#chatgpt-mermaid-\_r_735\_ [data-look="neo"].node path{stroke:url(#chatgpt-mermaid-\_r_735\_-gradient);stroke-width:1px;}#chatgpt-mermaid-\_r_735\_ [data-look="neo"].node .outer-path{filter:drop-shadow( 1px 2px 2px rgba(185,185,185,1));}#chatgpt-mermaid-\_r_735\_ [data-look="neo"].node .neo-line path{stroke:rgb(83, 154, 248);filter:none;}#chatgpt-mermaid-\_r_735\_ [data-look="neo"].node circle{stroke:url(#chatgpt-mermaid-\_r_735\_-gradient);filter:drop-shadow( 1px 2px 2px rgba(185,185,185,1));}#chatgpt-mermaid-\_r_735\_ [data-look="neo"].node circle .state-start{fill:#000000;}#chatgpt-mermaid-\_r_735\_ [data-look="neo"].icon-shape .icon{fill:url(#chatgpt-mermaid-\_r_735\_-gradient);filter:drop-shadow( 1px 2px 2px rgba(185,185,185,1));}#chatgpt-mermaid-\_r_735\_ [data-look="neo"].icon-shape .icon-neo path{stroke:url(#chatgpt-mermaid-\_r_735\_-gradient);filter:drop-shadow( 1px 2px 2px rgba(185,185,185,1));}#chatgpt-mermaid-\_r_735\_ :root{--mermaid-font-family:-apple-system-body,ui-sans-serif,-apple-system,system-ui,"Segoe UI",Helvetica,"Apple Color Emoji",Arial,sans-serif,"Segoe UI Emoji","Segoe UI Symbol";}execute_model(SchedulerOutput)execute_model()Model ForwardIntermediate ResultsSamplingSampled TokensModel OutputExecution Result

### **6.1&#x20;**`execute_model()`

主要职责：

- 接收 Scheduler 产生的执行输入。
- 将执行任务交给 GPUModelRunner。
- 处理模型执行结果。
- 支持 Pipeline Parallel 等执行场景。

注意：Scheduler 负责调度请求和 Token 预算，Worker 负责执行调度结果，二者职责不同。

### **6.2&#x20;**`sample_tokens()`

负责 Token 采样。

模型 Forward 得到 Logits 后，采样模块根据 SamplingParams 执行相应的采样策略，生成下一个 Token。

\#chatgpt-mermaid-\_r_73c\_{font-family:-apple-system-body,ui-sans-serif,-apple-system,system-ui,"Segoe UI",Helvetica,"Apple Color Emoji",Arial,sans-serif,"Segoe UI Emoji","Segoe UI Symbol";font-size:16px;fill:rgb(13, 13, 13);}@keyframes edge-animation-frame{from{stroke-dashoffset:0;}}@keyframes dash{to{stroke-dashoffset:0;}}#chatgpt-mermaid-\_r_73c\_ .edge-animation-slow{stroke-dasharray:9,5!important;stroke-dashoffset:900;animation:dash 50s linear infinite;stroke-linecap:round;}#chatgpt-mermaid-\_r_73c\_ .edge-animation-fast{stroke-dasharray:9,5!important;stroke-dashoffset:900;animation:dash 20s linear infinite;stroke-linecap:round;}#chatgpt-mermaid-\_r_73c\_ .error-icon{fill:rgb(243, 243, 243);}#chatgpt-mermaid-\_r_73c\_ .error-text{fill:rgb(13, 13, 13);stroke:rgb(13, 13, 13);}#chatgpt-mermaid-\_r_73c\_ .edge-thickness-normal{stroke-width:1px;}#chatgpt-mermaid-\_r_73c\_ .edge-thickness-thick{stroke-width:3.5px;}#chatgpt-mermaid-\_r_73c\_ .edge-pattern-solid{stroke-dasharray:0;}#chatgpt-mermaid-\_r_73c\_ .edge-thickness-invisible{stroke-width:0;fill:none;}#chatgpt-mermaid-\_r_73c\_ .edge-pattern-dashed{stroke-dasharray:3;}#chatgpt-mermaid-\_r_73c\_ .edge-pattern-dotted{stroke-dasharray:2;}#chatgpt-mermaid-\_r_73c\_ .marker{fill:rgb(143, 143, 143);stroke:rgb(143, 143, 143);}#chatgpt-mermaid-\_r_73c\_ .marker.cross{stroke:rgb(143, 143, 143);}#chatgpt-mermaid-\_r_73c\_ svg{font-family:-apple-system-body,ui-sans-serif,-apple-system,system-ui,"Segoe UI",Helvetica,"Apple Color Emoji",Arial,sans-serif,"Segoe UI Emoji","Segoe UI Symbol";font-size:16px;}#chatgpt-mermaid-\_r_73c\_ p{margin:0;}#chatgpt-mermaid-\_r_73c\_ .label{font-family:-apple-system-body,ui-sans-serif,-apple-system,system-ui,"Segoe UI",Helvetica,"Apple Color Emoji",Arial,sans-serif,"Segoe UI Emoji","Segoe UI Symbol";color:rgb(13, 13, 13);}#chatgpt-mermaid-\_r_73c\_ .cluster-label text{fill:rgb(13, 13, 13);}#chatgpt-mermaid-\_r_73c\_ .cluster-label span{color:rgb(13, 13, 13);}#chatgpt-mermaid-\_r_73c\_ .cluster-label span p{background-color:transparent;}#chatgpt-mermaid-\_r_73c\_ .label text,#chatgpt-mermaid-\_r_73c\_ span{fill:rgb(13, 13, 13);color:rgb(13, 13, 13);}#chatgpt-mermaid-\_r_73c\_ .node rect,#chatgpt-mermaid-\_r_73c\_ .node circle,#chatgpt-mermaid-\_r_73c\_ .node ellipse,#chatgpt-mermaid-\_r_73c\_ .node polygon,#chatgpt-mermaid-\_r_73c\_ .node path{fill:rgb(222, 234, 251);stroke:rgb(83, 154, 248);stroke-width:1px;}#chatgpt-mermaid-\_r_73c\_ .rough-node .label text,#chatgpt-mermaid-\_r_73c\_ .node .label text,#chatgpt-mermaid-\_r_73c\_ .image-shape .label,#chatgpt-mermaid-\_r_73c\_ .icon-shape .label{text-anchor:middle;}#chatgpt-mermaid-\_r_73c\_ .node .katex path{fill:#000;stroke:#000;stroke-width:1px;}#chatgpt-mermaid-\_r_73c\_ .rough-node .label,#chatgpt-mermaid-\_r_73c\_ .node .label,#chatgpt-mermaid-\_r_73c\_ .image-shape .label,#chatgpt-mermaid-\_r_73c\_ .icon-shape .label{text-align:center;}#chatgpt-mermaid-\_r_73c\_ .node.clickable{cursor:pointer;}#chatgpt-mermaid-\_r_73c\_ .root .anchor path{fill:rgb(143, 143, 143)!important;stroke-width:0;stroke:rgb(143, 143, 143);}#chatgpt-mermaid-\_r_73c\_ .arrowheadPath{fill:rgb(143, 143, 143);}#chatgpt-mermaid-\_r_73c\_ .edgePath .path{stroke:rgb(143, 143, 143);stroke-width:1px;}#chatgpt-mermaid-\_r_73c\_ .flowchart-link{stroke:rgb(143, 143, 143);fill:none;}#chatgpt-mermaid-\_r_73c\_ .edgeLabel{background-color:rgb(252, 252, 252);text-align:center;}#chatgpt-mermaid-\_r_73c\_ .edgeLabel p{background-color:rgb(252, 252, 252);}#chatgpt-mermaid-\_r_73c\_ .edgeLabel rect{opacity:0.5;background-color:rgb(252, 252, 252);fill:rgb(252, 252, 252);}#chatgpt-mermaid-\_r_73c\_ .labelBkg{background-color:rgba(252, 252, 252, 0.5);}#chatgpt-mermaid-\_r_73c\_ .cluster rect{fill:rgb(243, 243, 243);stroke:rgba(0, 0, 0, 0.1);stroke-width:1px;}#chatgpt-mermaid-\_r_73c\_ .cluster text{fill:rgb(13, 13, 13);}#chatgpt-mermaid-\_r_73c\_ .cluster span{color:rgb(13, 13, 13);}#chatgpt-mermaid-\_r_73c\_ div.mermaidTooltip{position:absolute;text-align:center;max-width:200px;padding:2px;font-family:-apple-system-body,ui-sans-serif,-apple-system,system-ui,"Segoe UI",Helvetica,"Apple Color Emoji",Arial,sans-serif,"Segoe UI Emoji","Segoe UI Symbol";font-size:12px;background:rgb(243, 243, 243);border:1px solid rgba(0, 0, 0, 0.1);border-radius:2px;pointer-events:none;z-index:100;}#chatgpt-mermaid-\_r_73c\_ .flowchartTitleText{text-anchor:middle;font-size:18px;fill:rgb(13, 13, 13);}#chatgpt-mermaid-\_r_73c\_ rect.text{fill:none;stroke-width:0;}#chatgpt-mermaid-\_r_73c\_ .icon-shape,#chatgpt-mermaid-\_r_73c\_ .image-shape{background-color:rgb(252, 252, 252);text-align:center;}#chatgpt-mermaid-\_r_73c\_ .icon-shape p,#chatgpt-mermaid-\_r_73c\_ .image-shape p{background-color:rgb(252, 252, 252);padding:2px;}#chatgpt-mermaid-\_r_73c\_ .icon-shape .label rect,#chatgpt-mermaid-\_r_73c\_ .image-shape .label rect{opacity:0.5;background-color:rgb(252, 252, 252);fill:rgb(252, 252, 252);}#chatgpt-mermaid-\_r_73c\_ .label-icon{display:inline-block;height:1em;overflow:visible;vertical-align:-0.125em;}#chatgpt-mermaid-\_r_73c\_ .node .label-icon path{fill:currentColor;stroke:revert;stroke-width:revert;}#chatgpt-mermaid-\_r_73c\_ .node .neo-node{stroke:rgb(83, 154, 248);}#chatgpt-mermaid-\_r_73c\_ [data-look="neo"].node rect,#chatgpt-mermaid-\_r_73c\_ [data-look="neo"].cluster rect,#chatgpt-mermaid-\_r_73c\_ [data-look="neo"].node polygon{stroke:url(#chatgpt-mermaid-\_r_73c\_-gradient);filter:drop-shadow( 1px 2px 2px rgba(185,185,185,1));}#chatgpt-mermaid-\_r_73c\_ [data-look="neo"].swimlane.cluster rect{filter:none;}#chatgpt-mermaid-\_r_73c\_ [data-look="neo"].node path{stroke:url(#chatgpt-mermaid-\_r_73c\_-gradient);stroke-width:1px;}#chatgpt-mermaid-\_r_73c\_ [data-look="neo"].node .outer-path{filter:drop-shadow( 1px 2px 2px rgba(185,185,185,1));}#chatgpt-mermaid-\_r_73c\_ [data-look="neo"].node .neo-line path{stroke:rgb(83, 154, 248);filter:none;}#chatgpt-mermaid-\_r_73c\_ [data-look="neo"].node circle{stroke:url(#chatgpt-mermaid-\_r_73c\_-gradient);filter:drop-shadow( 1px 2px 2px rgba(185,185,185,1));}#chatgpt-mermaid-\_r_73c\_ [data-look="neo"].node circle .state-start{fill:#000000;}#chatgpt-mermaid-\_r_73c\_ [data-look="neo"].icon-shape .icon{fill:url(#chatgpt-mermaid-\_r_73c\_-gradient);filter:drop-shadow( 1px 2px 2px rgba(185,185,185,1));}#chatgpt-mermaid-\_r_73c\_ [data-look="neo"].icon-shape .icon-neo path{stroke:url(#chatgpt-mermaid-\_r_73c\_-gradient);filter:drop-shadow( 1px 2px 2px rgba(185,185,185,1));}#chatgpt-mermaid-\_r_73c\_ .node text{font-size:14px;font-weight:600;letter-spacing:normal;fill:rgb(0, 79, 153);}#chatgpt-mermaid-\_r_73c\_ .edgeLabels text{font-size:13px;font-weight:600;letter-spacing:-0.08px;fill:rgb(0, 79, 153);}#chatgpt-mermaid-\_r_73c\_ .node tspan[font-weight="normal"],#chatgpt-mermaid-\_r_73c\_ .edgeLabels tspan[font-weight="normal"]{font-weight:600;}#chatgpt-mermaid-\_r_73c\_ .edgeLabel .label rect{opacity:1;rx:13px;ry:13px;fill:rgb(245, 250, 255);stroke:rgb(206, 219, 229);stroke-width:1px;}#chatgpt-mermaid-\_r_73c\_ .node rect,#chatgpt-mermaid-\_r_73c\_ .node circle,#chatgpt-mermaid-\_r_73c\_ .node ellipse,#chatgpt-mermaid-\_r_73c\_ .node polygon,#chatgpt-mermaid-\_r_73c\_ .node path{fill:rgb(229, 243, 255);stroke:rgba(0, 0, 0, 0.1);stroke-width:1px;}#chatgpt-mermaid-\_r_73c\_ .node rect{rx:16px;ry:16px;}#chatgpt-mermaid-\_r_73c\_ .mermaid-decision .label-container{fill:rgb(245, 250, 255);stroke:rgb(206, 219, 229);stroke-dasharray:2,2;}#chatgpt-mermaid-\_r_73c\_ .edgePaths .flowchart-link{stroke:rgb(143, 143, 143);stroke-width:1px;stroke-linecap:round;stroke-linejoin:round;}#chatgpt-mermaid-\_r_73c\_ .marker{fill:rgb(143, 143, 143);stroke:rgb(143, 143, 143);}#chatgpt-mermaid-\_r_73c\_ :root{--mermaid-font-family:-apple-system-body,ui-sans-serif,-apple-system,system-ui,"Segoe UI",Helvetica,"Apple Color Emoji",Arial,sans-serif,"Segoe UI Emoji","Segoe UI Symbol";}Model ForwardLogitsSamplingSampled Token返回执行结果

### **6.3&#x20;**`AsyncIntermediateTensors`

用于 Pipeline Parallel 场景。

当模型被切分到多个 Pipeline Stage 时，前一个 Stage 的输出需要传递给下一个 Stage。

\#chatgpt-mermaid-\_r_73j\_{font-family:-apple-system-body,ui-sans-serif,-apple-system,system-ui,"Segoe UI",Helvetica,"Apple Color Emoji",Arial,sans-serif,"Segoe UI Emoji","Segoe UI Symbol";font-size:16px;fill:rgb(13, 13, 13);}@keyframes edge-animation-frame{from{stroke-dashoffset:0;}}@keyframes dash{to{stroke-dashoffset:0;}}#chatgpt-mermaid-\_r_73j\_ .edge-animation-slow{stroke-dasharray:9,5!important;stroke-dashoffset:900;animation:dash 50s linear infinite;stroke-linecap:round;}#chatgpt-mermaid-\_r_73j\_ .edge-animation-fast{stroke-dasharray:9,5!important;stroke-dashoffset:900;animation:dash 20s linear infinite;stroke-linecap:round;}#chatgpt-mermaid-\_r_73j\_ .error-icon{fill:rgb(243, 243, 243);}#chatgpt-mermaid-\_r_73j\_ .error-text{fill:rgb(13, 13, 13);stroke:rgb(13, 13, 13);}#chatgpt-mermaid-\_r_73j\_ .edge-thickness-normal{stroke-width:1px;}#chatgpt-mermaid-\_r_73j\_ .edge-thickness-thick{stroke-width:3.5px;}#chatgpt-mermaid-\_r_73j\_ .edge-pattern-solid{stroke-dasharray:0;}#chatgpt-mermaid-\_r_73j\_ .edge-thickness-invisible{stroke-width:0;fill:none;}#chatgpt-mermaid-\_r_73j\_ .edge-pattern-dashed{stroke-dasharray:3;}#chatgpt-mermaid-\_r_73j\_ .edge-pattern-dotted{stroke-dasharray:2;}#chatgpt-mermaid-\_r_73j\_ .marker{fill:rgb(143, 143, 143);stroke:rgb(143, 143, 143);}#chatgpt-mermaid-\_r_73j\_ .marker.cross{stroke:rgb(143, 143, 143);}#chatgpt-mermaid-\_r_73j\_ svg{font-family:-apple-system-body,ui-sans-serif,-apple-system,system-ui,"Segoe UI",Helvetica,"Apple Color Emoji",Arial,sans-serif,"Segoe UI Emoji","Segoe UI Symbol";font-size:16px;}#chatgpt-mermaid-\_r_73j\_ p{margin:0;}#chatgpt-mermaid-\_r_73j\_ .label{font-family:-apple-system-body,ui-sans-serif,-apple-system,system-ui,"Segoe UI",Helvetica,"Apple Color Emoji",Arial,sans-serif,"Segoe UI Emoji","Segoe UI Symbol";color:rgb(13, 13, 13);}#chatgpt-mermaid-\_r_73j\_ .cluster-label text{fill:rgb(13, 13, 13);}#chatgpt-mermaid-\_r_73j\_ .cluster-label span{color:rgb(13, 13, 13);}#chatgpt-mermaid-\_r_73j\_ .cluster-label span p{background-color:transparent;}#chatgpt-mermaid-\_r_73j\_ .label text,#chatgpt-mermaid-\_r_73j\_ span{fill:rgb(13, 13, 13);color:rgb(13, 13, 13);}#chatgpt-mermaid-\_r_73j\_ .node rect,#chatgpt-mermaid-\_r_73j\_ .node circle,#chatgpt-mermaid-\_r_73j\_ .node ellipse,#chatgpt-mermaid-\_r_73j\_ .node polygon,#chatgpt-mermaid-\_r_73j\_ .node path{fill:rgb(222, 234, 251);stroke:rgb(83, 154, 248);stroke-width:1px;}#chatgpt-mermaid-\_r_73j\_ .rough-node .label text,#chatgpt-mermaid-\_r_73j\_ .node .label text,#chatgpt-mermaid-\_r_73j\_ .image-shape .label,#chatgpt-mermaid-\_r_73j\_ .icon-shape .label{text-anchor:middle;}#chatgpt-mermaid-\_r_73j\_ .node .katex path{fill:#000;stroke:#000;stroke-width:1px;}#chatgpt-mermaid-\_r_73j\_ .rough-node .label,#chatgpt-mermaid-\_r_73j\_ .node .label,#chatgpt-mermaid-\_r_73j\_ .image-shape .label,#chatgpt-mermaid-\_r_73j\_ .icon-shape .label{text-align:center;}#chatgpt-mermaid-\_r_73j\_ .node.clickable{cursor:pointer;}#chatgpt-mermaid-\_r_73j\_ .root .anchor path{fill:rgb(143, 143, 143)!important;stroke-width:0;stroke:rgb(143, 143, 143);}#chatgpt-mermaid-\_r_73j\_ .arrowheadPath{fill:rgb(143, 143, 143);}#chatgpt-mermaid-\_r_73j\_ .edgePath .path{stroke:rgb(143, 143, 143);stroke-width:1px;}#chatgpt-mermaid-\_r_73j\_ .flowchart-link{stroke:rgb(143, 143, 143);fill:none;}#chatgpt-mermaid-\_r_73j\_ .edgeLabel{background-color:rgb(252, 252, 252);text-align:center;}#chatgpt-mermaid-\_r_73j\_ .edgeLabel p{background-color:rgb(252, 252, 252);}#chatgpt-mermaid-\_r_73j\_ .edgeLabel rect{opacity:0.5;background-color:rgb(252, 252, 252);fill:rgb(252, 252, 252);}#chatgpt-mermaid-\_r_73j\_ .labelBkg{background-color:rgba(252, 252, 252, 0.5);}#chatgpt-mermaid-\_r_73j\_ .cluster rect{fill:rgb(243, 243, 243);stroke:rgba(0, 0, 0, 0.1);stroke-width:1px;}#chatgpt-mermaid-\_r_73j\_ .cluster text{fill:rgb(13, 13, 13);}#chatgpt-mermaid-\_r_73j\_ .cluster span{color:rgb(13, 13, 13);}#chatgpt-mermaid-\_r_73j\_ div.mermaidTooltip{position:absolute;text-align:center;max-width:200px;padding:2px;font-family:-apple-system-body,ui-sans-serif,-apple-system,system-ui,"Segoe UI",Helvetica,"Apple Color Emoji",Arial,sans-serif,"Segoe UI Emoji","Segoe UI Symbol";font-size:12px;background:rgb(243, 243, 243);border:1px solid rgba(0, 0, 0, 0.1);border-radius:2px;pointer-events:none;z-index:100;}#chatgpt-mermaid-\_r_73j\_ .flowchartTitleText{text-anchor:middle;font-size:18px;fill:rgb(13, 13, 13);}#chatgpt-mermaid-\_r_73j\_ rect.text{fill:none;stroke-width:0;}#chatgpt-mermaid-\_r_73j\_ .icon-shape,#chatgpt-mermaid-\_r_73j\_ .image-shape{background-color:rgb(252, 252, 252);text-align:center;}#chatgpt-mermaid-\_r_73j\_ .icon-shape p,#chatgpt-mermaid-\_r_73j\_ .image-shape p{background-color:rgb(252, 252, 252);padding:2px;}#chatgpt-mermaid-\_r_73j\_ .icon-shape .label rect,#chatgpt-mermaid-\_r_73j\_ .image-shape .label rect{opacity:0.5;background-color:rgb(252, 252, 252);fill:rgb(252, 252, 252);}#chatgpt-mermaid-\_r_73j\_ .label-icon{display:inline-block;height:1em;overflow:visible;vertical-align:-0.125em;}#chatgpt-mermaid-\_r_73j\_ .node .label-icon path{fill:currentColor;stroke:revert;stroke-width:revert;}#chatgpt-mermaid-\_r_73j\_ .node .neo-node{stroke:rgb(83, 154, 248);}#chatgpt-mermaid-\_r_73j\_ [data-look="neo"].node rect,#chatgpt-mermaid-\_r_73j\_ [data-look="neo"].cluster rect,#chatgpt-mermaid-\_r_73j\_ [data-look="neo"].node polygon{stroke:url(#chatgpt-mermaid-\_r_73j\_-gradient);filter:drop-shadow( 1px 2px 2px rgba(185,185,185,1));}#chatgpt-mermaid-\_r_73j\_ [data-look="neo"].swimlane.cluster rect{filter:none;}#chatgpt-mermaid-\_r_73j\_ [data-look="neo"].node path{stroke:url(#chatgpt-mermaid-\_r_73j\_-gradient);stroke-width:1px;}#chatgpt-mermaid-\_r_73j\_ [data-look="neo"].node .outer-path{filter:drop-shadow( 1px 2px 2px rgba(185,185,185,1));}#chatgpt-mermaid-\_r_73j\_ [data-look="neo"].node .neo-line path{stroke:rgb(83, 154, 248);filter:none;}#chatgpt-mermaid-\_r_73j\_ [data-look="neo"].node circle{stroke:url(#chatgpt-mermaid-\_r_73j\_-gradient);filter:drop-shadow( 1px 2px 2px rgba(185,185,185,1));}#chatgpt-mermaid-\_r_73j\_ [data-look="neo"].node circle .state-start{fill:#000000;}#chatgpt-mermaid-\_r_73j\_ [data-look="neo"].icon-shape .icon{fill:url(#chatgpt-mermaid-\_r_73j\_-gradient);filter:drop-shadow( 1px 2px 2px rgba(185,185,185,1));}#chatgpt-mermaid-\_r_73j\_ [data-look="neo"].icon-shape .icon-neo path{stroke:url(#chatgpt-mermaid-\_r_73j\_-gradient);filter:drop-shadow( 1px 2px 2px rgba(185,185,185,1));}#chatgpt-mermaid-\_r_73j\_ .node text{font-size:14px;font-weight:600;letter-spacing:normal;fill:rgb(0, 79, 153);}#chatgpt-mermaid-\_r_73j\_ .edgeLabels text{font-size:13px;font-weight:600;letter-spacing:-0.08px;fill:rgb(0, 79, 153);}#chatgpt-mermaid-\_r_73j\_ .node tspan[font-weight="normal"],#chatgpt-mermaid-\_r_73j\_ .edgeLabels tspan[font-weight="normal"]{font-weight:600;}#chatgpt-mermaid-\_r_73j\_ .edgeLabel .label rect{opacity:1;rx:13px;ry:13px;fill:rgb(245, 250, 255);stroke:rgb(206, 219, 229);stroke-width:1px;}#chatgpt-mermaid-\_r_73j\_ .node rect,#chatgpt-mermaid-\_r_73j\_ .node circle,#chatgpt-mermaid-\_r_73j\_ .node ellipse,#chatgpt-mermaid-\_r_73j\_ .node polygon,#chatgpt-mermaid-\_r_73j\_ .node path{fill:rgb(229, 243, 255);stroke:rgba(0, 0, 0, 0.1);stroke-width:1px;}#chatgpt-mermaid-\_r_73j\_ .node rect{rx:16px;ry:16px;}#chatgpt-mermaid-\_r_73j\_ .mermaid-decision .label-container{fill:rgb(245, 250, 255);stroke:rgb(206, 219, 229);stroke-dasharray:2,2;}#chatgpt-mermaid-\_r_73j\_ .edgePaths .flowchart-link{stroke:rgb(143, 143, 143);stroke-width:1px;stroke-linecap:round;stroke-linejoin:round;}#chatgpt-mermaid-\_r_73j\_ .marker{fill:rgb(143, 143, 143);stroke:rgb(143, 143, 143);}#chatgpt-mermaid-\_r_73j\_ :root{--mermaid-font-family:-apple-system-body,ui-sans-serif,-apple-system,system-ui,"Segoe UI",Helvetica,"Apple Color Emoji",Arial,sans-serif,"Segoe UI Emoji","Segoe UI Symbol";}Worker 0 / Stage 0Intermediate TensorsWorker 1 / Stage 1最终输出

它传递的是中间张量，不是最终生成的 Token。

## **7. 其他重要功能**

| 功能              | 相关接口                         | 作用              |
| --------------- | ---------------------------- | --------------- |
| GPU 休眠          | `sleep()`                    | 释放或降低 GPU 资源占用  |
| GPU 唤醒          | `wake_up()`                  | 恢复推理所需资源        |
| Checkpoint      | 相关 Checkpoint 接口             | 保存或恢复模型状态       |
| LoRA            | LoRA 相关接口                    | 管理 LoRA Adapter |
| Profiling       | `profile()` 等                | 性能分析            |
| Weight Transfer | 权重传输接口                       | 支持运行时权重更新       |
| Elastic EP      | Elastic Expert Parallel 相关接口 | 支持弹性专家并行相关能力    |

具体行为需要结合对应方法的实现和调用方分析，不能仅凭接口名称推断全部执行细节。

## **8. Worker 的整体生命周期**

\#chatgpt-mermaid-\_r_73q\_{font-family:-apple-system-body,ui-sans-serif,-apple-system,system-ui,"Segoe UI",Helvetica,"Apple Color Emoji",Arial,sans-serif,"Segoe UI Emoji","Segoe UI Symbol";font-size:16px;fill:rgb(13, 13, 13);}@keyframes edge-animation-frame{from{stroke-dashoffset:0;}}@keyframes dash{to{stroke-dashoffset:0;}}#chatgpt-mermaid-\_r_73q\_ .edge-animation-slow{stroke-dasharray:9,5!important;stroke-dashoffset:900;animation:dash 50s linear infinite;stroke-linecap:round;}#chatgpt-mermaid-\_r_73q\_ .edge-animation-fast{stroke-dasharray:9,5!important;stroke-dashoffset:900;animation:dash 20s linear infinite;stroke-linecap:round;}#chatgpt-mermaid-\_r_73q\_ .error-icon{fill:rgb(243, 243, 243);}#chatgpt-mermaid-\_r_73q\_ .error-text{fill:rgb(13, 13, 13);stroke:rgb(13, 13, 13);}#chatgpt-mermaid-\_r_73q\_ .edge-thickness-normal{stroke-width:1px;}#chatgpt-mermaid-\_r_73q\_ .edge-thickness-thick{stroke-width:3.5px;}#chatgpt-mermaid-\_r_73q\_ .edge-pattern-solid{stroke-dasharray:0;}#chatgpt-mermaid-\_r_73q\_ .edge-thickness-invisible{stroke-width:0;fill:none;}#chatgpt-mermaid-\_r_73q\_ .edge-pattern-dashed{stroke-dasharray:3;}#chatgpt-mermaid-\_r_73q\_ .edge-pattern-dotted{stroke-dasharray:2;}#chatgpt-mermaid-\_r_73q\_ .marker{fill:rgb(143, 143, 143);stroke:rgb(143, 143, 143);}#chatgpt-mermaid-\_r_73q\_ .marker.cross{stroke:rgb(143, 143, 143);}#chatgpt-mermaid-\_r_73q\_ svg{font-family:-apple-system-body,ui-sans-serif,-apple-system,system-ui,"Segoe UI",Helvetica,"Apple Color Emoji",Arial,sans-serif,"Segoe UI Emoji","Segoe UI Symbol";font-size:16px;}#chatgpt-mermaid-\_r_73q\_ p{margin:0;}#chatgpt-mermaid-\_r_73q\_ .label{font-family:-apple-system-body,ui-sans-serif,-apple-system,system-ui,"Segoe UI",Helvetica,"Apple Color Emoji",Arial,sans-serif,"Segoe UI Emoji","Segoe UI Symbol";color:rgb(13, 13, 13);}#chatgpt-mermaid-\_r_73q\_ .cluster-label text{fill:rgb(13, 13, 13);}#chatgpt-mermaid-\_r_73q\_ .cluster-label span{color:rgb(13, 13, 13);}#chatgpt-mermaid-\_r_73q\_ .cluster-label span p{background-color:transparent;}#chatgpt-mermaid-\_r_73q\_ .label text,#chatgpt-mermaid-\_r_73q\_ span{fill:rgb(13, 13, 13);color:rgb(13, 13, 13);}#chatgpt-mermaid-\_r_73q\_ .node rect,#chatgpt-mermaid-\_r_73q\_ .node circle,#chatgpt-mermaid-\_r_73q\_ .node ellipse,#chatgpt-mermaid-\_r_73q\_ .node polygon,#chatgpt-mermaid-\_r_73q\_ .node path{fill:rgb(222, 234, 251);stroke:rgb(83, 154, 248);stroke-width:1px;}#chatgpt-mermaid-\_r_73q\_ .rough-node .label text,#chatgpt-mermaid-\_r_73q\_ .node .label text,#chatgpt-mermaid-\_r_73q\_ .image-shape .label,#chatgpt-mermaid-\_r_73q\_ .icon-shape .label{text-anchor:middle;}#chatgpt-mermaid-\_r_73q\_ .node .katex path{fill:#000;stroke:#000;stroke-width:1px;}#chatgpt-mermaid-\_r_73q\_ .rough-node .label,#chatgpt-mermaid-\_r_73q\_ .node .label,#chatgpt-mermaid-\_r_73q\_ .image-shape .label,#chatgpt-mermaid-\_r_73q\_ .icon-shape .label{text-align:center;}#chatgpt-mermaid-\_r_73q\_ .node.clickable{cursor:pointer;}#chatgpt-mermaid-\_r_73q\_ .root .anchor path{fill:rgb(143, 143, 143)!important;stroke-width:0;stroke:rgb(143, 143, 143);}#chatgpt-mermaid-\_r_73q\_ .arrowheadPath{fill:rgb(143, 143, 143);}#chatgpt-mermaid-\_r_73q\_ .edgePath .path{stroke:rgb(143, 143, 143);stroke-width:1px;}#chatgpt-mermaid-\_r_73q\_ .flowchart-link{stroke:rgb(143, 143, 143);fill:none;}#chatgpt-mermaid-\_r_73q\_ .edgeLabel{background-color:rgb(252, 252, 252);text-align:center;}#chatgpt-mermaid-\_r_73q\_ .edgeLabel p{background-color:rgb(252, 252, 252);}#chatgpt-mermaid-\_r_73q\_ .edgeLabel rect{opacity:0.5;background-color:rgb(252, 252, 252);fill:rgb(252, 252, 252);}#chatgpt-mermaid-\_r_73q\_ .labelBkg{background-color:rgba(252, 252, 252, 0.5);}#chatgpt-mermaid-\_r_73q\_ .cluster rect{fill:rgb(243, 243, 243);stroke:rgba(0, 0, 0, 0.1);stroke-width:1px;}#chatgpt-mermaid-\_r_73q\_ .cluster text{fill:rgb(13, 13, 13);}#chatgpt-mermaid-\_r_73q\_ .cluster span{color:rgb(13, 13, 13);}#chatgpt-mermaid-\_r_73q\_ div.mermaidTooltip{position:absolute;text-align:center;max-width:200px;padding:2px;font-family:-apple-system-body,ui-sans-serif,-apple-system,system-ui,"Segoe UI",Helvetica,"Apple Color Emoji",Arial,sans-serif,"Segoe UI Emoji","Segoe UI Symbol";font-size:12px;background:rgb(243, 243, 243);border:1px solid rgba(0, 0, 0, 0.1);border-radius:2px;pointer-events:none;z-index:100;}#chatgpt-mermaid-\_r_73q\_ .flowchartTitleText{text-anchor:middle;font-size:18px;fill:rgb(13, 13, 13);}#chatgpt-mermaid-\_r_73q\_ rect.text{fill:none;stroke-width:0;}#chatgpt-mermaid-\_r_73q\_ .icon-shape,#chatgpt-mermaid-\_r_73q\_ .image-shape{background-color:rgb(252, 252, 252);text-align:center;}#chatgpt-mermaid-\_r_73q\_ .icon-shape p,#chatgpt-mermaid-\_r_73q\_ .image-shape p{background-color:rgb(252, 252, 252);padding:2px;}#chatgpt-mermaid-\_r_73q\_ .icon-shape .label rect,#chatgpt-mermaid-\_r_73q\_ .image-shape .label rect{opacity:0.5;background-color:rgb(252, 252, 252);fill:rgb(252, 252, 252);}#chatgpt-mermaid-\_r_73q\_ .label-icon{display:inline-block;height:1em;overflow:visible;vertical-align:-0.125em;}#chatgpt-mermaid-\_r_73q\_ .node .label-icon path{fill:currentColor;stroke:revert;stroke-width:revert;}#chatgpt-mermaid-\_r_73q\_ .node .neo-node{stroke:rgb(83, 154, 248);}#chatgpt-mermaid-\_r_73q\_ [data-look="neo"].node rect,#chatgpt-mermaid-\_r_73q\_ [data-look="neo"].cluster rect,#chatgpt-mermaid-\_r_73q\_ [data-look="neo"].node polygon{stroke:url(#chatgpt-mermaid-\_r_73q\_-gradient);filter:drop-shadow( 1px 2px 2px rgba(185,185,185,1));}#chatgpt-mermaid-\_r_73q\_ [data-look="neo"].swimlane.cluster rect{filter:none;}#chatgpt-mermaid-\_r_73q\_ [data-look="neo"].node path{stroke:url(#chatgpt-mermaid-\_r_73q\_-gradient);stroke-width:1px;}#chatgpt-mermaid-\_r_73q\_ [data-look="neo"].node .outer-path{filter:drop-shadow( 1px 2px 2px rgba(185,185,185,1));}#chatgpt-mermaid-\_r_73q\_ [data-look="neo"].node .neo-line path{stroke:rgb(83, 154, 248);filter:none;}#chatgpt-mermaid-\_r_73q\_ [data-look="neo"].node circle{stroke:url(#chatgpt-mermaid-\_r_73q\_-gradient);filter:drop-shadow( 1px 2px 2px rgba(185,185,185,1));}#chatgpt-mermaid-\_r_73q\_ [data-look="neo"].node circle .state-start{fill:#000000;}#chatgpt-mermaid-\_r_73q\_ [data-look="neo"].icon-shape .icon{fill:url(#chatgpt-mermaid-\_r_73q\_-gradient);filter:drop-shadow( 1px 2px 2px rgba(185,185,185,1));}#chatgpt-mermaid-\_r_73q\_ [data-look="neo"].icon-shape .icon-neo path{stroke:url(#chatgpt-mermaid-\_r_73q\_-gradient);filter:drop-shadow( 1px 2px 2px rgba(185,185,185,1));}#chatgpt-mermaid-\_r_73q\_ .node text{font-size:14px;font-weight:600;letter-spacing:normal;fill:rgb(0, 79, 153);}#chatgpt-mermaid-\_r_73q\_ .edgeLabels text{font-size:13px;font-weight:600;letter-spacing:-0.08px;fill:rgb(0, 79, 153);}#chatgpt-mermaid-\_r_73q\_ .node tspan[font-weight="normal"],#chatgpt-mermaid-\_r_73q\_ .edgeLabels tspan[font-weight="normal"]{font-weight:600;}#chatgpt-mermaid-\_r_73q\_ .edgeLabel .label rect{opacity:1;rx:13px;ry:13px;fill:rgb(245, 250, 255);stroke:rgb(206, 219, 229);stroke-width:1px;}#chatgpt-mermaid-\_r_73q\_ .node rect,#chatgpt-mermaid-\_r_73q\_ .node circle,#chatgpt-mermaid-\_r_73q\_ .node ellipse,#chatgpt-mermaid-\_r_73q\_ .node polygon,#chatgpt-mermaid-\_r_73q\_ .node path{fill:rgb(229, 243, 255);stroke:rgba(0, 0, 0, 0.1);stroke-width:1px;}#chatgpt-mermaid-\_r_73q\_ .node rect{rx:16px;ry:16px;}#chatgpt-mermaid-\_r_73q\_ .mermaid-decision .label-container{fill:rgb(245, 250, 255);stroke:rgb(206, 219, 229);stroke-dasharray:2,2;}#chatgpt-mermaid-\_r_73q\_ .edgePaths .flowchart-link{stroke:rgb(143, 143, 143);stroke-width:1px;stroke-linecap:round;stroke-linejoin:round;}#chatgpt-mermaid-\_r_73q\_ .marker{fill:rgb(143, 143, 143);stroke:rgb(143, 143, 143);}#chatgpt-mermaid-\_r_73q\_ :root{--mermaid-font-family:-apple-system-body,ui-sans-serif,-apple-system,system-ui,"Segoe UI",Helvetica,"Apple Color Emoji",Arial,sans-serif,"Segoe UI Emoji","Segoe UI Symbol";}Worker 创建设备初始化分布式初始化模型加载显存 ProfilingKV Cache 初始化Compile / Warmup等待 Scheduler 调度execute_model()GPUModelRunner ForwardSampling返回执行结果

## **9. 源码阅读重点**

建议重点理解以下几个问题：

| 优先级 | 研究方向                           | 重点问题                            |
| --- | ------------------------------ | ------------------------------- |
| P0  | `init_device()`                | GPU、Rank、分布式通信如何初始化？            |
| P0  | `load_model()`                 | Worker 如何委托 ModelRunner 加载模型？   |
| P0  | `determine_available_memory()` | KV Cache 显存预算如何计算？              |
| P0  | `initialize_from_config()`     | KV Cache 如何分配和初始化？              |
| P0  | `execute_model()`              | SchedulerOutput 如何转化为 GPU 执行任务？ |
| P1  | `compile_or_warm_up_model()`   | CUDA Graph 如何捕获和预热？             |
| P1  | `sample_tokens()`              | Forward 和 Sampling 如何衔接？        |
| P1  | `sleep()` / `wake_up()`        | GPU 资源如何释放和恢复？                  |
| P2  | Weight Transfer                | 权重如何动态更新？                       |
| P2  | Elastic EP                     | 专家并行如何弹性调整？                     |

## **10. 核心总结**

```
Worker
  |
  |-- 设备管理：GPU Device / CUDA
  |
  |-- 分布式管理：Rank / TP / PP / DP
  |
  |-- 模型生命周期：Load / Compile / Warmup
  |
  |-- 显存管理：Profiling / KV Cache
  |
  |-- 推理执行：execute_model()
  |       |
  |       +-- GPUModelRunner
  |               |
  |               +-- Model Forward
  |               +-- KV Cache
  |               +-- Sampling
  |
  +-- 资源管理：Sleep / Wakeup / Checkpoint
```

**一句话理解：**

- `Worker`：管理 GPU 执行环境，是 GPU 执行层的入口。
- `GPUModelRunner`：负责模型实际执行，是推理计算的核心。
- `Scheduler`：决定执行哪些请求、执行多少 Token。
- `Worker` 和 `GPUModelRunner`：将调度决策转化为实际 GPU 计算。

**下一步建议：** 深入阅读 `GPUModelRunner`，重点跟踪 `execute_model()`、`execute_model_state`、`execute_model` 对应的 Forward、KV Cache 和 Sampling 调用链，串起 vLLM 从调度到 GPU 执行的完整过程。
