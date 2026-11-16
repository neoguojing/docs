# **vLLM Core Engine Process Manager：架构、类图与关键流程**

这份代码的核心职责是：**管理 EngineCore 的创建、进程调度、启动握手、就绪检测、故障监控和资源回收。**

它位于 vLLM V1 Engine 的进程管理层，主要解决一个问题：

> API Frontend 如何找到并连接多个 EngineCore？如何保证这些 EngineCore 正确启动、分配 GPU、完成初始化，并在故障时及时处理？

代码最重要的入口：

- `launch_core_engines()`：整体启动入口。
- `CoreEngineProcManager`：Python multiprocessing 进程管理。
- `CoreEngineActorManager`：Ray Actor 管理。
- `wait_for_engine_startup()`：启动握手与 READY 屏障。
- `get_engine_zmq_addresses()`：生成 Engine 与 Frontend 的通信地址。

## **1. 整体架构**

这份代码可以分成五个子系统。

**Frontend / API Server**

AsyncLLM / LLMEngine / API Server

请求入口、ZMQ Client、接收 Engine 地址

ZMQ Handshake / HELLO / READY

**Engine Launch & Process Management**

**ProcManager**

multiprocessing

**ActorManager**

Ray Actor

**EngineCore 实例**

**EngineCore DP0**

**EngineCore DP1 ... DPn**

Executor / Worker / GPU

**GPU Worker 集群**

TP / PP / EP 等模型并行执行

**DP Coordinator（可选）**

负载统计、内部 LB、MoE Wave Coordination

注意区分两个层次：

- **DP Rank**：一个独立的 EngineCore 实例，负责一份数据并行模型副本。
- **TP/PP Rank**：一个 EngineCore 内部的 GPU Worker 并行执行单元。

例如：

```
DP Size = 2
TP Size = 4
PP Size = 1

EngineCore DP0
    ├── Worker TP0
    ├── Worker TP1
    ├── Worker TP2
    └── Worker TP3

EngineCore DP1
    ├── Worker TP0
    ├── Worker TP1
    ├── Worker TP2
    └── Worker TP3
```

总共是 2 个 EngineCore，8 个 GPU Worker（假设每个 TP Rank 对应一个 GPU）。

这里的 Process Manager 管理的是 EngineCore 这一层，不直接负责每个 TP Worker 的生命周期。

## **2. 核心类图**

这份代码的类关系可以归纳为三组：进程管理、Ray 管理、启动握手。

