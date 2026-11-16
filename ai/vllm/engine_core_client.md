# **vLLM EngineCoreClient 源码解析**

这份代码的核心是 vLLM V1 的 **EngineCoreClient 通信与管理层**。

它不是模型推理的执行主体，而是连接前端 API/AsyncLLM 与后台 EngineCore 的中间层，负责：

1. EngineCore 的创建与进程管理。
2. 请求从前端发送到 EngineCore。
3. EngineCore 推理结果返回前端。
4. 同步、异步、多进程通信适配。
5. Data Parallel（DP）EngineCore 的请求路由与负载均衡。
6. Utility RPC、引擎状态管理、故障监控和资源清理。

**阅读这份代码时，最重要的是抓住一条主线：**

`AsyncLLM → AsyncMPClient → ZMQ → EngineCoreProc → EngineCore`

其中，AsyncMPClient 负责异步通信，EngineCoreProc 负责后台进程运行，EngineCore 才负责调度和推理。

粘贴的文本 (1)(20261115-152447).txt

## **一、整体架构与类图**

### **1.1 类继承关系**

这份源码采用抽象接口 + 多种通信实现 + DP 扩展的设计。

\#chatgpt-mermaid-\_r_220\_{font-family:-apple-system-body,ui-sans-serif,-apple-system,system-ui,"Segoe UI",Helvetica,"Apple Color Emoji",Arial,sans-serif,"Segoe UI Emoji","Segoe UI Symbol";font-size:16px;fill:rgb(13, 13, 13);}@keyframes edge-animation-frame{from{stroke-dashoffset:0;}}@keyframes dash{to{stroke-dashoffset:0;}}#chatgpt-mermaid-\_r_220\_ .edge-animation-slow{stroke-dasharray:9,5!important;stroke-dashoffset:900;animation:dash 50s linear infinite;stroke-linecap:round;}#chatgpt-mermaid-\_r_220\_ .edge-animation-fast{stroke-dasharray:9,5!important;stroke-dashoffset:900;animation:dash 20s linear infinite;stroke-linecap:round;}#chatgpt-mermaid-\_r_220\_ .error-icon{fill:rgb(243, 243, 243);}#chatgpt-mermaid-\_r_220\_ .error-text{fill:rgb(13, 13, 13);stroke:rgb(13, 13, 13);}#chatgpt-mermaid-\_r_220\_ .edge-thickness-normal{stroke-width:1px;}#chatgpt-mermaid-\_r_220\_ .edge-thickness-thick{stroke-width:3.5px;}#chatgpt-mermaid-\_r_220\_ .edge-pattern-solid{stroke-dasharray:0;}#chatgpt-mermaid-\_r_220\_ .edge-thickness-invisible{stroke-width:0;fill:none;}#chatgpt-mermaid-\_r_220\_ .edge-pattern-dashed{stroke-dasharray:3;}#chatgpt-mermaid-\_r_220\_ .edge-pattern-dotted{stroke-dasharray:2;}#chatgpt-mermaid-\_r_220\_ .marker{fill:rgb(143, 143, 143);stroke:rgb(143, 143, 143);}#chatgpt-mermaid-\_r_220\_ .marker.cross{stroke:rgb(143, 143, 143);}#chatgpt-mermaid-\_r_220\_ svg{font-family:-apple-system-body,ui-sans-serif,-apple-system,system-ui,"Segoe UI",Helvetica,"Apple Color Emoji",Arial,sans-serif,"Segoe UI Emoji","Segoe UI Symbol";font-size:16px;}#chatgpt-mermaid-\_r_220\_ p{margin:0;}#chatgpt-mermaid-\_r_220\_ g.classGroup text{fill:rgb(83, 154, 248);stroke:none;font-family:-apple-system-body,ui-sans-serif,-apple-system,system-ui,"Segoe UI",Helvetica,"Apple Color Emoji",Arial,sans-serif,"Segoe UI Emoji","Segoe UI Symbol";font-size:10px;}#chatgpt-mermaid-\_r_220\_ g.classGroup text .title{font-weight:bolder;}#chatgpt-mermaid-\_r_220\_ .cluster-label text{fill:rgb(13, 13, 13);}#chatgpt-mermaid-\_r_220\_ .cluster-label span{color:rgb(13, 13, 13);}#chatgpt-mermaid-\_r_220\_ .cluster-label span p{background-color:transparent;}#chatgpt-mermaid-\_r_220\_ .cluster rect{fill:rgb(243, 243, 243);stroke:rgba(0, 0, 0, 0.1);stroke-width:1px;}#chatgpt-mermaid-\_r_220\_ .cluster text{fill:rgb(13, 13, 13);}#chatgpt-mermaid-\_r_220\_ .cluster span{color:rgb(13, 13, 13);}#chatgpt-mermaid-\_r_220\_ .nodeLabel,#chatgpt-mermaid-\_r_220\_ .edgeLabel{color:rgb(13, 13, 13);}#chatgpt-mermaid-\_r_220\_ .noteLabel .nodeLabel,#chatgpt-mermaid-\_r_220\_ .noteLabel .edgeLabel{color:rgb(13, 13, 13);}#chatgpt-mermaid-\_r_220\_ .edgeLabel .label rect{fill:rgb(222, 234, 251);}#chatgpt-mermaid-\_r_220\_ .label text{fill:rgb(13, 13, 13);}#chatgpt-mermaid-\_r_220\_ .labelBkg{background:rgb(222, 234, 251);}#chatgpt-mermaid-\_r_220\_ .edgeLabel .label span{background:rgb(222, 234, 251);}#chatgpt-mermaid-\_r_220\_ .classTitle{font-weight:bolder;}#chatgpt-mermaid-\_r_220\_ .node rect,#chatgpt-mermaid-\_r_220\_ .node circle,#chatgpt-mermaid-\_r_220\_ .node ellipse,#chatgpt-mermaid-\_r_220\_ .node polygon,#chatgpt-mermaid-\_r_220\_ .node path{fill:rgb(222, 234, 251);stroke:rgb(83, 154, 248);stroke-width:1;}#chatgpt-mermaid-\_r_220\_ .divider{stroke:rgb(83, 154, 248);stroke-width:1;}#chatgpt-mermaid-\_r_220\_ g.clickable{cursor:pointer;}#chatgpt-mermaid-\_r_220\_ g.classGroup rect{fill:rgb(222, 234, 251);stroke:rgb(83, 154, 248);}#chatgpt-mermaid-\_r_220\_ g.classGroup line{stroke:rgb(83, 154, 248);stroke-width:1;}#chatgpt-mermaid-\_r_220\_ .classLabel .box{stroke:none;stroke-width:0;fill:rgb(222, 234, 251);opacity:0.5;}#chatgpt-mermaid-\_r_220\_ .classLabel .label{fill:rgb(83, 154, 248);font-size:10px;}#chatgpt-mermaid-\_r_220\_ .relation{stroke:rgb(143, 143, 143);stroke-width:1;fill:none;}#chatgpt-mermaid-\_r_220\_ .dashed-line{stroke-dasharray:3;}#chatgpt-mermaid-\_r_220\_ .dotted-line{stroke-dasharray:1 2;}#chatgpt-mermaid-\_r_220\_ [id$="-compositionStart"],#chatgpt-mermaid-\_r_220\_ .composition{fill:rgb(143, 143, 143)!important;stroke:rgb(143, 143, 143)!important;stroke-width:1;}#chatgpt-mermaid-\_r_220\_ [id$="-compositionEnd"],#chatgpt-mermaid-\_r_220\_ .composition{fill:rgb(143, 143, 143)!important;stroke:rgb(143, 143, 143)!important;stroke-width:1;}#chatgpt-mermaid-\_r_220\_ [id$="-dependencyStart"],#chatgpt-mermaid-\_r_220\_ .dependency{fill:rgb(143, 143, 143)!important;stroke:rgb(143, 143, 143)!important;stroke-width:1;}#chatgpt-mermaid-\_r_220\_ [id$="-dependencyEnd"],#chatgpt-mermaid-\_r_220\_ .dependency{fill:rgb(143, 143, 143)!important;stroke:rgb(143, 143, 143)!important;stroke-width:1;}#chatgpt-mermaid-\_r_220\_ [id$="-extensionStart"],#chatgpt-mermaid-\_r_220\_ .extension{fill:transparent!important;stroke:rgb(143, 143, 143)!important;stroke-width:1;}#chatgpt-mermaid-\_r_220\_ [id$="-extensionEnd"],#chatgpt-mermaid-\_r_220\_ .extension{fill:transparent!important;stroke:rgb(143, 143, 143)!important;stroke-width:1;}#chatgpt-mermaid-\_r_220\_ [id$="-aggregationStart"],#chatgpt-mermaid-\_r_220\_ .aggregation{fill:transparent!important;stroke:rgb(143, 143, 143)!important;stroke-width:1;}#chatgpt-mermaid-\_r_220\_ [id$="-aggregationEnd"],#chatgpt-mermaid-\_r_220\_ .aggregation{fill:transparent!important;stroke:rgb(143, 143, 143)!important;stroke-width:1;}#chatgpt-mermaid-\_r_220\_ [id$="-lollipopStart"],#chatgpt-mermaid-\_r_220\_ .lollipop{fill:rgb(222, 234, 251)!important;stroke:rgb(143, 143, 143)!important;stroke-width:1;}#chatgpt-mermaid-\_r_220\_ [id$="-lollipopEnd"],#chatgpt-mermaid-\_r_220\_ .lollipop{fill:rgb(222, 234, 251)!important;stroke:rgb(143, 143, 143)!important;stroke-width:1;}#chatgpt-mermaid-\_r_220\_ .edgeTerminals{font-size:11px;line-height:initial;}#chatgpt-mermaid-\_r_220\_ .classTitleText{text-anchor:middle;font-size:18px;fill:rgb(13, 13, 13);}#chatgpt-mermaid-\_r_220\_ .edgeLabel[data-look="neo"]{background-color:rgb(252, 252, 252);text-align:center;}#chatgpt-mermaid-\_r_220\_ .edgeLabel[data-look="neo"] p{background-color:rgb(252, 252, 252);}#chatgpt-mermaid-\_r_220\_ .edgeLabel[data-look="neo"] rect{opacity:0.5;background-color:rgb(252, 252, 252);fill:rgb(252, 252, 252);}#chatgpt-mermaid-\_r_220\_ .label-icon{display:inline-block;height:1em;overflow:visible;vertical-align:-0.125em;}#chatgpt-mermaid-\_r_220\_ .node .label-icon path{fill:currentColor;stroke:revert;stroke-width:revert;}#chatgpt-mermaid-\_r_220\_ .node .neo-node{stroke:rgb(83, 154, 248);}#chatgpt-mermaid-\_r_220\_ [data-look="neo"].node rect,#chatgpt-mermaid-\_r_220\_ [data-look="neo"].cluster rect,#chatgpt-mermaid-\_r_220\_ [data-look="neo"].node polygon{stroke:url(#chatgpt-mermaid-\_r_220\_-gradient);filter:drop-shadow( 1px 2px 2px rgba(185,185,185,1));}#chatgpt-mermaid-\_r_220\_ [data-look="neo"].swimlane.cluster rect{filter:none;}#chatgpt-mermaid-\_r_220\_ [data-look="neo"].node path{stroke:url(#chatgpt-mermaid-\_r_220\_-gradient);stroke-width:1px;}#chatgpt-mermaid-\_r_220\_ [data-look="neo"].node .outer-path{filter:drop-shadow( 1px 2px 2px rgba(185,185,185,1));}#chatgpt-mermaid-\_r_220\_ [data-look="neo"].node .neo-line path{stroke:rgb(83, 154, 248);filter:none;}#chatgpt-mermaid-\_r_220\_ [data-look="neo"].node circle{stroke:url(#chatgpt-mermaid-\_r_220\_-gradient);filter:drop-shadow( 1px 2px 2px rgba(185,185,185,1));}#chatgpt-mermaid-\_r_220\_ [data-look="neo"].node circle .state-start{fill:#000000;}#chatgpt-mermaid-\_r_220\_ [data-look="neo"].icon-shape .icon{fill:url(#chatgpt-mermaid-\_r_220\_-gradient);filter:drop-shadow( 1px 2px 2px rgba(185,185,185,1));}#chatgpt-mermaid-\_r_220\_ [data-look="neo"].icon-shape .icon-neo path{stroke:url(#chatgpt-mermaid-\_r_220\_-gradient);filter:drop-shadow( 1px 2px 2px rgba(185,185,185,1));}#chatgpt-mermaid-\_r_220\_ :root{--mermaid-font-family:-apple-system-body,ui-sans-serif,-apple-system,system-ui,"Segoe UI",Helvetica,"Apple Color Emoji",Arial,sans-serif,"Segoe UI Emoji","Segoe UI Symbol";}«abstract»EngineCoreClient+make_client()+make_async_mp_client()+add_request()+get_output()+abort_requests()+shutdown()+collective_rpc()InprocClient+engine_core: EngineCore+add_request()+get_output()«abstract»MPClient+input_socket+output_socket+encoder+decoder+utility_results+start_engine_core_monitor()+call_utility()SyncMPClient+outputs_queue+output_queue_thread+\_send_input()+call_utility()+get_output()AsyncMPClient+outputs_queue+\_ensure_output_queue_task()+\_send_input()+call_utility_async()+get_output_async()DPAsyncMPClient+current_wave+lb_engines+\_ensure_stats_update_task()+get_core_engine_for_request()DPLBAsyncMPClient+reqs_in_flight+engine_inflight+get_core_engine_for_request()+process_engine_outputs()+prepare_elastic_ep()+commit_elastic_ep()BackgroundResourcesEngineCoreProcEngineCore

类图根据本文件中的继承、成员和调用关系整理；EngineCoreProc 与 EngineCore 的内部实现不在本文件中。

### **1.2 各类的职责**

| 类                   | 核心职责                          | 适用场景     |
| ------------------- | ----------------------------- | -------- |
| `EngineCoreClient`  | 定义统一接口、工厂选择                   | 所有模式     |
| `InprocClient`      | 直接调用 EngineCore               | 同进程同步    |
| `MPClient`          | 多进程通信基础设施                     | 同步/异步共用  |
| `SyncMPClient`      | ZMQ + 线程 + 同步 Queue           | LLM      |
| `AsyncMPClient`     | ZMQ + asyncio Task + 异步 Queue | AsyncLLM |
| `DPAsyncMPClient`   | DP 状态同步、统计订阅                  | 外部 LB    |
| `DPLBAsyncMPClient` | 内部负载均衡、请求跟踪                   | 内部 LB    |

这里有三个重要的设计层次：

- `EngineCoreClient`：解决接口统一问题。
- `MPClient`：解决跨进程通信问题。
- `AsyncMPClient / DPAsyncMPClient / DPLBAsyncMPClient`：逐步增加异步和分布式能力。

注意：`DPAsyncMPClient` 并不负责内部请求均衡，它默认将请求发给自己的 `core_engine`。真正的内部多 EngineCore 负载均衡在 `DPLBAsyncMPClient`。

粘贴的文本 (1)(20261115-152447).txt

## **二、工厂模式：客户端是如何选择的？**

核心入口：

```
EngineCoreClient.make_client(    multiprocess_mode,    asyncio_mode,    vllm_config,    executor_class,    log_stats,    renderer,)
```

### **2.1 第一层：同步/异步与多进程**

| multiprocess | asyncio | 创建对象                      |
| ------------ | ------- | ------------------------- |
| False        | False   | `InprocClient`            |
| True         | False   | `SyncMPClient`            |
| True         | True    | 进入 DP 判断或 `AsyncMPClient` |
| False        | True    | 不支持，抛出异常                  |

### **2.2 第二层：AsyncMPClient 的 DP 分流**

```
if parallel_config.data_parallel_size > 1:    if parallel_config.data_parallel_external_lb:        return DPAsyncMPClient(...)    return DPLBAsyncMPClient(...)return AsyncMPClient(...)
```

对应架构：

