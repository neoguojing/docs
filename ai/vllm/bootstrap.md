# vLLM 启动全景 · 总纲

> 一张全景图看懂启动主线，各阶段只讲「是什么 / 为什么」，关键名词逐条解析

**一句话总纲**：vLLM 的启动不是「加载模型」，而是在构建一个高性能推理运行时——依次完成 配置模型 → 选执行后端 → 建并行环境 → 建结构 → 加载/量化权重 → 规划 CPU/GPU 权重 → 多模态 profiling → 编译计算图 → 捕获 CUDA Graph → 规划 KV Cache → Warmup，最终 Engine Ready。

## 断点测试

### 问题：
- vlm --omni 什么意思
- vllm bench 做什么？如何做

### 代码点：
#### /home/neo/workspace/vllm/vllm/entrypoints/cli/main.py  入口，注册以下cmd
vllm.entrypoints.cli.openai,
vllm.entrypoints.cli.serve,
vllm.entrypoints.cli.launch,
vllm.entrypoints.cli.benchmark.main,
vllm.entrypoints.cli.collect_env,
vllm.entrypoints.cli.run_batch,

几个关键点：
每个模块的注册：cmd_init() 返回CLISubcommand 对象
CLISubcommand：注册以下函数：
cmd() validate subparser_init（用于解析子命令参数）

#### /home/neo/workspace/vllm/vllm/entrypoints/cli/serve.py
ServeSubcommand.cmd()
```
[ 用户执行 `vllm serve [model] [options]` ]
                   │
                   ▼
       参数解析与校验 (validate_parsed_serve_args)
                   │
                   ▼
         命令行位置参数与配置对齐
                   │
                   ├─► 【条件 1】是否配置了 `--grpc`？
                   │      ├── Yes ──► 启动 gRPC 异步服务 (`serve_grpc`)
                   │      └── No  ──► 继续向下判断
                   │
                   ├─► 【条件 2】是否为 `--headless` 模式（或 `api_server_count < 1`）？
                   │      ├── Yes ──► 进入【无头/Worker 模式】(`run_headless`)
                   │      └── No  ──► 继续向下判断
                   │
                   ├─► 【条件 3】是否启用多端口外部负载均衡 (`--data-parallel-multi-port-external-lb`)？
                   │      ├── Yes ──► 进入【DP 监督者模式】(`run_dp_supervisor`)
                   │      └── No  ──► 继续向下判断
                   │
                   ├─► 【条件 4】是否满足 `api_server_count > 1` 或启用 Rust 前端？
                   │      ├── Yes ──► 进入【多 API Server 模式】(`run_multi_api_server`)
                   │      │            (初始化 ZMQ 通信与多进程/Rust 前端管理器)
                   │      │
                   │      └── No  ──► 【默认分支：单进程标准服务模式】
                   │                           │
                   │                           ▼
                   │                   启动单进程 FastAPI 服务 (`run_server`)
                   │
                   ▼
         【生命周期管理与优雅退出】
         (捕获 SIGTERM/SIGINT 信号，安全释放引擎与子进程)
```
```
             Supervisor
                  |
        +---------+---------+
        |         |         |
        v         v         v
     Server 0  Server 1  Server 2  Server 3
     Port 8000 Port 8001 Port 8002 Port 8003
        |         |         |         |
        v         v         v         v
     Rank 0    Rank 1    Rank 2    Rank 3

特点：
- 每个 Rank 有独立 API Server。
- 每个 API Server 监听独立端口。
- Supervisor 负责启动这些 Server。
- 外部可以直接访问对应端口，或者通过外部负载均衡器访问。
适合需要独立暴露每个 DP Rank 的场景。

                 Client
                    |
                    v
          +-------------------+
          |   Rust Frontend   |
          |                   |
          | Thread 1          |
          | Thread 2          |
          | Thread 3          |
          +-------------------+
                    |
                    v
                DP Engine
              /    |    \
          Rank 0 Rank 1 Rank 2
原因：

Rust Frontend 本身采用多线程架构，不需要通过启动多个 API Server 进程来实现并发处理。

注意：

这里的 1 表示一个 Rust Frontend 实例，而不是只能处理一个请求。

                   Client
                      |
                      v
              Nginx / Envoy / LB
                      |
          +-----------+-----------+
          |           |           |
          v           v           v
       Server 0    Server 1    Server 2
          |           |           |
          v           v           v
        Rank 0      Rank 1      Rank 2
这里的 1 指每个本地 DP Rank 配置一个 API Server。

外部 LB 负责：
- 请求分发。
- 负载均衡。
- 健康检查。
- 故障转移。
API Server 不需要再承担内部的 DP 负载均衡职责。

                  Client
                     |
                     v
               External LB
                     |
             +-------+-------+
             |               |
             v               v
          Node 0           Node 1
             |               |
       Internal LB      Internal LB
             |               |
       +-----+-----+     +---+-----+
       |     |     |     |   |     |
       v     v     v     v   v     v
      API   API   API   API API   API
       |     |     |     |   |     |
       v     v     v     v   v     v
      DP0   DP1   DP2   DP3 DP4   DP5
分发分成两级：
1. External LB：把请求分发到不同 Node。
2. Internal LB：在 Node 内部分发到本地 DP Rank。
这种模式适合多节点 DP 部署。

                  Client
                     |
                     v
               API Frontend
                     |
                Internal LB
                     |
          +----------+----------+
          |          |          |
          v          v          v
        Rank 0     Rank 1     Rank 2     Rank 3
其核心区别是：
- 不依赖外部 LB 进行 DP 分发。
- 由 vLLM 内部机制负责 DP 请求分发。
- API Server 数量按照完整 DP 规模设置。
```
#### /home/neo/workspace/vllm/vllm/entrypoints/launchers/api_server/entry.py
- run_server 函数入口
- EngineClient协议类，作为http后端，和底层engine通信，核心函数入generate，encode等
- AsyncLLM：
class AsyncLLM(EngineClient): AsyncLLM继承了该类
- 在 forkserver 启动时，先把 vllm 的核心推理引擎模块加载到内存里
好处：以后真正开子进程时，这些大模块已经加载好了，子进程可以直接拿来用，不用重新导入
启动一个服务进程，将vllm.v1.engine.async_llm模块的依赖提前加载好
    multiprocessing.set_forkserver_preload(["vllm.v1.engine.async_llm"])
AsyncLLM.from_vllm_config 创建async_llm
init_app_state 将各种变量挂载在fastapi服务器中，方便使用
engine = request.app.state.engine_client



