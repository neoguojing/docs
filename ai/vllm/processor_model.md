### 主进程
- serve_http
- AsyncLLM(EngineClient)._run_output_handler()
- AsyncMPClient._ensure_output_queue_task()

#### 服务进程：用于fork
- vllm.v1.engine.async_llm
#### engine core 进程 CoreEngineProcManager初始化函数负责启动
- EngineCoreProc.run_engine_core 入口函数
- EngineCoreProc.run_busy_loop 事件循环
- 线程：EngineCoreProc.process_input_sockets
- 线程：EngineCoreProc.process_output_sockets