\#chatgpt-mermaid-\_r_f2\_{font-family:-apple-system-body,ui-sans-serif,-apple-system,system-ui,"Segoe UI",Helvetica,"Apple Color Emoji",Arial,sans-serif,"Segoe UI Emoji","Segoe UI Symbol";font-size:16px;fill:rgb(13, 13, 13);}@keyframes edge-animation-frame{from{stroke-dashoffset:0;}}@keyframes dash{to{stroke-dashoffset:0;}}#chatgpt-mermaid-\_r_f2\_ .edge-animation-slow{stroke-dasharray:9,5!important;stroke-dashoffset:900;animation:dash 50s linear infinite;stroke-linecap:round;}#chatgpt-mermaid-\_r_f2\_ .edge-animation-fast{stroke-dasharray:9,5!important;stroke-dashoffset:900;animation:dash 20s linear infinite;stroke-linecap:round;}#chatgpt-mermaid-\_r_f2\_ .error-icon{fill:rgb(243, 243, 243);}#chatgpt-mermaid-\_r_f2\_ .error-text{fill:rgb(13, 13, 13);stroke:rgb(13, 13, 13);}#chatgpt-mermaid-\_r_f2\_ .edge-thickness-normal{stroke-width:1px;}#chatgpt-mermaid-\_r_f2\_ .edge-thickness-thick{stroke-width:3.5px;}#chatgpt-mermaid-\_r_f2\_ .edge-pattern-solid{stroke-dasharray:0;}#chatgpt-mermaid-\_r_f2\_ .edge-thickness-invisible{stroke-width:0;fill:none;}#chatgpt-mermaid-\_r_f2\_ .edge-pattern-dashed{stroke-dasharray:3;}#chatgpt-mermaid-\_r_f2\_ .edge-pattern-dotted{stroke-dasharray:2;}#chatgpt-mermaid-\_r_f2\_ .marker{fill:rgb(143, 143, 143);stroke:rgb(143, 143, 143);}#chatgpt-mermaid-\_r_f2\_ .marker.cross{stroke:rgb(143, 143, 143);}#chatgpt-mermaid-\_r_f2\_ svg{font-family:-apple-system-body,ui-sans-serif,-apple-system,system-ui,"Segoe UI",Helvetica,"Apple Color Emoji",Arial,sans-serif,"Segoe UI Emoji","Segoe UI Symbol";font-size:16px;}#chatgpt-mermaid-\_r_f2\_ p{margin:0;}#chatgpt-mermaid-\_r_f2\_ g.classGroup text{fill:rgb(83, 154, 248);stroke:none;font-family:-apple-system-body,ui-sans-serif,-apple-system,system-ui,"Segoe UI",Helvetica,"Apple Color Emoji",Arial,sans-serif,"Segoe UI Emoji","Segoe UI Symbol";font-size:10px;}#chatgpt-mermaid-\_r_f2\_ g.classGroup text .title{font-weight:bolder;}#chatgpt-mermaid-\_r_f2\_ .cluster-label text{fill:rgb(13, 13, 13);}#chatgpt-mermaid-\_r_f2\_ .cluster-label span{color:rgb(13, 13, 13);}#chatgpt-mermaid-\_r_f2\_ .cluster-label span p{background-color:transparent;}#chatgpt-mermaid-\_r_f2\_ .cluster rect{fill:rgb(243, 243, 243);stroke:rgba(0, 0, 0, 0.1);stroke-width:1px;}#chatgpt-mermaid-\_r_f2\_ .cluster text{fill:rgb(13, 13, 13);}#chatgpt-mermaid-\_r_f2\_ .cluster span{color:rgb(13, 13, 13);}#chatgpt-mermaid-\_r_f2\_ .nodeLabel,#chatgpt-mermaid-\_r_f2\_ .edgeLabel{color:rgb(13, 13, 13);}#chatgpt-mermaid-\_r_f2\_ .noteLabel .nodeLabel,#chatgpt-mermaid-\_r_f2\_ .noteLabel .edgeLabel{color:rgb(13, 13, 13);}#chatgpt-mermaid-\_r_f2\_ .edgeLabel .label rect{fill:rgb(222, 234, 251);}#chatgpt-mermaid-\_r_f2\_ .label text{fill:rgb(13, 13, 13);}#chatgpt-mermaid-\_r_f2\_ .labelBkg{background:rgb(222, 234, 251);}#chatgpt-mermaid-\_r_f2\_ .edgeLabel .label span{background:rgb(222, 234, 251);}#chatgpt-mermaid-\_r_f2\_ .classTitle{font-weight:bolder;}#chatgpt-mermaid-\_r_f2\_ .node rect,#chatgpt-mermaid-\_r_f2\_ .node circle,#chatgpt-mermaid-\_r_f2\_ .node ellipse,#chatgpt-mermaid-\_r_f2\_ .node polygon,#chatgpt-mermaid-\_r_f2\_ .node path{fill:rgb(222, 234, 251);stroke:rgb(83, 154, 248);stroke-width:1;}#chatgpt-mermaid-\_r_f2\_ .divider{stroke:rgb(83, 154, 248);stroke-width:1;}#chatgpt-mermaid-\_r_f2\_ g.clickable{cursor:pointer;}#chatgpt-mermaid-\_r_f2\_ g.classGroup rect{fill:rgb(222, 234, 251);stroke:rgb(83, 154, 248);}#chatgpt-mermaid-\_r_f2\_ g.classGroup line{stroke:rgb(83, 154, 248);stroke-width:1;}#chatgpt-mermaid-\_r_f2\_ .classLabel .box{stroke:none;stroke-width:0;fill:rgb(222, 234, 251);opacity:0.5;}#chatgpt-mermaid-\_r_f2\_ .classLabel .label{fill:rgb(83, 154, 248);font-size:10px;}#chatgpt-mermaid-\_r_f2\_ .relation{stroke:rgb(143, 143, 143);stroke-width:1;fill:none;}#chatgpt-mermaid-\_r_f2\_ .dashed-line{stroke-dasharray:3;}#chatgpt-mermaid-\_r_f2\_ .dotted-line{stroke-dasharray:1 2;}#chatgpt-mermaid-\_r_f2\_ [id$="-compositionStart"],#chatgpt-mermaid-\_r_f2\_ .composition{fill:rgb(143, 143, 143)!important;stroke:rgb(143, 143, 143)!important;stroke-width:1;}#chatgpt-mermaid-\_r_f2\_ [id$="-compositionEnd"],#chatgpt-mermaid-\_r_f2\_ .composition{fill:rgb(143, 143, 143)!important;stroke:rgb(143, 143, 143)!important;stroke-width:1;}#chatgpt-mermaid-\_r_f2\_ [id$="-dependencyStart"],#chatgpt-mermaid-\_r_f2\_ .dependency{fill:rgb(143, 143, 143)!important;stroke:rgb(143, 143, 143)!important;stroke-width:1;}#chatgpt-mermaid-\_r_f2\_ [id$="-dependencyEnd"],#chatgpt-mermaid-\_r_f2\_ .dependency{fill:rgb(143, 143, 143)!important;stroke:rgb(143, 143, 143)!important;stroke-width:1;}#chatgpt-mermaid-\_r_f2\_ [id$="-extensionStart"],#chatgpt-mermaid-\_r_f2\_ .extension{fill:transparent!important;stroke:rgb(143, 143, 143)!important;stroke-width:1;}#chatgpt-mermaid-\_r_f2\_ [id$="-extensionEnd"],#chatgpt-mermaid-\_r_f2\_ .extension{fill:transparent!important;stroke:rgb(143, 143, 143)!important;stroke-width:1;}#chatgpt-mermaid-\_r_f2\_ [id$="-aggregationStart"],#chatgpt-mermaid-\_r_f2\_ .aggregation{fill:transparent!important;stroke:rgb(143, 143, 143)!important;stroke-width:1;}#chatgpt-mermaid-\_r_f2\_ [id$="-aggregationEnd"],#chatgpt-mermaid-\_r_f2\_ .aggregation{fill:transparent!important;stroke:rgb(143, 143, 143)!important;stroke-width:1;}#chatgpt-mermaid-\_r_f2\_ [id$="-lollipopStart"],#chatgpt-mermaid-\_r_f2\_ .lollipop{fill:rgb(222, 234, 251)!important;stroke:rgb(143, 143, 143)!important;stroke-width:1;}#chatgpt-mermaid-\_r_f2\_ [id$="-lollipopEnd"],#chatgpt-mermaid-\_r_f2\_ .lollipop{fill:rgb(222, 234, 251)!important;stroke:rgb(143, 143, 143)!important;stroke-width:1;}#chatgpt-mermaid-\_r_f2\_ .edgeTerminals{font-size:11px;line-height:initial;}#chatgpt-mermaid-\_r_f2\_ .classTitleText{text-anchor:middle;font-size:18px;fill:rgb(13, 13, 13);}#chatgpt-mermaid-\_r_f2\_ .edgeLabel[data-look="neo"]{background-color:rgb(252, 252, 252);text-align:center;}#chatgpt-mermaid-\_r_f2\_ .edgeLabel[data-look="neo"] p{background-color:rgb(252, 252, 252);}#chatgpt-mermaid-\_r_f2\_ .edgeLabel[data-look="neo"] rect{opacity:0.5;background-color:rgb(252, 252, 252);fill:rgb(252, 252, 252);}#chatgpt-mermaid-\_r_f2\_ .label-icon{display:inline-block;height:1em;overflow:visible;vertical-align:-0.125em;}#chatgpt-mermaid-\_r_f2\_ .node .label-icon path{fill:currentColor;stroke:revert;stroke-width:revert;}#chatgpt-mermaid-\_r_f2\_ .node .neo-node{stroke:rgb(83, 154, 248);}#chatgpt-mermaid-\_r_f2\_ [data-look="neo"].node rect,#chatgpt-mermaid-\_r_f2\_ [data-look="neo"].cluster rect,#chatgpt-mermaid-\_r_f2\_ [data-look="neo"].node polygon{stroke:url(#chatgpt-mermaid-\_r_f2\_-gradient);filter:drop-shadow( 1px 2px 2px rgba(185,185,185,1));}#chatgpt-mermaid-\_r_f2\_ [data-look="neo"].swimlane.cluster rect{filter:none;}#chatgpt-mermaid-\_r_f2\_ [data-look="neo"].node path{stroke:url(#chatgpt-mermaid-\_r_f2\_-gradient);stroke-width:1px;}#chatgpt-mermaid-\_r_f2\_ [data-look="neo"].node .outer-path{filter:drop-shadow( 1px 2px 2px rgba(185,185,185,1));}#chatgpt-mermaid-\_r_f2\_ [data-look="neo"].node .neo-line path{stroke:rgb(83, 154, 248);filter:none;}#chatgpt-mermaid-\_r_f2\_ [data-look="neo"].node circle{stroke:url(#chatgpt-mermaid-\_r_f2\_-gradient);filter:drop-shadow( 1px 2px 2px rgba(185,185,185,1));}#chatgpt-mermaid-\_r_f2\_ [data-look="neo"].node circle .state-start{fill:#000000;}#chatgpt-mermaid-\_r_f2\_ [data-look="neo"].icon-shape .icon{fill:url(#chatgpt-mermaid-\_r_f2\_-gradient);filter:drop-shadow( 1px 2px 2px rgba(185,185,185,1));}#chatgpt-mermaid-\_r_f2\_ [data-look="neo"].icon-shape .icon-neo path{stroke:url(#chatgpt-mermaid-\_r_f2\_-gradient);filter:drop-shadow( 1px 2px 2px rgba(185,185,185,1));}#chatgpt-mermaid-\_r_f2\_ :root{--mermaid-font-family:-apple-system-body,ui-sans-serif,-apple-system,system-ui,"Segoe UI",Helvetica,"Apple Color Emoji",Arial,sans-serif,"Segoe UI Emoji","Segoe UI Symbol";}CoreEngine+bool local+bytes identity+CoreEngineState state«enumeration»CoreEngineStateNEWCONNECTEDREADYCoreEngineProcManager+list processes+Event manager_stopped+str failed_proc_name+shutdown()+monitor_engine_liveness()+sentinels()+finished_procs()CoreEngineActorManager+list local_engine_actors+list remote_engine_actors+list run_refs+shutdown()+monitor_engine_liveness()+create_dp_placement_groups()+scale_up_elastic_ep()+scale_down_elastic_ep()EngineZmqAddresses+list inputs+list outputs+str coordinator_input+str coordinator_output+str frontend_stats_publish_addressEngineHandshakeMetadata+EngineZmqAddresses addresses+dict parallel_configCoreEngineLaunch+engine_manager+coordinator+addresses+tensor_queue+watched_frontend_processesDPCoordinator+get_engine_socket_addresses()+get_stats_publish_address()EngineCoreProc+run_engine_core()multiprocessingstartup trackingstartup tracking