```
运行 `run_server(args)`
                           │
                           ├─► 1. 注册 SIGTERM 终端信号拦截
                           ├─► 2. 调用 `setup_server()` 绑定 Socket 监听端口
                           │
                           ▼
             进入 `run_server_worker()` 异步上下文
                           │
                           ├─► 检查并导入插件 (Tool / Reasoning Parser)
                           │
                           ▼
               构建异步引擎客户端 (`build_async_engine_client`)
               │ (检查 forkserver 优化，实例化 `AsyncLLM`)
                           │
                           ▼
               构建 FastAPI 应用与服务 (`build_and_serve`)
               │  ├── 调用 `build_app()` 创建 FastAPI 实例
               │  ├── 调用 `init_app_state()` 将 `engine_client` 挂载到 `app.state`
               │  └── 调用 `serve_http()` 启动 Uvicorn HTTP 服务器
                           │
                           ▼
         【HTTP 服务运行中】 (接收 OpenAI 兼容请求，通过 `engine_client` 调用大模型)
                           │
                           ▼
               【优雅退出与资源清理】 (收到停止信号)
                ├── 等待 `shutdown_task` 完成
                └── 安全关闭 `async_llm` 引擎与 Socket 连接

async_llm = AsyncLLM.from_vllm_config(
            vllm_config=vllm_config,
            usage_context=usage_context,
            enable_log_requests=engine_args.enable_log_requests,
            aggregate_engine_logging=engine_args.aggregate_engine_logging,
            disable_log_stats=engine_args.disable_log_stats,
            client_addresses=client_config,
            client_count=client_count,
            client_index=client_index,
        )
```

#### /home/neo/workspace/vllm/vllm/v1/engine/async_llm.py
- 在init的函数中初始化类
- 如下创建engine_core client用于和其他通信：通过zmq
- 启动_run_output_handler 循环，接收底层消息
```
self.engine_core = EngineCoreClient.make_async_mp_client(
            vllm_config=vllm_config,
            executor_class=executor_class,
            log_stats=self.log_stats,
            client_addresses=client_addresses,
            client_count=client_count,
            client_index=client_index,
            renderer=renderer,
        )
```
#### /home/neo/workspace/vllm/vllm/v1/engine/core_client.py
- make_async_mp_client：创建AsyncMPClient 对象
- 调用MPClient._init_:
- AsyncMPClient：
- - 创建output——queue

```
`MPClient.__init__(...)` 初始化
                           │
                           ├─► 1. 初始化 ZMQ 上下文与资源管理器 (`BackgroundResources`, `weakref.finalize`)
                           ├─► 2. 检查多进程并行配置 (`enable_elastic_ep`) 决定是否启用 socket handover
                           │
                           ▼
             判断引擎管理模式 (`client_addresses`)
                           │
                           ├─► 【外部管理模式】
                           │    ├── 绑定传入的 `input_address` (ROUTER) 与 `output_address` (PULL)
                           │    ├── 通过管道回传实际绑定的 ZMQ 监听地址
                           │    └── 启动前端多模态预热 (`_start_mm_warmup()`)
                           │
                           └─► 【内部管理模式】
                                ├── 获取默认 ZMQ 引擎通信地址并解析占位符: ipc:///tmp/%%%
                                ├── 调用 `launch_core_engines()` 拉起后台多进程 EngineCore
                                └── 启动前端多模态预热 (`_start_mm_warmup()`)
                           │
                           ▼
             初始化编解码器与多进程 IPC
                           │
                           ├─► 实例化 `MsgpackEncoder` 与 `MsgpackDecoder`
                           ├─► 根据多模态配置初始化 `TensorIpcSender` (支持 torch_shm 共享内存)
                           └─► 计算数据并行 Rank 范围与 Core Engines 身份标识 (`core_engines`)
                           │
                           ▼
           【阻塞等待引擎就绪握手】
                           │
                           ├─► 轮询 `sync_input_socket.poll()` 等待所有后台 Engine 发送 Ready 信号
                           │   (受 `VLLM_ENGINE_READY_TIMEOUT_S` 控制，超时则抛出 `TimeoutError`)
                           └─► 收到就绪响应并调用 `_apply_ready_response()`
                           │
                           ▼
             启动引擎监控与收尾清理
                           │
                           ├─► 调用 `start_engine_core_monitor()` 监控后台核心进程状态
                           └─► 【异常保护】设置 `success = True`；若初始化中途失败，在 `finally` 触发清理
```

#### /home/neo/workspace/vllm/vllm/v1/engine/utils.py 
- launch_core_engines：核心启动函数
- CoreEngineProcManager__init__ 创建进程
- EngineCoreProc.run_engine_core： 新的进程的入口
- 
```
local_engine_manager = CoreEngineProcManager(
                vllm_config=vllm_config,
                executor_class=executor_class,
                log_stats=log_stats,
                handshake_address=handshake_address,
                client_handshake_address=client_handshake_address,
                local_client=True,
                local_engine_count=local_engine_count,
                start_index=dp_rank,
                local_start_index=local_start_index or 0,
                tensor_queue=tensor_queue,
            )
`launch_core_engines(vllm_config, ...)` 启动流程
                           │
                           ├─► 1. 检查多模态配置，必要时创建跨进程 `tensor_queue` (torch_shm)
                           ├─► 2. 检查并启动 `DPCoordinator` (仅当在线 DP 且 `dp_rank == 0` 时)
                           │
                           ▼
             判断数据并行后端类型 (`parallel_config.data_parallel_backend`)
                           │
                           ├─► 【Ray 后端】
                           │    ├── 实例化 `CoreEngineActorManager`
                           │    └── `yield CoreEngineLaunch` 并返回
                           │
                           └─► 【原生多进程后端】
                                ├── 根据 `offline_mode` 与 `dp_rank` 计算需要握手的引擎列表 (`engines_to_handshake`)
                                ├── 计算 `handshake_local_only` 与生成 ZMQ 握手地址 (支持 TCP 或 IPC)
                                └── 创建 ZMQ 握手 ROUTER Socket (`handshake_socket`)
                           │
                           ▼
             拉起本地引擎并进入上下文挂载
                           │
                           ├─► 若 `local_engine_count > 0`：实例化 `CoreEngineProcManager` (拉起子进程)
                           ├─► 封装 `CoreEngineLaunch` 并执行 `yield launch` (将控制权交回调用方)
                           └─► 执行 `wait_for_engine_startup()` 阻塞等待所有后台引擎完成握手连接

`CoreEngineProcManager.__init__(...)` 执行流程
                           │
                           ├─► 1. 获取多进程上下文并组装通用参数 (`common_kwargs`)
                           ├─► 2. 根据 `local_engine_count` 循环实例化 `context.Process` 
                           │   (目标函数: `EngineCoreProc.run_engine_core`)
                           ├─► 3. 注册 `weakref.finalize` 资源清理终结器
                           │
                           ▼
             遍历并逐个启动后台引擎子进程
                           │
                           ├─► 检查并配置 DP 物理 GPU 映射关系 (`set_assigned_physical_gpu_ids_for_dp_rank`)
                           ├─► 通过 `numa_utils.configure_subprocess` 配置 NUMA 亲和性
                           └─► 调用 `proc.start()` 真正拉起子进程
                           │
                           ▼
             【异常保护检查 (finally)】
                           │
                           ├─► 检查是否有进程启动失败 (`self.finished_procs()`)
                           └─► 若有失败：立即调用 `self.shutdown()` 清理并终止所有已拉起的进程
```

