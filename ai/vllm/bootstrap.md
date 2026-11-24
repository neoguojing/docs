# vLLM 启动全景 
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
#### /home/neo/workspace/vllm/vllm/model_executor/model_loader/base_loader.py
- /home/neo/workspace/vllm/vllm/model_executor/model_loader/utils.py
- /home/neo/workspace/vllm/vllm/model_executor/model_loader/default_loader.py
- load_model
- get_model_architecture
- load_weights
- create_model
- process_weights_after_loading

```
[开始: load_model]
 ├── 1. 解析配置与目标设备
 │    ├── 获取 device_config、load_config
 │    └── 设置 target_device (CPU/GPU)
 ├── 2. 设置全局默认 Torch 数据类型 (set_default_torch_dtype)
 ├── 3. 在目标设备上下文中创建模型 (initialize_model)
 │    ├── 获取模型架构与配置 (get_model_architecture)
 │    ├── 配置量化参数 (configure_quant_config)
 │    ├── 检查模型类签名 (__init__)
 │    │    ├── [新版] 接受 vllm_config & prefix -> 直接实例化模型
 │    │    └── [旧版/兼容] 发出 DeprecationWarning -> 映射旧参数 kwargs 实例化
 │    └── 记录重载元数据 (record_metadata_for_reloading)
 ├── 4. 加载权重 (load_weights)
 │    ├── [TorchAO 判定] 若为 torchao 且版本 >= 0.15.0，设置 safetensors_load_strategy = "torchao"
 │    ├── 初始化专家并行(EP)权重过滤器 (_init_ep_weight_filter)
 │    ├── 调用 model.load_weights() 载入所有权重
 │    ├── 计算并记录权重加载耗时
 │    └── [可选] 根据条件判断是否执行权重加载跟踪检查 (track_weights_loading)
 ├── 5. 显存监控 (CUDA/XPU 环境)
 │    └── 记录并打印加载权重后的峰值显存 (max_memory_allocated)
 ├── 6. 在线量化处理 (若存在)
 │    └── 执行层级在线量化处理 (finalize_layerwise_processing)
 ├── 7. 权重后处理 (process_weights_after_loading)
 │    ├── 词嵌入权重重绑定检查 (maybe_retie_word_embeddings)
 │    ├── 遍历所有子模块 (named_modules)
 │    │    └── 若存在 quant_method：
 │    │         ├── 检查是否支持预处理权重 (supports_pre_processed_weights)
 │    │         ├── 按需应用 device_loading_context 移动到设备
 │    │         ├── 调用 quant_method.process_weights_after_loading()
 │    │         ├── 更新张量并行状态 (update_param_tp_status)
 │    │         └── 释放内存 (release_device_memory_under_pressure)
 │    ├── 初始化延迟注意力层权重 (is_deferred_attention_layer)
 │    ├── 执行模型级别的 process_weights_after_loading() 钩子
 │    └── [TorchAO] 若量化类型为 torchao，设置重载属性 (set_torchao_reload_attrs)
 └── [结束] 返回评估模式模型 (model.eval())
```
#### /home/neo/workspace/vllm/vllm/model_executor/models/gemma4_mm.py
- /home/neo/workspace/vllm/vllm/multimodal/registry.py
- /home/neo/workspace/vllm/vllm/model_executor/models/gemma4.py
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