### **2.1 核心类职责**

| 类                         | 职责                   | 关键属性/方法                       |
| ------------------------- | -------------------- | ----------------------------- |
| `CoreEngine`              | 跟踪每个 DP Engine 的握手状态 | `identity`、`state`            |
| `CoreEngineState`         | 握手状态枚举               | NEW / CONNECTED / READY       |
| `CoreEngineProcManager`   | 管理本地 Python 子进程      | `Process`、`shutdown()`        |
| `CoreEngineActorManager`  | 管理 Ray Engine Actor  | `ActorHandle`、PlacementGroup  |
| `EngineZmqAddresses`      | 保存通信地址               | inputs / outputs              |
| `EngineHandshakeMetadata` | 向 Engine 下发初始化信息     | addresses / parallel_config   |
| `CoreEngineLaunch`        | 启动阶段资源上下文            | manager / coordinator / queue |
| `SignalCallback`          | 将信号处理回调转交独立线程        | Event / Thread                |

其中最值得重点研究的是三个类：

**① CoreEngineProcManager**

它解决本地进程生命周期问题：

- 创建多少个 EngineCore 进程？
- 每个进程对应哪个 DP Rank？
- 如何设置 GPU 映射？
- 进程是否异常退出？
- 如何统一关闭？

**② CoreEngineActorManager**

它解决分布式资源调度问题：

- Ray 集群中如何分配 GPU？
- 一个 DP Rank 如何绑定多个 GPU？
- 如何创建 Placement Group？
- 如何跨节点创建 Engine Actor？
- 如何动态扩缩容？

**③ CoreEngine**

它解决启动状态跟踪问题。

```
class CoreEngine:    def __init__(self, index=0, local=True):        self.local = local        self.identity = index.to_bytes(2, "little")        self.state = CoreEngineState.NEW
```

这里的 `identity` 是 ZMQ ROUTER/DEALER 通信中的身份标识。

为什么需要它？

因为多个 EngineCore 都会连接同一个 ROUTER Socket，Frontend 必须知道：

- 哪个 Engine 发来的消息？
- 对应哪个 DP Rank？
- 哪个 Engine 已经完成初始化？

`identity` 就是消息路由和状态关联的依据。

## **3. 关键概念剖析**

### **3.1 Process Manager 和 Actor Manager 的区别**

这是代码最重要的架构分支。

| 对比       | ProcManager               | ActorManager    |
| -------- | ------------------------- | --------------- |
| 执行环境     | 单机/本地多进程                  | Ray 集群          |
| 进程创建     | `multiprocessing.Process` | `ray.remote()`  |
| 实例标识     | Process name              | ActorHandle     |
| GPU 资源管理 | 本地 GPU ID 映射              | Placement Group |
| 运行监控     | Process Sentinel          | Ray ObjectRef   |
| 退出方式     | shutdown / terminate      | ray.kill        |
| 跨节点      | 不负责跨节点调度                  | 支持              |

两者并不是同时工作的两个管理器，而是根据配置选择其一：