\#chatgpt-mermaid-\_r_22l\_{font-family:-apple-system-body,ui-sans-serif,-apple-system,system-ui,"Segoe UI",Helvetica,"Apple Color Emoji",Arial,sans-serif,"Segoe UI Emoji","Segoe UI Symbol";font-size:16px;fill:rgb(13, 13, 13);}@keyframes edge-animation-frame{from{stroke-dashoffset:0;}}@keyframes dash{to{stroke-dashoffset:0;}}#chatgpt-mermaid-\_r_22l\_ .edge-animation-slow{stroke-dasharray:9,5!important;stroke-dashoffset:900;animation:dash 50s linear infinite;stroke-linecap:round;}#chatgpt-mermaid-\_r_22l\_ .edge-animation-fast{stroke-dasharray:9,5!important;stroke-dashoffset:900;animation:dash 20s linear infinite;stroke-linecap:round;}#chatgpt-mermaid-\_r_22l\_ .error-icon{fill:rgb(243, 243, 243);}#chatgpt-mermaid-\_r_22l\_ .error-text{fill:rgb(13, 13, 13);stroke:rgb(13, 13, 13);}#chatgpt-mermaid-\_r_22l\_ .edge-thickness-normal{stroke-width:1px;}#chatgpt-mermaid-\_r_22l\_ .edge-thickness-thick{stroke-width:3.5px;}#chatgpt-mermaid-\_r_22l\_ .edge-pattern-solid{stroke-dasharray:0;}#chatgpt-mermaid-\_r_22l\_ .edge-thickness-invisible{stroke-width:0;fill:none;}#chatgpt-mermaid-\_r_22l\_ .edge-pattern-dashed{stroke-dasharray:3;}#chatgpt-mermaid-\_r_22l\_ .edge-pattern-dotted{stroke-dasharray:2;}#chatgpt-mermaid-\_r_22l\_ .marker{fill:rgb(143, 143, 143);stroke:rgb(143, 143, 143);}#chatgpt-mermaid-\_r_22l\_ .marker.cross{stroke:rgb(143, 143, 143);}#chatgpt-mermaid-\_r_22l\_ svg{font-family:-apple-system-body,ui-sans-serif,-apple-system,system-ui,"Segoe UI",Helvetica,"Apple Color Emoji",Arial,sans-serif,"Segoe UI Emoji","Segoe UI Symbol";font-size:16px;}#chatgpt-mermaid-\_r_22l\_ p{margin:0;}#chatgpt-mermaid-\_r_22l\_ .label{font-family:-apple-system-body,ui-sans-serif,-apple-system,system-ui,"Segoe UI",Helvetica,"Apple Color Emoji",Arial,sans-serif,"Segoe UI Emoji","Segoe UI Symbol";color:rgb(13, 13, 13);}#chatgpt-mermaid-\_r_22l\_ .cluster-label text{fill:rgb(13, 13, 13);}#chatgpt-mermaid-\_r_22l\_ .cluster-label span{color:rgb(13, 13, 13);}#chatgpt-mermaid-\_r_22l\_ .cluster-label span p{background-color:transparent;}#chatgpt-mermaid-\_r_22l\_ .label text,#chatgpt-mermaid-\_r_22l\_ span{fill:rgb(13, 13, 13);color:rgb(13, 13, 13);}#chatgpt-mermaid-\_r_22l\_ .node rect,#chatgpt-mermaid-\_r_22l\_ .node circle,#chatgpt-mermaid-\_r_22l\_ .node ellipse,#chatgpt-mermaid-\_r_22l\_ .node polygon,#chatgpt-mermaid-\_r_22l\_ .node path{fill:rgb(222, 234, 251);stroke:rgb(83, 154, 248);stroke-width:1px;}#chatgpt-mermaid-\_r_22l\_ .rough-node .label text,#chatgpt-mermaid-\_r_22l\_ .node .label text,#chatgpt-mermaid-\_r_22l\_ .image-shape .label,#chatgpt-mermaid-\_r_22l\_ .icon-shape .label{text-anchor:middle;}#chatgpt-mermaid-\_r_22l\_ .node .katex path{fill:#000;stroke:#000;stroke-width:1px;}#chatgpt-mermaid-\_r_22l\_ .rough-node .label,#chatgpt-mermaid-\_r_22l\_ .node .label,#chatgpt-mermaid-\_r_22l\_ .image-shape .label,#chatgpt-mermaid-\_r_22l\_ .icon-shape .label{text-align:center;}#chatgpt-mermaid-\_r_22l\_ .node.clickable{cursor:pointer;}#chatgpt-mermaid-\_r_22l\_ .root .anchor path{fill:rgb(143, 143, 143)!important;stroke-width:0;stroke:rgb(143, 143, 143);}#chatgpt-mermaid-\_r_22l\_ .arrowheadPath{fill:rgb(143, 143, 143);}#chatgpt-mermaid-\_r_22l\_ .edgePath .path{stroke:rgb(143, 143, 143);stroke-width:1px;}#chatgpt-mermaid-\_r_22l\_ .flowchart-link{stroke:rgb(143, 143, 143);fill:none;}#chatgpt-mermaid-\_r_22l\_ .edgeLabel{background-color:rgb(252, 252, 252);text-align:center;}#chatgpt-mermaid-\_r_22l\_ .edgeLabel p{background-color:rgb(252, 252, 252);}#chatgpt-mermaid-\_r_22l\_ .edgeLabel rect{opacity:0.5;background-color:rgb(252, 252, 252);fill:rgb(252, 252, 252);}#chatgpt-mermaid-\_r_22l\_ .labelBkg{background-color:rgba(252, 252, 252, 0.5);}#chatgpt-mermaid-\_r_22l\_ .cluster rect{fill:rgb(243, 243, 243);stroke:rgba(0, 0, 0, 0.1);stroke-width:1px;}#chatgpt-mermaid-\_r_22l\_ .cluster text{fill:rgb(13, 13, 13);}#chatgpt-mermaid-\_r_22l\_ .cluster span{color:rgb(13, 13, 13);}#chatgpt-mermaid-\_r_22l\_ div.mermaidTooltip{position:absolute;text-align:center;max-width:200px;padding:2px;font-family:-apple-system-body,ui-sans-serif,-apple-system,system-ui,"Segoe UI",Helvetica,"Apple Color Emoji",Arial,sans-serif,"Segoe UI Emoji","Segoe UI Symbol";font-size:12px;background:rgb(243, 243, 243);border:1px solid rgba(0, 0, 0, 0.1);border-radius:2px;pointer-events:none;z-index:100;}#chatgpt-mermaid-\_r_22l\_ .flowchartTitleText{text-anchor:middle;font-size:18px;fill:rgb(13, 13, 13);}#chatgpt-mermaid-\_r_22l\_ rect.text{fill:none;stroke-width:0;}#chatgpt-mermaid-\_r_22l\_ .icon-shape,#chatgpt-mermaid-\_r_22l\_ .image-shape{background-color:rgb(252, 252, 252);text-align:center;}#chatgpt-mermaid-\_r_22l\_ .icon-shape p,#chatgpt-mermaid-\_r_22l\_ .image-shape p{background-color:rgb(252, 252, 252);padding:2px;}#chatgpt-mermaid-\_r_22l\_ .icon-shape .label rect,#chatgpt-mermaid-\_r_22l\_ .image-shape .label rect{opacity:0.5;background-color:rgb(252, 252, 252);fill:rgb(252, 252, 252);}#chatgpt-mermaid-\_r_22l\_ .label-icon{display:inline-block;height:1em;overflow:visible;vertical-align:-0.125em;}#chatgpt-mermaid-\_r_22l\_ .node .label-icon path{fill:currentColor;stroke:revert;stroke-width:revert;}#chatgpt-mermaid-\_r_22l\_ .node .neo-node{stroke:rgb(83, 154, 248);}#chatgpt-mermaid-\_r_22l\_ [data-look="neo"].node rect,#chatgpt-mermaid-\_r_22l\_ [data-look="neo"].cluster rect,#chatgpt-mermaid-\_r_22l\_ [data-look="neo"].node polygon{stroke:url(#chatgpt-mermaid-\_r_22l\_-gradient);filter:drop-shadow( 1px 2px 2px rgba(185,185,185,1));}#chatgpt-mermaid-\_r_22l\_ [data-look="neo"].swimlane.cluster rect{filter:none;}#chatgpt-mermaid-\_r_22l\_ [data-look="neo"].node path{stroke:url(#chatgpt-mermaid-\_r_22l\_-gradient);stroke-width:1px;}#chatgpt-mermaid-\_r_22l\_ [data-look="neo"].node .outer-path{filter:drop-shadow( 1px 2px 2px rgba(185,185,185,1));}#chatgpt-mermaid-\_r_22l\_ [data-look="neo"].node .neo-line path{stroke:rgb(83, 154, 248);filter:none;}#chatgpt-mermaid-\_r_22l\_ [data-look="neo"].node circle{stroke:url(#chatgpt-mermaid-\_r_22l\_-gradient);filter:drop-shadow( 1px 2px 2px rgba(185,185,185,1));}#chatgpt-mermaid-\_r_22l\_ [data-look="neo"].node circle .state-start{fill:#000000;}#chatgpt-mermaid-\_r_22l\_ [data-look="neo"].icon-shape .icon{fill:url(#chatgpt-mermaid-\_r_22l\_-gradient);filter:drop-shadow( 1px 2px 2px rgba(185,185,185,1));}#chatgpt-mermaid-\_r_22l\_ [data-look="neo"].icon-shape .icon-neo path{stroke:url(#chatgpt-mermaid-\_r_22l\_-gradient);filter:drop-shadow( 1px 2px 2px rgba(185,185,185,1));}#chatgpt-mermaid-\_r_22l\_ .node text{font-size:14px;font-weight:600;letter-spacing:normal;fill:rgb(0, 79, 153);}#chatgpt-mermaid-\_r_22l\_ .edgeLabels text{font-size:13px;font-weight:600;letter-spacing:-0.08px;fill:rgb(0, 79, 153);}#chatgpt-mermaid-\_r_22l\_ .node tspan[font-weight="normal"],#chatgpt-mermaid-\_r_22l\_ .edgeLabels tspan[font-weight="normal"]{font-weight:600;}#chatgpt-mermaid-\_r_22l\_ .edgeLabel .label rect{opacity:1;rx:13px;ry:13px;fill:rgb(245, 250, 255);stroke:rgb(206, 219, 229);stroke-width:1px;}#chatgpt-mermaid-\_r_22l\_ .node rect,#chatgpt-mermaid-\_r_22l\_ .node circle,#chatgpt-mermaid-\_r_22l\_ .node ellipse,#chatgpt-mermaid-\_r_22l\_ .node polygon,#chatgpt-mermaid-\_r_22l\_ .node path{fill:rgb(229, 243, 255);stroke:rgba(0, 0, 0, 0.1);stroke-width:1px;}#chatgpt-mermaid-\_r_22l\_ .node rect{rx:16px;ry:16px;}#chatgpt-mermaid-\_r_22l\_ .mermaid-decision .label-container{fill:rgb(245, 250, 255);stroke:rgb(206, 219, 229);stroke-dasharray:2,2;}#chatgpt-mermaid-\_r_22l\_ .edgePaths .flowchart-link{stroke:rgb(143, 143, 143);stroke-width:1px;stroke-linecap:round;stroke-linejoin:round;}#chatgpt-mermaid-\_r_22l\_ .marker{fill:rgb(143, 143, 143);stroke:rgb(143, 143, 143);}#chatgpt-mermaid-\_r_22l\_ :root{--mermaid-font-family:-apple-system-body,ui-sans-serif,-apple-system,system-ui,"Segoe UI",Helvetica,"Apple Color Emoji",Arial,sans-serif,"Segoe UI Emoji","Segoe UI Symbol";}EngineCoreClient.make_clientmultiprocess_mode?InprocClientasyncio_mode?SyncMPClientmake_async_mp_clientDP size 1?AsyncMPClientExternal LB?DPAsyncMPClientDPLBAsyncMPClientFalseTrueFalseTrueNoYesYesNo

### **2.3 为什么要这么设计？**

因为不同场景需要不同的通信模型：

- Inproc：不需要跨进程通信，直接调用对象。
- Sync：同步调用接口，后台线程接收结果。
- Async：异步调用接口，使用 asyncio Task 接收结果。
- DP External LB：每个客户端通常负责自己的 DP Rank。
- DP Internal LB：一个客户端需要在多个 DP Rank 之间分配请求。

**重点：DP 是模型执行实例的并行部署方式，不是把一个请求拆成多个 Rank 再汇总结果。**

这里的 DP 主要体现为多个独立 EngineCore 接收不同请求。

粘贴的文本 (1)(20261115-152447).txt

## **三、MPClient：整个文件最重要的基础类**

建议重点阅读 `MPClient.__init__()`，它决定了多进程通信架构。

### **3.1 核心对象关系**

**Frontend Process**

`MPClient`

ZMQ Context / Encoder / Decoder / Monitor

**Input Socket**

`ROUTER`

发送请求

**Output Socket**

`PULL`

接收结果

跨进程 ZMQ 通信

**EngineCore Process**

`EngineCoreProc`

`EngineCore → Scheduler → Executor → Worker`

注意：上图中 Scheduler、Executor、Worker 是 EngineCore 执行侧的逻辑示意，不是当前文件的类定义。

### **3.2 初始化关键步骤**

源码主线：

```
sync_ctx = zmq.Context(io_threads=2)self.ctx = (    zmq.asyncio.Context(sync_ctx)    if asyncio_mode else sync_ctx)self.resources = BackgroundResources(ctx=sync_ctx)self._finalizer = weakref.finalize(    self, self.resources)
```

接下来：

1. 创建 ZMQ Context。
2. 创建输入 ROUTER Socket。
3. 创建输出 PULL Socket。
4. 调用 `launch_core_engines()` 启动 EngineCore 进程。
5. 初始化序列化 Encoder/Decoder。
6. 等待所有 EngineCore Ready。
7. 启动 EngineCore 监控线程。

其中最值得注意的是：

```
self.input_socket = make_zmq_socket(    self.ctx,    addresses.inputs[0],    zmq.ROUTER,    bind=True,)self.resources.output_socket = make_zmq_socket(    self.ctx,    addresses.outputs[0],    zmq.PULL,)
```

为什么使用 ROUTER 和 PULL？

| Socket | 作用                     | 方向              |
| ------ | ---------------------- | --------------- |
| ROUTER | 按 Engine Identity 路由请求 | Client → Engine |
| PULL   | 汇聚 Engine 输出消息         | Engine → Client |

在客户端这一侧，ROUTER 可以根据消息中的 Engine Identity 将请求发送给指定的 EngineCore。

### **3.3 Engine Identity 是什么？**

```
EngineIdentity = bytesself.core_engines = [    rank.to_bytes(2, "little")    for rank in self.engine_ranks_managed]
```

例如：

```
rank = 0  # b'\x00\x00'rank = 1  # b'\x01\x00'rank = 2  # b'\x02\x00'
```

它是 EngineCore 在 ZMQ 通信中的身份标识。

注意区分：

| 概念              | 含义                     |
| --------------- | ---------------------- |
| DP Rank         | DP 维度的逻辑编号             |
| Engine Identity | 客户端用于路由通信的字节标识         |
| Process PID     | 操作系统进程编号               |
| TP Rank         | Tensor Parallel 中的逻辑编号 |

**DP Rank 不等于 PID，也不等于 TP Rank。**

### **3.4 Ready 握手机制**

```
while identities:    if not sync_input_socket.poll(timeout=...):        raise TimeoutError(...)    identity, payload = sync_input_socket.recv_multipart()    identities.remove(identity)    self._apply_ready_response(payload)
```

这个阶段非常重要。

EngineCore 进程启动不代表模型已经可用。

客户端必须等待每个 EngineCore 发送 Ready 消息。

Ready 消息中的 `EngineCoreReadyResponse` 会同步以下配置：

- `max_model_len`
- `num_gpu_blocks`
- `block_size`
- `mamba_block_size`
- KV Cache 容量和并发信息

因此，**Ready 握手也是前后端配置同步的过程，而不只是进程存活检测。**

粘贴的文本 (1)(20261115-152447).txt

## **四、关键流程一：请求发送与结果接收**

这是整个文件最值得掌握的流程。

### **4.1 请求发送**

以 `AsyncMPClient.add_request_async()` 为例：

```
async def add_request_async(self, request):    request.client_index = self.client_index    await self._send_input(        EngineCoreRequestType.ADD,        request    )    self._ensure_output_queue_task()
```

实际发送：

```
async def _send_input(self, request_type, request, engine=None):    if engine is None:        engine = self.core_engine    message = (        request_type.value,        *self.encoder.encode(request)    )    return self._send_input_message(message, engine)
```

```
async def _send_input_message(self, message, engine):    self.ensure_alive()    return await self.input_socket.send_multipart(        (engine,) + message,        copy=False    )
```

请求结构：

```
(engine_identity,
 request_type,
 serialized_request)
```

注意：实际消息可能包含多个 Frame，例如多模态 Tensor 的带外数据，不应简单理解为只有三个字节帧。

### **4.2 请求发送流程图**