#### /home/neo/workspace/vllm/vllm/v1/engine/core.py
- run_engine_core
- EngineCoreProc的初始化函数和EngineCore的初始化函数
- - self.model_executor = executor_class(vllm_config) ：完成模型权重的加载
  - kv_cache_config = self._initialize_kv_caches(vllm_config) 完成kv cache创建
- engine_core.run_busy_loop()：进程进入循环
```
run_engine_core() [ 入口函数：后台进程启动 ]
│
├── [1. 进程环境与配置准备]
│    ├─ maybe_register_config_serialize_by_value() ──> 注册配置序列化
│    ├─ 检测数据并行 (DP) ──> 设置进程标题 (EngineCore / EngineCore_DP{rank})
│    ├─ 检查 numa_bind ──> 记录 NUMA 亲和性状态
│    └─ KV 传输隔离 ──> 动态修改 kv_transfer_config.engine_id (追加 _dp 标识)
│
├── [2. 引擎核心实例化分支]
│    ├─ MoE 模型 ──> 设置 data_parallel_rank ──> 实例化 DPEngineCoreProc
│    └─ 非 MoE ──> reconfigure_for_independent_dp_rank() ──> 实例化 EngineCoreProc
│
└── [3. EngineCoreProc.__init__ 初始化]
     ├─ 创建 ZMQ 队列、Identity 与多模态 Tensor 接收器
     ├─ 执行 ZMQ 握手与 DP 负载均衡/协调器配置
     ├─ 启动后台 I/O 线程:
     │    ├─ input_thread (process_input_sockets) ──> 异步接收 ZMQ 输入并写入队列
     │    └─ output_thread (process_output_sockets) ──> 异步发送输出结果
     └─ 等待 DP Coordinator 的 READY 信号 (ready_event.wait)
          │
          ▼
     [4. 父类 EngineCore.__init__ 初始化]
          ├─ 加载通用插件 (load_general_plugins)
          ├─ 初始化模型执行器 (model_executor) 与失败回调
          ├─ KV 缓存 Profiling 与显存分配 (_initialize_kv_caches)
          ├─ 实例化调度器 (Scheduler) 与前缀缓存哈希器
          ├─ 初始化多模态接收器与流水线并行批处理队列 (batch_queue)
          └─ 系统优化: 冻结 Python GC 堆 (freeze_gc_heap) 减少 GC 暂停
               │
               ▼
          [5. run_busy_loop() ──> 核心业务忙碌循环 ]
               │
               while self._handle_shutdown(): ──> 持续监听关闭信号
               ├─ 1) self._process_input_queue() ──> 处理输入队列中的新请求/指令
               ├─ 2) self._maybe_publish_request_counts() (前置) ──> 发布队列统计给负载均衡器
               ├─ 3) self._process_engine_step() ──> 执行调度与 GPU 模型前向计算
               └─ 4) self._maybe_publish_request_counts() (后置) ──> 更新发布最新统计状态

```
#### /home/neo/workspace/vllm/vllm/v1/executor/uniproc_executor.py
-  self.driver_worker = WorkerWrapperBase(rpc_rank=0) 创建worker
```
[ 开始: UniProcExecutor._init_executor ]
         │
         ▼
[ 1. 创建驱动 Worker 包装器 ]
  └─ self.driver_worker = WorkerWrapperBase(rpc_rank=0)
         │
         ▼
[ 2. 收集分布式参数与构建 kwargs ]
  ├─ 获取 distributed_init_method, rank, local_rank
  └─ 组装 vllm_config、is_driver_worker、shared_worker_lock 等参数
         │
         ▼
[ 3. 配置网卡设备环境变量 ]
  └─ set_worker_net_device(local_rank, self.vllm_config) (若适用)
         │
         ▼
[ 4. 初始化 Worker 与计算设备 ]
  ├─ self.driver_worker.init_worker(all_kwargs=[kwargs])
  └─ self.driver_worker.init_device()
         │
         ▼
 ┌────────────────────────────────────────────────────────┐
 │ 检查是否启用弹性 EP 扩缩容 launch                      │
 │ (envs.VLLM_ELASTIC_EP_SCALE_UP_LAUNCH)                 │
 └───────────────────────┬────────────────────────────────┘
                         │
          ┌──────────────┴──────────────┐
          │                             │
       [ 是 ]                        [ 否 ]
          │                             │
          ▼                             ▼
  [ elastic_ep_execute ]        [ self.driver_worker ]
  [ ("load_model")     ]        [ .load_model()      ]
          │                             │
          └──────────────┬──────────────┘
                         │
                         ▼
[ 5. 更新后端块大小 ]
  └─ current_platform.update_block_size_for_backend(self.vllm_config)
         │
         ▼
[ 结束: UniProcExecutor 初始化完成 ]
```
#### /home/neo/workspace/vllm/vllm/v1/worker/worker_base.py
- WorkerWrapperBase.init_worker
- worker_class(**kwargs) : 创建worker
```
[ 开始: init_worker ]
         │
         ▼
[ 1. 提取当前 Rank 的参数 ]
  └─ kwargs = all_kwargs[self.rpc_rank]
         │
         ▼
[ 2. 验证并保存 vllm_config ]
  └─ 确保 vllm_config 不为空（否则断言报错）
         │
         ▼
[ 3. 初始化全局环境 ]
  ├─ 开启线程追踪: enable_trace_function_call_for_thread()
  └─ 加载插件: load_general_plugins()
         │
         ▼
[ 4. 解析 Worker 类 (worker_cls) ]
  └─ 通过 resolve_obj_by_qualname 解析全限定名类
         │
         ▼
 ┌────────────────────────────────────────────────────────┐
 │ 检查是否有 Worker 扩展类 (worker_extension_cls)         │
 └───────────────────────┬────────────────────────────────┘
                         │
          ┌──────────────┴──────────────┐
          │                             │
       [ 是 ]                        [ 否 ]
          │                             │
          ▼                             ▼
  [ 检查属性冲突 ]               [ 跳过 ]
  [ 动态混入 worker_class.__bases__ ]
          │                             │
          └──────────────┬──────────────┘
                         │
                         ▼
[ 5. 处理物理 GPU ID 与多模态锁 ]
  ├─ 更新 assigned_physical_gpu_ids
  └─ 检查 shared_worker_lock (缺失时若为 shm 则报错，否则警告)
         │
         ▼
[ 6. 初始化多模态接收器缓存 ]
  └─ mm_receiver_cache = MULTIMODAL_REGISTRY.worker_receiver_cache_from_config(...)
         │
         ▼
[ 7. 实例化真实的 Worker 对象 ]
  └─ 在 set_current_vllm_config 上下文中执行: self.worker = worker_class(**kwargs)
         │
         ▼
[ 结束: init_worker 执行完毕 ]
```
#### /home/neo/workspace/vllm/vllm/v1/worker/gpu_worker.py
- init_device: 执行设备初始化
- load_model: 执行模型加载
```
[ 开始: init_device ]
         │
         ▼
 ┌────────────────────────────────────────┐
 │ 设备类型是否为 "cuda" ？                │
 └───────────────────┬────────────────────┘
                     │
          ┌──────────┴──────────┐
       [ 否 ]                [ 是 ]
          │                     │
          ▼                     ▼
  [ 抛出 RuntimeError ]  [ 1. 清理 Ray 的 NCCL 异步异常环境变量 ]
                         │
                         ▼
                       [ 2. 计算调整后的 local_rank ]
                         │ (结合数据并行 DP rank 与 TP/PP 世界大小)
                         ▼
                       [ 3. 校验并绑定物理 GPU ID (assigned_physical_gpu_ids) ]
                         │
                         ▼
                       [ 4. 获取可见设备索引并设置 PyTorch 当前设备 ]
                         │
                         ▼
                       [ 5. 校验数据类型支持度 (check_if_supports_dtype) ]
                         │
                         ▼
                       [ 6. 初始化分布式通信环境 (NCCL) ]
                         │ (确保 NCCL 缓冲区在显存分析前分配完成)
                         ▼
                       [ 7. 设置随机种子 (set_random_seed) ]
                         │
                         ▼
                       [ 8. 显存清理与内存快照采集 ]
                         │ (gc.collect() -> empty_cache() -> MemorySnapshot)
                         ▼
                       [ 9. 初始化工作空间管理器 (init_workspace_manager) ]
                         │
                         ▼
                       [ 10. 构建模型运行器 (Model Runner) ]
                         ├─ V2: GPUModelRunner / MMEncoderModelRunner
                         └─ V1: GPUModelRunnerV1
                         │
                         ▼
 ┌────────────────────────────────────────┐
 │ 如果 rank == 0：上报使用统计信息        │
 └───────────────────┬────────────────────┘
                     │
                     ▼
             [ 结束: 设备初始化完成 ]

[ 开始: load_model ]
         │
         ▼
 ┌────────────────────────────────────────────────────────┐
 │ 进入多重上下文环境 (with 语句)                          │
 │  ├─ 1. _maybe_get_memory_pool_context(tag="weights")    │
 │  ├─ 2. set_current_vllm_config(self.vllm_config)       │
 │  └─ 3. _scoped_allocator_max_split(max_split_size_mb=20) │
 └───────────────────────┬────────────────────────────────┘
                         │
                         ▼
[ 1. 调用模型运行器加载模型权重 ]
  └─ self.model_runner.load_model(load_dummy_weights=load_dummy_weights)
                         │
                         ▼
 ┌────────────────────────────────────────┐
 │ 是否启用 EC 传输 (has_ec_transfer()) ？  │
 └───────────────────┬────────────────────┘
                     │
          ┌──────────┴──────────┐
       [ 是 ]                [ 否 ]
          │                     │
          ▼                     ▼
  [ 启动 Worker 服务的 ]       [ 跳过 ]
  [ EC 传输服务: ]
  [ get_ec_transfer() ]
  [ .start_worker_services() ]
          │
          └──────────┬──────────┘
                     │
                     ▼
 ┌────────────────────────────────────────┐
 │ 是否配置了 weight_transfer_config ？    │
 └───────────────────┬────────────────────┘
                     │
          ┌──────────┴──────────┐
       [ 是 ]                [ 否 ]
          │                     │
          ▼                     ▼
  [ 创建并初始化权重传输引擎 ]  [ 跳过 ]
  [ WeightTransferEngineFactory ]
  [ .create_engine(...) ]
          │
          └──────────┬──────────┘
                     │
                     ▼
             [ 结束: 模型加载完成 ]
```