```
if parallel_config.data_parallel_backend == "ray":    engine_actor_manager = CoreEngineActorManager(...)else:    local_engine_manager = CoreEngineProcManager(...)
```

### **3.2 Rank 的概念**

这份代码主要涉及三种 Rank。

| Rank            | 含义             | 作用                      |
| --------------- | -------------- | ----------------------- |
| `dp_rank`       | 全局 DP Rank     | 唯一标识一个 EngineCore       |
| `local_dp_rank` | 本机 DP Rank     | 确定本机第几个 DP Engine       |
| `tp_rank`       | TP Worker Rank | 一个 EngineCore 内部的模型并行编号 |

例如：

```
两台机器，每台 8 张 GPU

机器 A：
  DP0 -> TP0~TP7
  DP1 -> TP0~TP7

机器 B：
  DP2 -> TP0~TP7
  DP3 -> TP0~TP7
```

此时：

- `dp_rank`：0、1、2、3。
- `local_dp_rank`：机器 A 是 0、1；机器 B 也是 0、1。
- `tp_rank`：每个 DP Engine 内部都是 0\~7。

**注意：DP Rank 标识的是模型副本，不是 GPU Worker。**

### **3.3 Placement Group 是什么？**

这是 Ray 后端的重要概念。

Ray Placement Group 可以把一组资源作为一个整体申请，避免一个 DP Engine 所需的 GPU 被零散分配到不合适的节点。

例如：

```
bundles = [    {"GPU": 1.0},    {"GPU": 1.0},    {"GPU": 1.0},    {"GPU": 1.0},    {"CPU": 1.0},]pg = ray.util.placement_group(    bundles=bundles,    strategy="STRICT_PACK",)
```

含义：

- 前四个 Bundle：为 TP=4 的 Worker 预留 GPU。
- 最后一个 Bundle：为 EngineCore Actor 预留 CPU。
- `STRICT_PACK`：要求整个 Placement Group 放在同一节点。

代码通过：

```
PlacementGroupSchedulingStrategy(    placement_group=pg,    placement_group_bundle_index=world_size,)
```

把 EngineCore Actor 放在最后一个 CPU Bundle 上。

这里 `world_size` 对应最后一个 Bundle 的索引，因为索引从 0 开始，前面恰好有 `world_size` 个 GPU Bundle。

### **3.4 ZMQ Handshake 是什么？**

Handshake 是 EngineCore 启动时与管理端进行的通信协议。

它不是推理请求通信，而是**进程发现、初始化参数交换和启动状态确认**。

核心消息：

| 消息    | 发送方          | 含义               |
| ----- | ------------ | ---------------- |
| HELLO | EngineCore   | 我已经启动，可以建立连接     |
| 初始化信息 | Frontend/管理端 | 下发 ZMQ 地址和 DP 配置 |
| READY | EngineCore   | 初始化完成，可以投入运行     |

## **4. 关键流程一：EngineCore 启动流程**

核心入口是：

```
@contextlib.contextmanagerdef launch_core_engines(    vllm_config,    executor_class,    log_stats,    addresses,):
```

整体调用关系：