EngineCoreEngineCoreProcZMQ ROUTERAsyncMPClientAsyncLLMEngineCoreEngineCoreProcZMQ ROUTERAsyncMPClientAsyncLLM#chatgpt-mermaid-\_r_24s\_{font-family:-apple-system-body,ui-sans-serif,-apple-system,system-ui,"Segoe UI",Helvetica,"Apple Color Emoji",Arial,sans-serif,"Segoe UI Emoji","Segoe UI Symbol";font-size:16px;fill:rgb(13, 13, 13);}@keyframes edge-animation-frame{from{stroke-dashoffset:0;}}@keyframes dash{to{stroke-dashoffset:0;}}#chatgpt-mermaid-\_r_24s\_ .edge-animation-slow{stroke-dasharray:9,5!important;stroke-dashoffset:900;animation:dash 50s linear infinite;stroke-linecap:round;}#chatgpt-mermaid-\_r_24s\_ .edge-animation-fast{stroke-dasharray:9,5!important;stroke-dashoffset:900;animation:dash 20s linear infinite;stroke-linecap:round;}#chatgpt-mermaid-\_r_24s\_ .error-icon{fill:rgb(243, 243, 243);}#chatgpt-mermaid-\_r_24s\_ .error-text{fill:rgb(13, 13, 13);stroke:rgb(13, 13, 13);}#chatgpt-mermaid-\_r_24s\_ .edge-thickness-normal{stroke-width:1px;}#chatgpt-mermaid-\_r_24s\_ .edge-thickness-thick{stroke-width:3.5px;}#chatgpt-mermaid-\_r_24s\_ .edge-pattern-solid{stroke-dasharray:0;}#chatgpt-mermaid-\_r_24s\_ .edge-thickness-invisible{stroke-width:0;fill:none;}#chatgpt-mermaid-\_r_24s\_ .edge-pattern-dashed{stroke-dasharray:3;}#chatgpt-mermaid-\_r_24s\_ .edge-pattern-dotted{stroke-dasharray:2;}#chatgpt-mermaid-\_r_24s\_ .marker{fill:rgb(143, 143, 143);stroke:rgb(143, 143, 143);}#chatgpt-mermaid-\_r_24s\_ .marker.cross{stroke:rgb(143, 143, 143);}#chatgpt-mermaid-\_r_24s\_ svg{font-family:-apple-system-body,ui-sans-serif,-apple-system,system-ui,"Segoe UI",Helvetica,"Apple Color Emoji",Arial,sans-serif,"Segoe UI Emoji","Segoe UI Symbol";font-size:16px;}#chatgpt-mermaid-\_r_24s\_ p{margin:0;}#chatgpt-mermaid-\_r_24s\_ .actor{stroke:rgb(83, 154, 248);fill:rgb(222, 234, 251);stroke-width:1;}#chatgpt-mermaid-\_r_24s\_ rect.actor.outer-path[data-look="neo"]{filter:drop-shadow( 1px 2px 2px rgba(185,185,185,1));}#chatgpt-mermaid-\_r_24s\_ rect.note[data-look="neo"]{stroke:rgb(107, 198, 127);fill:rgb(243, 243, 243);filter:drop-shadow( 1px 2px 2px rgba(185,185,185,1));}#chatgpt-mermaid-\_r_24s\_ text.actor>tspan{fill:rgb(13, 13, 13);stroke:none;}#chatgpt-mermaid-\_r_24s\_ .actor-line{stroke:rgb(143, 143, 143);}#chatgpt-mermaid-\_r_24s\_ .innerArc{stroke-width:1.5;stroke-dasharray:none;}#chatgpt-mermaid-\_r_24s\_ .messageLine0{stroke-width:1.5;stroke-dasharray:none;stroke:rgb(143, 143, 143);}#chatgpt-mermaid-\_r_24s\_ .messageLine1{stroke-width:1.5;stroke-dasharray:2,2;stroke:rgb(143, 143, 143);}#chatgpt-mermaid-\_r_24s\_ [id$="-arrowhead"] path{fill:rgb(143, 143, 143);stroke:rgb(143, 143, 143);}#chatgpt-mermaid-\_r_24s\_ .sequenceNumber{fill:#707070;}#chatgpt-mermaid-\_r_24s\_ [id$="-sequencenumber"]{fill:rgb(143, 143, 143);}#chatgpt-mermaid-\_r_24s\_ [id$="-crosshead"] path{fill:rgb(143, 143, 143);stroke:rgb(143, 143, 143);}#chatgpt-mermaid-\_r_24s\_ .messageText{fill:rgb(13, 13, 13);stroke:none;}#chatgpt-mermaid-\_r_24s\_ .labelBox{stroke:rgba(0, 0, 0, 0.1);fill:rgb(252, 252, 252);filter:none;}#chatgpt-mermaid-\_r_24s\_ .labelText,#chatgpt-mermaid-\_r_24s\_ .labelText>tspan{fill:rgb(13, 13, 13);stroke:none;}#chatgpt-mermaid-\_r_24s\_ .loopText,#chatgpt-mermaid-\_r_24s\_ .loopText>tspan{fill:rgb(13, 13, 13);stroke:none;}#chatgpt-mermaid-\_r_24s\_ .sectionTitle,#chatgpt-mermaid-\_r_24s\_ .sectionTitle>tspan{fill:rgb(13, 13, 13);stroke:none;}#chatgpt-mermaid-\_r_24s\_ .loopLine{stroke-width:2px;stroke-dasharray:2,2;stroke:rgba(0, 0, 0, 0.1);fill:rgba(0, 0, 0, 0.1);}#chatgpt-mermaid-\_r_24s\_ .note{stroke:rgb(107, 198, 127);fill:rgb(243, 243, 243);}#chatgpt-mermaid-\_r_24s\_ .noteText,#chatgpt-mermaid-\_r_24s\_ .noteText>tspan{fill:rgb(13, 13, 13);stroke:none;font-weight:normal;}#chatgpt-mermaid-\_r_24s\_ .activation0{fill:rgb(243, 243, 243);stroke:hsl(0, 0%, 85.2941176471%);}#chatgpt-mermaid-\_r_24s\_ .activation1{fill:rgb(243, 243, 243);stroke:hsl(0, 0%, 85.2941176471%);}#chatgpt-mermaid-\_r_24s\_ .activation2{fill:rgb(243, 243, 243);stroke:hsl(0, 0%, 85.2941176471%);}#chatgpt-mermaid-\_r_24s\_ .actorPopupMenu{position:absolute;}#chatgpt-mermaid-\_r_24s\_ .actorPopupMenuPanel{position:absolute;fill:rgb(222, 234, 251);box-shadow:0px 8px 16px 0px rgba(0,0,0,0.2);filter:drop-shadow(3px 5px 2px rgb(0 0 0 / 0.4));}#chatgpt-mermaid-\_r_24s\_ .actor-man circle,#chatgpt-mermaid-\_r_24s\_ line{fill:rgb(222, 234, 251);stroke-width:2px;}#chatgpt-mermaid-\_r_24s\_ g rect.rect{filter:drop-shadow( 1px 2px 2px rgba(185,185,185,1));stroke:rgb(83, 154, 248);}#chatgpt-mermaid-\_r_24s\_ .node .neo-node{stroke:rgb(83, 154, 248);}#chatgpt-mermaid-\_r_24s\_ [data-look="neo"].node rect,#chatgpt-mermaid-\_r_24s\_ [data-look="neo"].cluster rect,#chatgpt-mermaid-\_r_24s\_ [data-look="neo"].node polygon{stroke:url(#chatgpt-mermaid-\_r_24s\_-gradient);filter:drop-shadow( 1px 2px 2px rgba(185,185,185,1));}#chatgpt-mermaid-\_r_24s\_ [data-look="neo"].swimlane.cluster rect{filter:none;}#chatgpt-mermaid-\_r_24s\_ [data-look="neo"].node path{stroke:url(#chatgpt-mermaid-\_r_24s\_-gradient);stroke-width:1px;}#chatgpt-mermaid-\_r_24s\_ [data-look="neo"].node .outer-path{filter:drop-shadow( 1px 2px 2px rgba(185,185,185,1));}#chatgpt-mermaid-\_r_24s\_ [data-look="neo"].node .neo-line path{stroke:rgb(83, 154, 248);filter:none;}#chatgpt-mermaid-\_r_24s\_ [data-look="neo"].node circle{stroke:url(#chatgpt-mermaid-\_r_24s\_-gradient);filter:drop-shadow( 1px 2px 2px rgba(185,185,185,1));}#chatgpt-mermaid-\_r_24s\_ [data-look="neo"].node circle .state-start{fill:#000000;}#chatgpt-mermaid-\_r_24s\_ [data-look="neo"].icon-shape .icon{fill:url(#chatgpt-mermaid-\_r_24s\_-gradient);filter:drop-shadow( 1px 2px 2px rgba(185,185,185,1));}#chatgpt-mermaid-\_r_24s\_ [data-look="neo"].icon-shape .icon-neo path{stroke:url(#chatgpt-mermaid-\_r_24s\_-gradient);filter:drop-shadow( 1px 2px 2px rgba(185,185,185,1));}#chatgpt-mermaid-\_r_24s\_ :root{--mermaid-font-family:-apple-system-body,ui-sans-serif,-apple-system,system-ui,"Segoe UI",Helvetica,"Apple Color Emoji",Arial,sans-serif,"Segoe UI Emoji","Segoe UI Symbol";}add_request_async(request)设置 client_indexEncoder.encode(request)send_multipart(identity, ADD, data)路由到指定 Engine处理 EngineCoreRequestEngineCoreOutputs返回输出PULL 接收Decoder.decode()outputs_queue.put_nowait()get_output_async()

这里需要区分两个动作：

- `add_request_async()`：把请求发送出去。
- `get_output_async()`：从异步队列获取结果。

它们不是一个同步的请求-响应函数。

请求发送后，前端可以继续处理其他请求，不需要阻塞等待当前请求生成完毕。

### **4.3 结果接收**

`AsyncMPClient._ensure_output_queue_task()` 创建后台 Task：

```
async def process_outputs_socket():    while True:        frames = await output_socket.recv_multipart(            copy=False        )        outputs = decoder.decode(frames)        if outputs.utility_output:            _process_utility_output(                outputs.utility_output,                utility_results            )            continue        if outputs.outputs or outputs.scheduler_stats:            outputs_queue.put_nowait(outputs)
```

这里有一个关键设计：

**结果接收与业务读取解耦。**

```
ZMQ Socket
    ↓
后台 asyncio Task
    ↓
Decoder
    ↓
分类处理
    ├── UtilityOutput → Future
    └── 普通推理结果 → asyncio.Queue
                              ↓
                       get_output_async()
```

这意味着：

- ZMQ 接收任务不会等待上层 API 消费结果。
- 上层通过 Queue 获取结果。
- Utility RPC 通过 Future 获取对应调用结果。

粘贴的文本 (1)(20261115-152447).txt

## **五、关键流程二：Utility RPC**

这是第二个必须掌握的机制。

EngineCoreClient 不只是发送推理请求，还需要执行各种控制操作。

例如：

```
await client.pause_scheduler_async()await client.resume_scheduler_async()await client.reset_prefix_cache_async()await client.add_lora_async(...)await client.collective_rpc_async(...)
```

这些操作不是普通的 `ADD` 请求，而是通过 `UTILITY` 消息发送。

### **5.1 Utility RPC 的核心实现**

```
async def _call_utility_async(self, method, *args, engine):    call_id = uuid.uuid1().int >> 64    future = asyncio.get_running_loop().create_future()    self.utility_results[call_id] = future    message = (        EngineCoreRequestType.UTILITY.value,        *self.encoder.encode(            (self.client_index, call_id, method, args)        )    )    await self._send_input_message(message, engine)    self._ensure_output_queue_task()    return await future
```

这里有三个核心元素：

| 对象                | 作用                      |
| ----------------- | ----------------------- |
| `call_id`         | 唯一标识一次 RPC 调用           |
| `utility_results` | 保存 call_id 到 Future 的映射 |
| `Future`          | 等待后台执行结果                |

### **5.2 RPC 完整流程**

Output HandlerEngineCoreutility_resultsAsyncMPClientOutput HandlerEngineCoreutility_resultsAsyncMPClient#chatgpt-mermaid-\_r_25r\_{font-family:-apple-system-body,ui-sans-serif,-apple-system,system-ui,"Segoe UI",Helvetica,"Apple Color Emoji",Arial,sans-serif,"Segoe UI Emoji","Segoe UI Symbol";font-size:16px;fill:rgb(13, 13, 13);}@keyframes edge-animation-frame{from{stroke-dashoffset:0;}}@keyframes dash{to{stroke-dashoffset:0;}}#chatgpt-mermaid-\_r_25r\_ .edge-animation-slow{stroke-dasharray:9,5!important;stroke-dashoffset:900;animation:dash 50s linear infinite;stroke-linecap:round;}#chatgpt-mermaid-\_r_25r\_ .edge-animation-fast{stroke-dasharray:9,5!important;stroke-dashoffset:900;animation:dash 20s linear infinite;stroke-linecap:round;}#chatgpt-mermaid-\_r_25r\_ .error-icon{fill:rgb(243, 243, 243);}#chatgpt-mermaid-\_r_25r\_ .error-text{fill:rgb(13, 13, 13);stroke:rgb(13, 13, 13);}#chatgpt-mermaid-\_r_25r\_ .edge-thickness-normal{stroke-width:1px;}#chatgpt-mermaid-\_r_25r\_ .edge-thickness-thick{stroke-width:3.5px;}#chatgpt-mermaid-\_r_25r\_ .edge-pattern-solid{stroke-dasharray:0;}#chatgpt-mermaid-\_r_25r\_ .edge-thickness-invisible{stroke-width:0;fill:none;}#chatgpt-mermaid-\_r_25r\_ .edge-pattern-dashed{stroke-dasharray:3;}#chatgpt-mermaid-\_r_25r\_ .edge-pattern-dotted{stroke-dasharray:2;}#chatgpt-mermaid-\_r_25r\_ .marker{fill:rgb(143, 143, 143);stroke:rgb(143, 143, 143);}#chatgpt-mermaid-\_r_25r\_ .marker.cross{stroke:rgb(143, 143, 143);}#chatgpt-mermaid-\_r_25r\_ svg{font-family:-apple-system-body,ui-sans-serif,-apple-system,system-ui,"Segoe UI",Helvetica,"Apple Color Emoji",Arial,sans-serif,"Segoe UI Emoji","Segoe UI Symbol";font-size:16px;}#chatgpt-mermaid-\_r_25r\_ p{margin:0;}#chatgpt-mermaid-\_r_25r\_ .actor{stroke:rgb(83, 154, 248);fill:rgb(222, 234, 251);stroke-width:1;}#chatgpt-mermaid-\_r_25r\_ rect.actor.outer-path[data-look="neo"]{filter:drop-shadow( 1px 2px 2px rgba(185,185,185,1));}#chatgpt-mermaid-\_r_25r\_ rect.note[data-look="neo"]{stroke:rgb(107, 198, 127);fill:rgb(243, 243, 243);filter:drop-shadow( 1px 2px 2px rgba(185,185,185,1));}#chatgpt-mermaid-\_r_25r\_ text.actor>tspan{fill:rgb(13, 13, 13);stroke:none;}#chatgpt-mermaid-\_r_25r\_ .actor-line{stroke:rgb(143, 143, 143);}#chatgpt-mermaid-\_r_25r\_ .innerArc{stroke-width:1.5;stroke-dasharray:none;}#chatgpt-mermaid-\_r_25r\_ .messageLine0{stroke-width:1.5;stroke-dasharray:none;stroke:rgb(143, 143, 143);}#chatgpt-mermaid-\_r_25r\_ .messageLine1{stroke-width:1.5;stroke-dasharray:2,2;stroke:rgb(143, 143, 143);}#chatgpt-mermaid-\_r_25r\_ [id$="-arrowhead"] path{fill:rgb(143, 143, 143);stroke:rgb(143, 143, 143);}#chatgpt-mermaid-\_r_25r\_ .sequenceNumber{fill:#707070;}#chatgpt-mermaid-\_r_25r\_ [id$="-sequencenumber"]{fill:rgb(143, 143, 143);}#chatgpt-mermaid-\_r_25r\_ [id$="-crosshead"] path{fill:rgb(143, 143, 143);stroke:rgb(143, 143, 143);}#chatgpt-mermaid-\_r_25r\_ .messageText{fill:rgb(13, 13, 13);stroke:none;}#chatgpt-mermaid-\_r_25r\_ .labelBox{stroke:rgba(0, 0, 0, 0.1);fill:rgb(252, 252, 252);filter:none;}#chatgpt-mermaid-\_r_25r\_ .labelText,#chatgpt-mermaid-\_r_25r\_ .labelText>tspan{fill:rgb(13, 13, 13);stroke:none;}#chatgpt-mermaid-\_r_25r\_ .loopText,#chatgpt-mermaid-\_r_25r\_ .loopText>tspan{fill:rgb(13, 13, 13);stroke:none;}#chatgpt-mermaid-\_r_25r\_ .sectionTitle,#chatgpt-mermaid-\_r_25r\_ .sectionTitle>tspan{fill:rgb(13, 13, 13);stroke:none;}#chatgpt-mermaid-\_r_25r\_ .loopLine{stroke-width:2px;stroke-dasharray:2,2;stroke:rgba(0, 0, 0, 0.1);fill:rgba(0, 0, 0, 0.1);}#chatgpt-mermaid-\_r_25r\_ .note{stroke:rgb(107, 198, 127);fill:rgb(243, 243, 243);}#chatgpt-mermaid-\_r_25r\_ .noteText,#chatgpt-mermaid-\_r_25r\_ .noteText>tspan{fill:rgb(13, 13, 13);stroke:none;font-weight:normal;}#chatgpt-mermaid-\_r_25r\_ .activation0{fill:rgb(243, 243, 243);stroke:hsl(0, 0%, 85.2941176471%);}#chatgpt-mermaid-\_r_25r\_ .activation1{fill:rgb(243, 243, 243);stroke:hsl(0, 0%, 85.2941176471%);}#chatgpt-mermaid-\_r_25r\_ .activation2{fill:rgb(243, 243, 243);stroke:hsl(0, 0%, 85.2941176471%);}#chatgpt-mermaid-\_r_25r\_ .actorPopupMenu{position:absolute;}#chatgpt-mermaid-\_r_25r\_ .actorPopupMenuPanel{position:absolute;fill:rgb(222, 234, 251);box-shadow:0px 8px 16px 0px rgba(0,0,0,0.2);filter:drop-shadow(3px 5px 2px rgb(0 0 0 / 0.4));}#chatgpt-mermaid-\_r_25r\_ .actor-man circle,#chatgpt-mermaid-\_r_25r\_ line{fill:rgb(222, 234, 251);stroke-width:2px;}#chatgpt-mermaid-\_r_25r\_ g rect.rect{filter:drop-shadow( 1px 2px 2px rgba(185,185,185,1));stroke:rgb(83, 154, 248);}#chatgpt-mermaid-\_r_25r\_ .node .neo-node{stroke:rgb(83, 154, 248);}#chatgpt-mermaid-\_r_25r\_ [data-look="neo"].node rect,#chatgpt-mermaid-\_r_25r\_ [data-look="neo"].cluster rect,#chatgpt-mermaid-\_r_25r\_ [data-look="neo"].node polygon{stroke:url(#chatgpt-mermaid-\_r_25r\_-gradient);filter:drop-shadow( 1px 2px 2px rgba(185,185,185,1));}#chatgpt-mermaid-\_r_25r\_ [data-look="neo"].swimlane.cluster rect{filter:none;}#chatgpt-mermaid-\_r_25r\_ [data-look="neo"].node path{stroke:url(#chatgpt-mermaid-\_r_25r\_-gradient);stroke-width:1px;}#chatgpt-mermaid-\_r_25r\_ [data-look="neo"].node .outer-path{filter:drop-shadow( 1px 2px 2px rgba(185,185,185,1));}#chatgpt-mermaid-\_r_25r\_ [data-look="neo"].node .neo-line path{stroke:rgb(83, 154, 248);filter:none;}#chatgpt-mermaid-\_r_25r\_ [data-look="neo"].node circle{stroke:url(#chatgpt-mermaid-\_r_25r\_-gradient);filter:drop-shadow( 1px 2px 2px rgba(185,185,185,1));}#chatgpt-mermaid-\_r_25r\_ [data-look="neo"].node circle .state-start{fill:#000000;}#chatgpt-mermaid-\_r_25r\_ [data-look="neo"].icon-shape .icon{fill:url(#chatgpt-mermaid-\_r_25r\_-gradient);filter:drop-shadow( 1px 2px 2px rgba(185,185,185,1));}#chatgpt-mermaid-\_r_25r\_ [data-look="neo"].icon-shape .icon-neo path{stroke:url(#chatgpt-mermaid-\_r_25r\_-gradient);filter:drop-shadow( 1px 2px 2px rgba(185,185,185,1));}#chatgpt-mermaid-\_r_25r\_ :root{--mermaid-font-family:-apple-system-body,ui-sans-serif,-apple-system,system-ui,"Segoe UI",Helvetica,"Apple Color Emoji",Arial,sans-serif,"Segoe UI Emoji","Segoe UI Symbol";}生成 call_id保存 call_id → Future发送 UTILITY(method,args)执行对应方法UtilityOutput(call_id,result)根据 call_id 查找 Futurefuture.set_result(result)await future 返回