#### /home/neo/workspace/vllm/vllm/v1/worker/gpu/model_runner.py
- GPUModelRunner.__init__
- load_model 真正执行模型加载
```
为您整理了 `GPUModelRunner.__init__` 构造函数中设置的核心参数与对象列表：

### 1. 配置参数引用（Config References）

* `self.vllm_config`：vLLM 整体配置对象。
* `self.model_config`：模型架构、词表、最大长度等配置。
* `self.cache_config`：KV 缓存类型、量化及内存配置。
* `self.compilation_config`：编译优化相关配置。
* `self.lora_config`：LoRA 动态微调配置。
* `self.load_config`：模型权重加载配置。
* `self.parallel_config`：分布式并行配置（TP、PP、DP、EP 等）。
* `self.scheduler_config`：调度器配置（批处理大小、Token 上限等）。
* `self.speculative_config`：投机解码配置。
* `self.observability_config`：可观测性与监控配置。

### 2. 计算设备与数据类型（Device & Dtype）

* `self.device`：绑定的 PyTorch 计算设备（如 `cuda:0`）。
* `self.dtype`：模型的基础数据类型（如 FP16 / BF16）。
* `self.kv_cache_dtype`：KV 缓存的数据类型（若开启量化则为量化类型，否则与 `dtype` 一致）。
* `self.kv_block_zeroer`：KV 块清零器（用于 FP8 等混合模型的 KV 缓存初始化）。

### 3. 模型基础尺寸与流控（Dimensions & Streams）

* `self.vocab_size`：词表大小。
* `self.max_model_len`：最大上下文长度限制。
* `self.max_num_tokens`：单批次最大 Token 数（`max_num_batched_tokens`）。
* `self.max_num_reqs`：单批次最大请求数（`max_num_seqs`）。
* `self.is_encoder_decoder`：是否为 Encoder-Decoder 架构模型。
* `self.output_copy_stream`：专用的 CUDA 异步输出流（用于重叠拷贝与计算）。

### 4. 并行策略参数（Distributed Parallelism）

* **流水线并行 (PP)**：
* `self.use_pp`：是否启用流水线并行。
* `self.is_first_pp_rank` / `self.is_last_pp_rank`：当前 Rank 是否为首个或末尾 PP 节点。
* `self.pp_handler`：PP 节点间通信辅助器。
* `self.intermediate_tensors`：PP 中间张量持久化缓冲区。


* **数据并行 (DP) 与专家并行 (EP)**：
* `self.dp_size` / `self.dp_rank`：数据并行大小与当前 Rank。
* `self.check_ep_fault`：MoE + DP 架构下的 EP All2All 容错检测开关。


* **解码上下文并行 (DCP)**：
* `self.use_dcp` / `self.dcp_size` / `self.dcp_rank`：DCP 相关状态。
* `self.cp_interleave`：KV 缓存交织大小。



### 5. 高级特性与生态支持（Advanced Features）

* **多模态与投机解码**：
* `self.supports_mm_inputs` / `self.uses_inputs_embeds`：多模态输入支持标志。
* `self.encoder_cache` / `self.ec_connector`：多模态编码器缓存与连接器。
* `self.speculator`：投机解码草稿模型运行器。
* `self.use_aux_hidden_state_outputs`：是否需要输出辅助隐藏状态。
* `self.draft_tokens_handler`：草稿 Token 传播处理器。


* **状态管理与缓冲区**：
* `self.req_states`：通用请求状态管理器。
* `self.input_buffers`：输入张量缓冲区。
* `self.is_pooling_model` / `self.pooling_runner`：Pooling 模型的支持标记与运行器。


* **专家负载均衡与 LoRA**：
* `self.eplb`：EPLB 专家并行负载均衡控制器。
* `self.lora_state` / `self.lora_capture_cases`：LoRA 运行状态与捕获用例。
* `self.kv_connector`：KV 传输连接器。

[ 开始: GPUModelRunner.load_model ]
         │
         ▼
[ 1. 准备加载与计时 ]
  ├─ 记录起始时间 (time.perf_counter)
  ├─ 若 load_dummy_weights 为真，设置 load_format = "dummy"
  └─ self.eplb.prepare_load()
         │
         ▼
 ┌────────────────────────────────────────────────────────┐
 │ 使用 DeviceMemoryProfiler 显存分析器加载模型             │
 │  ├─ 获取模型加载器 (model_loader)                      │
 │  ├─ 激活 JitWarmupRegistry，加载主模型 (model_loader.load_model) │
 │  ├─ 若有 LoRA 配置，加载 LoRA 模型                        │
 │  ├─ 若启用 aux 隐藏状态，配置 Eagle3/PP 隐藏状态中继         │
 │  └─ 若有草稿模型投机解码器 (DraftModelSpeculator)，加载模型并注册 EPLB │
 └───────────────────────┬────────────────────────────────┘
                         │
                         ▼
[ 2. 统计并记录模型显存开销与加载耗时 ]
         │
         ▼
[ 3. 初始化模型状态与解码参数 ]
  ├─ self.model_state = init_model_state(...)
  ├─ 计算 decode_query_len (投机步数 + 每步新采样 Token 数)
  └─ 若启用批分片采样 (batch_sharded_sampling)，初始化 BatchSharder
         │
         ▼
 ┌────────────────────────────────────────────────────────┐
 │ 是否为末尾流水线并行节点 (is_last_pp_rank) 且非 Pooling 模型？ │
 └───────────────────────┬────────────────────────────────┘
                         │
          ┌──────────────┴──────────────┐
          │                             │
       [ 是 ]                        [ 否 ]
          │                             │
          ▼                             ▼
  [ 4. 初始化采样器与辅助工作器 ]       [ 跳过 ]
   ├─ 根据 watermark_config 创建标准 Sampler 或 GPUWatermarkSampler
   ├─ 检查自定义采样器 (custom_sampler) 或创建 RejectionSampler (投机解码)
   ├─ 初始化 PromptLogprobsWorker (提示词对数概率工作器)
   └─ 初始化 StructuredOutputsWorker (结构化输出工作器)
          │
          └──────────────┬──────────────┘
                         │
                         ▼
 ┌────────────────────────────────────────┐
 │ 是否为 Pooling 模型且处于末尾 PP ランク？ │
 └───────────────────┬────────────────────┘
                     │
          ┌──────────┴──────────┐
       [ 是 ]                [ 否 ]
          │                     │
          ▼                     ▼
  [ 初始化 PoolingRunner ]      [ 跳过 ]
          │                     │
          └──────────────┬──────────────┘
                         │
                         ▼
[ 5. EPLB 专家负载均衡注册与异步循环启动 ]
  ├─ self.eplb.maybe_register_model(...)
  └─ self.eplb.maybe_start_async_loop(eplb_models_added)
                         │
                         ▼
 ┌────────────────────────────────────────┐
 │ 是否为非首个流水线并行节点 (!is_first_pp_rank)？│
 └───────────────────┬────────────────────┘
                     │
          ┌──────────┴──────────┐
       [ 是 ]                [ 否 ]
          │                     │
          ▼                     ▼
  [ 创建持久化中间张量缓存 ]    [ 跳过 ]
  [ self.intermediate_tensors ]
          │                     │
          └──────────────┬──────────────┘
                         │
                         ▼
[ 6. 卸载器后置初始化 ]
  └─ get_offloader().post_init()
                         │
                         ▼
[ 结束: load_model 执行完成 ]
```
#### /home/neo/workspace/vllm/vllm/entrypoints/launchers/launcher.py
启动http服务
```
[ 调用 `serve_http()` 启动服务 ]
                   │
                   ├─► 1. 遍历并打印 FastAPI 所有注册的路由和方法
                   ├─► 2. 配置 Uvicorn 参数 (如 H11 协议头大小、并发限制等)
                   ├─► 3. 实例化 `NoSignalServer` (接管底层服务，禁用 Uvicorn 自带信号)
                   │
                   ▼
       初始化并创建两大核心异步任务 (`loop.create_task`)
                   │
                   ├─► 1. `watchdog_loop` (后台看门狗)
                   │      └── 每隔 5 秒检查一次引擎状态，若引擎意外崩溃则触发退出
                   │
                   └─► 2. `server_task` (Uvicorn 服务任务)
                          └── 在绑定的 Socket 上开始监听并处理客户端的 HTTP 请求
                   │
                   ▼
         注册系统信号监听 (`SIGINT` / `SIGTERM`)
                   │
                   ▼
         【HTTP 服务正常运行中 ⇄ 持续响应请求】
                   │
                   ├─► 收到中断信号 (如用户按下 `Ctrl+C` 或收到 `SIGTERM`)
                   │         │
                   │         ▼
                   │   触发 `handle_shutdown()` 优雅退出流程 (Graceful Shutdown)
                   │         ├── ① 停止大模型引擎客户端 (`engine_client.shutdown`)
                   │         ├── ② 取消 `server_task` 和 `watchdog_task`
                   │         └── ③ 释放网络 Socket 资源
                   │
                   └─► 异常捕获与强制清理
                          └── 确保最终安全关闭 Uvicorn 服务并退出进程
```