\#chatgpt-mermaid-\_r_gb\_{font-family:-apple-system-body,ui-sans-serif,-apple-system,system-ui,"Segoe UI",Helvetica,"Apple Color Emoji",Arial,sans-serif,"Segoe UI Emoji","Segoe UI Symbol";font-size:16px;fill:rgb(13, 13, 13);}@keyframes edge-animation-frame{from{stroke-dashoffset:0;}}@keyframes dash{to{stroke-dashoffset:0;}}#chatgpt-mermaid-\_r_gb\_ .edge-animation-slow{stroke-dasharray:9,5!important;stroke-dashoffset:900;animation:dash 50s linear infinite;stroke-linecap:round;}#chatgpt-mermaid-\_r_gb\_ .edge-animation-fast{stroke-dasharray:9,5!important;stroke-dashoffset:900;animation:dash 20s linear infinite;stroke-linecap:round;}#chatgpt-mermaid-\_r_gb\_ .error-icon{fill:rgb(243, 243, 243);}#chatgpt-mermaid-\_r_gb\_ .error-text{fill:rgb(13, 13, 13);stroke:rgb(13, 13, 13);}#chatgpt-mermaid-\_r_gb\_ .edge-thickness-normal{stroke-width:1px;}#chatgpt-mermaid-\_r_gb\_ .edge-thickness-thick{stroke-width:3.5px;}#chatgpt-mermaid-\_r_gb\_ .edge-pattern-solid{stroke-dasharray:0;}#chatgpt-mermaid-\_r_gb\_ .edge-thickness-invisible{stroke-width:0;fill:none;}#chatgpt-mermaid-\_r_gb\_ .edge-pattern-dashed{stroke-dasharray:3;}#chatgpt-mermaid-\_r_gb\_ .edge-pattern-dotted{stroke-dasharray:2;}#chatgpt-mermaid-\_r_gb\_ .marker{fill:rgb(143, 143, 143);stroke:rgb(143, 143, 143);}#chatgpt-mermaid-\_r_gb\_ .marker.cross{stroke:rgb(143, 143, 143);}#chatgpt-mermaid-\_r_gb\_ svg{font-family:-apple-system-body,ui-sans-serif,-apple-system,system-ui,"Segoe UI",Helvetica,"Apple Color Emoji",Arial,sans-serif,"Segoe UI Emoji","Segoe UI Symbol";font-size:16px;}#chatgpt-mermaid-\_r_gb\_ p{margin:0;}#chatgpt-mermaid-\_r_gb\_ .label{font-family:-apple-system-body,ui-sans-serif,-apple-system,system-ui,"Segoe UI",Helvetica,"Apple Color Emoji",Arial,sans-serif,"Segoe UI Emoji","Segoe UI Symbol";color:rgb(13, 13, 13);}#chatgpt-mermaid-\_r_gb\_ .cluster-label text{fill:rgb(13, 13, 13);}#chatgpt-mermaid-\_r_gb\_ .cluster-label span{color:rgb(13, 13, 13);}#chatgpt-mermaid-\_r_gb\_ .cluster-label span p{background-color:transparent;}#chatgpt-mermaid-\_r_gb\_ .label text,#chatgpt-mermaid-\_r_gb\_ span{fill:rgb(13, 13, 13);color:rgb(13, 13, 13);}#chatgpt-mermaid-\_r_gb\_ .node rect,#chatgpt-mermaid-\_r_gb\_ .node circle,#chatgpt-mermaid-\_r_gb\_ .node ellipse,#chatgpt-mermaid-\_r_gb\_ .node polygon,#chatgpt-mermaid-\_r_gb\_ .node path{fill:rgb(222, 234, 251);stroke:rgb(83, 154, 248);stroke-width:1px;}#chatgpt-mermaid-\_r_gb\_ .rough-node .label text,#chatgpt-mermaid-\_r_gb\_ .node .label text,#chatgpt-mermaid-\_r_gb\_ .image-shape .label,#chatgpt-mermaid-\_r_gb\_ .icon-shape .label{text-anchor:middle;}#chatgpt-mermaid-\_r_gb\_ .node .katex path{fill:#000;stroke:#000;stroke-width:1px;}#chatgpt-mermaid-\_r_gb\_ .rough-node .label,#chatgpt-mermaid-\_r_gb\_ .node .label,#chatgpt-mermaid-\_r_gb\_ .image-shape .label,#chatgpt-mermaid-\_r_gb\_ .icon-shape .label{text-align:center;}#chatgpt-mermaid-\_r_gb\_ .node.clickable{cursor:pointer;}#chatgpt-mermaid-\_r_gb\_ .root .anchor path{fill:rgb(143, 143, 143)!important;stroke-width:0;stroke:rgb(143, 143, 143);}#chatgpt-mermaid-\_r_gb\_ .arrowheadPath{fill:rgb(143, 143, 143);}#chatgpt-mermaid-\_r_gb\_ .edgePath .path{stroke:rgb(143, 143, 143);stroke-width:1px;}#chatgpt-mermaid-\_r_gb\_ .flowchart-link{stroke:rgb(143, 143, 143);fill:none;}#chatgpt-mermaid-\_r_gb\_ .edgeLabel{background-color:rgb(252, 252, 252);text-align:center;}#chatgpt-mermaid-\_r_gb\_ .edgeLabel p{background-color:rgb(252, 252, 252);}#chatgpt-mermaid-\_r_gb\_ .edgeLabel rect{opacity:0.5;background-color:rgb(252, 252, 252);fill:rgb(252, 252, 252);}#chatgpt-mermaid-\_r_gb\_ .labelBkg{background-color:rgba(252, 252, 252, 0.5);}#chatgpt-mermaid-\_r_gb\_ .cluster rect{fill:rgb(243, 243, 243);stroke:rgba(0, 0, 0, 0.1);stroke-width:1px;}#chatgpt-mermaid-\_r_gb\_ .cluster text{fill:rgb(13, 13, 13);}#chatgpt-mermaid-\_r_gb\_ .cluster span{color:rgb(13, 13, 13);}#chatgpt-mermaid-\_r_gb\_ div.mermaidTooltip{position:absolute;text-align:center;max-width:200px;padding:2px;font-family:-apple-system-body,ui-sans-serif,-apple-system,system-ui,"Segoe UI",Helvetica,"Apple Color Emoji",Arial,sans-serif,"Segoe UI Emoji","Segoe UI Symbol";font-size:12px;background:rgb(243, 243, 243);border:1px solid rgba(0, 0, 0, 0.1);border-radius:2px;pointer-events:none;z-index:100;}#chatgpt-mermaid-\_r_gb\_ .flowchartTitleText{text-anchor:middle;font-size:18px;fill:rgb(13, 13, 13);}#chatgpt-mermaid-\_r_gb\_ rect.text{fill:none;stroke-width:0;}#chatgpt-mermaid-\_r_gb\_ .icon-shape,#chatgpt-mermaid-\_r_gb\_ .image-shape{background-color:rgb(252, 252, 252);text-align:center;}#chatgpt-mermaid-\_r_gb\_ .icon-shape p,#chatgpt-mermaid-\_r_gb\_ .image-shape p{background-color:rgb(252, 252, 252);padding:2px;}#chatgpt-mermaid-\_r_gb\_ .icon-shape .label rect,#chatgpt-mermaid-\_r_gb\_ .image-shape .label rect{opacity:0.5;background-color:rgb(252, 252, 252);fill:rgb(252, 252, 252);}#chatgpt-mermaid-\_r_gb\_ .label-icon{display:inline-block;height:1em;overflow:visible;vertical-align:-0.125em;}#chatgpt-mermaid-\_r_gb\_ .node .label-icon path{fill:currentColor;stroke:revert;stroke-width:revert;}#chatgpt-mermaid-\_r_gb\_ .node .neo-node{stroke:rgb(83, 154, 248);}#chatgpt-mermaid-\_r_gb\_ [data-look="neo"].node rect,#chatgpt-mermaid-\_r_gb\_ [data-look="neo"].cluster rect,#chatgpt-mermaid-\_r_gb\_ [data-look="neo"].node polygon{stroke:url(#chatgpt-mermaid-\_r_gb\_-gradient);filter:drop-shadow( 1px 2px 2px rgba(185,185,185,1));}#chatgpt-mermaid-\_r_gb\_ [data-look="neo"].swimlane.cluster rect{filter:none;}#chatgpt-mermaid-\_r_gb\_ [data-look="neo"].node path{stroke:url(#chatgpt-mermaid-\_r_gb\_-gradient);stroke-width:1px;}#chatgpt-mermaid-\_r_gb\_ [data-look="neo"].node .outer-path{filter:drop-shadow( 1px 2px 2px rgba(185,185,185,1));}#chatgpt-mermaid-\_r_gb\_ [data-look="neo"].node .neo-line path{stroke:rgb(83, 154, 248);filter:none;}#chatgpt-mermaid-\_r_gb\_ [data-look="neo"].node circle{stroke:url(#chatgpt-mermaid-\_r_gb\_-gradient);filter:drop-shadow( 1px 2px 2px rgba(185,185,185,1));}#chatgpt-mermaid-\_r_gb\_ [data-look="neo"].node circle .state-start{fill:#000000;}#chatgpt-mermaid-\_r_gb\_ [data-look="neo"].icon-shape .icon{fill:url(#chatgpt-mermaid-\_r_gb\_-gradient);filter:drop-shadow( 1px 2px 2px rgba(185,185,185,1));}#chatgpt-mermaid-\_r_gb\_ [data-look="neo"].icon-shape .icon-neo path{stroke:url(#chatgpt-mermaid-\_r_gb\_-gradient);filter:drop-shadow( 1px 2px 2px rgba(185,185,185,1));}#chatgpt-mermaid-\_r_gb\_ .node text{font-size:14px;font-weight:600;letter-spacing:normal;fill:rgb(0, 79, 153);}#chatgpt-mermaid-\_r_gb\_ .edgeLabels text{font-size:13px;font-weight:600;letter-spacing:-0.08px;fill:rgb(0, 79, 153);}#chatgpt-mermaid-\_r_gb\_ .node tspan[font-weight="normal"],#chatgpt-mermaid-\_r_gb\_ .edgeLabels tspan[font-weight="normal"]{font-weight:600;}#chatgpt-mermaid-\_r_gb\_ .edgeLabel .label rect{opacity:1;rx:13px;ry:13px;fill:rgb(245, 250, 255);stroke:rgb(206, 219, 229);stroke-width:1px;}#chatgpt-mermaid-\_r_gb\_ .node rect,#chatgpt-mermaid-\_r_gb\_ .node circle,#chatgpt-mermaid-\_r_gb\_ .node ellipse,#chatgpt-mermaid-\_r_gb\_ .node polygon,#chatgpt-mermaid-\_r_gb\_ .node path{fill:rgb(229, 243, 255);stroke:rgba(0, 0, 0, 0.1);stroke-width:1px;}#chatgpt-mermaid-\_r_gb\_ .node rect{rx:16px;ry:16px;}#chatgpt-mermaid-\_r_gb\_ .mermaid-decision .label-container{fill:rgb(245, 250, 255);stroke:rgb(206, 219, 229);stroke-dasharray:2,2;}#chatgpt-mermaid-\_r_gb\_ .edgePaths .flowchart-link{stroke:rgb(143, 143, 143);stroke-width:1px;stroke-linecap:round;stroke-linejoin:round;}#chatgpt-mermaid-\_r_gb\_ .marker{fill:rgb(143, 143, 143);stroke:rgb(143, 143, 143);}#chatgpt-mermaid-\_r_gb\_ :root{--mermaid-font-family:-apple-system-body,ui-sans-serif,-apple-system,system-ui,"Segoe UI",Helvetica,"Apple Color Emoji",Arial,sans-serif,"Segoe UI Emoji","Segoe UI Symbol";}launch_core_engines()读取 ParallelConfig判断 Offline / Online创建 tensor_queue（可选）需要 DP Coordinator?启动 DPCoordinator跳过 CoordinatorDP BackendCoreEngineActorManager创建 ZMQ ROUTERCoreEngineProcManager启动 EngineCore 子进程返回 CoreEngineLaunchFrontend 启动及连接wait_for_engine_startup全部 Engine READY是否RayMP