结果匹配：

```
def _process_utility_output(output, utility_results):    future = utility_results.pop(output.call_id)    if output.failure_message is not None:        future.set_exception(...)    else:        future.set_result(output.result.result)
```

### **5.3 为什么需要 Future 映射？**

假设同时发出三个控制请求：

```
call_id=101 → pause_scheduler
call_id=102 → reset_prefix_cache
call_id=103 → add_lora
```

后台返回顺序可能是：

```
103 → 101 → 102
```

如果采用简单的 FIFO 队列，就可能将结果分配给错误的调用者。

使用 `call_id → Future` 映射，可以做到：

- 多个 RPC 并发执行。
- 返回顺序不必与请求顺序一致。
- 每个调用者等待自己的 Future。
- 失败通过 `set_exception()` 传递。

这实际上是一个典型的异步 RPC 请求关联机制。

粘贴的文本 (1)(20261115-152447).txt

## **六、SyncMPClient 与 AsyncMPClient 的区别**

这两个类继承同一个 `MPClient`，但结果接收模型不同。

### **6.1 对比**

| 维度     | SyncMPClient                | AsyncMPClient       |
| ------ | --------------------------- | ------------------- |
| Socket | zmq.Socket                  | zmq.asyncio.Socket  |
| 接收方式   | 独立 Thread                   | asyncio Task        |
| 结果队列   | `queue.Queue`               | `asyncio.Queue`     |
| 等待结果   | `queue.get()`               | `await queue.get()` |
| RPC 等待 | `concurrent.futures.Future` | `asyncio.Future`    |
| 发送     | 同步 send                     | await send          |
| 典型使用   | LLM                         | AsyncLLM            |

### **6.2 SyncMPClient 的线程模型**

\#chatgpt-mermaid-\_r_26i\_{font-family:-apple-system-body,ui-sans-serif,-apple-system,system-ui,"Segoe UI",Helvetica,"Apple Color Emoji",Arial,sans-serif,"Segoe UI Emoji","Segoe UI Symbol";font-size:16px;fill:rgb(13, 13, 13);}@keyframes edge-animation-frame{from{stroke-dashoffset:0;}}@keyframes dash{to{stroke-dashoffset:0;}}#chatgpt-mermaid-\_r_26i\_ .edge-animation-slow{stroke-dasharray:9,5!important;stroke-dashoffset:900;animation:dash 50s linear infinite;stroke-linecap:round;}#chatgpt-mermaid-\_r_26i\_ .edge-animation-fast{stroke-dasharray:9,5!important;stroke-dashoffset:900;animation:dash 20s linear infinite;stroke-linecap:round;}#chatgpt-mermaid-\_r_26i\_ .error-icon{fill:rgb(243, 243, 243);}#chatgpt-mermaid-\_r_26i\_ .error-text{fill:rgb(13, 13, 13);stroke:rgb(13, 13, 13);}#chatgpt-mermaid-\_r_26i\_ .edge-thickness-normal{stroke-width:1px;}#chatgpt-mermaid-\_r_26i\_ .edge-thickness-thick{stroke-width:3.5px;}#chatgpt-mermaid-\_r_26i\_ .edge-pattern-solid{stroke-dasharray:0;}#chatgpt-mermaid-\_r_26i\_ .edge-thickness-invisible{stroke-width:0;fill:none;}#chatgpt-mermaid-\_r_26i\_ .edge-pattern-dashed{stroke-dasharray:3;}#chatgpt-mermaid-\_r_26i\_ .edge-pattern-dotted{stroke-dasharray:2;}#chatgpt-mermaid-\_r_26i\_ .marker{fill:rgb(143, 143, 143);stroke:rgb(143, 143, 143);}#chatgpt-mermaid-\_r_26i\_ .marker.cross{stroke:rgb(143, 143, 143);}#chatgpt-mermaid-\_r_26i\_ svg{font-family:-apple-system-body,ui-sans-serif,-apple-system,system-ui,"Segoe UI",Helvetica,"Apple Color Emoji",Arial,sans-serif,"Segoe UI Emoji","Segoe UI Symbol";font-size:16px;}#chatgpt-mermaid-\_r_26i\_ p{margin:0;}#chatgpt-mermaid-\_r_26i\_ .label{font-family:-apple-system-body,ui-sans-serif,-apple-system,system-ui,"Segoe UI",Helvetica,"Apple Color Emoji",Arial,sans-serif,"Segoe UI Emoji","Segoe UI Symbol";color:rgb(13, 13, 13);}#chatgpt-mermaid-\_r_26i\_ .cluster-label text{fill:rgb(13, 13, 13);}#chatgpt-mermaid-\_r_26i\_ .cluster-label span{color:rgb(13, 13, 13);}#chatgpt-mermaid-\_r_26i\_ .cluster-label span p{background-color:transparent;}#chatgpt-mermaid-\_r_26i\_ .label text,#chatgpt-mermaid-\_r_26i\_ span{fill:rgb(13, 13, 13);color:rgb(13, 13, 13);}#chatgpt-mermaid-\_r_26i\_ .node rect,#chatgpt-mermaid-\_r_26i\_ .node circle,#chatgpt-mermaid-\_r_26i\_ .node ellipse,#chatgpt-mermaid-\_r_26i\_ .node polygon,#chatgpt-mermaid-\_r_26i\_ .node path{fill:rgb(222, 234, 251);stroke:rgb(83, 154, 248);stroke-width:1px;}#chatgpt-mermaid-\_r_26i\_ .rough-node .label text,#chatgpt-mermaid-\_r_26i\_ .node .label text,#chatgpt-mermaid-\_r_26i\_ .image-shape .label,#chatgpt-mermaid-\_r_26i\_ .icon-shape .label{text-anchor:middle;}#chatgpt-mermaid-\_r_26i\_ .node .katex path{fill:#000;stroke:#000;stroke-width:1px;}#chatgpt-mermaid-\_r_26i\_ .rough-node .label,#chatgpt-mermaid-\_r_26i\_ .node .label,#chatgpt-mermaid-\_r_26i\_ .image-shape .label,#chatgpt-mermaid-\_r_26i\_ .icon-shape .label{text-align:center;}#chatgpt-mermaid-\_r_26i\_ .node.clickable{cursor:pointer;}#chatgpt-mermaid-\_r_26i\_ .root .anchor path{fill:rgb(143, 143, 143)!important;stroke-width:0;stroke:rgb(143, 143, 143);}#chatgpt-mermaid-\_r_26i\_ .arrowheadPath{fill:rgb(143, 143, 143);}#chatgpt-mermaid-\_r_26i\_ .edgePath .path{stroke:rgb(143, 143, 143);stroke-width:1px;}#chatgpt-mermaid-\_r_26i\_ .flowchart-link{stroke:rgb(143, 143, 143);fill:none;}#chatgpt-mermaid-\_r_26i\_ .edgeLabel{background-color:rgb(252, 252, 252);text-align:center;}#chatgpt-mermaid-\_r_26i\_ .edgeLabel p{background-color:rgb(252, 252, 252);}#chatgpt-mermaid-\_r_26i\_ .edgeLabel rect{opacity:0.5;background-color:rgb(252, 252, 252);fill:rgb(252, 252, 252);}#chatgpt-mermaid-\_r_26i\_ .labelBkg{background-color:rgba(252, 252, 252, 0.5);}#chatgpt-mermaid-\_r_26i\_ .cluster rect{fill:rgb(243, 243, 243);stroke:rgba(0, 0, 0, 0.1);stroke-width:1px;}#chatgpt-mermaid-\_r_26i\_ .cluster text{fill:rgb(13, 13, 13);}#chatgpt-mermaid-\_r_26i\_ .cluster span{color:rgb(13, 13, 13);}#chatgpt-mermaid-\_r_26i\_ div.mermaidTooltip{position:absolute;text-align:center;max-width:200px;padding:2px;font-family:-apple-system-body,ui-sans-serif,-apple-system,system-ui,"Segoe UI",Helvetica,"Apple Color Emoji",Arial,sans-serif,"Segoe UI Emoji","Segoe UI Symbol";font-size:12px;background:rgb(243, 243, 243);border:1px solid rgba(0, 0, 0, 0.1);border-radius:2px;pointer-events:none;z-index:100;}#chatgpt-mermaid-\_r_26i\_ .flowchartTitleText{text-anchor:middle;font-size:18px;fill:rgb(13, 13, 13);}#chatgpt-mermaid-\_r_26i\_ rect.text{fill:none;stroke-width:0;}#chatgpt-mermaid-\_r_26i\_ .icon-shape,#chatgpt-mermaid-\_r_26i\_ .image-shape{background-color:rgb(252, 252, 252);text-align:center;}#chatgpt-mermaid-\_r_26i\_ .icon-shape p,#chatgpt-mermaid-\_r_26i\_ .image-shape p{background-color:rgb(252, 252, 252);padding:2px;}#chatgpt-mermaid-\_r_26i\_ .icon-shape .label rect,#chatgpt-mermaid-\_r_26i\_ .image-shape .label rect{opacity:0.5;background-color:rgb(252, 252, 252);fill:rgb(252, 252, 252);}#chatgpt-mermaid-\_r_26i\_ .label-icon{display:inline-block;height:1em;overflow:visible;vertical-align:-0.125em;}#chatgpt-mermaid-\_r_26i\_ .node .label-icon path{fill:currentColor;stroke:revert;stroke-width:revert;}#chatgpt-mermaid-\_r_26i\_ .node .neo-node{stroke:rgb(83, 154, 248);}#chatgpt-mermaid-\_r_26i\_ [data-look="neo"].node rect,#chatgpt-mermaid-\_r_26i\_ [data-look="neo"].cluster rect,#chatgpt-mermaid-\_r_26i\_ [data-look="neo"].node polygon{stroke:url(#chatgpt-mermaid-\_r_26i\_-gradient);filter:drop-shadow( 1px 2px 2px rgba(185,185,185,1));}#chatgpt-mermaid-\_r_26i\_ [data-look="neo"].swimlane.cluster rect{filter:none;}#chatgpt-mermaid-\_r_26i\_ [data-look="neo"].node path{stroke:url(#chatgpt-mermaid-\_r_26i\_-gradient);stroke-width:1px;}#chatgpt-mermaid-\_r_26i\_ [data-look="neo"].node .outer-path{filter:drop-shadow( 1px 2px 2px rgba(185,185,185,1));}#chatgpt-mermaid-\_r_26i\_ [data-look="neo"].node .neo-line path{stroke:rgb(83, 154, 248);filter:none;}#chatgpt-mermaid-\_r_26i\_ [data-look="neo"].node circle{stroke:url(#chatgpt-mermaid-\_r_26i\_-gradient);filter:drop-shadow( 1px 2px 2px rgba(185,185,185,1));}#chatgpt-mermaid-\_r_26i\_ [data-look="neo"].node circle .state-start{fill:#000000;}#chatgpt-mermaid-\_r_26i\_ [data-look="neo"].icon-shape .icon{fill:url(#chatgpt-mermaid-\_r_26i\_-gradient);filter:drop-shadow( 1px 2px 2px rgba(185,185,185,1));}#chatgpt-mermaid-\_r_26i\_ [data-look="neo"].icon-shape .icon-neo path{stroke:url(#chatgpt-mermaid-\_r_26i\_-gradient);filter:drop-shadow( 1px 2px 2px rgba(185,185,185,1));}#chatgpt-mermaid-\_r_26i\_ .node text{font-size:14px;font-weight:600;letter-spacing:normal;fill:rgb(0, 79, 153);}#chatgpt-mermaid-\_r_26i\_ .edgeLabels text{font-size:13px;font-weight:600;letter-spacing:-0.08px;fill:rgb(0, 79, 153);}#chatgpt-mermaid-\_r_26i\_ .node tspan[font-weight="normal"],#chatgpt-mermaid-\_r_26i\_ .edgeLabels tspan[font-weight="normal"]{font-weight:600;}#chatgpt-mermaid-\_r_26i\_ .edgeLabel .label rect{opacity:1;rx:13px;ry:13px;fill:rgb(245, 250, 255);stroke:rgb(206, 219, 229);stroke-width:1px;}#chatgpt-mermaid-\_r_26i\_ .node rect,#chatgpt-mermaid-\_r_26i\_ .node circle,#chatgpt-mermaid-\_r_26i\_ .node ellipse,#chatgpt-mermaid-\_r_26i\_ .node polygon,#chatgpt-mermaid-\_r_26i\_ .node path{fill:rgb(229, 243, 255);stroke:rgba(0, 0, 0, 0.1);stroke-width:1px;}#chatgpt-mermaid-\_r_26i\_ .node rect{rx:16px;ry:16px;}#chatgpt-mermaid-\_r_26i\_ .mermaid-decision .label-container{fill:rgb(245, 250, 255);stroke:rgb(206, 219, 229);stroke-dasharray:2,2;}#chatgpt-mermaid-\_r_26i\_ .edgePaths .flowchart-link{stroke:rgb(143, 143, 143);stroke-width:1px;stroke-linecap:round;stroke-linejoin:round;}#chatgpt-mermaid-\_r_26i\_ .marker{fill:rgb(143, 143, 143);stroke:rgb(143, 143, 143);}#chatgpt-mermaid-\_r_26i\_ :root{--mermaid-font-family:-apple-system-body,ui-sans-serif,-apple-system,system-ui,"Segoe UI",Helvetica,"Apple Color Emoji",Arial,sans-serif,"Segoe UI Emoji","Segoe UI Symbol";}SyncMPClient发送请求get_output()Output ThreadZMQ PULLDecoder消息类型Future.set_resultqueue.QueueUtility普通结果

关键代码：

```
self.output_queue_thread = Thread(    target=process_outputs_socket,    daemon=True,)self.output_queue_thread.start()
```

线程负责持续接收 ZMQ 消息。

而 `get_output()`：

```
outputs = self.outputs_queue.get()
```

会阻塞等待结果。

### **6.3 AsyncMPClient 的 Task 模型**