## 一、启动全景流程

主线来自真实启动日志（vLLM V1 Engine + Gemma 多模态 + AWQ-4bit + Prefetch Offload + Triton Attention + torch.compile + CUDA Graph + KV Cache）：

```
vllm serve
 │  ① API Server：解析参数 → EngineArgs → 各类 Config
 ▼  ② EngineCore：创建 V1 LLM Engine（独立进程，与 API 层解耦）
 ▼  ③ Parallel/Device：初始化 TP/PP/DP 与 NCCL
 ▼  ④ Model Runner：按 config 构建模型结构
 ▼  ⑤ Weight Loader：safetensors → GPU，AWQ 走 Marlin Kernel
 ▼  ⑥ Offload：CPU↔GPU 权重预取，为 GPU 腾显存
 ▼  ⑦ Multimodal Warmup：Vision Encoder profiling
 ▼  ⑧ torch.compile：Dynamo → Inductor 生成优化 kernel
 ▼  ⑨ CUDA Graph Capture：按 batch/shape 批量捕获图
 ▼  ⑩ KV Cache Profile：测算 GPU 可容纳 token 数
 ▼  JIT Warmup → 第二轮 CUDA Graph → Engine Ready → OpenAI API
```

## 二、各阶段核心概念速查（是什么 · 为什么）

