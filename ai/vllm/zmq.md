# **vLLM 中的 ZMQ 通信原理与源码分析**

本文以你提供的 vLLM 源码为例，解释 ZMQ 的 Socket 模式、通信机制、消息格式、Poll、握手及完整请求链路。

## **1. ZMQ 是什么？**

ZMQ（ZeroMQ）是消息通信库，提供 Socket、消息队列、路由和异步通信机制。

它不是 TCP 的替代品，而是构建在 TCP、IPC 等传输方式之上的消息通信抽象。

vLLM 使用 ZMQ 实现 Frontend 与 EngineCore 多进程之间的通信。

```
应用层：EngineCoreRequest / EngineCoreOutput
                    |
消息层：MessagePack / Multipart / Tensor IPC
                    |
通信层：ROUTER / DEALER / PUSH / PULL / XSUB
                    |
传输层：TCP / IPC
                    |
操作系统：Socket / 网络
```

## **2. 四种核心 Socket 通信原理**

### **2.1 ROUTER / DEALER：异步双向通信**

vLLM：请求与握手通道

```
              Frontend
             ROUTER Socket
                  |
          +-------+-------+
          |       |       |
        Rank0   Rank1   Rank2
          |       |       |
        DEALER  DEALER  DEALER
```

**ROUTER 特点：**

- 可以连接多个 DEALER。
- 接收消息时自动附加发送方 Identity。
- 发送消息时根据 Identity 找到目标连接。
- 支持双向、异步通信。

**DEALER 特点：**

- 异步发送和接收。
- 不强制一发一收。
- 可以连接 ROUTER。
- 不负责指定返回消息的目标 Identity。

vLLM 对应代码：

```
# Frontendself.input_socket = make_zmq_socket(    self.ctx,    input_address,    zmq.ROUTER,    bind=True,)# EngineCoreinput_socket = make_zmq_socket(    ctx,    input_address,    zmq.DEALER,    identity=identity,    bind=False,)
```

Identity 对应：

```
identity = rank.to_bytes(2, "little")
```

ROUTER 收到 DEALER 消息：

```
identity, payload = router.recv_multipart()
```

ROUTER 向指定 DEALER 发送：

```
router.send_multipart([    identity,    payload,])
```

因此 ROUTER/DEALER 解决的是：**多个 EngineCore 如何被识别、选择和双向通信。**