\#chatgpt-mermaid-\_r_275\_{font-family:-apple-system-body,ui-sans-serif,-apple-system,system-ui,"Segoe UI",Helvetica,"Apple Color Emoji",Arial,sans-serif,"Segoe UI Emoji","Segoe UI Symbol";font-size:16px;fill:rgb(13, 13, 13);}@keyframes edge-animation-frame{from{stroke-dashoffset:0;}}@keyframes dash{to{stroke-dashoffset:0;}}#chatgpt-mermaid-\_r_275\_ .edge-animation-slow{stroke-dasharray:9,5!important;stroke-dashoffset:900;animation:dash 50s linear infinite;stroke-linecap:round;}#chatgpt-mermaid-\_r_275\_ .edge-animation-fast{stroke-dasharray:9,5!important;stroke-dashoffset:900;animation:dash 20s linear infinite;stroke-linecap:round;}#chatgpt-mermaid-\_r_275\_ .error-icon{fill:rgb(243, 243, 243);}#chatgpt-mermaid-\_r_275\_ .error-text{fill:rgb(13, 13, 13);stroke:rgb(13, 13, 13);}#chatgpt-mermaid-\_r_275\_ .edge-thickness-normal{stroke-width:1px;}#chatgpt-mermaid-\_r_275\_ .edge-thickness-thick{stroke-width:3.5px;}#chatgpt-mermaid-\_r_275\_ .edge-pattern-solid{stroke-dasharray:0;}#chatgpt-mermaid-\_r_275\_ .edge-thickness-invisible{stroke-width:0;fill:none;}#chatgpt-mermaid-\_r_275\_ .edge-pattern-dashed{stroke-dasharray:3;}#chatgpt-mermaid-\_r_275\_ .edge-pattern-dotted{stroke-dasharray:2;}#chatgpt-mermaid-\_r_275\_ .marker{fill:rgb(143, 143, 143);stroke:rgb(143, 143, 143);}#chatgpt-mermaid-\_r_275\_ .marker.cross{stroke:rgb(143, 143, 143);}#chatgpt-mermaid-\_r_275\_ svg{font-family:-apple-system-body,ui-sans-serif,-apple-system,system-ui,"Segoe UI",Helvetica,"Apple Color Emoji",Arial,sans-serif,"Segoe UI Emoji","Segoe UI Symbol";font-size:16px;}#chatgpt-mermaid-\_r_275\_ p{margin:0;}#chatgpt-mermaid-\_r_275\_ .label{font-family:-apple-system-body,ui-sans-serif,-apple-system,system-ui,"Segoe UI",Helvetica,"Apple Color Emoji",Arial,sans-serif,"Segoe UI Emoji","Segoe UI Symbol";color:rgb(13, 13, 13);}#chatgpt-mermaid-\_r_275\_ .cluster-label text{fill:rgb(13, 13, 13);}#chatgpt-mermaid-\_r_275\_ .cluster-label span{color:rgb(13, 13, 13);}#chatgpt-mermaid-\_r_275\_ .cluster-label span p{background-color:transparent;}#chatgpt-mermaid-\_r_275\_ .label text,#chatgpt-mermaid-\_r_275\_ span{fill:rgb(13, 13, 13);color:rgb(13, 13, 13);}#chatgpt-mermaid-\_r_275\_ .node rect,#chatgpt-mermaid-\_r_275\_ .node circle,#chatgpt-mermaid-\_r_275\_ .node ellipse,#chatgpt-mermaid-\_r_275\_ .node polygon,#chatgpt-mermaid-\_r_275\_ .node path{fill:rgb(222, 234, 251);stroke:rgb(83, 154, 248);stroke-width:1px;}#chatgpt-mermaid-\_r_275\_ .rough-node .label text,#chatgpt-mermaid-\_r_275\_ .node .label text,#chatgpt-mermaid-\_r_275\_ .image-shape .label,#chatgpt-mermaid-\_r_275\_ .icon-shape .label{text-anchor:middle;}#chatgpt-mermaid-\_r_275\_ .node .katex path{fill:#000;stroke:#000;stroke-width:1px;}#chatgpt-mermaid-\_r_275\_ .rough-node .label,#chatgpt-mermaid-\_r_275\_ .node .label,#chatgpt-mermaid-\_r_275\_ .image-shape .label,#chatgpt-mermaid-\_r_275\_ .icon-shape .label{text-align:center;}#chatgpt-mermaid-\_r_275\_ .node.clickable{cursor:pointer;}#chatgpt-mermaid-\_r_275\_ .root .anchor path{fill:rgb(143, 143, 143)!important;stroke-width:0;stroke:rgb(143, 143, 143);}#chatgpt-mermaid-\_r_275\_ .arrowheadPath{fill:rgb(143, 143, 143);}#chatgpt-mermaid-\_r_275\_ .edgePath .path{stroke:rgb(143, 143, 143);stroke-width:1px;}#chatgpt-mermaid-\_r_275\_ .flowchart-link{stroke:rgb(143, 143, 143);fill:none;}#chatgpt-mermaid-\_r_275\_ .edgeLabel{background-color:rgb(252, 252, 252);text-align:center;}#chatgpt-mermaid-\_r_275\_ .edgeLabel p{background-color:rgb(252, 252, 252);}#chatgpt-mermaid-\_r_275\_ .edgeLabel rect{opacity:0.5;background-color:rgb(252, 252, 252);fill:rgb(252, 252, 252);}#chatgpt-mermaid-\_r_275\_ .labelBkg{background-color:rgba(252, 252, 252, 0.5);}#chatgpt-mermaid-\_r_275\_ .cluster rect{fill:rgb(243, 243, 243);stroke:rgba(0, 0, 0, 0.1);stroke-width:1px;}#chatgpt-mermaid-\_r_275\_ .cluster text{fill:rgb(13, 13, 13);}#chatgpt-mermaid-\_r_275\_ .cluster span{color:rgb(13, 13, 13);}#chatgpt-mermaid-\_r_275\_ div.mermaidTooltip{position:absolute;text-align:center;max-width:200px;padding:2px;font-family:-apple-system-body,ui-sans-serif,-apple-system,system-ui,"Segoe UI",Helvetica,"Apple Color Emoji",Arial,sans-serif,"Segoe UI Emoji","Segoe UI Symbol";font-size:12px;background:rgb(243, 243, 243);border:1px solid rgba(0, 0, 0, 0.1);border-radius:2px;pointer-events:none;z-index:100;}#chatgpt-mermaid-\_r_275\_ .flowchartTitleText{text-anchor:middle;font-size:18px;fill:rgb(13, 13, 13);}#chatgpt-mermaid-\_r_275\_ rect.text{fill:none;stroke-width:0;}#chatgpt-mermaid-\_r_275\_ .icon-shape,#chatgpt-mermaid-\_r_275\_ .image-shape{background-color:rgb(252, 252, 252);text-align:center;}#chatgpt-mermaid-\_r_275\_ .icon-shape p,#chatgpt-mermaid-\_r_275\_ .image-shape p{background-color:rgb(252, 252, 252);padding:2px;}#chatgpt-mermaid-\_r_275\_ .icon-shape .label rect,#chatgpt-mermaid-\_r_275\_ .image-shape .label rect{opacity:0.5;background-color:rgb(252, 252, 252);fill:rgb(252, 252, 252);}#chatgpt-mermaid-\_r_275\_ .label-icon{display:inline-block;height:1em;overflow:visible;vertical-align:-0.125em;}#chatgpt-mermaid-\_r_275\_ .node .label-icon path{fill:currentColor;stroke:revert;stroke-width:revert;}#chatgpt-mermaid-\_r_275\_ .node .neo-node{stroke:rgb(83, 154, 248);}#chatgpt-mermaid-\_r_275\_ [data-look="neo"].node rect,#chatgpt-mermaid-\_r_275\_ [data-look="neo"].cluster rect,#chatgpt-mermaid-\_r_275\_ [data-look="neo"].node polygon{stroke:url(#chatgpt-mermaid-\_r_275\_-gradient);filter:drop-shadow( 1px 2px 2px rgba(185,185,185,1));}#chatgpt-mermaid-\_r_275\_ [data-look="neo"].swimlane.cluster rect{filter:none;}#chatgpt-mermaid-\_r_275\_ [data-look="neo"].node path{stroke:url(#chatgpt-mermaid-\_r_275\_-gradient);stroke-width:1px;}#chatgpt-mermaid-\_r_275\_ [data-look="neo"].node .outer-path{filter:drop-shadow( 1px 2px 2px rgba(185,185,185,1));}#chatgpt-mermaid-\_r_275\_ [data-look="neo"].node .neo-line path{stroke:rgb(83, 154, 248);filter:none;}#chatgpt-mermaid-\_r_275\_ [data-look="neo"].node circle{stroke:url(#chatgpt-mermaid-\_r_275\_-gradient);filter:drop-shadow( 1px 2px 2px rgba(185,185,185,1));}#chatgpt-mermaid-\_r_275\_ [data-look="neo"].node circle .state-start{fill:#000000;}#chatgpt-mermaid-\_r_275\_ [data-look="neo"].icon-shape .icon{fill:url(#chatgpt-mermaid-\_r_275\_-gradient);filter:drop-shadow( 1px 2px 2px rgba(185,185,185,1));}#chatgpt-mermaid-\_r_275\_ [data-look="neo"].icon-shape .icon-neo path{stroke:url(#chatgpt-mermaid-\_r_275\_-gradient);filter:drop-shadow( 1px 2px 2px rgba(185,185,185,1));}#chatgpt-mermaid-\_r_275\_ .node text{font-size:14px;font-weight:600;letter-spacing:normal;fill:rgb(0, 79, 153);}#chatgpt-mermaid-\_r_275\_ .edgeLabels text{font-size:13px;font-weight:600;letter-spacing:-0.08px;fill:rgb(0, 79, 153);}#chatgpt-mermaid-\_r_275\_ .node tspan[font-weight="normal"],#chatgpt-mermaid-\_r_275\_ .edgeLabels tspan[font-weight="normal"]{font-weight:600;}#chatgpt-mermaid-\_r_275\_ .edgeLabel .label rect{opacity:1;rx:13px;ry:13px;fill:rgb(245, 250, 255);stroke:rgb(206, 219, 229);stroke-width:1px;}#chatgpt-mermaid-\_r_275\_ .node rect,#chatgpt-mermaid-\_r_275\_ .node circle,#chatgpt-mermaid-\_r_275\_ .node ellipse,#chatgpt-mermaid-\_r_275\_ .node polygon,#chatgpt-mermaid-\_r_275\_ .node path{fill:rgb(229, 243, 255);stroke:rgba(0, 0, 0, 0.1);stroke-width:1px;}#chatgpt-mermaid-\_r_275\_ .node rect{rx:16px;ry:16px;}#chatgpt-mermaid-\_r_275\_ .mermaid-decision .label-container{fill:rgb(245, 250, 255);stroke:rgb(206, 219, 229);stroke-dasharray:2,2;}#chatgpt-mermaid-\_r_275\_ .edgePaths .flowchart-link{stroke:rgb(143, 143, 143);stroke-width:1px;stroke-linecap:round;stroke-linejoin:round;}#chatgpt-mermaid-\_r_275\_ .marker{fill:rgb(143, 143, 143);stroke:rgb(143, 143, 143);}#chatgpt-mermaid-\_r_275\_ :root{--mermaid-font-family:-apple-system-body,ui-sans-serif,-apple-system,system-ui,"Segoe UI",Helvetica,"Apple Color Emoji",Arial,sans-serif,"Segoe UI Emoji","Segoe UI Symbol";}AsyncMPClientadd_request_async()get_output_async()Output Taskawait ZMQ recvDecoder消息类型Future.set_resultasyncio.QueueUtility普通结果

核心区别：

- Sync：线程负责 I/O，Queue 负责跨线程交付。
- Async：Task 负责 I/O，asyncio.Queue 负责异步交付。

**不要把 AsyncMPClient 理解为没有后台接收循环，它同样有一个持续运行的接收任务。**

## **七、关键流程三：Data Parallel 请求负载均衡**

这是这份代码中最有价值的分布式系统设计。

重点阅读：

```
DPLBAsyncMPClient.get_core_engine_for_request()
```

### **7.1 DP 内部 LB 与外部 LB**

**External Load Balancing**

Load Balancer

Client 0

DP Rank 0

Client 1

DP Rank 1

**Internal Load Balancing**

One DPLBAsyncMPClient

**Engine 0**

DP Rank 0

**Engine 1**

DP Rank 1

**Engine 2**

DP Rank 2

内部 LB 由客户端选择具体 EngineCore；外部 LB 由外部机制决定请求落在哪个 DP Rank。

区别：

- `DPAsyncMPClient`：请求默认发往自己的 `core_engine`。
- `DPLBAsyncMPClient`：客户端自己在多个 EngineCore 中选择目标。

### **7.2 负载均衡依据**

DPLBAsyncMPClient 保存三个重要变量：

```
self.lb_enginesself.reqs_in_flightself.engine_inflight
```

含义：

| 变量                | 含义                              |
| ----------------- | ------------------------------- |
| `lb_engines`      | 每个 Engine 的等待数、运行数、KV Cache 使用率 |
| `reqs_in_flight`  | request_id → EngineIdentity     |
| `engine_inflight` | 每个 Engine 当前未完成请求数量             |

例如：

```
lb_engines = [    [10, 5, 0.9],    [2, 3, 0.3],    [4, 2, 0.5],]
```

代表：

| Engine | Waiting | Running | KV 使用率 |
| ------ | ------- | ------- | ------ |
| 0      | 10      | 5       | 90%    |
| 1      | 2       | 3       | 30%    |
| 2      | 4       | 2       | 50%    |

Engine 0 有较高的等待量和 KV Cache 压力，因此新请求应该优先考虑其他 Engine。

### **7.3 负载评分公式**

源码中：

```
score = max(    self.client_count * inflight,    waiting + running)if waiting:    score += waiting * 6.0 * max(        0.0,        kv_cache_usage - 0.5    )
```

可以写成：

\\[ S_i = \max(C \times I_i,\ W_i+R_i) + 6W_i\max(0,K_i-0.5) \\]

其中：

| 符号        | 含义                       |
| --------- | ------------------------ |
| \\(C\\)   | 客户端数量                    |
| \\(I_i\\) | 当前客户端在 Engine i 上的未完成请求数 |
| \\(W_i\\) | Engine i 的等待请求数          |
| \\(R_i\\) | Engine i 的运行请求数          |
| \\(K_i\\) | Engine i 的 KV Cache 使用率  |

最终选择：

\\[ Engine\_{selected}=\arg\min_i S_i \\]

### **7.4 为什么要乘 KV Cache 惩罚项？**

假设两个 Engine：

| 指标      | Engine A | Engine B |
| ------- | -------- | -------- |
| Waiting | 5        | 5        |
| Running | 3        | 3        |
| KV 使用率  | 90%      | 40%      |

Engine A 的等待队列虽然和 B 一样，但它的 KV Cache 已经接近满载。

继续向 A 分配请求可能导致：

- KV Cache 资源不足。
- 请求等待时间增加。
- Scheduler 调度空间受限。
- 请求吞吐下降。

因此增加惩罚项：

\\[ Penalty=6W\max(0,K-0.5) \\]

A：

\\[ 5\times6\times(0.9-0.5)=12 \\]

B：

\\[ 5\times6\times\max(0,0.4-0.5)=0 \\]

这意味着相同等待队列长度下，A 会获得更高负载评分。

### **7.5 一个重要细节：本地请求计数**

```
current_counts[eng_index][0] += self.client_count
```

为什么要本地增加 Waiting？

因为 Coordinator 的统计数据并不是实时更新的，源码注释指出统计更新周期约为 100ms。

如果短时间内连续到达多个请求，而本地不更新计数，客户端可能连续把请求发到同一个 Engine。

因此：

- Coordinator 提供全局统计快照。
- Client 使用本地计数弥补统计延迟。
- `engine_inflight` 提供当前客户端的精确请求计数。
- `eng_start_index` 轮转起始位置，避免相同评分时总是优先选择同一个 Engine。

这是一个典型的**全局近似统计 + 本地精确状态**的负载均衡设计。

粘贴的文本 (1)(20261115-152447).txt

### **7.6 为什么需要 reqs_in_flight？**

请求分配后：

```
self.reqs_in_flight[request.request_id] = chosen_engineself.engine_inflight[chosen_engine] += 1
```

当请求完成：

```
async def process_engine_outputs(self, outputs):    for req_id in outputs.finished_requests:        engine = self.reqs_in_flight.pop(req_id, None)        if engine is not None:            self.engine_inflight[engine] -= 1
```

这两个操作构成完整的生命周期：