| 阶段 | 是什么 | 为什么 |
| --- | --- | --- |
| **① API Server 参数解析** | 入口层。命令行参数 → EngineArgs → 各类 Config，再交给 EngineCore。此时还没真正跑 Transformer。 | 把用户输入统一成内部配置：模型/长度/dtype/量化/GPU 数/并行/KV Cache/backend 等，Engine 才能据此决策。 |
| **② 解析 Model** | 读 config.json → 解析 architecture → 命中对应实现类（Gemma4ForConditionalGeneration）。 | vLLM 不靠模型名猜结构，而是按声明的 architecture 动态选择模型实现，便于扩展新模型。 |
| **③ Scheduler 与 Chunked Prefill** | max_model_len=单请求最大长度(16000)；max_num_batched_tokens=一次调度最多处理的 token(2496)。二者勿混淆。 | 一次吞下超长 prompt 会占满算力并阻塞 decode。把大 prefill 切成 chunk，让 prefill 与 decode 更易混排调度。 |
| **④ Attention Backend** | Attention 有多种后端实现（FlashAttention / FlashInfer / Triton…），按 GPU、head dim、dtype、KV layout 选择。 | Gemma 异构 head dim（sliding 256 / full 512）→ FA4 不支持 → fallback 到 TRITON_ATTN。 |
| **⑤ EngineCore（进程解耦）** | APIServer 与 EngineCore 是两个进程（如 pid 79016 / 79241）：前者管 HTTP/OpenAI，后者管 GPU 推理。 | V1 核心思想：API 层与执行层解耦，请求调度与 GPU 计算互不阻塞。 |
| **⑥ Parallel State** | 初始化 TP/PP/DP 并行维度（本例均为 1）与通信后端 NCCL。 | 即便单卡也统一初始化并行抽象：抽象统一、便于扩到多卡，通信层一致。 |
| **⑦ Model Runner / 建模型结构** | "Loading from scratch" 不是从头训练，而是按 config 建 PyTorch Module，再灌入 checkpoint 权重。 | 结构与权重分离：先有骨架（Vision Tower/Embedding/Layers/Attention/MLP/LM Head），再填参数。 |
| **⑧ 量化 Kernel** | AWQ-4bit 权重 → Compressed-Tensors → MarlinLinearKernel → GPU GEMM，而非普通 FP16 Linear。 | 既省权重显存，又用专用 CUDA kernel 保持量化 Linear 的速度。 |
| **⑨ Weight Loader + dtype 转换** | safetensors 分片(4 shards)经 mmap 加载到 GPU。大量 FP16→BF16 warning = checkpoint dtype ≠ 参数 dtype。 | 加载时按模型参数 dtype 做转换对齐；本例多集中在 vision_tower 部分，属正常匹配，非报错。 |
| **⑩ Prefetch Offload** | 不把全部权重常驻 GPU；部分权重留在 CPU，执行前预取(prefetch)上 GPU，用完让位下一组。 | 24GB 显存装不下 19.47GiB 权重 + KV Cache + activation + CUDA Graph，用 CPU 换 GPU 显存，代价是 PCIe 传输延迟。 |
| **⑪ Multimodal Warmup** | 模型是 Image/Video → Vision Encoder → 多模态 Embedding → LLM，启动时预建 encoder 并分配 encoder cache、profile。 | 多模态输入规模不确定，需提前 warmup 定好预算（本例 encoder cache 预算 2496 tokens），避免首请求抖动。 |
| **⑫ torch.compile** | Dynamo 抓取计算图 → Inductor 生成优化 kernel（本例耗时 72s，是最大耗时点之一）。 | 消除 Python/dispatcher 逐算子开销，做算子融合与 kernel 优化，直接提升吞吐、降低延迟。 |
| **⑬ CUDA Graph** | 把一串 kernel 打包成图一次提交；需按不同 batch/token shape 各捕获一张图（1,2,4,…,512）。 | 省掉每次 kernel launch 的 CPU 开销。运行时按当前 batch size 找对应图直接 replay。 |
| **⑭ KV Cache Profile** | 把过去 token 的 K/V 缓存复用，按 block 分块管理（block size 32）。启动测算可得 KV 容量(本例 17059 tokens)。 | KV Cache 是 LLM 推理核心瓶颈。分块管理实现细粒度、可共享的显存复用；决定最大并发（本例 ≈1.07× 一个满长请求）。 |

## 三、核心名词解析（是什么 · 为什么）

> 上面速查表把名词「用」了一遍，这里把它们逐条「讲清」。按所属阶段归类，仍只讲是什么 / 为什么。

### 1. Attention 后端类（对应阶段 ④）

| 名词 | 是什么 | 为什么 |
| --- | --- | --- |
| **Attention Backend** | 计算注意力（Attention）的具体 GPU kernel 实现，vLLM 里做成可插拔「后端」。 | 不同硬件、head dim、dtype、KV 内存布局下最优实现不同，所以需要多套后端并按条件择优/回退。 |
| **FlashAttention（FA2/FA3/FA4）** | 一种「IO-aware」的精确注意力算法/kernel：把序列分块(tiling)在片上缓存 SRAM 里逐块算 softmax，不把 N×N 注意力大矩阵写回显存 HBM。 | 显存占用从 O(N²) 降到 O(N)、大幅减少访存往返，长序列提速明显；代价是对 head dim、内存布局、硬件版本有白名单约束。 |
| **FlashInfer** | 面向 LLM 推理的高性能 Attention / KV-Cache kernel 库，支持变长(varlen)、分页(paged)、量化 KV，并可按需定制 kernel。 | 针对 prefill/decode 不同特性做了更激进的优化，提供比通用实现更高的吞吐，是 FA 之外的另一条高性能后端路线。 |
| **Triton / TRITON_ATTN** | Triton 是 OpenAI 开源的 GPU DSL（用 Python 方言写 CUDA kernel）；TRITON_ATTN 指用 Triton 实现的那套 attention 后端。 | 当 FA4 不支持某些 head dim（如本例 sliding 256 / full 512 异构）时，Triton 能灵活生成兼容 kernel，作为通用回退方案。 |
| **head dim / 异构 head dim** | 每个注意力头的向量维度（head_dim）。Gemma 的滑窗层与全注意力层 head_dim 不同（256 vs 512），即「异构」。 | 后端只对特定 head_dim 做了向量化优化并列入白名单，异构值不在支持列表就会触发 fallback。 |
| **SRAM / HBM** | SRAM=SM 上的片上高速缓存（小、快）；HBM=高带宽显存（大、相对慢），即平时说的「显存」。 | FlashAttention 的核心思路就是尽量在 SRAM 内完成计算、少往返 HBM，这是它省显存提速的物理基础。 |