### **第一步：准备配置和通信资源**

主要变量：

```
dp_size = parallel_config.data_parallel_sizelocal_engine_count = parallel_config.data_parallel_size_localdp_rank = parallel_config.data_parallel_rank
```

它们分别表示：

- `dp_size`：全局 DP Engine 数量。
- `local_engine_count`：当前节点负责启动的 Engine 数量。
- `dp_rank`：当前启动进程所处的 DP Rank 起始位置。

随后可能创建多模态 Tensor IPC Queue：

```
if multimodal_config is not None \    and multimodal_config.mm_tensor_ipc == "torch_shm":    tensor_queue = get_mp_context().Queue()
```

它用于 API Server 与 EngineCore 之间共享多模态 Tensor。

### **第二步：启动 DP Coordinator**

```
run_coordinator = (    vllm_config.needs_dp_coordinator    and not offline_mode    and dp_rank == 0)
```

只有满足条件才启动。

Coordinator 的主要职责：

1. 收集和发布 Engine 队列负载统计。
2. 支持 Internal / Hybrid Load Balancing。
3. 对 MoE 模型进行 Wave Coordination。

需要注意：Coordinator 不是每个 DP Rank 都创建一个，而是在这里由在线模式的 Rank 0 启动。

### **第三步：选择 Engine 管理器**

- Ray Backend：创建 `CoreEngineActorManager`。
- 非 Ray Backend：创建 `CoreEngineProcManager`。

非 Ray 路径会先创建一个 ZMQ ROUTER Socket，用于等待 EngineCore 的 HELLO 和 READY 消息，然后启动本地进程。

### **第四步：启动 EngineCore**

```
context.Process(    target=EngineCoreProc.run_engine_core,    name=f"EngineCore_DP{global_index}",    kwargs={        "dp_rank": global_index,        "local_dp_rank": local_index,        ...    },)
```

注意，这里并没有直接执行 EngineCore 的推理逻辑，而是创建后台进程，并让它执行：

```
EngineCoreProc.run_engine_core()
```

真正的 EngineCore 初始化和 Worker 创建需要继续阅读 `vllm/v1/engine/core.py`。

## **5. 关键流程二：HELLO / READY 握手**

这是整份代码最值得断点阅读的流程。

核心方法：

```
wait_for_engine_startup(    handshake_socket,    core_engines,    parallel_config,    coordinated_dp,    cache_config,    launch,)
```

### **5.1 状态机**

**NEW**

EngineCore 已被管理端登记，尚未连接

HELLO

EngineCore → ROUTER

**CONNECTED**

管理端已识别 Engine，并下发初始化元数据

READY

EngineCore → ROUTER

**READY**

初始化完成，通过启动屏障

对应代码：

```
if status == "HELLO" and engine.state == CoreEngineState.NEW:    # 发送初始化信息    handshake_socket.send_multipart(        (eng_identity, init_message),        copy=False,    )    engine.state = CoreEngineState.CONNECTEDelif status == "READY" and engine.state == CoreEngineState.CONNECTED:    engine.state = CoreEngineState.READY
```