参考：[ZeroMQ Guide：ROUTER/DEALER](https://zguide.zeromq.org/docs/chapter3/?utm_source=chatgpt.com)。

### **2.2 PUSH / PULL：生产者与消费者**

vLLM：推理结果通道

```
EngineCore 0 PUSH ---+
                     |
EngineCore 1 PUSH ---+----> Frontend PULL
                     |
EngineCore 2 PUSH ---+
```

特点：

- PUSH 只能发送。
- PULL 只能接收。
- 一个 PULL 可以连接多个 PUSH。
- PUSH 向可用的下游 PULL 分发消息。
- PULL 从多个上游公平地接收消息。

vLLM 对应代码：

```
# Frontendself.resources.output_socket = make_zmq_socket(    self.ctx,    output_address,    zmq.PULL,)# EngineCoresocket = make_zmq_socket(    ctx,    output_path,    zmq.PUSH,)
```

这里不是 ROUTER/DEALER 的 Identity 路由模式，而是结果汇聚模式。

**关键：PULL 不识别 request_id，也不负责请求匹配。它只接收消息，具体请求由上层协议识别。**

参考：[ZeroMQ Socket API：PUSH/PULL](https://zeromq.org/socket-api/?utm_source=chatgpt.com)。

### **2.3 XSUB / PUB：发布订阅**

vLLM：DP Coordinator 通信

```
       DP Coordinator
            PUB
             |
       +-----+-----+
       |           |
     XSUB        XSUB
   EngineCore 0 EngineCore 1
```

特点：

- PUB 发布消息。
- SUB 订阅消息。
- XSUB 是暴露订阅控制能力的底层订阅端。
- 订阅端可以通过发送订阅帧控制订阅内容。

vLLM：

```
coord_socket = make_zmq_socket(    ctx,    coord_input_address,    zmq.XSUB,    identity=identity,    bind=False,)coord_socket.send(b"\x01")
```

`b"\x01"` 表示订阅空前缀，即接收所有主题的消息。

注意：XSUB 不能直接和 PUSH/PULL 配对，它属于发布订阅模式。[ZeroMQ 协议规范：XSUB 订阅帧](https://rfc.zeromq.org/spec/39/?utm_source=chatgpt.com)。

## **3. ZMQ 的其他重要机制**

### **3.1 bind 和 connect**

```
# 服务端通常创建监听端点router.bind("tcp://127.0.0.1:5555")# 客户端连接服务端dealer.connect("tcp://127.0.0.1:5555")
```

在 vLLM 中：

| Socket            | 操作      |
| ----------------- | ------- |
| Frontend ROUTER   | bind    |
| EngineCore DEALER | connect |
| Frontend PULL     | bind    |
| EngineCore PUSH   | connect |

bind 表示创建监听端点，connect 表示建立到端点的连接。它们不是发送端和接收端的同义词。

### **3.2 Multipart：多帧消息**

ZMQ 支持一条消息包含多个 Frame：

```
socket.send_multipart([    b"ADD",    b"request_data",])
```

接收：

```
frames = socket.recv_multipart()
```

vLLM 输入侧：

```
type_frame, *data_frames = input_socket.recv_multipart(    copy=False)
```

消息格式可以理解为：

```
Frame 0：消息类型
Frame 1：请求元数据
Frame 2：其他数据
...
```

ROUTER 还会在 DEALER 消息前增加 Identity 帧：

```
ROUTER 接收到：

[Identity][Message Type][Payload]
```

### **3.3 Poller：事件轮询**

Poller 用于同时监听多个 Socket，而不是为每个 Socket 创建接收线程。

```
poller = zmq.Poller()poller.register(socket0, zmq.POLLIN)poller.register(socket1, zmq.POLLIN)while True:    events = dict(poller.poll())    if socket0 in events:        msg = socket0.recv_multipart()    if socket1 in events:        msg = socket1.recv_multipart()
```

`POLLIN` 表示 Socket 可读。

vLLM 有两种典型使用：

| 源码                          | Poll 目的                        |
| --------------------------- | ------------------------------ |
| `wait_for_engine_startup()` | 等待握手消息，同时监控进程退出                |
| `process_input_sockets()`   | 监听多个 DEALER 和 Coordinator XSUB |

### **3.4 Identity：连接身份**

```
identity = rank.to_bytes(2, "little")
```

Identity 是 ROUTER 使用的连接路由标识。

它与以下概念不同：

- `request_id`：请求标识。
- `client_index`：Frontend Client 索引。
- `engine_index`：EngineCore 索引。

不要把它们混为一谈。

## **4. vLLM 完整通信链路**

### **4.1 启动阶段：HELLO / INIT / READY**

对应 `wait_for_engine_startup()`：

```
EngineCore DEALER              Frontend ROUTER
       |                              |
       | -------- HELLO ------------> |
       |                              |
       | <--------- INIT ------------- |
       |                              |
       | -------- READY ------------> |
       |                              |
       |                         State = READY
```

源码逻辑：

```
if status == "HELLO":    handshake_socket.send_multipart(        (eng_identity, init_message)    )    engine.state = CoreEngineState.CONNECTEDelif status == "READY":    engine.state = CoreEngineState.READY
```

作用：

- HELLO：EngineCore 发起握手。
- INIT：Frontend 下发初始化元数据。
- READY：EngineCore 初始化完成。

Poller 等待握手消息，同时监控 EngineCore、Coordinator 等进程的退出事件。

注意：你提供的 `process_input_sockets()` 还有一条初始 Ready 消息，用于建立输入路由可用性。它与上述握手流程相关，但不能仅凭当前片段认定是同一条消息。

### **4.2 请求阶段：Frontend → EngineCore**

对应 `process_input_sockets()`：

```
Frontend ROUTER
       |
       | [Identity][ADD][Request]
       v
EngineCore DEALER
       |
       v
process_input_sockets()
       |
       +-- recv_multipart()
       |
       +-- MsgpackDecoder
       |
       +-- preprocess_add_request()
       |
       v
input_queue
       |
       v
EngineCore Busy Loop
       |
       v
Scheduler
       |
       v
Executor / Worker / ModelRunner
```

消息对象：

```
class EngineCoreRequest:    request_id: str    prompt_token_ids: list[int] | None    sampling_params: SamplingParams | None    client_index: int = 0    data_parallel_rank: int | None
```

这里的 MessagePack 负责序列化请求；Tensor IPC 可以额外传递多模态 Tensor。

### **4.3 结果阶段：EngineCore → Frontend**

对应 `process_output_sockets()`：

```
ModelRunner
    |
    v
Scheduler
    |
    v
output_queue
    |
    v
process_output_sockets()
    |
    +-- MsgpackEncoder
    +-- Buffer 复用
    +-- MessageTracker
    |
    v
EngineCore PUSH
    |
    v
Frontend PULL
    |
    v
AsyncMPClient
    |
    v
请求结果分发
```

结果对象：

```
class EngineCoreOutput:    request_id: str    new_token_ids: list[int]    finish_reason: FinishReason | None    engine_index: int
```

### **4.4 PULL 如何区分请求？**

PULL 本身不区分，依靠应用层字段：

```
client_index, outputs = outputoutputs.engine_index = engine_indexbuffers = encoder.encode_into(outputs, buffer)tracker = self._send_msg_tracking_payload(    sockets[client_index], buffers)
```

三个字段的职责：

| 字段             | 作用                            |
| -------------- | ----------------------------- |
| `client_index` | 选择发往哪个 Frontend Client Socket |
| `engine_index` | 标识输出来自哪个 EngineCore           |
| `request_id`   | 关联具体的推理请求                     |

例如 PULL 收到：

```
Output(request_id=A, engine_index=0)
Output(request_id=B, engine_index=1)
Output(request_id=A, engine_index=0)
```

Frontend 根据 `request_id` 将结果分别归入 A、B 请求。

当 `finish_reason != None` 时，该请求完成。

## **5. 最终架构图**

```
                    Frontend / MPClient
                 +-----------------------+
                 |                       |
                 | ROUTER          PULL  |
                 +-----------------------+
                    |               ^
                    |               |
                 DEALER             PUSH
                    |               |
          +---------+---------------+---------+
          |                                   |
     EngineCore 0                        EngineCore 1
          |                                   |
   process_input_sockets()             process_input_sockets()
          |                                   |
     input_queue                         input_queue
          |                                   |
          +---------- Busy Loop -------------+
                         |
                    Scheduler
                         |
                      Executor
                         |
                     ModelRunner
                         |
                    output_queue
                         |
                 process_output_sockets()
                         |
                       PUSH
                         |
                    Frontend PULL

       DP Coordinator：
       PUB -> EngineCore XSUB
       EngineCore PUSH -> Coordinator 接收端
```

## **6. 核心知识总结**

| ZMQ 概念          | vLLM 中的实际用途         |
| --------------- | ------------------- |
| Context         | Socket 和 IO 资源管理    |
| ROUTER / DEALER | 请求路由、握手、控制通信        |
| PUSH / PULL     | 推理结果传输与汇聚           |
| XSUB / PUB      | DP Coordinator 发布订阅 |
| Identity        | EngineCore 连接身份     |
| Multipart       | 消息类型与数据分帧           |
| Poller          | 多 Socket 事件监听       |
| bind / connect  | 建立通信端点              |
| MessageTracker  | 追踪发送完成状态            |
| LINGER          | Socket 关闭时等待未完成发送   |

**理解 vLLM ZMQ 最重要的四句话：**

1. ROUTER/DEALER 解决 EngineCore 之间的定向通信。
2. PUSH/PULL 解决推理结果的传输与汇聚。
3. MessagePack 和 Tensor IPC 解决消息内容如何传输。
4. `request_id`、`client_index`、`engine_index` 解决业务层如何关联消息。

参考资料：[ZeroMQ Guide：高级请求应答模式](https://zguide.zeromq.org/docs/chapter3/?utm_source=chatgpt.com)、[ZeroMQ Socket API](https://zeromq.org/socket-api/?utm_source=chatgpt.com)。