### 2. 量化类（对应阶段 ⑧）

| 名词 | 是什么 | 为什么 |
| --- | --- | --- |
| **量化（Quantization）/ 4-bit** | 把权重从 16-bit（FP16/BF16）压到更低比特（如 4-bit）表示。 | 显存占用与访存带宽成倍下降，推理更快更省；难点是压完不掉精度、且要跑得比反量化再算更快。 |
| **AWQ（Activation-aware Weight Quantization）** | 一种 4-bit 权重量化方法：依据激活值分布，识别并「保护」少数关键权重通道（给更高精度/缩放），其余统一低比特。 | 在几乎不掉精度前提下把权重压到约 1/4，是当前主流 4-bit 量化算法之一。 |
| **Compressed-Tensors** | 一种量化模型的标准化打包与加载格式/框架（Neural Magic 提出）：用元数据描述量化方案、scale、group size 等。 | 不同量化算法（AWQ/GPTQ 等）的 checkpoint 描述五花八门，统一格式后 vLLM 能据元数据还原量化结构并选对 kernel。 |
| **Marlin / MarlinLinearKernel** | 针对 INT4（AWQ/GPTQ）权重 + FP16 激活的高性能 GEMM CUDA kernel：在 kernel 内部直接处理量化权重做混合精度矩阵乘。 | 若先「反量化成 FP16 再矩阵乘」会多一次大张量读写、反而变慢；Marlin 让量化 Linear 的速度逼近甚至超过 FP16，真正把量化变成提速而非纯省显存。 |
| **GEMM** | General Matrix Multiply，通用矩阵乘（Linear/全连接层的计算本质）。 | 矩阵乘是 LLM 前向的计算主体，量化带来的显存/带宽收益最终主要落在 GEMM 上，所以量化 kernel 的质量直接决定性能。 |
| **Linear 层（FP16 Linear）** | 全连接矩阵乘层，是模型里数量最多的可计算层。 | 量化版与普通版的差别就体现在这里——同一层是用普通 FP16 Linear，还是走 Marlin 这类量化 kernel。 |

### 3. 权重加载类（对应阶段 ⑤⑨）

| 名词 | 是什么 | 为什么 |
| --- | --- | --- |
| **safetensors** | HuggingFace 推出的安全权重文件格式：张量名→张量的扁平映射，不含 pickle、支持 mmap/懒加载与分片。 | 比传统 .bin/pickle 更安全（不执行任意代码）、加载更快、天然支持分片与按需读取。 |
| **shard / 分片（4 shards）** | 权重太大时拆成多个 safetensors 文件（如 model-00001-of-00004…）。 | 单文件过大不利分发与并行加载，分片后可多线程/多进程并行读、降低单次内存峰值。 |
| **mmap（内存映射）** | 把文件按页映射进地址空间，访问到哪页才读哪页。 | 不必一次性把整份权重读进内存，加载更快、内存峰值更低，适合超大 checkpoint。 |
| **dtype / FP16 / BF16** | 数据精度类型。FP16=半精度（10 位尾数，范围小）；BF16=「脑浮点」（8 位指数位，动态范围与 FP32 同、尾数更短）。 | checkpoint 存的 dtype 可能与模型参数 dtype 不一致，加载时需转换对齐（大量 FP16→BF16 warning 即源于此）。BF16 动态范围大、不易溢出，是 Ampere+ GPU 的常用训练/推理精度，属正常匹配而非报错。 |
| **Weight Loader / default_loader** | 负责把 checkpoint 读到 GPU、并把张量正确映射进模型各参数的组件。 | 结构与权重分离，需要一个统一的「灌权重」环节处理分片、dtype 转换、量化还原、CPU/GPU 放置等杂务。 |
| **Loading from scratch** | 先按 config 搭一个空的模型结构，再把 checkpoint 权重灌进去——不是「从头训练」。 | 强调「结构」与「权重」是两件事：日志里的 scratch 指先建骨架，随后才加载参数，避免误解为重新训练。 |

### 4. 编译与计算图类（对应阶段 ⑧⑨）

| 名词 | 是什么 | 为什么 |
| --- | --- | --- |
| **torch.compile** | PyTorch 2.x 的图编译入口，把逐行执行的 eager 代码编译成优化后的 kernel。 | 消除 Python/dispatcher 逐算子调度开销、做算子融合，提升吞吐降低延迟；本例耗时约 72s，是启动最大耗时点之一。 |
| **Dynamo** | torch.compile 的前端：抓取 Python 字节码，构建出可优化的计算图。 | 把动态、带分支的 Python 代码「翻译」成可优化的静态图，是后续 kernel 生成的前提。 |
| **Inductor** | torch.compile 的后端代码生成器，产出 Triton / C++ 优化 kernel。 | 在图层面做算子融合与内存规划，生成高效 kernel，是真正产生性能收益的环节。 |
| **kernel / 算子融合** | kernel=GPU 上执行的一段计算程序；融合=把多个连续算子合并进一个 kernel。 | 减少 kernel launch 次数、避免中间结果反复往返 HBM，是 Inductor 提速的主要手段。 |
| **CUDA Graph** | 把一串固定的 kernel 序列「捕获」成一张图，之后一次性「replay」提交执行。 | 省掉每次单独 launch 的 CPU 开销；小 kernel 数量多、batch 小的时候收益尤为明显。因图内 shape 固定，需按不同 batch/shape 各捕获一张图（1,2,4,…,512）。 |
| **capture / replay / kernel launch** | launch=CPU 向 GPU 提交一个 kernel；capture=录制一串 kernel 成图；replay=整图一次提交。 | 逐 kernel launch 的 CPU 开销在 decode 小 batch 时会成为瓶颈，CUDA Graph 用「一次提交」摊薄这部分开销。 |
| **JIT Warmup** | just-in-time 预热：先空跑/真实跑一遍，触发延迟编译与缓存填充。 | 把编译、内存分配等一次性开销放到启动阶段完成，避免首个真实请求时卡顿抖动。 |

### 5. KV Cache 与调度类（对应阶段 ③⑩⑭）