这里有一个重要设计：

**HELLO 代表进程已经能够通信，不代表模型已经加载完成。READY 才代表 EngineCore 完成了它的启动初始化。**

### **5.2 为什么需要两个阶段？**

EngineCore 的启动通常涉及：

1. 创建 Executor。
2. 启动 GPU Worker。
3. 初始化分布式通信。
4. 加载模型。
5. 初始化 KV Cache。
6. 完成运行前准备。

这些操作可能耗时较长。

因此，进程创建成功和模型初始化成功是两个不同的状态。

管理端不能只通过 `Process.start()` 就认为 EngineCore 可以接受请求。

### **5.3 为什么使用 ROUTER？**

代码：

```
with zmq_socket_ctx(    local_handshake_address,    zmq.ROUTER,    bind=True,) as handshake_socket:
```

因为多个 EngineCore 需要连接同一个管理端。

ROUTER 可以根据每个连接的 Identity 识别消息发送方。

消息逻辑：

```
                ROUTER
                  |
       +----------+----------+
       |          |          |
    Engine0    Engine1    Engine2
       |          |          |
     HELLO      HELLO      HELLO
       |          |          |
    READY      READY      READY
```

这里 `recv_multipart()` 接收到：

```
eng_identity, ready_msg_bytes = (    handshake_socket.recv_multipart())
```

通过 `eng_identity` 找到对应的 `CoreEngine`：

```
engine = next(    (e for e in core_engines if e.identity == eng_identity),    None,)
```

因此，消息顺序不需要和 DP Rank 顺序一致。

### **5.4 启动屏障**

代码维护两个计数器：

```
conn_pending = [local_count, remote_count]start_pending = [0, 0]
```

分别表示：

| 变量              | 含义                            |
| --------------- | ----------------------------- |
| `conn_pending`  | 尚未完成 HELLO 的 Engine 数量        |
| `start_pending` | 已经 HELLO、尚未 READY 的 Engine 数量 |

只有当：

```
while any(conn_pending) or any(start_pending):
```

条件不再成立时，才算完成启动屏障。

例如 DP=4，其中本地 2 个、远程 2 个：

| 阶段         | Local Pending | Remote Pending |
| ---------- | ------------- | -------------- |
| 初始连接       | 2             | 2              |
| 全部 HELLO   | 0             | 2              |
| 远程全部 HELLO | 0             | 0              |
| 部分 READY   | 0             | 0              |
| 全部 READY   | 0             | 0              |

注意：`start_pending` 会在 HELLO 时增加，收到 READY 时减少。因此两个阶段可以交错进行。

## **6. 关键流程三：GPU ID 映射**

重点方法：

```
set_assigned_physical_gpu_ids_for_dp_rank()get_physical_gpu_ids_for_local_dp_rank()
```

目的：将 DP Rank 对应的逻辑 GPU 范围映射到实际物理 GPU ID。

例如：

```
GPU 物理编号：

[0, 1, 2, 3, 4, 5, 6, 7]

DP Size = 2
World Size = 4
Local World Size = 4

DP0 -> [0, 1, 2, 3]
DP1 -> [4, 5, 6, 7]
```

核心计算：

```
start = local_dp_rank * world_sizestop = start + local_world_sizereturn user_assigned_gpu_ids[start:stop]
```

如果用户没有通过 `--device-ids` 指定 GPU，则通过平台设备映射接口获取物理 GPU ID。

注意两个概念：

- `local_dp_rank`：当前机器内的 DP 编号。
- `world_size`：一个 DP Engine 所需要的 GPU Worker 总数。

代码还特别处理了同一个 `VllmConfig` 被多个进程依次启动时，GPU ID 映射可能被覆盖的问题，所以在启动每个进程前单独更新配置。

## **7. 关键流程四：Ray Placement Group 创建**

Ray 路径最重要的方法：

```
create_dp_placement_groups()
```

它支持三种放置策略：

| 策略       | 行为                    | 适用情况                   |
| -------- | --------------------- | ---------------------- |
| `strict` | 一个 DP Engine 的资源放在单节点 | 节点内资源充足                |
| `fill`   | 尽量填满当前节点，再使用其他节点      | 提高资源利用率                |
| `span`   | 一个 DP Engine 可以跨多个节点  | 单节点 GPU 不足以容纳一个 Engine |

其中：

- `STRICT_PACK`：整个 Placement Group 放在同一节点。
- `PACK`：尽可能将 Bundle 放在较少节点上，但允许跨节点。

代码还对 DeepEP 有特殊限制：`fill` 不保证 EP Rank 的节点分布满足 DeepEP 要求，因此不能用于对应的 DeepEP All2All 后端。

### **Ray Actor 创建过程**

```
actor = (    ray.remote(actor_class)    .options(        scheduling_strategy=PlacementGroupSchedulingStrategy(            placement_group=pg,            placement_group_bundle_index=world_size,        ),        runtime_env=runtime_env,    )    .remote(        vllm_config=dp_vllm_config,        executor_class=executor_class,        log_stats=log_stats,        local_client=local_client,        addresses=addresses,        dp_rank=index,        local_dp_rank=local_index,    ))
```

可以理解为：

```
Ray Cluster
    |
    +-- PlacementGroup DP0
    |      +-- GPU Bundle 0
    |      +-- GPU Bundle 1
    |      +-- GPU Bundle 2
    |      +-- GPU Bundle 3
    |      +-- CPU Bundle
    |              |
    |          EngineCoreActor DP0
    |
    +-- PlacementGroup DP1
           +-- GPU Bundle 0
           +-- GPU Bundle 1
           +-- GPU Bundle 2
           +-- GPU Bundle 3
           +-- CPU Bundle
                   |
               EngineCoreActor DP1
```

Actor 创建后：

```
ray.get(refs)for actor in self.local_engine_actors + self.remote_engine_actors:    ref = actor.run.remote()    self.run_refs.append(ref)
```

这里分为两个阶段：

1. `wait_for_init.remote()`：等待 Actor 初始化完成。
2. `run.remote()`：启动 Actor 的持续运行任务。