\#chatgpt-mermaid-\_r_294\_{font-family:-apple-system-body,ui-sans-serif,-apple-system,system-ui,"Segoe UI",Helvetica,"Apple Color Emoji",Arial,sans-serif,"Segoe UI Emoji","Segoe UI Symbol";font-size:16px;fill:rgb(13, 13, 13);}@keyframes edge-animation-frame{from{stroke-dashoffset:0;}}@keyframes dash{to{stroke-dashoffset:0;}}#chatgpt-mermaid-\_r_294\_ .edge-animation-slow{stroke-dasharray:9,5!important;stroke-dashoffset:900;animation:dash 50s linear infinite;stroke-linecap:round;}#chatgpt-mermaid-\_r_294\_ .edge-animation-fast{stroke-dasharray:9,5!important;stroke-dashoffset:900;animation:dash 20s linear infinite;stroke-linecap:round;}#chatgpt-mermaid-\_r_294\_ .error-icon{fill:rgb(243, 243, 243);}#chatgpt-mermaid-\_r_294\_ .error-text{fill:rgb(13, 13, 13);stroke:rgb(13, 13, 13);}#chatgpt-mermaid-\_r_294\_ .edge-thickness-normal{stroke-width:1px;}#chatgpt-mermaid-\_r_294\_ .edge-thickness-thick{stroke-width:3.5px;}#chatgpt-mermaid-\_r_294\_ .edge-pattern-solid{stroke-dasharray:0;}#chatgpt-mermaid-\_r_294\_ .edge-thickness-invisible{stroke-width:0;fill:none;}#chatgpt-mermaid-\_r_294\_ .edge-pattern-dashed{stroke-dasharray:3;}#chatgpt-mermaid-\_r_294\_ .edge-pattern-dotted{stroke-dasharray:2;}#chatgpt-mermaid-\_r_294\_ .marker{fill:rgb(143, 143, 143);stroke:rgb(143, 143, 143);}#chatgpt-mermaid-\_r_294\_ .marker.cross{stroke:rgb(143, 143, 143);}#chatgpt-mermaid-\_r_294\_ svg{font-family:-apple-system-body,ui-sans-serif,-apple-system,system-ui,"Segoe UI",Helvetica,"Apple Color Emoji",Arial,sans-serif,"Segoe UI Emoji","Segoe UI Symbol";font-size:16px;}#chatgpt-mermaid-\_r_294\_ p{margin:0;}#chatgpt-mermaid-\_r_294\_ .label{font-family:-apple-system-body,ui-sans-serif,-apple-system,system-ui,"Segoe UI",Helvetica,"Apple Color Emoji",Arial,sans-serif,"Segoe UI Emoji","Segoe UI Symbol";color:rgb(13, 13, 13);}#chatgpt-mermaid-\_r_294\_ .cluster-label text{fill:rgb(13, 13, 13);}#chatgpt-mermaid-\_r_294\_ .cluster-label span{color:rgb(13, 13, 13);}#chatgpt-mermaid-\_r_294\_ .cluster-label span p{background-color:transparent;}#chatgpt-mermaid-\_r_294\_ .label text,#chatgpt-mermaid-\_r_294\_ span{fill:rgb(13, 13, 13);color:rgb(13, 13, 13);}#chatgpt-mermaid-\_r_294\_ .node rect,#chatgpt-mermaid-\_r_294\_ .node circle,#chatgpt-mermaid-\_r_294\_ .node ellipse,#chatgpt-mermaid-\_r_294\_ .node polygon,#chatgpt-mermaid-\_r_294\_ .node path{fill:rgb(222, 234, 251);stroke:rgb(83, 154, 248);stroke-width:1px;}#chatgpt-mermaid-\_r_294\_ .rough-node .label text,#chatgpt-mermaid-\_r_294\_ .node .label text,#chatgpt-mermaid-\_r_294\_ .image-shape .label,#chatgpt-mermaid-\_r_294\_ .icon-shape .label{text-anchor:middle;}#chatgpt-mermaid-\_r_294\_ .node .katex path{fill:#000;stroke:#000;stroke-width:1px;}#chatgpt-mermaid-\_r_294\_ .rough-node .label,#chatgpt-mermaid-\_r_294\_ .node .label,#chatgpt-mermaid-\_r_294\_ .image-shape .label,#chatgpt-mermaid-\_r_294\_ .icon-shape .label{text-align:center;}#chatgpt-mermaid-\_r_294\_ .node.clickable{cursor:pointer;}#chatgpt-mermaid-\_r_294\_ .root .anchor path{fill:rgb(143, 143, 143)!important;stroke-width:0;stroke:rgb(143, 143, 143);}#chatgpt-mermaid-\_r_294\_ .arrowheadPath{fill:rgb(143, 143, 143);}#chatgpt-mermaid-\_r_294\_ .edgePath .path{stroke:rgb(143, 143, 143);stroke-width:1px;}#chatgpt-mermaid-\_r_294\_ .flowchart-link{stroke:rgb(143, 143, 143);fill:none;}#chatgpt-mermaid-\_r_294\_ .edgeLabel{background-color:rgb(252, 252, 252);text-align:center;}#chatgpt-mermaid-\_r_294\_ .edgeLabel p{background-color:rgb(252, 252, 252);}#chatgpt-mermaid-\_r_294\_ .edgeLabel rect{opacity:0.5;background-color:rgb(252, 252, 252);fill:rgb(252, 252, 252);}#chatgpt-mermaid-\_r_294\_ .labelBkg{background-color:rgba(252, 252, 252, 0.5);}#chatgpt-mermaid-\_r_294\_ .cluster rect{fill:rgb(243, 243, 243);stroke:rgba(0, 0, 0, 0.1);stroke-width:1px;}#chatgpt-mermaid-\_r_294\_ .cluster text{fill:rgb(13, 13, 13);}#chatgpt-mermaid-\_r_294\_ .cluster span{color:rgb(13, 13, 13);}#chatgpt-mermaid-\_r_294\_ div.mermaidTooltip{position:absolute;text-align:center;max-width:200px;padding:2px;font-family:-apple-system-body,ui-sans-serif,-apple-system,system-ui,"Segoe UI",Helvetica,"Apple Color Emoji",Arial,sans-serif,"Segoe UI Emoji","Segoe UI Symbol";font-size:12px;background:rgb(243, 243, 243);border:1px solid rgba(0, 0, 0, 0.1);border-radius:2px;pointer-events:none;z-index:100;}#chatgpt-mermaid-\_r_294\_ .flowchartTitleText{text-anchor:middle;font-size:18px;fill:rgb(13, 13, 13);}#chatgpt-mermaid-\_r_294\_ rect.text{fill:none;stroke-width:0;}#chatgpt-mermaid-\_r_294\_ .icon-shape,#chatgpt-mermaid-\_r_294\_ .image-shape{background-color:rgb(252, 252, 252);text-align:center;}#chatgpt-mermaid-\_r_294\_ .icon-shape p,#chatgpt-mermaid-\_r_294\_ .image-shape p{background-color:rgb(252, 252, 252);padding:2px;}#chatgpt-mermaid-\_r_294\_ .icon-shape .label rect,#chatgpt-mermaid-\_r_294\_ .image-shape .label rect{opacity:0.5;background-color:rgb(252, 252, 252);fill:rgb(252, 252, 252);}#chatgpt-mermaid-\_r_294\_ .label-icon{display:inline-block;height:1em;overflow:visible;vertical-align:-0.125em;}#chatgpt-mermaid-\_r_294\_ .node .label-icon path{fill:currentColor;stroke:revert;stroke-width:revert;}#chatgpt-mermaid-\_r_294\_ .node .neo-node{stroke:rgb(83, 154, 248);}#chatgpt-mermaid-\_r_294\_ [data-look="neo"].node rect,#chatgpt-mermaid-\_r_294\_ [data-look="neo"].cluster rect,#chatgpt-mermaid-\_r_294\_ [data-look="neo"].node polygon{stroke:url(#chatgpt-mermaid-\_r_294\_-gradient);filter:drop-shadow( 1px 2px 2px rgba(185,185,185,1));}#chatgpt-mermaid-\_r_294\_ [data-look="neo"].swimlane.cluster rect{filter:none;}#chatgpt-mermaid-\_r_294\_ [data-look="neo"].node path{stroke:url(#chatgpt-mermaid-\_r_294\_-gradient);stroke-width:1px;}#chatgpt-mermaid-\_r_294\_ [data-look="neo"].node .outer-path{filter:drop-shadow( 1px 2px 2px rgba(185,185,185,1));}#chatgpt-mermaid-\_r_294\_ [data-look="neo"].node .neo-line path{stroke:rgb(83, 154, 248);filter:none;}#chatgpt-mermaid-\_r_294\_ [data-look="neo"].node circle{stroke:url(#chatgpt-mermaid-\_r_294\_-gradient);filter:drop-shadow( 1px 2px 2px rgba(185,185,185,1));}#chatgpt-mermaid-\_r_294\_ [data-look="neo"].node circle .state-start{fill:#000000;}#chatgpt-mermaid-\_r_294\_ [data-look="neo"].icon-shape .icon{fill:url(#chatgpt-mermaid-\_r_294\_-gradient);filter:drop-shadow( 1px 2px 2px rgba(185,185,185,1));}#chatgpt-mermaid-\_r_294\_ [data-look="neo"].icon-shape .icon-neo path{stroke:url(#chatgpt-mermaid-\_r_294\_-gradient);filter:drop-shadow( 1px 2px 2px rgba(185,185,185,1));}#chatgpt-mermaid-\_r_294\_ .node text{font-size:14px;font-weight:600;letter-spacing:normal;fill:rgb(0, 79, 153);}#chatgpt-mermaid-\_r_294\_ .edgeLabels text{font-size:13px;font-weight:600;letter-spacing:-0.08px;fill:rgb(0, 79, 153);}#chatgpt-mermaid-\_r_294\_ .node tspan[font-weight="normal"],#chatgpt-mermaid-\_r_294\_ .edgeLabels tspan[font-weight="normal"]{font-weight:600;}#chatgpt-mermaid-\_r_294\_ .edgeLabel .label rect{opacity:1;rx:13px;ry:13px;fill:rgb(245, 250, 255);stroke:rgb(206, 219, 229);stroke-width:1px;}#chatgpt-mermaid-\_r_294\_ .node rect,#chatgpt-mermaid-\_r_294\_ .node circle,#chatgpt-mermaid-\_r_294\_ .node ellipse,#chatgpt-mermaid-\_r_294\_ .node polygon,#chatgpt-mermaid-\_r_294\_ .node path{fill:rgb(229, 243, 255);stroke:rgba(0, 0, 0, 0.1);stroke-width:1px;}#chatgpt-mermaid-\_r_294\_ .node rect{rx:16px;ry:16px;}#chatgpt-mermaid-\_r_294\_ .mermaid-decision .label-container{fill:rgb(245, 250, 255);stroke:rgb(206, 219, 229);stroke-dasharray:2,2;}#chatgpt-mermaid-\_r_294\_ .edgePaths .flowchart-link{stroke:rgb(143, 143, 143);stroke-width:1px;stroke-linecap:round;stroke-linejoin:round;}#chatgpt-mermaid-\_r_294\_ .marker{fill:rgb(143, 143, 143);stroke:rgb(143, 143, 143);}#chatgpt-mermaid-\_r_294\_ :root{--mermaid-font-family:-apple-system-body,ui-sans-serif,-apple-system,system-ui,"Segoe UI",Helvetica,"Apple Color Emoji",Arial,sans-serif,"Segoe UI Emoji","Segoe UI Symbol";}收到新请求选择 Engine记录 request_id → Engineinflight + 1Engine 执行请求收到 finished_requests删除 request_id 映射inflight - 1

为什么不能只看 Coordinator 统计？

因为请求可能已经发送出去，但 Coordinator 尚未更新统计数据。

如果本地没有记录请求分配情况：

1. 不能准确跟踪当前客户端的负载。
2. 无法知道某个请求应该向哪个 Engine 发送 ABORT。
3. 无法在请求完成后正确回收计数。

### **7.7 ABORT 如何路由？**

```
engine = self.reqs_in_flight.get(request_id)await self._send_input(    EngineCoreRequestType.ABORT,    request_ids,    engine)
```

如果一次取消多个请求：

```
by_engine = defaultdict(list)for req_id in request_ids:    if engine := self.reqs_in_flight.get(req_id):        by_engine[engine].append(req_id)
```

再按 Engine 分组发送。

例如：

```
请求 A → Engine 0
请求 B → Engine 1
请求 C → Engine 0
请求 D → Engine 2
```

ABORT 会变成：

```
Engine 0: [A, C]
Engine 1: [B]
Engine 2: [D]
```

**这也是为什么请求路由信息必须保留到请求真正结束。**

## **八、关键流程四：DP 状态同步与 First Request**

DPAsyncMPClient 还有一个容易忽略的机制：统计同步。

### **8.1 统计同步**

```
async def _ensure_stats_update_task(self):
```

这个任务使用：

```
zmq.XSUBzmq.PAIR
```

其中：

- XSUB：订阅 Coordinator 发布的统计数据。
- PAIR：客户端内部用于接收首次请求通知和弹性扩缩容通知。

统计消息结构：

```
counts, wave, running = msgspec.msgpack.decode(buf)
```

它更新：

```
self.current_wave = waveself.engines_running = runningself.lb_engines = sliced_counts
```

所以这里存在两类消息：

| 消息        | 用途                    |
| --------- | --------------------- |
| Stats     | 更新负载均衡状态              |
| FIRST_REQ | 通知 Coordinator 有新请求进入 |

### **8.2 为什么需要 FIRST_REQ？**

代码：

```
if not self.engines_running:    req_msg = msgspec.msgpack.encode(        ("FIRST_REQ", chosen_engine)    )    await self.first_req_send_socket.send(req_msg)
```

当 EngineCore 处于暂停状态时，新的请求需要通知 Coordinator。

Coordinator 再向 EngineCore 发送相应通知，使其恢复执行必要的调度循环，例如 EP 场景下的 dummy loop。

流程：

EngineCoreDPCoordinatorDPAsyncMPClientEngineCoreDPCoordinatorDPAsyncMPClient#chatgpt-mermaid-\_r_2at\_{font-family:-apple-system-body,ui-sans-serif,-apple-system,system-ui,"Segoe UI",Helvetica,"Apple Color Emoji",Arial,sans-serif,"Segoe UI Emoji","Segoe UI Symbol";font-size:16px;fill:rgb(13, 13, 13);}@keyframes edge-animation-frame{from{stroke-dashoffset:0;}}@keyframes dash{to{stroke-dashoffset:0;}}#chatgpt-mermaid-\_r_2at\_ .edge-animation-slow{stroke-dasharray:9,5!important;stroke-dashoffset:900;animation:dash 50s linear infinite;stroke-linecap:round;}#chatgpt-mermaid-\_r_2at\_ .edge-animation-fast{stroke-dasharray:9,5!important;stroke-dashoffset:900;animation:dash 20s linear infinite;stroke-linecap:round;}#chatgpt-mermaid-\_r_2at\_ .error-icon{fill:rgb(243, 243, 243);}#chatgpt-mermaid-\_r_2at\_ .error-text{fill:rgb(13, 13, 13);stroke:rgb(13, 13, 13);}#chatgpt-mermaid-\_r_2at\_ .edge-thickness-normal{stroke-width:1px;}#chatgpt-mermaid-\_r_2at\_ .edge-thickness-thick{stroke-width:3.5px;}#chatgpt-mermaid-\_r_2at\_ .edge-pattern-solid{stroke-dasharray:0;}#chatgpt-mermaid-\_r_2at\_ .edge-thickness-invisible{stroke-width:0;fill:none;}#chatgpt-mermaid-\_r_2at\_ .edge-pattern-dashed{stroke-dasharray:3;}#chatgpt-mermaid-\_r_2at\_ .edge-pattern-dotted{stroke-dasharray:2;}#chatgpt-mermaid-\_r_2at\_ .marker{fill:rgb(143, 143, 143);stroke:rgb(143, 143, 143);}#chatgpt-mermaid-\_r_2at\_ .marker.cross{stroke:rgb(143, 143, 143);}#chatgpt-mermaid-\_r_2at\_ svg{font-family:-apple-system-body,ui-sans-serif,-apple-system,system-ui,"Segoe UI",Helvetica,"Apple Color Emoji",Arial,sans-serif,"Segoe UI Emoji","Segoe UI Symbol";font-size:16px;}#chatgpt-mermaid-\_r_2at\_ p{margin:0;}#chatgpt-mermaid-\_r_2at\_ .actor{stroke:rgb(83, 154, 248);fill:rgb(222, 234, 251);stroke-width:1;}#chatgpt-mermaid-\_r_2at\_ rect.actor.outer-path[data-look="neo"]{filter:drop-shadow( 1px 2px 2px rgba(185,185,185,1));}#chatgpt-mermaid-\_r_2at\_ rect.note[data-look="neo"]{stroke:rgb(107, 198, 127);fill:rgb(243, 243, 243);filter:drop-shadow( 1px 2px 2px rgba(185,185,185,1));}#chatgpt-mermaid-\_r_2at\_ text.actor>tspan{fill:rgb(13, 13, 13);stroke:none;}#chatgpt-mermaid-\_r_2at\_ .actor-line{stroke:rgb(143, 143, 143);}#chatgpt-mermaid-\_r_2at\_ .innerArc{stroke-width:1.5;stroke-dasharray:none;}#chatgpt-mermaid-\_r_2at\_ .messageLine0{stroke-width:1.5;stroke-dasharray:none;stroke:rgb(143, 143, 143);}#chatgpt-mermaid-\_r_2at\_ .messageLine1{stroke-width:1.5;stroke-dasharray:2,2;stroke:rgb(143, 143, 143);}#chatgpt-mermaid-\_r_2at\_ [id$="-arrowhead"] path{fill:rgb(143, 143, 143);stroke:rgb(143, 143, 143);}#chatgpt-mermaid-\_r_2at\_ .sequenceNumber{fill:#707070;}#chatgpt-mermaid-\_r_2at\_ [id$="-sequencenumber"]{fill:rgb(143, 143, 143);}#chatgpt-mermaid-\_r_2at\_ [id$="-crosshead"] path{fill:rgb(143, 143, 143);stroke:rgb(143, 143, 143);}#chatgpt-mermaid-\_r_2at\_ .messageText{fill:rgb(13, 13, 13);stroke:none;}#chatgpt-mermaid-\_r_2at\_ .labelBox{stroke:rgba(0, 0, 0, 0.1);fill:rgb(252, 252, 252);filter:none;}#chatgpt-mermaid-\_r_2at\_ .labelText,#chatgpt-mermaid-\_r_2at\_ .labelText>tspan{fill:rgb(13, 13, 13);stroke:none;}#chatgpt-mermaid-\_r_2at\_ .loopText,#chatgpt-mermaid-\_r_2at\_ .loopText>tspan{fill:rgb(13, 13, 13);stroke:none;}#chatgpt-mermaid-\_r_2at\_ .sectionTitle,#chatgpt-mermaid-\_r_2at\_ .sectionTitle>tspan{fill:rgb(13, 13, 13);stroke:none;}#chatgpt-mermaid-\_r_2at\_ .loopLine{stroke-width:2px;stroke-dasharray:2,2;stroke:rgba(0, 0, 0, 0.1);fill:rgba(0, 0, 0, 0.1);}#chatgpt-mermaid-\_r_2at\_ .note{stroke:rgb(107, 198, 127);fill:rgb(243, 243, 243);}#chatgpt-mermaid-\_r_2at\_ .noteText,#chatgpt-mermaid-\_r_2at\_ .noteText>tspan{fill:rgb(13, 13, 13);stroke:none;font-weight:normal;}#chatgpt-mermaid-\_r_2at\_ .activation0{fill:rgb(243, 243, 243);stroke:hsl(0, 0%, 85.2941176471%);}#chatgpt-mermaid-\_r_2at\_ .activation1{fill:rgb(243, 243, 243);stroke:hsl(0, 0%, 85.2941176471%);}#chatgpt-mermaid-\_r_2at\_ .activation2{fill:rgb(243, 243, 243);stroke:hsl(0, 0%, 85.2941176471%);}#chatgpt-mermaid-\_r_2at\_ .actorPopupMenu{position:absolute;}#chatgpt-mermaid-\_r_2at\_ .actorPopupMenuPanel{position:absolute;fill:rgb(222, 234, 251);box-shadow:0px 8px 16px 0px rgba(0,0,0,0.2);filter:drop-shadow(3px 5px 2px rgb(0 0 0 / 0.4));}#chatgpt-mermaid-\_r_2at\_ .actor-man circle,#chatgpt-mermaid-\_r_2at\_ line{fill:rgb(222, 234, 251);stroke-width:2px;}#chatgpt-mermaid-\_r_2at\_ g rect.rect{filter:drop-shadow( 1px 2px 2px rgba(185,185,185,1));stroke:rgb(83, 154, 248);}#chatgpt-mermaid-\_r_2at\_ .node .neo-node{stroke:rgb(83, 154, 248);}#chatgpt-mermaid-\_r_2at\_ [data-look="neo"].node rect,#chatgpt-mermaid-\_r_2at\_ [data-look="neo"].cluster rect,#chatgpt-mermaid-\_r_2at\_ [data-look="neo"].node polygon{stroke:url(#chatgpt-mermaid-\_r_2at\_-gradient);filter:drop-shadow( 1px 2px 2px rgba(185,185,185,1));}#chatgpt-mermaid-\_r_2at\_ [data-look="neo"].swimlane.cluster rect{filter:none;}#chatgpt-mermaid-\_r_2at\_ [data-look="neo"].node path{stroke:url(#chatgpt-mermaid-\_r_2at\_-gradient);stroke-width:1px;}#chatgpt-mermaid-\_r_2at\_ [data-look="neo"].node .outer-path{filter:drop-shadow( 1px 2px 2px rgba(185,185,185,1));}#chatgpt-mermaid-\_r_2at\_ [data-look="neo"].node .neo-line path{stroke:rgb(83, 154, 248);filter:none;}#chatgpt-mermaid-\_r_2at\_ [data-look="neo"].node circle{stroke:url(#chatgpt-mermaid-\_r_2at\_-gradient);filter:drop-shadow( 1px 2px 2px rgba(185,185,185,1));}#chatgpt-mermaid-\_r_2at\_ [data-look="neo"].node circle .state-start{fill:#000000;}#chatgpt-mermaid-\_r_2at\_ [data-look="neo"].icon-shape .icon{fill:url(#chatgpt-mermaid-\_r_2at\_-gradient);filter:drop-shadow( 1px 2px 2px rgba(185,185,185,1));}#chatgpt-mermaid-\_r_2at\_ [data-look="neo"].icon-shape .icon-neo path{stroke:url(#chatgpt-mermaid-\_r_2at\_-gradient);filter:drop-shadow( 1px 2px 2px rgba(185,185,185,1));}#chatgpt-mermaid-\_r_2at\_ :root{--mermaid-font-family:-apple-system-body,ui-sans-serif,-apple-system,system-ui,"Segoe UI",Helvetica,"Apple Color Emoji",Arial,sans-serif,"Segoe UI Emoji","Segoe UI Symbol";}ADD RequestFIRST_REQ(target_engine)唤醒/通知Stats Update更新 engines_running