| 名词 | 是什么 | 为什么 |
| --- | --- | --- |
| **KV Cache** | 缓存历史 token 计算出的 Key/Value 张量，下一步生成时直接复用、不重复算。 | decode 每步只需算「新 token」的 K/V，这是自回归提速的关键；代价是它随序列长度增长、最吃显存，是 LLM 推理核心瓶颈。 |
| **PagedAttention / block / block size 32 / Block Table** | 像操作系统分页那样管理 KV Cache：切成固定大小 block（本例 32），用「页表」Block Table 记录每个请求用了哪些物理块。 | 彻底消除按连续大块预分配造成的碎片与浪费，做到按需分配、块间可共享，从而提升显存利用率与并发。 |
| **prefix caching** | 复用不同请求之间「相同前缀」已有的 KV Cache。 | 多个请求共享同一 system prompt / few-shot 前缀时，省去重复的 prefill 计算，降延迟提吞吐。 |
| **Prefill / Decode** | prefill=并行处理整段 prompt、算出全部 K/V（算力密集）；decode=逐 token 自回归生成（访存密集）。 | 两阶段计算特性差异巨大，把它们高效混排调度是连续批处理与延迟控制的核心难点。 |
| **Chunked Prefill** | 把过长的 prefill 切成若干 chunk，分多个调度步处理。 | 防止一个超长 prompt 独占一步调度、阻塞正在进行的 decode，从而压低 decode 的延迟抖动。 |
| **continuous batching** | 请求级动态组批：有请求算完就立即退出，新请求随时补入同一批。 | 不必等整批齐步，GPU 利用率和吞吐更高，是 vLLM 高吞吐的关键机制之一。 |
| **Scheduler** | 决定每一步「跑哪些请求、各处理多少 token」的调度器。 | 在吞吐（多塞请求）与延迟（别饿死 decode）之间做实时权衡，是 Engine 的大脑。 |
| **max_model_len / max_num_batched_tokens** | 前者=单请求最大上下文长度(16000)；后者=单步最多调度的 token 数(2496)。 | 一个约束「单请求能有多长」，一个约束「单步算力预算多大」，混淆二者会导致对显存/吞吐的误判。 |
| **gpu_memory_utilization** | 允许 vLLM 占用的总显存比例（本例 0.9）。 | 不能设成 1.0：必须给 Activation、CUDA Graph、内存碎片留余量，否则运行期 OOM。 |

### 6. 多模态与并行/进程类（对应阶段 ⑥⑦⑪）

| 名词 | 是什么 | 为什么 |
| --- | --- | --- |
| **Vision Encoder / Vision Tower** | 把图像/视频编码成 embedding 向量的子模块（常为 ViT），输出喂给 LLM 主干。 | 多模态模型需要先把非文本输入转成与文本同空间的 embedding，再走统一 LLM。 |
| **encoder cache / Multimodal Warmup** | 缓存视觉编码结果；启动时预建 encoder 并 profile 出可分配预算（本例 2496 tokens）。 | 图像/视频 token 数不固定，需提前估算预算，避免首请求因显存/容量不确定而抖动或 OOM。 |
| **TP / PP / DP** | 张量并行(TP)/流水线并行(PP)/数据并行(DP)。 | 单卡装不下或想更快时，把大模型在维度/层/样本三个方向切到多卡。 |
| **NCCL** | NVIDIA 的多 GPU 集合通信库（all-reduce、broadcast 等）。 | TP/PP 的切分需要跨卡同步张量，NCCL 提供高效 GPU 间通信后端。 |
| **APIServer / EngineCore / 进程解耦** | APIServer 管 HTTP/OpenAI 接口，EngineCore 管 GPU 推理，二者为两个进程，经 IPC 通信。 | V1 核心设计：把网络/序列化这类 CPU 工作从 GPU 关键路径上移出，互不阻塞。 |
| **EngineArgs / Config / arg_utils** | 命令行参数汇总成的内部配置对象与解析工具。 | 所有决策（模型/长度/量化/并行/backend/KV）都源自配置，统一入口便于校验与传播。 |
| **GPUWorker / ModelRunner** | Worker 管一个 GPU 上的设备与显存，ModelRunner 负责把一批请求跑成一次 forward。 | 把「设备资源管理」与「前向执行」分层，便于多卡与不同后端复用。 |
| **architecture / Resolved architecture / config.json** | config.json 声明模型架构名，vLLM 据此解析并绑定到对应实现类。 | 不靠模型名猜结构，按声明的 architecture 动态选实现，新增模型只需注册实现类。 |
| **LM Head / Sampler** | LM Head=最后输出到词表 logits 的投影层；Sampler=按 logits 采样出下一个 token（贪心/top-k/top-p 等）。 | 一个把隐藏状态映射到词概率，一个从概率里决定实际 token，是生成「最后一环」。 |

## 四、启动完成后的显存账（GPU Memory Utilization=0.9 → 21.2 GiB）

利用率不是「模型大小 / 显存」，而是权重 + Runtime + Activation + CUDA Graph + KV Cache 综合决定。

| 占用项 | 容量 |
| --- | --- |
| 模型权重 + 非 torch 开销 | 13.61 GiB |
| Activation 峰值 | 1.38 GiB |
| CUDA Graph | 0.44 GiB |
| KV Cache | 6.21 GiB |

> 注：Activation = 前向计算时各层临时生成的中间张量（hidden states、QKV/FFN 中间结果等），随并发与序列长度变化、用完即释放；正因要给 Activation 与 CUDA Graph 留余量，KV Cache 才不能把剩余显存吃满。

## 五、源码地图（日志可直接对应的模块）

最值得深入的 5 个核心对象链：**EngineCore → GPUWorker → ModelRunner → Scheduler → KV Cache**。

| 入口 / 配置 | arg_utils.py · config.py · models/model.py（Resolved architecture） |
| --- | --- |
| Engine / 并行 | v1 core · parallel_state.py |
| 执行 / 模型 | gpu_worker.py · model_runner.py · model_executor/models/ |
| 权重 / 量化 | default_loader.py · weight_utils.py · compressed_tensors · Marlin |
| Attention | attention backend · Triton kernel |
| Offload / 多模态 | prefetch.py · encoder_runner.py |
| 编译 / 图 / 缓存 | torch.compile(Dynamo+Inductor) · CUDA Graph · kv_cache_utils.py · scheduler.py |

## 六、源码学习路径（自顶向下，按调用链追）

1. **启动主链**：vllm serve → APIServer → EngineCore → GPUWorker → ModelRunner：搞清一个 serve 怎么走到 ModelRunner。
2. **模型加载**：load_model() → Model → WeightLoader → safetensors → GPU：结构何时建、权重何时灌、量化在哪介入。
3. **显存与缓存**：Weights + Activation + CUDA Graph + KV Cache：重点 KV Cache / Block / Block Table / Cache Manager / Scheduler。
4. **执行优化**：torch.compile → Inductor → CUDA Graph → Kernel，再进 Triton / FlashAttention / FlashInfer / Marlin。
5. **请求执行**：HTTP → APIServer → EngineCore → Scheduler → KV Cache → ModelRunner → Sampler → Response，此时再看 continuous batching / chunked prefill / prefix caching。