这和本地 multiprocessing 路径的 HELLO/READY 握手不能简单地视为同一套实现。

## **8. 关键流程五：故障监控与关闭**

### **8.1 本地进程监控**

```
def monitor_engine_liveness(self):    sentinel_to_proc = {        proc.sentinel: proc for proc in self.processes    }    while sentinels and not self.manager_stopped.is_set():        died_sentinels = connection.wait(            sentinels,            timeout=1,        )
```

**Sentinel 是什么？**

它是 multiprocessing 提供的进程退出监控句柄。

当子进程退出时，对应 Sentinel 会变为可读状态。

管理器就可以：

```
exitcode = proc.exitcodeif exitcode != 0:    self.failed_proc_name = proc.name
```

检测到异常后，退出监控循环并触发统一关闭。

### **8.2 Ray Actor 监控**

```
actor_done_refs, _ = ray.wait(    actor_run_refs,    timeout=5,)
```

通过 ObjectRef 监控 Actor 的运行任务。

如果出现：

```
ray.get(actor_ref)
```

抛出 `RayActorError`，就认为发生异常。

### **8.3 关闭流程**

| 管理对象            | 关闭方式                                  |
| --------------- | ------------------------------------- |
| multiprocessing | `shutdown(self.processes)`            |
| Ray Actor       | `ray.kill(actor)`                     |
| Placement Group | `ray.util.remove_placement_group(pg)` |
| 监控线程/循环         | `manager_stopped.set()`               |

特别值得研究：

```
get_engine_process_shutdown_timeout()
```

它区分了两个超时：

- `request_timeout`：允许在途请求继续排空的时间。
- `process_timeout`：进程释放设备资源的等待时间。

ROCm 在两个超时都为 0 的特定情况下，会额外获得 15 秒清理时间，避免强制杀进程导致 VRAM 残留。

## **9. 其他重要方法索引**

| 方法                                            | 功能                        | 阅读优先级 |
| --------------------------------------------- | ------------------------- | ----- |
| `get_engine_zmq_addresses()`                  | 分配 Engine 与 Frontend 通信地址 | 高     |
| `launch_core_engines()`                       | 总体启动流程                    | 最高    |
| `wait_for_engine_startup()`                   | 握手与 READY 屏障              | 最高    |
| `CoreEngineProcManager.__init__()`            | 创建子进程、分配 GPU              | 最高    |
| `CoreEngineActorManager.__init__()`           | 创建 Ray Actor              | 高     |
| `create_dp_placement_groups()`                | DP 资源分配                   | 高     |
| `set_assigned_physical_gpu_ids_for_dp_rank()` | GPU 映射                    | 高     |
| `monitor_engine_liveness()`                   | 进程/Actor 监控               | 中     |
| `scale_up_elastic_ep()`                       | 弹性扩容                      | 中     |
| `scale_down_elastic_ep()`                     | 弹性缩容                      | 中     |
| `SignalCallback`                              | 信号处理与线程协作                 | 低     |

## **10. 建议的 VS Code 断点阅读顺序**

不要从文件第一行开始逐行阅读。建议围绕一次 EngineCore 启动建立调用链。

1. `get_engine_zmq_addresses()`

   观察 IPC/TCP 地址如何生成，为什么有些端口使用 `tcp://host:0`。
2. `launch_core_engines()`

   观察 DP 配置、Coordinator、Backend 分支。
3. `CoreEngineProcManager.__init__()`

   观察进程创建、DP Rank、GPU 映射和 `proc.start()`。
4. `EngineCoreProc.run_engine_core()`

   跳转到 `vllm/v1/engine/core.py`，研究 EngineCore 内部初始化。
5. **EngineCore HELLO 发送位置**

   追踪 EngineCore 如何连接 ROUTER、发送 HELLO、接收初始化元数据。
6. `wait_for_engine_startup()`

   观察 HELLO、READY 状态变化和启动屏障。
7. `monitor_engine_liveness()`

   观察异常退出、进程清理和资源回收。

### **建议重点观察的变量**

| 变量                   | 需要搞清楚的问题                         |
| -------------------- | -------------------------------- |
| `dp_size`            | 总共有多少个 EngineCore？               |
| `local_engine_count` | 当前节点启动几个？                        |
| `dp_rank`            | 全局 DP 编号如何分配？                    |
| `local_dp_rank`      | 本地 DP 编号如何分配？                    |
| `world_size`         | 每个 EngineCore 使用多少个 Worker Rank？ |
| `addresses`          | Frontend 和 EngineCore 如何建立通信？    |
| `identity`           | 如何识别不同 EngineCore？               |
| `conn_pending`       | 还有多少 Engine 没有连接？                |
| `start_pending`      | 还有多少 Engine 没有 READY？            |

## **11. 最终总结：这份代码应该掌握什么？**

我建议把这份源码归纳成四个核心知识点：

1. **进程模型**：EngineCore 为什么独立运行在后台进程或 Ray Actor 中？
2. **资源调度**：DP Rank 如何对应 GPU 资源，Ray Placement Group 如何组织资源？
3. **通信机制**：Frontend 如何通过 ZMQ ROUTER 与 EngineCore 完成启动握手？
4. **生命周期管理**：如何保证所有 EngineCore READY，如何检测异常和释放资源？

这份代码没有直接实现模型推理调度，真正的推理执行逻辑还在 EngineCore 和 Executor 内部。

**下一步最值得读的是&#x20;**`vllm/v1/engine/core.py`**&#x20;中的&#x20;**`EngineCoreProc.run_engine_core()`**。** 把它与本文件的 `launch_core_engines()`、`wait_for_engine_startup()` 串起来，就能理解 vLLM 从创建 EngineCore 到模型真正初始化完成的完整过程。

参考源码：本次上传文件 `CoreEngineProcManager / CoreEngineActorManager`，主要对应第 144–274、401–1035、1039–1416 行。

如果继续推进源码学习，可以下一步把 `core.py` 的 EngineCore、Executor、Worker 调用关系整理成一张完整类图和启动时序图。

明天继续研究 EngineCore 内部初始化和 Executor 创建流程，可以使用下面的学习计划：

📚明天继续学习 EngineCore 源码轻点设置