这里需要特别注意：`FIRST_REQ` 不是普通推理请求，也不是 ADD 请求的替代品，它是协调器层面的运行状态通知。

粘贴的文本 (1)(20261115-152447).txt

## **九、关键流程五：Elastic EP 动态扩缩容**

这是文件中复杂度最高的一块。

它解决的问题是：

**运行期间动态增加或减少 Data Parallel EngineCore 数量，并重新配置分布式 Expert Parallel 环境。**

对应接口：

```
prepare_elastic_ep(new_data_parallel_size)commit_elastic_ep()
```

### **9.1 为什么拆成 Prepare 和 Commit？**

因为动态扩缩容不是简单创建或销毁进程。

还需要处理：

- 新旧 Rank 的通信配置。
- Expert 分布式布局。
- EngineCore 的重新初始化。
- 新 Engine Ready。
- Scheduler 暂停与恢复。
- Coordinator 统计信息更新。

因此拆成两个阶段：

| 阶段      | 作用                          |
| ------- | --------------------------- |
| Prepare | 准备新的 DP 拓扑，暂不将请求路由到新 Engine |
| Commit  | 正式切换拓扑，更新 Engine 列表并恢复调度    |

### **9.2 Scale Up 流程**

\#chatgpt-mermaid-\_r_2bc\_{font-family:-apple-system-body,ui-sans-serif,-apple-system,system-ui,"Segoe UI",Helvetica,"Apple Color Emoji",Arial,sans-serif,"Segoe UI Emoji","Segoe UI Symbol";font-size:16px;fill:rgb(13, 13, 13);}@keyframes edge-animation-frame{from{stroke-dashoffset:0;}}@keyframes dash{to{stroke-dashoffset:0;}}#chatgpt-mermaid-\_r_2bc\_ .edge-animation-slow{stroke-dasharray:9,5!important;stroke-dashoffset:900;animation:dash 50s linear infinite;stroke-linecap:round;}#chatgpt-mermaid-\_r_2bc\_ .edge-animation-fast{stroke-dasharray:9,5!important;stroke-dashoffset:900;animation:dash 20s linear infinite;stroke-linecap:round;}#chatgpt-mermaid-\_r_2bc\_ .error-icon{fill:rgb(243, 243, 243);}#chatgpt-mermaid-\_r_2bc\_ .error-text{fill:rgb(13, 13, 13);stroke:rgb(13, 13, 13);}#chatgpt-mermaid-\_r_2bc\_ .edge-thickness-normal{stroke-width:1px;}#chatgpt-mermaid-\_r_2bc\_ .edge-thickness-thick{stroke-width:3.5px;}#chatgpt-mermaid-\_r_2bc\_ .edge-pattern-solid{stroke-dasharray:0;}#chatgpt-mermaid-\_r_2bc\_ .edge-thickness-invisible{stroke-width:0;fill:none;}#chatgpt-mermaid-\_r_2bc\_ .edge-pattern-dashed{stroke-dasharray:3;}#chatgpt-mermaid-\_r_2bc\_ .edge-pattern-dotted{stroke-dasharray:2;}#chatgpt-mermaid-\_r_2bc\_ .marker{fill:rgb(143, 143, 143);stroke:rgb(143, 143, 143);}#chatgpt-mermaid-\_r_2bc\_ .marker.cross{stroke:rgb(143, 143, 143);}#chatgpt-mermaid-\_r_2bc\_ svg{font-family:-apple-system-body,ui-sans-serif,-apple-system,system-ui,"Segoe UI",Helvetica,"Apple Color Emoji",Arial,sans-serif,"Segoe UI Emoji","Segoe UI Symbol";font-size:16px;}#chatgpt-mermaid-\_r_2bc\_ p{margin:0;}#chatgpt-mermaid-\_r_2bc\_ .label{font-family:-apple-system-body,ui-sans-serif,-apple-system,system-ui,"Segoe UI",Helvetica,"Apple Color Emoji",Arial,sans-serif,"Segoe UI Emoji","Segoe UI Symbol";color:rgb(13, 13, 13);}#chatgpt-mermaid-\_r_2bc\_ .cluster-label text{fill:rgb(13, 13, 13);}#chatgpt-mermaid-\_r_2bc\_ .cluster-label span{color:rgb(13, 13, 13);}#chatgpt-mermaid-\_r_2bc\_ .cluster-label span p{background-color:transparent;}#chatgpt-mermaid-\_r_2bc\_ .label text,#chatgpt-mermaid-\_r_2bc\_ span{fill:rgb(13, 13, 13);color:rgb(13, 13, 13);}#chatgpt-mermaid-\_r_2bc\_ .node rect,#chatgpt-mermaid-\_r_2bc\_ .node circle,#chatgpt-mermaid-\_r_2bc\_ .node ellipse,#chatgpt-mermaid-\_r_2bc\_ .node polygon,#chatgpt-mermaid-\_r_2bc\_ .node path{fill:rgb(222, 234, 251);stroke:rgb(83, 154, 248);stroke-width:1px;}#chatgpt-mermaid-\_r_2bc\_ .rough-node .label text,#chatgpt-mermaid-\_r_2bc\_ .node .label text,#chatgpt-mermaid-\_r_2bc\_ .image-shape .label,#chatgpt-mermaid-\_r_2bc\_ .icon-shape .label{text-anchor:middle;}#chatgpt-mermaid-\_r_2bc\_ .node .katex path{fill:#000;stroke:#000;stroke-width:1px;}#chatgpt-mermaid-\_r_2bc\_ .rough-node .label,#chatgpt-mermaid-\_r_2bc\_ .node .label,#chatgpt-mermaid-\_r_2bc\_ .image-shape .label,#chatgpt-mermaid-\_r_2bc\_ .icon-shape .label{text-align:center;}#chatgpt-mermaid-\_r_2bc\_ .node.clickable{cursor:pointer;}#chatgpt-mermaid-\_r_2bc\_ .root .anchor path{fill:rgb(143, 143, 143)!important;stroke-width:0;stroke:rgb(143, 143, 143);}#chatgpt-mermaid-\_r_2bc\_ .arrowheadPath{fill:rgb(143, 143, 143);}#chatgpt-mermaid-\_r_2bc\_ .edgePath .path{stroke:rgb(143, 143, 143);stroke-width:1px;}#chatgpt-mermaid-\_r_2bc\_ .flowchart-link{stroke:rgb(143, 143, 143);fill:none;}#chatgpt-mermaid-\_r_2bc\_ .edgeLabel{background-color:rgb(252, 252, 252);text-align:center;}#chatgpt-mermaid-\_r_2bc\_ .edgeLabel p{background-color:rgb(252, 252, 252);}#chatgpt-mermaid-\_r_2bc\_ .edgeLabel rect{opacity:0.5;background-color:rgb(252, 252, 252);fill:rgb(252, 252, 252);}#chatgpt-mermaid-\_r_2bc\_ .labelBkg{background-color:rgba(252, 252, 252, 0.5);}#chatgpt-mermaid-\_r_2bc\_ .cluster rect{fill:rgb(243, 243, 243);stroke:rgba(0, 0, 0, 0.1);stroke-width:1px;}#chatgpt-mermaid-\_r_2bc\_ .cluster text{fill:rgb(13, 13, 13);}#chatgpt-mermaid-\_r_2bc\_ .cluster span{color:rgb(13, 13, 13);}#chatgpt-mermaid-\_r_2bc\_ div.mermaidTooltip{position:absolute;text-align:center;max-width:200px;padding:2px;font-family:-apple-system-body,ui-sans-serif,-apple-system,system-ui,"Segoe UI",Helvetica,"Apple Color Emoji",Arial,sans-serif,"Segoe UI Emoji","Segoe UI Symbol";font-size:12px;background:rgb(243, 243, 243);border:1px solid rgba(0, 0, 0, 0.1);border-radius:2px;pointer-events:none;z-index:100;}#chatgpt-mermaid-\_r_2bc\_ .flowchartTitleText{text-anchor:middle;font-size:18px;fill:rgb(13, 13, 13);}#chatgpt-mermaid-\_r_2bc\_ rect.text{fill:none;stroke-width:0;}#chatgpt-mermaid-\_r_2bc\_ .icon-shape,#chatgpt-mermaid-\_r_2bc\_ .image-shape{background-color:rgb(252, 252, 252);text-align:center;}#chatgpt-mermaid-\_r_2bc\_ .icon-shape p,#chatgpt-mermaid-\_r_2bc\_ .image-shape p{background-color:rgb(252, 252, 252);padding:2px;}#chatgpt-mermaid-\_r_2bc\_ .icon-shape .label rect,#chatgpt-mermaid-\_r_2bc\_ .image-shape .label rect{opacity:0.5;background-color:rgb(252, 252, 252);fill:rgb(252, 252, 252);}#chatgpt-mermaid-\_r_2bc\_ .label-icon{display:inline-block;height:1em;overflow:visible;vertical-align:-0.125em;}#chatgpt-mermaid-\_r_2bc\_ .node .label-icon path{fill:currentColor;stroke:revert;stroke-width:revert;}#chatgpt-mermaid-\_r_2bc\_ .node .neo-node{stroke:rgb(83, 154, 248);}#chatgpt-mermaid-\_r_2bc\_ [data-look="neo"].node rect,#chatgpt-mermaid-\_r_2bc\_ [data-look="neo"].cluster rect,#chatgpt-mermaid-\_r_2bc\_ [data-look="neo"].node polygon{stroke:url(#chatgpt-mermaid-\_r_2bc\_-gradient);filter:drop-shadow( 1px 2px 2px rgba(185,185,185,1));}#chatgpt-mermaid-\_r_2bc\_ [data-look="neo"].swimlane.cluster rect{filter:none;}#chatgpt-mermaid-\_r_2bc\_ [data-look="neo"].node path{stroke:url(#chatgpt-mermaid-\_r_2bc\_-gradient);stroke-width:1px;}#chatgpt-mermaid-\_r_2bc\_ [data-look="neo"].node .outer-path{filter:drop-shadow( 1px 2px 2px rgba(185,185,185,1));}#chatgpt-mermaid-\_r_2bc\_ [data-look="neo"].node .neo-line path{stroke:rgb(83, 154, 248);filter:none;}#chatgpt-mermaid-\_r_2bc\_ [data-look="neo"].node circle{stroke:url(#chatgpt-mermaid-\_r_2bc\_-gradient);filter:drop-shadow( 1px 2px 2px rgba(185,185,185,1));}#chatgpt-mermaid-\_r_2bc\_ [data-look="neo"].node circle .state-start{fill:#000000;}#chatgpt-mermaid-\_r_2bc\_ [data-look="neo"].icon-shape .icon{fill:url(#chatgpt-mermaid-\_r_2bc\_-gradient);filter:drop-shadow( 1px 2px 2px rgba(185,185,185,1));}#chatgpt-mermaid-\_r_2bc\_ [data-look="neo"].icon-shape .icon-neo path{stroke:url(#chatgpt-mermaid-\_r_2bc\_-gradient);filter:drop-shadow( 1px 2px 2px rgba(185,185,185,1));}#chatgpt-mermaid-\_r_2bc\_ .node text{font-size:14px;font-weight:600;letter-spacing:normal;fill:rgb(0, 79, 153);}#chatgpt-mermaid-\_r_2bc\_ .edgeLabels text{font-size:13px;font-weight:600;letter-spacing:-0.08px;fill:rgb(0, 79, 153);}#chatgpt-mermaid-\_r_2bc\_ .node tspan[font-weight="normal"],#chatgpt-mermaid-\_r_2bc\_ .edgeLabels tspan[font-weight="normal"]{font-weight:600;}#chatgpt-mermaid-\_r_2bc\_ .edgeLabel .label rect{opacity:1;rx:13px;ry:13px;fill:rgb(245, 250, 255);stroke:rgb(206, 219, 229);stroke-width:1px;}#chatgpt-mermaid-\_r_2bc\_ .node rect,#chatgpt-mermaid-\_r_2bc\_ .node circle,#chatgpt-mermaid-\_r_2bc\_ .node ellipse,#chatgpt-mermaid-\_r_2bc\_ .node polygon,#chatgpt-mermaid-\_r_2bc\_ .node path{fill:rgb(229, 243, 255);stroke:rgba(0, 0, 0, 0.1);stroke-width:1px;}#chatgpt-mermaid-\_r_2bc\_ .node rect{rx:16px;ry:16px;}#chatgpt-mermaid-\_r_2bc\_ .mermaid-decision .label-container{fill:rgb(245, 250, 255);stroke:rgb(206, 219, 229);stroke-dasharray:2,2;}#chatgpt-mermaid-\_r_2bc\_ .edgePaths .flowchart-link{stroke:rgb(143, 143, 143);stroke-width:1px;stroke-linecap:round;stroke-linejoin:round;}#chatgpt-mermaid-\_r_2bc\_ .marker{fill:rgb(143, 143, 143);stroke:rgb(143, 143, 143);}#chatgpt-mermaid-\_r_2bc\_ :root{--mermaid-font-family:-apple-system-body,ui-sans-serif,-apple-system,system-ui,"Segoe UI",Helvetica,"Apple Color Emoji",Arial,sans-serif,"Segoe UI Emoji","Segoe UI Symbol";}prepare_elastic_ep(N)计算新的 Expert 配置创建分布式重配置 Bootstrap通知现有 Engine 重新配置创建新的 EngineCore等待所有新 Engine Readycommit_elastic_ep()pause_scheduler(mode=keep)现有 Engine 提交新配置等待RECONFIGURE_FINISHED等待新增 Engine Ready更新 core_engines 和 DP size通知 Coordinatorresume_scheduler()

关键方法：

```
_prepare_scale_up_elastic_ep()_commit_scale_up_elastic_ep()
```

### **9.3 Scale Down 流程**

