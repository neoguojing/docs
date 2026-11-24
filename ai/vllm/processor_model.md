```
vLLM 单机进程模型
│
├── 1. 主进程（Main Process）
│   │
│   ├── HTTP Server
│   │   └── serve_http()
│   │
│   ├── AsyncLLM
│   │   └── _run_output_handler()
│   │       └── 异步输出处理
│   │
│   └── AsyncMPClient
│       └── _ensure_output_queue_task()
│           └── 异步输出队列接收
│
├── 2. Forkserver 辅助进程（条件性存在）
│   │
│   └── 预加载模块
│       └── vllm.v1.engine.async_llm
│
└── 3. Engine Core 进程
    │
    ├── EngineCoreProcManager
    │   └── 进程启动与管理
    │
    └── EngineCoreProc
        │
        ├── run_engine_core()
        │   └── 进程入口
        │
        ├── run_busy_loop()
        │   └── 主事件循环
        │
        ├── process_input_sockets()
        │   └── 输入 Socket 处理线程
        │
        └── process_output_sockets()
            └── 输出 Socket 处理线程
```
