# vLLM 为什么使用 ZeroMQ（ZMQ）？

先明确一个核心结论：

vLLM 使用 ZeroMQ（简称 ZMQ），主要是为了解决多进程、多组件之间的高性能异步消息通信问题。

vLLM 不只是一个模型推理函数，它内部包含 API Server、Engine、Scheduler、Model Executor、Worker 等多个组件。这些组件可能运行在不同进程中，甚至分布在不同 GPU 设备上。

它们需要频繁交换：

- 请求提交、请求取消等控制消息。
- Scheduler 的调度结果。
- Token 生成结果。
- Engine 状态和执行反馈。
- 多进程之间的任务分发和同步信息。

如果全部使用 Python Queue、Multiprocessing Pipe 或传统 TCP Socket，需要自己处理大量通信细节。

ZMQ 提供了一套更高层次的消息通信抽象。

## 1. ZMQ 是什么？

[Zeromq SVG Icon - Free Zeromq PNG & SVG Download](https://images.openai.com/static-rsc-4/NThVpfVwyrGdHZ0xY-kAUffVvEoFlFaXttQ7lflyuRMhCTScCO4j1VP9UqivT_w37slRKuMeIsLmCVb2vHBtjZfPn9uUOOQu-IJBkpPC1Vwl73sJ7b-a4ORAd2l0afTvWDewFHdU8nhNpi2TXhB_EX0IY220A_jFBJ71OLJpfgQ?purpose=inline)

ZeroMQ

简称 ZMQ / ØMQ

一个高性能、异步、面向消息的通信库。

不是传统意义上的消息队列服务器，也不是独立的消息 Broker。

它的核心定位：

在不同进程、线程或者机器之间，提供统一的消息通信接口。

可以把它理解为：

传统 Socket：

```
应用程序
   |
TCP Socket
   |
自己处理连接、收发、消息边界、
缓冲、重连、消息分发等
```

ZeroMQ：

```
应用程序
   |
ZMQ Socket
   |
ZMQ 内部处理：
连接管理、消息队列、异步收发、
消息路由、通信模式
   |
TCP / IPC / Inproc
```

注意：ZMQ 底层仍然使用 TCP、IPC 等传输机制，并不是取代 TCP。

## 2. 为什么 vLLM 需要 ZMQ？

主要有四个原因。

| 需求      | ZMQ 提供的能力                 |
| ------- | ------------------------- |
| 多进程通信   | IPC、TCP 通信                |
| 高并发消息收发 | 异步消息队列                    |
| 不同通信关系  | PUB/SUB、PUSH/PULL、REQ/REP |
| 降低通信开销  | 消息批量处理、异步 I/O、内部缓冲        |

尤其是 vLLM 的多进程架构。

例如：

```
              API Server
                   |
                   v
              Engine Core
                   |
             Scheduler
                   |
             Model Executor
                   |
          GPU Worker Processes
                   |
                  GPU
```

这些组件之间并不一定在同一个进程。

如果使用普通函数调用，只能在同一进程内直接调用；跨进程就需要 IPC 或网络通信。

ZMQ 让这些组件可以通过 Socket 进行消息交互，而不必直接管理底层通信细节。

## 3. ZeroMQ 的核心设计架构

理解 ZMQ，重点掌握下面五个概念：

1. Context
2. Socket
3. Transport
4. Message Queue
5. Messaging Pattern

整体结构：

[Kafka vs ZeroMQ: Architectures, Performance, Use Cases · AutoMQ/automq Wiki · GitHub](https://images.openai.com/static-rsc-4/cnGTAP_a07-s1rSeuIOJBLVQ0mHgkBoxF7S5FqlKtDIJQQxBlKMIfblO4hzqIveERhWXPXd74yY0HYsH76y1HIhWMj0t_Q62WDoHTXRSX8r8zDhcYx4ohmNdiigupFIaZd0mswbtWfHpjYottudmK0etoLg1go4dOt4ecOtWw-A?purpose=inline)

[github.com](https://github.com/AutoMQ/automq/wiki/Kafka-vs-ZeroMQ%3A-Architectures%2C-Performance%2C-Use-Cases?utm_source=chatgpt.com)

### 3.1 Context：通信上下文

Context 是 ZMQ 的运行环境。

```
import zmqctx = zmq.Context()
```

作用：

- 管理 ZMQ 内部的 I/O 线程。
- 管理 Socket 的生命周期。
- 为 Socket 提供通信环境。
- 同一个 Context 下的 Socket 可以通过 `inproc://` 通信。

例如：

```
ctx = zmq.Context(io_threads=2)socket1 = ctx.socket(zmq.PUSH)socket2 = ctx.socket(zmq.PULL)
```

这里的 `io_threads=2` 表示 ZMQ 内部使用两个 I/O 工作线程处理通信任务。

注意：它不是说应用程序只能创建两个 Socket，也不是说只有两个业务线程。

### 3.2 Socket：通信端点

Socket 是 ZMQ 最核心的抽象。

传统 Socket 主要关注网络连接，而 ZMQ Socket 还包含通信模式、消息路由和内部消息队列等语义。

例如：

```
socket = ctx.socket(zmq.PUSH)socket.bind("tcp://*:5555")
```

常用接口：

| 接口                 | 作用        |
| ------------------ | --------- |
| `ctx.socket()`     | 创建 Socket |
| `bind()`           | 绑定通信地址    |
| `connect()`        | 连接远端地址    |
| `send()`           | 发送消息      |
| `recv()`           | 接收消息      |
| `send_multipart()` | 发送多帧消息    |
| `recv_multipart()` | 接收多帧消息    |
| `close()`          | 关闭 Socket |

这里有一个容易混淆的概念：

bind 和 connect 并不完全等同于传统 TCP Server 和 Client。

- bind：在指定地址上建立监听端点。
- connect：发起与目标端点的连接。
- 连接建立、断开、重连等由 ZMQ 在后台处理。

一个 Socket 可以连接多个远端 Socket，而不是传统 TCP Socket 那样通常对应一个连接。

[image](https://www.google.com/s2/favicons?domain=https://zguide.zeromq.org\&sz=32)

ØMQ - The Guide+1

### 3.3 Transport：底层传输方式

ZMQ 不限定必须通过 TCP 通信，而是提供统一的 Socket API，底层可以使用不同传输方式。

| 类型     | 地址格式                   | 应用场景      |
| ------ | ---------------------- | --------- |
| inproc | `inproc://name`        | 同一进程内线程通信 |
| ipc    | `ipc:///tmp/a.sock`    | 同一机器不同进程  |
| tcp    | `tcp://127.0.0.1:5555` | 本地或跨机器通信  |

可以理解为：

```
             ZMQ Socket API
                    |
         +----------+----------+
         |          |          |
       inproc      ipc        tcp
         |          |          |
       内存队列    Unix域套接字  TCP网络
```

对于 vLLM：

- 同进程线程之间可以使用 inproc。
- 同一台机器的多进程通信可以使用 IPC。
- 跨机器通信可以使用 TCP。

这让上层业务不需要针对不同传输协议重新设计通信代码。

### 3.4 Message Queue：内部消息队列

这是 ZMQ 的重要设计。

发送端调用：

```
socket.send(b"hello")
```

不意味着应用程序必须等待接收端立即处理这条消息。

ZMQ 会在内部维护消息队列：

```
发送端
  |
  | send(message)
  v
ZMQ 内部发送队列
  |
  | 异步 I/O
  v
底层传输
  |
  v
ZMQ 内部接收队列
  |
  v
接收端 recv()
```

它解决了两个问题：

问题一：发送和接收解耦

发送方不必和接收方严格同步执行。

问题二：缓冲和流量控制

当接收端处理不过来时，ZMQ 可以通过队列缓冲消息，并使用 HWM（High Water Mark，高水位）限制队列容量。

需要注意：

- ZMQ 队列不是无限的。
- 达到 HWM 后，不同 Socket 类型的行为可能不同，例如阻塞或丢弃。
- ZMQ 的异步发送不代表消息已经被对端业务处理。
- 不要把 ZMQ 当成具有持久化和消费确认机制的 Kafka。

[image](https://www.google.com/s2/favicons?domain=https://zguide.zeromq.org\&sz=32)

ØMQ - The Guide+1

## 4. ZMQ 的六种重要通信模式

ZMQ 的核心设计思想之一是：

Socket 类型决定通信模式，而不仅仅是通信连接。

常见模式如下。

4.1 REQ / REP：请求-响应

[7. Building an Asynchronous Microservice Architecture | PHP 7 Programming Blueprints](https://images.openai.com/static-rsc-4/L8VqVq2fXgruieSI-teA2ITjb6LCfT2KBXjfJLJwAGe0qZioABmPd-aXz5jBSdqdpt8c4cFtgJ0adZtHmelo2-_0zfL6OZpG-RHLbeUaM0IiAvfj2D9a3kN0WVHO23G6meOCwPf3VANkcmON84-kRjVFkmWF3iVKVBjjIfmiwsQ?purpose=inline)

[subscription.packtpub.com](https://subscription.packtpub.com/book/web-development/9781785889714/7/ch07lvl1sec36/zeromq-patterns?utm_source=chatgpt.com)

```
Client                 Server
  |                       |
  |------ Request ------->|
  |                       |
  |<----- Response -------|
```

- REQ：发送请求，等待响应。
- REP：接收请求，返回响应。
- 严格遵循 Send → Receive → Send → Receive。

适合简单 RPC。

4.2 DEALER / ROUTER：异步请求-响应

[2.21.2.2. Asynchronous Request Reply Pattern](https://images.openai.com/static-rsc-4/1mgUVMhlpkd0muUkH6mo8WEzLzi3c6dWHdpqaOiid6EXl98JmwenPI9eqhk7xZpG2VLlMPkbrpMlZO-0RHSXx-pA1LX5jQ3Fkp1vyKt-MOHwmpzeY7ICqpgmDf_MFb_IhteCu41ErTXG0lnyddPTE-1txahlXsQ5SMV8UBGi0Yc?purpose=inline)

[d3s.mff.cuni.cz](https://d3s.mff.cuni.cz/files/teaching/nswi080/lectures/slides/ch02s21s02s02.html?utm_source=chatgpt.com)

```
Client A ----\
              \
               ROUTER
              /
Client B ----/
                |
              DEALER
                |
              Worker
```

- DEALER：异步发送和接收，支持多个对端。
- ROUTER：根据消息中的 Identity 路由到指定对端。
- 不需要像 REQ/REP 一样严格交替收发。

适合多客户端、多服务端、异步任务调度。

4.3 PUB / SUB：发布-订阅

[2.20.2.4. Dynamic Publish Subscribe Pattern](https://images.openai.com/static-rsc-4/akaqMHyXq13xLH5qVBp5KTynM6DMhdeQrkAZOfCuTMfAzFTDzhnZ1CPQPSsWgHA7Dhx78VPtMm666hfRu3kbN-Hil_2I_wHVdT-zanBJW9HdFGIQ7mbvh4rYDFSDXSyNHbZ3iVxWnGJbTj7hi-mzQfCs7q__CW97dLcJPLGwxmM?purpose=inline)

[d3s.mff.cuni.cz](https://d3s.mff.cuni.cz/files/teaching/nswi080/lectures/slides/ch02s20s02s04.html?utm_source=chatgpt.com)

```
            Publisher
            /   |   \
           /    |    \
         SUB   SUB   SUB
```

- PUB：发布消息。
- SUB：订阅消息。
- 可以根据 Topic 过滤消息。
- 一对多广播。

适合状态广播、事件通知。

4.4 PUSH / PULL：任务分发

[ZeroMQ – ssup2 Blog / Software](https://images.openai.com/static-rsc-4/o8bv6SKJmzecKqu2DTbOeAA-tWersJPxeIWdbLDn-8doxbVCUL7XIe5moAUwWMZ4no08RVzuJL8peIWi6P4TEj8LbygKFAImhVguvSvvqvxDfYIs3d7yePh1ggBZExYBj_uMWlTxpY1bFQkVADAU4XRGHWaFH7GJaq44i3rJ9y0?purpose=inline)

[ssup2.github.io](https://ssup2.github.io/blog-software/docs/theory-analysis/zeromq/?utm_source=chatgpt.com)

```
              PUSH
             /    \
            /      \
         PULL      PULL
        Worker1   Worker2
```

- PUSH：向多个 Worker 分发任务。
- PULL：接收任务。
- 多个 Worker 之间通常采用轮询分发。

适合任务队列、并行计算。

4.5 PAIR：点对点通信

```
Socket A <----------> Socket B
```

一对一双向通信，通常用于两个线程之间的协调。

4.6 XPUB / XSUB：扩展发布订阅

在 PUB/SUB 基础上暴露订阅事件，常用于构建消息代理和订阅转发系统。

这些模式的关键差异：

| 模式            | 方向   | 核心用途      |
| ------------- | ---- | --------- |
| REQ/REP       | 双向   | 同步请求响应    |
| DEALER/ROUTER | 双向   | 异步请求响应、路由 |
| PUB/SUB       | 单向发布 | 广播        |
| PUSH/PULL     | 单向   | 负载分发      |
| PAIR          | 双向   | 一对一       |
| XPUB/XSUB     | 单向发布 | 扩展订阅管理    |

参考：

[image](https://www.google.com/s2/favicons?domain=https://zguide.zeromq.org\&sz=32)

ØMQ - The Guide+2



## 5. vLLM 中 ZMQ 的实际应用架构

这是最重要的部分：理解 ZMQ 如何参与 vLLM 的推理流程。

以下以 vLLM V1 多进程架构为主。

官方架构中，API Server 通过 ZMQ 和 Engine Core 通信；Engine Core 负责 Scheduler、KV Cache 管理和 GPU Worker 协调。

[image](https://www.google.com/s2/favicons?domain=https://docs.vllm.ai\&sz=32)

vLLM+1



### 5.1 vLLM 的进程架构

[Architecture Overview - vLLM](https://images.openai.com/static-rsc-4/xullan2hhO1nQ794PEI2tpTilEFxL1k21GI-62RRNVd1iU_rS9epa8fSlX4gk3XMa5Wa8j0phtlTZHjUSsZxABr0gbCTukjpOUuDyGOq8_fZWgnnI8mu9ik6L-t-qvOHv7kgW5YBmzlf0SCGYvECl1jQC42g-MdqPu4jjTGstyE?purpose=inline)

[docs.vllm.ai](https://docs.vllm.ai/en/latest/design/arch_overview.html?utm_source=chatgpt.com)

可以简化成：

```
              API Server Process
                      |
                 ZMQ Socket
                      |
                      v
              Engine Core Process
                      |
             +--------+--------+
             |                 |
         Scheduler        KV Cache
             |
             v
         Executor
             |
       +-----+-----+
       |           |
    Worker 0    Worker 1
       |           |
      GPU 0       GPU 1
```

不同组件的职责：

| 进程          | 职责                       |
| ----------- | ------------------------ |
| API Server  | HTTP 请求、Tokenizer、输入输出处理 |
| Engine Core | 请求管理、Scheduler、KV Cache  |
| GPU Worker  | 模型加载、Forward、GPU 执行      |

### 5.2 API Server 与 Engine Core 如何通信？

这里重点看两个方向。

方向一：API Server → Engine Core

ROUTER / DEALER

```
API Server
    |
  ROUTER
    |
    | ZMQ
    |
  DEALER
    |
Engine Core
    |
Scheduler
```

API Server 通过 ROUTER 发送请求，Engine Core 通过 DEALER 接收请求。

请求可能包括：

- Add Request
- Abort Request
- Scheduler 相关控制请求
- 其他 Engine Core 控制消息

方向二：Engine Core → API Server

PUSH / PULL

```
Engine Core
    |
   PUSH
    |
    | ZMQ
    |
   PULL
    |
API Server
    |
输出处理
    |
HTTP Streaming
    |
   Client
```

Engine Core 将执行结果发送给 API Server。

例如：

- 新生成的 Token。
- 请求完成状态。
- 执行结果。
- 其他 Engine 输出事件。

注意：这里是 vLLM V1 的典型本地多进程通信关系，不应将所有 vLLM 版本、所有部署模式都视为完全相同的拓扑。

### 5.3 从源码理解 ZMQ

重点阅读：

`vllm/v1/engine/core_client.py`

其中 `MPClient` 是多进程 Engine Core 的客户端通信实现。

源码中可以看到：

```
sync_ctx = zmq.Context(io_threads=2)self.input_socket = make_zmq_socket(    self.ctx,    addresses.inputs[0],    zmq.ROUTER,    bind=True,)self.resources.output_socket = make_zmq_socket(    self.ctx,    addresses.outputs[0],    zmq.PULL,)
```

这里做了两件事：

1. 创建 ZMQ Context。
2. 创建两个 Socket，分别负责输入和输出。

对应的 Engine Core 端使用匹配的 DEALER 和 PUSH Socket。

接下来，重点看请求发送：

```
def _send_input(    self,    request_type,    request,):    msg = (        self.core_engine,        request_type.value,        *self.encoder.encode(request),    )    self.input_socket.send_multipart(        msg,        copy=False,    )
```

这个接口非常值得理解。

消息结构：

```
+------------------+
| Engine Identity  |
+------------------+
| Request Type     |
+------------------+
| Serialized Data  |
+------------------+
```

解释：

- `core_engine`：目标 Engine Core 的身份标识。
- `request_type`：请求类型。
- `encoder.encode()`：对请求进行序列化。
- `send_multipart()`：以多帧形式发送消息。
- `copy=False`：尽可能避免发送过程中的额外数据复制。

为什么使用 ROUTER？

因为 ROUTER 可以根据 Identity 将消息路由到指定的 Engine Core，适合多 Engine Core 的场景。

为什么使用 DEALER？

因为 DEALER 支持异步通信，不要求严格的一问一答。

为什么使用 PUSH/PULL？

因为输出通道是 Engine Core 向 API Server 单向推送结果，PUSH/PULL 正好符合这个模型。

源码参考：[vLLM core_client.py](https://github.com/vllm-project/vllm/blob/main/vllm/v1/engine/core_client.py?utm_source=chatgpt.com)。

[image](https://www.google.com/s2/favicons?domain=https://github.com\&sz=32)

GitHub+1



### 5.4 vLLM 的通信不只有 ZMQ

需要特别注意：

ZMQ 负责消息通信，不代表 vLLM 内部所有数据都通过 ZMQ 传输。

vLLM 根据不同数据类型和通信场景，采用不同机制：

| 通信机制                         | 主要用途                             |
| ---------------------------- | -------------------------------- |
| ZMQ                          | API Server 与 Engine Core 之间的消息通信 |
| Python Multiprocessing       | 进程创建、进程管理、部分控制通信                 |
| Shared Memory / Tensor Queue | 部分跨进程数据共享                        |
| NCCL                         | GPU 之间的集合通信                      |
| CUDA                         | GPU 计算与显存访问                      |

例如：

- 请求元信息可以通过 ZMQ 传递。
- GPU Worker 的模型计算由 GPU 执行。
- Tensor Parallel 场景下，GPU 间的张量通信主要由 NCCL 等机制完成。

这也是为什么不能把 ZMQ 理解成 vLLM 的 GPU 通信框架。

## 6. ZMQ 的核心优势与局限

### 6.1 优势

| 特性     | 解释                    | 对 vLLM 的价值       |
| ------ | --------------------- | ---------------- |
| 异步通信   | 发送和接收不需要同步执行          | 避免组件相互等待         |
| 多种通信模式 | REQ/REP、PUB/SUB 等     | 不同业务选择不同通信模型     |
| 多传输协议  | IPC、TCP、inproc        | 适配不同部署方式         |
| 内部消息队列 | 缓冲待发送和接收消息            | 解耦生产者和消费者        |
| 自动重连   | 连接中断后可自动尝试恢复          | 提高连接层面的弹性        |
| 消息路由   | ROUTER 根据 Identity 路由 | 支持多个 Engine Core |
| 多帧消息   | 一个消息包含多个 Frame        | 方便传递结构化消息        |
| 低复制开销  | 支持消息缓冲区和零拷贝接口         | 降低部分数据传输开销       |

### 6.2 局限

ZMQ 并不是万能的消息系统。

| 局限            | 说明                          |
| ------------- | --------------------------- |
| 不持久化          | 默认不提供磁盘消息存储                 |
| 不保证业务处理成功     | send 成功不代表对端已经处理            |
| 队列容量有限        | HWM 限制缓冲区大小                 |
| 不提供完整事务       | 不提供 Kafka 式消费提交机制           |
| Socket 线程安全限制 | 普通 ZMQ Socket 通常不能被多个线程并发使用 |
| 消息丢失风险        | 进程异常退出时，尚未处理的消息可能丢失         |

尤其要记住：

自动重连 ≠ 消息可靠投递。

如果 Engine Core 崩溃，ZMQ 可以感知连接异常，但不能自动恢复之前未完成的推理请求。这需要 vLLM 自身的进程管理和异常处理机制。

[image](https://www.google.com/s2/favicons?domain=https://zguide.zeromq.org\&sz=32)

ØMQ - The Guide+1

## 7. ZMQ 与其他通信技术的区别

| 对比项         | ZMQ    | TCP Socket | Kafka             | gRPC      |
| ----------- | ------ | ---------- | ----------------- | --------- |
| 定位          | 消息通信库  | 网络传输协议接口   | 分布式消息系统           | RPC 框架    |
| 是否需要 Broker | 不需要    | 不需要        | 需要集群服务            | 不需要       |
| 消息队列        | 内存队列   | 需要自己实现     | 持久化队列             | 不以消息队列为核心 |
| 通信模式        | 多种内置模式 | 自行实现       | Producer/Consumer | RPC       |
| 异步通信        | 原生支持   | 自行实现       | 原生支持              | 支持        |
| 持久化         | 无      | 无          | 有                 | 无         |
| 典型场景        | 进程间通信  | 底层网络通信     | 分布式事件流            | 服务间 RPC   |

我的理解：

- TCP：解决数据如何通过网络传输。
- ZMQ：解决不同组件之间如何组织消息通信。
- Kafka：解决分布式消息的持久化、缓冲和消费。
- gRPC：解决服务之间如何定义和调用远程接口。

## 8. 建议掌握的 ZMQ 知识体系

如果你是为了深入理解 vLLM 源码，不需要一开始就研究 ZMQ 的所有底层实现。

建议按下面的顺序学习：

1. 第一阶段：核心概念
   - Context / Socket
   - bind / connect
   - IPC / TCP / inproc
   - Message / Multipart
2. 第二阶段：通信模式
   - ROUTER / DEALER
   - PUSH / PULL
   - PUB / SUB
   - 理解 Identity、消息路由、负载分发
3. 第三阶段：内部机制
   - I/O Thread
   - Message Queue
   - HWM
   - Poller
   - 自动重连与异常处理
   - Zero-copy
4. 第四阶段：结合 vLLM 源码
   - `vllm/v1/engine/core_client.py`
   - `vllm/v1/engine/core.py`
   - `vllm/v1/engine/utils.py`
   - `vllm/v1/executor/multiproc_executor.py`

### 9. 推荐阅读资料

1. [ØMQ Guide - Sockets and Patterns](https://zguide.zeromq.org/docs/chapter2/?utm_source=chatgpt.com)：重点看通信模式、Socket、异步 I/O。
2. [ZeroMQ Socket API](https://zeromq.org/socket-api/?utm_source=chatgpt.com)：查阅各种 Socket 的具体行为。
3. [vLLM Architecture Overview](https://docs.vllm.ai/en/latest/design/arch_overview/?utm_source=chatgpt.com)：理解 vLLM V1 多进程架构。
4. [vLLM core_client.py](https://github.com/vllm-project/vllm/blob/main/vllm/v1/engine/core_client.py?utm_source=chatgpt.com)：重点研究 ZMQ 的实际代码实现。

最后总结：

我建议你重点记住这三个层次：

```
第一层：底层传输
TCP / IPC / inproc

第二层：ZMQ 通信抽象
Context / Socket / Queue / Message

第三层：通信模式
ROUTER-DEALER：异步路由
PUSH-PULL：任务分发
PUB-SUB：事件广播
```

在 vLLM 中，ZMQ 的主要价值是将 API Server 和 Engine Core 解耦，使请求提交、推理结果返回以及多 Engine Core 路由可以通过统一的异步消息机制实现。

最值得深入阅读的是 `core_client.py` 中的 `MPClient`，它是理解 vLLM 如何使用 ZMQ 的一个很好的入口。