\#chatgpt-mermaid-\_r_2bp\_{font-family:-apple-system-body,ui-sans-serif,-apple-system,system-ui,"Segoe UI",Helvetica,"Apple Color Emoji",Arial,sans-serif,"Segoe UI Emoji","Segoe UI Symbol";font-size:16px;fill:rgb(13, 13, 13);}@keyframes edge-animation-frame{from{stroke-dashoffset:0;}}@keyframes dash{to{stroke-dashoffset:0;}}#chatgpt-mermaid-\_r_2bp\_ .edge-animation-slow{stroke-dasharray:9,5!important;stroke-dashoffset:900;animation:dash 50s linear infinite;stroke-linecap:round;}#chatgpt-mermaid-\_r_2bp\_ .edge-animation-fast{stroke-dasharray:9,5!important;stroke-dashoffset:900;animation:dash 20s linear infinite;stroke-linecap:round;}#chatgpt-mermaid-\_r_2bp\_ .error-icon{fill:rgb(243, 243, 243);}#chatgpt-mermaid-\_r_2bp\_ .error-text{fill:rgb(13, 13, 13);stroke:rgb(13, 13, 13);}#chatgpt-mermaid-\_r_2bp\_ .edge-thickness-normal{stroke-width:1px;}#chatgpt-mermaid-\_r_2bp\_ .edge-thickness-thick{stroke-width:3.5px;}#chatgpt-mermaid-\_r_2bp\_ .edge-pattern-solid{stroke-dasharray:0;}#chatgpt-mermaid-\_r_2bp\_ .edge-thickness-invisible{stroke-width:0;fill:none;}#chatgpt-mermaid-\_r_2bp\_ .edge-pattern-dashed{stroke-dasharray:3;}#chatgpt-mermaid-\_r_2bp\_ .edge-pattern-dotted{stroke-dasharray:2;}#chatgpt-mermaid-\_r_2bp\_ .marker{fill:rgb(143, 143, 143);stroke:rgb(143, 143, 143);}#chatgpt-mermaid-\_r_2bp\_ .marker.cross{stroke:rgb(143, 143, 143);}#chatgpt-mermaid-\_r_2bp\_ svg{font-family:-apple-system-body,ui-sans-serif,-apple-system,system-ui,"Segoe UI",Helvetica,"Apple Color Emoji",Arial,sans-serif,"Segoe UI Emoji","Segoe UI Symbol";font-size:16px;}#chatgpt-mermaid-\_r_2bp\_ p{margin:0;}#chatgpt-mermaid-\_r_2bp\_ .label{font-family:-apple-system-body,ui-sans-serif,-apple-system,system-ui,"Segoe UI",Helvetica,"Apple Color Emoji",Arial,sans-serif,"Segoe UI Emoji","Segoe UI Symbol";color:rgb(13, 13, 13);}#chatgpt-mermaid-\_r_2bp\_ .cluster-label text{fill:rgb(13, 13, 13);}#chatgpt-mermaid-\_r_2bp\_ .cluster-label span{color:rgb(13, 13, 13);}#chatgpt-mermaid-\_r_2bp\_ .cluster-label span p{background-color:transparent;}#chatgpt-mermaid-\_r_2bp\_ .label text,#chatgpt-mermaid-\_r_2bp\_ span{fill:rgb(13, 13, 13);color:rgb(13, 13, 13);}#chatgpt-mermaid-\_r_2bp\_ .node rect,#chatgpt-mermaid-\_r_2bp\_ .node circle,#chatgpt-mermaid-\_r_2bp\_ .node ellipse,#chatgpt-mermaid-\_r_2bp\_ .node polygon,#chatgpt-mermaid-\_r_2bp\_ .node path{fill:rgb(222, 234, 251);stroke:rgb(83, 154, 248);stroke-width:1px;}#chatgpt-mermaid-\_r_2bp\_ .rough-node .label text,#chatgpt-mermaid-\_r_2bp\_ .node .label text,#chatgpt-mermaid-\_r_2bp\_ .image-shape .label,#chatgpt-mermaid-\_r_2bp\_ .icon-shape .label{text-anchor:middle;}#chatgpt-mermaid-\_r_2bp\_ .node .katex path{fill:#000;stroke:#000;stroke-width:1px;}#chatgpt-mermaid-\_r_2bp\_ .rough-node .label,#chatgpt-mermaid-\_r_2bp\_ .node .label,#chatgpt-mermaid-\_r_2bp\_ .image-shape .label,#chatgpt-mermaid-\_r_2bp\_ .icon-shape .label{text-align:center;}#chatgpt-mermaid-\_r_2bp\_ .node.clickable{cursor:pointer;}#chatgpt-mermaid-\_r_2bp\_ .root .anchor path{fill:rgb(143, 143, 143)!important;stroke-width:0;stroke:rgb(143, 143, 143);}#chatgpt-mermaid-\_r_2bp\_ .arrowheadPath{fill:rgb(143, 143, 143);}#chatgpt-mermaid-\_r_2bp\_ .edgePath .path{stroke:rgb(143, 143, 143);stroke-width:1px;}#chatgpt-mermaid-\_r_2bp\_ .flowchart-link{stroke:rgb(143, 143, 143);fill:none;}#chatgpt-mermaid-\_r_2bp\_ .edgeLabel{background-color:rgb(252, 252, 252);text-align:center;}#chatgpt-mermaid-\_r_2bp\_ .edgeLabel p{background-color:rgb(252, 252, 252);}#chatgpt-mermaid-\_r_2bp\_ .edgeLabel rect{opacity:0.5;background-color:rgb(252, 252, 252);fill:rgb(252, 252, 252);}#chatgpt-mermaid-\_r_2bp\_ .labelBkg{background-color:rgba(252, 252, 252, 0.5);}#chatgpt-mermaid-\_r_2bp\_ .cluster rect{fill:rgb(243, 243, 243);stroke:rgba(0, 0, 0, 0.1);stroke-width:1px;}#chatgpt-mermaid-\_r_2bp\_ .cluster text{fill:rgb(13, 13, 13);}#chatgpt-mermaid-\_r_2bp\_ .cluster span{color:rgb(13, 13, 13);}#chatgpt-mermaid-\_r_2bp\_ div.mermaidTooltip{position:absolute;text-align:center;max-width:200px;padding:2px;font-family:-apple-system-body,ui-sans-serif,-apple-system,system-ui,"Segoe UI",Helvetica,"Apple Color Emoji",Arial,sans-serif,"Segoe UI Emoji","Segoe UI Symbol";font-size:12px;background:rgb(243, 243, 243);border:1px solid rgba(0, 0, 0, 0.1);border-radius:2px;pointer-events:none;z-index:100;}#chatgpt-mermaid-\_r_2bp\_ .flowchartTitleText{text-anchor:middle;font-size:18px;fill:rgb(13, 13, 13);}#chatgpt-mermaid-\_r_2bp\_ rect.text{fill:none;stroke-width:0;}#chatgpt-mermaid-\_r_2bp\_ .icon-shape,#chatgpt-mermaid-\_r_2bp\_ .image-shape{background-color:rgb(252, 252, 252);text-align:center;}#chatgpt-mermaid-\_r_2bp\_ .icon-shape p,#chatgpt-mermaid-\_r_2bp\_ .image-shape p{background-color:rgb(252, 252, 252);padding:2px;}#chatgpt-mermaid-\_r_2bp\_ .icon-shape .label rect,#chatgpt-mermaid-\_r_2bp\_ .image-shape .label rect{opacity:0.5;background-color:rgb(252, 252, 252);fill:rgb(252, 252, 252);}#chatgpt-mermaid-\_r_2bp\_ .label-icon{display:inline-block;height:1em;overflow:visible;vertical-align:-0.125em;}#chatgpt-mermaid-\_r_2bp\_ .node .label-icon path{fill:currentColor;stroke:revert;stroke-width:revert;}#chatgpt-mermaid-\_r_2bp\_ .node .neo-node{stroke:rgb(83, 154, 248);}#chatgpt-mermaid-\_r_2bp\_ [data-look="neo"].node rect,#chatgpt-mermaid-\_r_2bp\_ [data-look="neo"].cluster rect,#chatgpt-mermaid-\_r_2bp\_ [data-look="neo"].node polygon{stroke:url(#chatgpt-mermaid-\_r_2bp\_-gradient);filter:drop-shadow( 1px 2px 2px rgba(185,185,185,1));}#chatgpt-mermaid-\_r_2bp\_ [data-look="neo"].swimlane.cluster rect{filter:none;}#chatgpt-mermaid-\_r_2bp\_ [data-look="neo"].node path{stroke:url(#chatgpt-mermaid-\_r_2bp\_-gradient);stroke-width:1px;}#chatgpt-mermaid-\_r_2bp\_ [data-look="neo"].node .outer-path{filter:drop-shadow( 1px 2px 2px rgba(185,185,185,1));}#chatgpt-mermaid-\_r_2bp\_ [data-look="neo"].node .neo-line path{stroke:rgb(83, 154, 248);filter:none;}#chatgpt-mermaid-\_r_2bp\_ [data-look="neo"].node circle{stroke:url(#chatgpt-mermaid-\_r_2bp\_-gradient);filter:drop-shadow( 1px 2px 2px rgba(185,185,185,1));}#chatgpt-mermaid-\_r_2bp\_ [data-look="neo"].node circle .state-start{fill:#000000;}#chatgpt-mermaid-\_r_2bp\_ [data-look="neo"].icon-shape .icon{fill:url(#chatgpt-mermaid-\_r_2bp\_-gradient);filter:drop-shadow( 1px 2px 2px rgba(185,185,185,1));}#chatgpt-mermaid-\_r_2bp\_ [data-look="neo"].icon-shape .icon-neo path{stroke:url(#chatgpt-mermaid-\_r_2bp\_-gradient);filter:drop-shadow( 1px 2px 2px rgba(185,185,185,1));}#chatgpt-mermaid-\_r_2bp\_ .node text{font-size:14px;font-weight:600;letter-spacing:normal;fill:rgb(0, 79, 153);}#chatgpt-mermaid-\_r_2bp\_ .edgeLabels text{font-size:13px;font-weight:600;letter-spacing:-0.08px;fill:rgb(0, 79, 153);}#chatgpt-mermaid-\_r_2bp\_ .node tspan[font-weight="normal"],#chatgpt-mermaid-\_r_2bp\_ .edgeLabels tspan[font-weight="normal"]{font-weight:600;}#chatgpt-mermaid-\_r_2bp\_ .edgeLabel .label rect{opacity:1;rx:13px;ry:13px;fill:rgb(245, 250, 255);stroke:rgb(206, 219, 229);stroke-width:1px;}#chatgpt-mermaid-\_r_2bp\_ .node rect,#chatgpt-mermaid-\_r_2bp\_ .node circle,#chatgpt-mermaid-\_r_2bp\_ .node ellipse,#chatgpt-mermaid-\_r_2bp\_ .node polygon,#chatgpt-mermaid-\_r_2bp\_ .node path{fill:rgb(229, 243, 255);stroke:rgba(0, 0, 0, 0.1);stroke-width:1px;}#chatgpt-mermaid-\_r_2bp\_ .node rect{rx:16px;ry:16px;}#chatgpt-mermaid-\_r_2bp\_ .mermaid-decision .label-container{fill:rgb(245, 250, 255);stroke:rgb(206, 219, 229);stroke-dasharray:2,2;}#chatgpt-mermaid-\_r_2bp\_ .edgePaths .flowchart-link{stroke:rgb(143, 143, 143);stroke-width:1px;stroke-linecap:round;stroke-linejoin:round;}#chatgpt-mermaid-\_r_2bp\_ .marker{fill:rgb(143, 143, 143);stroke:rgb(143, 143, 143);}#chatgpt-mermaid-\_r_2bp\_ :root{--mermaid-font-family:-apple-system-body,ui-sans-serif,-apple-system,system-ui,"Segoe UI",Helvetica,"Apple Color Emoji",Arial,sans-serif,"Segoe UI Emoji","Segoe UI Symbol";}prepare_elastic_ep(N)重新配置保留的 Engine等待分布式 Bootstrap 完成commit_elastic_ep()先缩减可路由的 core_engines保留的 Engine: pause keep删除的 Engine: pause abort提交重新配置等待重配置通知Coordinator 更新统计resume_scheduler()

### **9.4 为什么 Scale Down 要先缩减 core_engines？**

代码：

```
old_core_engines = self.core_enginesself.core_engines = old_core_engines[:new_data_parallel_size]self.lb_engines = self.lb_engines[:new_data_parallel_size]
```

注意注释：

```
# Immediately stop sending requests to the removing engines.
```

这是一种重要的安全策略：

**先停止向待删除的 Engine 分配新请求，再进行实际资源回收。**

否则可能发生：

- 新请求被分配到正在销毁的 Rank。
- 请求发送成功但 Engine 已经退出。
- 负载统计仍然包含已经移除的 Engine。

粘贴的文本 (1)(20261115-152447).txt

## **十、进程生命周期、故障监控与资源回收**

这部分属于生产级系统设计。

### **10.1 EngineCore 进程监控**

```
def start_engine_core_monitor(self):    ...    Thread(        target=monitor_engine_cores,        daemon=True,    ).start()
```

监控线程：

```
engine_manager.monitor_engine_liveness()
```

发现 EngineCore 异常退出：

```
_self.resources.engine_dead = True_self.shutdown()
```

之后调用：

```
ensure_alive()
```

就会抛出：

```
EngineDeadError
```

这意味着客户端不会继续把请求发往已经失效的 Engine。

### **10.2 BackgroundResources 为什么存在？**

```
@dataclassclass BackgroundResources:    ctx: zmq.Context    engine_manager: ...    coordinator: ...    output_socket: ...    input_socket: ...    output_queue_task: ...    stats_update_task: ...    engine_dead: bool = False
```

它集中管理后台资源：

- Engine 进程管理器。
- Coordinator。
- ZMQ Socket。
- asyncio Task。
- 关闭通知地址。
- Engine 死亡状态。

配合：

```
self._finalizer = weakref.finalize(    self,    self.resources)
```

即使客户端在构造中途发生异常，也可以清理已经创建的资源。

### **10.3 为什么不用简单的析构函数？**

因为这里存在多种生命周期：

- Python 对象生命周期。
- 线程生命周期。
- asyncio Task 生命周期。
- ZMQ Socket 生命周期。
- EngineCore 子进程生命周期。

它们不一定同时结束。

例如 AsyncMPClient 的 Socket 需要在所属 Event Loop 中关闭，不能随意从其他线程直接操作。

所以代码会判断：

```
if in_loop(loop):    close_sockets_and_tasks()elif not loop.is_closed():    loop.call_soon_threadsafe(        close_sockets_and_tasks    )
```

这体现了一个重要原则：

**资源释放必须遵循资源所属执行环境的生命周期约束。**

粘贴的文本 (1)(20261115-152447).txt

## **十一、关键概念速查表**

把整份文件浓缩成下面这张表，方便复习。

| 概念                  | 是什么                | 为什么需要            |
| ------------------- | ------------------ | ---------------- |
| EngineCoreClient    | 抽象客户端接口            | 统一同步/异步调用        |
| InprocClient        | 同进程客户端             | 避免 IPC 开销        |
| MPClient            | 多进程通信基础类           | 统一通信和生命周期管理      |
| Engine Identity     | Engine 的通信身份       | 确定请求发送给哪个 Engine |
| ROUTER              | ZMQ 路由 Socket      | 向指定 Engine 发送消息  |
| PULL                | ZMQ 接收 Socket      | 接收多个 Engine 的输出  |
| MsgpackEncoder      | 消息编码器              | Python 对象序列化     |
| MsgpackDecoder      | 消息解码器              | 将消息还原为输出对象       |
| EngineCoreOutputs   | Engine 输出结构        | 承载生成结果和控制结果      |
| Utility RPC         | Engine 控制调用        | 执行非推理类操作         |
| call_id             | RPC 调用标识           | 匹配异步返回结果         |
| Future              | 异步结果占位对象           | 等待特定 RPC 完成      |
| asyncio.Queue       | 异步结果队列             | 接收推理输出           |
| BackgroundResources | 后台资源管理对象           | 统一清理线程、Socket、进程 |
| Ready Handshake     | Engine 就绪握手        | 同步初始化配置          |
| DP Rank             | 数据并行编号             | 区分不同 DP Engine   |
| Coordinator         | DP 协调器             | 统计、协调 Engine 状态  |
| Load Balancing      | 负载均衡               | 将请求分配给合适的 Engine |
| First Request       | 首个请求通知             | 唤醒必要的协调执行流程      |
| Elastic EP          | 弹性 Expert Parallel | 动态调整分布式拓扑        |

## **十二、源码阅读路线：真正值得断点的地方**

不建议从第一行一直读到最后一行。建议按下面的顺序阅读。

## **推荐断点清单**

1. **make_client()**

   第一优先级

   搞清楚不同配置下创建哪个 Client，建立整体架构认知。

   约 L90–165
2. **MPClient.init()**

   第一优先级

   搞清楚 ZMQ Socket、进程启动、Ready 握手。

   约 L558–739
3. **AsyncMPClient.add_request_async()**

   第一优先级

   跟踪 ADD 请求如何序列化和发送。

   约 L1219–1222
4. **\_ensure_output_queue_task()**

   第一优先级

   理解结果接收、消息分类和 Future 匹配。

   约 L1090–1164
5. **\_call_utility_async()**

   第二优先级

   理解异步 RPC 请求关联机制。

   约 L1199–1214
6. **get_core_engine_for_request()**

   第二优先级

   理解 DP 请求分配算法。

   约 L1550 附近
7. **prepare_elastic_ep() / commit_elastic_ep()**

   进阶

   理解分布式动态扩缩容。

   约 L1720–1969
8. **BackgroundResources / monitor**

   进阶

   理解生产环境的资源回收和故障处理。

   约 L447–535、L774–801

## **十三、还需要继续追踪哪些源码？**

这份文件只是客户端层。要真正理解 vLLM 的请求生命周期，还需要向下追踪：

| 下一步源码                                   | 要回答的问题                        |
| --------------------------------------- | ----------------------------- |
| `vllm/v1/engine/core.py`                | EngineCoreProc 如何接收请求？        |
| `EngineCore`                            | ADD、ABORT、UTILITY 如何分发？       |
| `EngineCore.step_fn()`                  | Scheduler 如何驱动推理？             |
| `EngineCoreProc.run_engine_core()` 相关逻辑 | 后台进程如何持续运行？                   |
| `vllm/v1/engine/coordinator.py`         | DP Coordinator 如何统计和协调？       |
| `vllm/v1/engine/utils.py`               | CoreEngineProcManager 如何启动进程？ |
| `vllm/v1/engine/output_processor.py`    | Engine 输出如何进入上层请求生命周期？        |

### **最后：这份源码需要建立的五个认知**

1. **客户端与执行引擎分离**：Client 负责通信，EngineCore 负责执行。
2. **请求与控制命令分离**：ADD/ABORT 是请求类消息，UTILITY 是控制类 RPC。
3. **发送与接收分离**：后台接收任务持续消费 ZMQ，前端通过 Queue/Future 获取结果。
4. **通信与调度分离**：DPLBAsyncMPClient 负责选择 Engine，真正的请求调度发生在 EngineCore。
5. **状态与资源分离**：BackgroundResources 管理生命周期，避免客户端销毁后遗留后台资源。

**我的建议：下一步不要继续钻 Elastic EP。先进入&#x20;**`core.py`**，把&#x20;**`EngineCoreProc → EngineCore → Scheduler → Executor`**&#x20;这条执行链路打通。** 这条链路才是理解 vLLM 推理引擎的核心。

可以继续安排下一次源码学习，重点完成 EngineCoreProc 和 EngineCore 的类图、启动流程及请求分发断点分析。

🔬明天继续解析 EngineCore 执行链路轻点设置
