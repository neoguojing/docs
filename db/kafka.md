# Apache Kafka 底层机制技术文档

> 本文档基于 Apache Kafka 3.x/4.x 版本架构编写，涵盖 KRaft（Kafka Raft Metadata）模式等最新特性。实际配置请参照官方文档和具体版本说明。

---

## 1. 定位与数据模型

### 1.1 它是什么？

Apache Kafka 是一个**分布式流式数据处理平台**，最初由 LinkedIn 开发并于 2011 年开源，2012 年成为 Apache 顶级项目。它的核心定位经历了从「高性能消息队列」到「分布式流式平台」再到「AI 数据总线」的演进：

- **消息队列（Message Queue）**：解耦生产者和消费者，支持异步通信和缓冲。
- **分布式流式平台（Distributed Streaming Platform）**：提供发布/订阅消息流、持久化存储、流式数据处理能力。
- **AI 数据总线（AI Data Bus）**：Kafka 4.0+ 面向 AI 场景，支持流式优先湖仓（Kafka + Iceberg/Hudi）、实时特征管道及模型上下文协议（MCP）集成。

### 1.2 解决什么问题？

| 问题场景 | Kafka 的解决方案 |
|---------|----------------|
| 系统间异步解耦 | 生产者写入 Topic，消费者独立拉取，无需直接依赖 |
| 流量削峰填谷 | 消息持久化在磁盘，消费者按自身速率消费，不受生产者峰值影响 |
| 日志/事件采集 | 统一接入点采集各系统日志、埋点事件，持久化后供下游分析 |
| 实时流式计算 | 与 Flink/Spark Streaming 等引擎集成，实现实时数据管道 |
| 数据同步与备份 | 通过 MirrorMaker/CDC 实现跨机房/云的数据同步 |
| 审计与合规 | 所有事件持久化存储，支持任意时间点回放和审计 |

### 1.3 核心数据模型

Kafka 的数据模型围绕以下几个核心概念构建：

- **Topic（主题）**：消息的逻辑分类，生产者向 Topic 发布消息，消费者从 Topic 订阅消息。
- **Partition（分区）**：Topic 的物理拆分单元，每个 Partition 是一个有序的、不可变的顺序日志（Append-Only Log）。Partition 可分布在不同 Broker 上，实现水平扩展。
- **Message（消息）**：由 Key（可选）、Value、Timestamp、Headers 等字段组成，按分区内严格有序。
- **Offset（偏移量）**：消息在 Partition 中的唯一整数标识，自增且不可变。
- **Broker（节点）**：Kafka 集群中的服务器节点，负责消息的存储和转发。
- **Producer（生产者）**：向 Topic 发布消息的客户端。
- **Consumer（消费者）**：从 Topic 订阅并处理消息的客户端，以 Consumer Group 形式组织。
- **Consumer Group（消费者组）**：一组消费者共享一个 Group ID，组内每条消息仅被一个消费者处理，实现负载均衡。

---

## 2. 存储引擎

### 2.1 数据组织方式

Kafka 的存储引擎基于**日志结构（Log-Structured）**设计，核心思想是「顺序追加、分段存储、按段清理」：

```
Topic
 └── Partition-0  (Broker-A)
 │    ├── 00000000000000000000.log    (Segment 1: Offset 0~999999)
 │    ├── 00000000000000000000.index  (偏移量稀疏索引)
 │    ├── 00000000000000000000.timeindex (时间戳索引)
 │    ├── 00000000000000010000.log    (Segment 2: Offset 1000000~...)
 │    └── ...
 └── Partition-1  (Broker-B)
      └── ...
```

- **Log Segment（日志段）**：每个 Partition 由多个 Log Segment 文件组成，每个 Segment 包含三个文件：
  - `.log`：存储消息的实际数据（二进制格式）。
  - `.index`：**偏移量稀疏索引**，记录 Offset 到该消息在 .log 文件中物理位置（字节偏移）的映射。索引文件是稀疏的（默认每隔 4KB 写入一个索引项），通过二分查找定位消息。
  - `.timeindex`：**时间戳索引**，记录 Timestamp 到 Offset 的映射，支持按时间范围查询消息。

### 2.2 Log Segment 文件结构详解

每个 Segment 由三个同名文件组成，共享同一个**基准偏移量（Base Offset）**作为文件名前缀。下面以一个具体场景来拆解这三个文件的**真实物理结构**。

**场景假设**：当前有一个 Topic 的 Partition-0，它的一个 Log Segment 的基准偏移量（Base Offset）为 `1000`。这意味着这个 Segment 的第一条消息的全局 Offset 是 1000。

在物理磁盘上，这个 Segment 由以下三个文件组成：

```
my-topic-0/
 ├── 00000000000000001000.log          # 消息数据文件 (实际内容)
 ├── 00000000000000001000.index        # 偏移量稀疏索引
 └── 00000000000000001000.timeindex    # 时间戳索引
```

#### 2.2.1 .log 文件结构（实际数据）

`.log` 文件是纯二进制文件，消息并不是单条独立存储的，而是以 **RecordBatch（消息批次）** 为单位进行顺序追加。

**文本图示例：**

```
[RecordBatch-1]
  ├── Header: [baseOffset=1000, lastOffset=1002, count=3, crc, compressCodec...]
  ├── Record: [offset=1000, key="A", value="Hello"]
  ├── Record: [offset=1001, key="B", value="Kafka"]
  └── Record: [offset=1002, key="C", value="World"]

[RecordBatch-2]
  ├── Header: [baseOffset=1003, lastOffset=1004, count=2, crc, compressCodec...]
  ├── Record: [offset=1003, key="D", value="Test"]
  └── Record: [offset=1004, key="E", value="Data"]
```

**说明**：Kafka 通过批次机制减少网络请求和磁盘 I/O。物理上，这些 RecordBatch 紧密相连，顺序排列。每个 Batch 的 Header 包含 baseOffset（批次起始偏移量）、lastOffset（批次结束偏移量）、count（消息条数）、crc（校验和）、compressCodec（压缩编码）等固定字节的头部信息。

#### 2.2.2 .index 文件结构（偏移量稀疏索引）

`.index` 文件也是纯二进制文件，它**不为每一条消息建立索引**，而是采用**稀疏索引**机制（默认每写入 4KB 消息数据，才记录一条索引）。

它的内部是由固定长度的键值对组成的数组，每个条目固定为 **8 字节**：

- **前 4 字节**：相对偏移量（Relative Offset）= `消息的全局 Offset - Base Offset`
- **后 4 字节**：物理位置（Position）= 该消息在 `.log` 文件中的字节偏移量

**文本图示例：**

```
00000000000000001000.index 内容：

+---------------------+---------------------+
| Relative Offset (4B)| Position in .log(4B)|
+---------------------+---------------------+
| 00000000 (即 0)     | 00000000 (即 0)     | <-- 第 0 条索引 (Base Offset处)
+---------------------+---------------------+
| 00000018 (即 24)    | 00000400 (即 1024)  | <-- 第 1 条索引 (间隔4KB触发)
+---------------------+---------------------+
| 00000030 (即 48)    | 00000800 (即 2048)  | <-- 第 2 条索引 (间隔4KB触发)
+---------------------+---------------------+
```

**查找示例：如果要找全局 Offset = 1025 的消息。**

1. 算出相对偏移量：1025 - 1000 = 25。
2. 在 `.index` 中二分查找，找到**小于等于 25 的最大值**，即 `24`。
3. 获取对应的物理位置：`1024`。
4. 去 `.log` 文件的 `1024` 字节处开始**顺序向后扫描**，直到找到 Offset=1025 的消息。

#### 2.2.3 .timeindex 文件结构（时间戳索引）

`.timeindex` 同样是稀疏索引，结构与 `.index` 类似，但键值对的含义不同。每个条目固定为 **12 字节**：

- **前 8 字节**：时间戳（Timestamp）
- **后 4 字节**：相对偏移量（Relative Offset）

**文本图示例：**

```
00000000000000001000.timeindex 内容：

+-------------------------+---------------------+
| Timestamp (8B)          | Relative Offset(4B)|
+-------------------------+---------------------+
| 1690000000000           | 00000000 (即 0)     | <-- 第 0 条索引
+-------------------------+---------------------+
| 1690000060000           | 00000018 (即 24)    | <-- 1分钟后触发新索引
+-------------------------+---------------------+
| 1690000120000           | 00000030 (即 48)    | <-- 2分钟后触发新索引
+-------------------------+---------------------+
```

**查找示例：如果要找时间戳 >= `1690000070000` 的消息。**

1. 在 `.timeindex` 中二分查找，找到**小于等于目标时间戳的最大值**，即 `1690000060000`。
2. 获取对应的相对偏移量：`24`。
3. 拿着相对偏移量 `24`，去 `.index` 文件中查找对应的物理位置（如 `1024`）。
4. 去 `.log` 文件的 `1024` 字节处开始**顺序向后扫描**，直到找到时间戳 >= `1690000070000` 的消息。

### 2.3 三个文件如何配合（完整检索举例）

`.log`、`.index` 和 `.timeindex` 并不是孤立存在的，它们通过**相对偏移量（Relative Offset）**紧密串联。当 Kafka 需要检索消息时，会根据查询条件（按 Offset 或按时间戳）进行多级联动查找。

**场景举例：按时间戳查找消息**

假设消费者请求获取时间戳 `T = 1690000070000` 之后的第一条消息。

**第一步：定位时间戳索引（`.timeindex`）**

Kafka 在 `.timeindex` 文件中进行二分查找，寻找"小于等于目标时间戳"的最大索引项。

- 找到索引项：`Timestamp = 1690000060000`，`Relative Offset = 24`。
- 此时，Kafka 知道了目标消息的相对偏移量至少是 `24`。

**第二步：定位物理地址（`.index`）**

Kafka 拿着上一步得到的 `Relative Offset = 24`，去 `.index` 文件中进行二分查找，寻找"小于等于 24"的最大索引项。

- 找到索引项：`Relative Offset = 24`，`Position = 1024`。
- 此时，Kafka 知道了目标消息在 `.log` 文件中的物理起始字节位置是 `1024`。

**第三步：顺序扫描数据（`.log`）**

Kafka 直接跳转到 `.log` 文件的 `1024` 字节处，开始向后顺序读取消息，直到找到时间戳真正 `>= 1690000070000` 的消息并返回。

这种设计将原本需要遍历海量数据的"顺序查找"，优化为了"**两次二分查找 + 极少量的顺序扫描**"，极大地兼顾了检索效率与磁盘空间。

### 2.4 写入流程

1. 生产者将消息发送到指定 Topic 的某个 Partition 的 Leader 副本。
2. Broker 将消息**顺序追加**写入操作系统 Page Cache（页缓存），同时更新 Partition 的 LEO（Log End Offset）。
3. 消息数据通过 **sendfile 零拷贝**机制直接发送给 Follower 副本进行同步复制。
4. 根据 `acks` 配置，Producer 等待 ISR 副本确认后才返回成功。
5. OS 内核根据页面置换策略（如 LRU）**异步刷盘**至物理磁盘。

### 2.5 落盘与刷盘策略

- **默认策略**：依赖 OS 异步刷写，不主动调用 `fsync`，刷盘频率由 `os.pagecache` 和 `flush.messages`/`flush.ms` 配置控制。
- **强制刷盘**：配置 `log.flush.interval.messages`（每 N 条消息刷盘）或 `log.flush.interval.ms`（每 N 毫秒刷盘），但**生产环境不建议开启**，因为会严重降低吞吐量。可靠性主要依赖副本机制保障。
- **Segment 滚动（Rolling）**：新 Segment 生成的触发条件（满足任一即触发）：
  - `log.segment.bytes`（默认 1GB）：当前 Segment 文件大小达到阈值。
  - `log.segment.ms`（默认 7 天 = 604800000ms）：当前 Segment 创建时间超过阈值。
  - `log.roll.{hours,milliseconds,jitter}`：基于时间的滚动（含随机抖动避免全部分区同时滚动）。

### 2.6 日志清理策略

| 策略 | 配置参数 | 说明 | 适用场景 |
|------|---------|------|---------|
| **Delete（删除）** | `log.retention.{hours/minutes/bytes}` | 基于时间或大小过期删除旧 Segment | 日志采集、监控数据等允许丢弃旧数据的场景 |
| **Compact（压缩）** | `log.cleanup.policy=compact` | 保留每个 Key 的最新版本，删除同一 Key 的旧版本 | 状态存储、变更事件流等需要保留最新状态的场景 |
| **Compact + Delete** | `log.cleanup.policy=compact,delete` | 先压缩再按保留策略删除 | 兼顾状态保留和过期清理 |

---

## 3. 内存与数据结构

### 3.1 JVM 堆内内存

Kafka Broker 基于 JVM 运行，堆内内存主要用于：

- **网络层**：Netty 的 DirectBuffer（堆外内存）用于网络 I/O 缓冲，不占用堆内存，避免 GC 影响。
- **请求/响应对象**：序列化后的请求和响应对象。
- **元数据缓存**：本地维护的 Topic、Partition、Broker 元数据信息。
- **生产者缓冲区**：`buffer.memory`（默认 32MB）用于缓存待发送消息。
- **消费者缓冲区**：`fetch.max.bytes` 控制单次拉取最大字节数。

### 3.2 操作系统页缓存（Page Cache）——核心加速手段

**Page Cache（页缓存）是 Linux 内核中用于缓存文件系统数据的内存管理机制。** 它的核心思想是：将尽可能多的空闲物理内存用作磁盘缓存，以弥合 CPU/内存与磁盘 I/O 之间的巨大速度差异。

**核心机制：**

1. **写入缓冲（延迟写回）**：当应用程序发起写请求时，数据并不会立即写入物理磁盘，而是先写入 Page Cache，并将对应的内存页标记为"脏页（Dirty Page）"。系统调用立即返回，随后由操作系统内核的后台线程在合适的时机异步将脏页批量刷回磁盘。
2. **读取预读（Read-Ahead）**：当发生读请求时，内核先检查 Page Cache 中是否存在该数据。若命中（Cache Hit），直接从内存返回；若未命中（Cache Miss），则从磁盘加载数据到 Page Cache，并基于局部性原理提前预读相邻的数据页，以加速后续的连续读取。

**Kafka 是如何使用 Page Cache 的？**

Kafka 放弃了在 JVM 堆内存中维护消息缓存的传统做法，而是将数据缓存全权交由操作系统的 Page Cache 管理。这种设计带来了三大核心优势：

1. **写入极速化**：Producer 发送消息到 Broker 时，数据直接写入 Page Cache 即视为写入成功并立即返回，将耗时的磁盘 I/O 转化为了接近内存速度的操作。
2. **零 GC 压力**：由于消息数据不存储在 JVM 堆内，Kafka 彻底避免了海量对象创建带来的 JVM GC（垃圾回收）停顿问题，保证了高吞吐下的延迟稳定性。
3. **读写缓存复用**：当 Consumer 消费消息时，优先从 Page Cache 读取。如果生产与消费速率相近，整个链路几乎完全在内存中完成，物理磁盘的访问极少。

**Kafka 调用了哪些系统调用？**

Kafka 对 Page Cache 的深度依赖，体现在它对底层系统调用的精准选择上：

- **生产者写入消息：调用 `pwrite()`**

  当 Producer 发送消息时，Broker 使用 `pwrite()` 系统调用将数据写入磁盘。由于操作系统的 Page Cache 机制，`pwrite()` 实际上只是将数据写入了内存中的页缓存，并标记为脏页，随后由 OS 异步刷盘。

- **消费者读取消息：调用 `sendfile()`**

  当 Consumer 拉取消息时，Kafka 使用了 `sendfile()` 系统调用实现"零拷贝（Zero-Copy）"。

  - **传统读取路径**（4次拷贝，2次上下文切换）：
    ```
    磁盘 → Page Cache(内核) → 用户态缓冲区 → Socket Buffer(内核) → NIC Buffer → 网络
    ```
  - **sendfile 优化路径**（2次DMA拷贝，2次上下文切换）：
    ```
    磁盘 → Page Cache(内核) → Socket Buffer(内核) → NIC Buffer → 网络
    ```

  数据直接从内核态的 Page Cache 传输到 Socket Buffer（配合 DMA 控制器发往网卡），完全跳过了用户空间，实现了 0 次 CPU 拷贝和极低的上下文切换开销。

- **索引文件检索：调用 `mmap()`**

  对于 `.index` 和 `.timeindex` 这种需要频繁进行随机二分查找的小文件，Kafka 使用了 `mmap()`（内存映射）系统调用。`mmap()` 将索引文件直接映射到进程的虚拟地址空间。Kafka 可以像访问普通内存数组一样直接读取索引数据，无需频繁触发 `read()` 等系统调用，极大地提升了随机检索的效率。

### 3.3 NIC Buffer 是什么？

**NIC Buffer（Network Interface Card Buffer，网卡缓冲区）** 是位于物理网卡硬件上的内存。当内核通过 DMA（直接内存访问）将数据从 Page Cache 或 Socket Buffer 转移到网卡时，数据会暂存在 NIC Buffer 中，等待网卡将其序列化为电信号或光信号发送到物理网络上。

在零拷贝（sendfile + SG-DMA）场景下，数据最终就是由 DMA 控制器直接写入 NIC Buffer 并发出的。整个数据路径可以概括为：

```
Page Cache(内核内存) → Socket Buffer(内核内存) → DMA控制器 → NIC Buffer(网卡硬件内存) → 物理网络
```

理解 NIC Buffer 有助于理解为什么零拷贝能节省 CPU：因为从 Page Cache 到 NIC Buffer 的整段数据搬运完全由 DMA 控制器硬件完成，不需要 CPU 逐字节拷贝。

### 3.4 Reactor 网络模型与异步发送

Kafka Producer 的高吞吐不仅依赖于攒批，更得益于其底层的**双线程异步模型（Reactor 模式）**：

- **业务线程解耦**：当业务代码调用 `send()` 时，消息仅经过序列化和分区路由，随后被迅速追加到内存中的 `RecordAccumulator`（记录累加器）对应的 Batch 中。此过程完全不触碰网络 I/O，业务线程几乎无阻塞。
- **Sender 线程（IO 线程）**：后台独立的 Sender 线程通过 NIO 多路复用（Selector）统一管理所有网络连接。它负责从累加器中拉取满足条件（大小达到 `batch.size` 或时间达到 `linger.ms`）的 Batch，打包成 Request 批量发送。
- **异步回调**：发送完成后，Sender 线程根据 Broker 的响应结果触发用户注册的 Callback 接口，通知业务方消息是否成功。

这种"生产数据"与"传输数据"在时间和线程上的彻底隔离，使得 Producer 能够用有限的内存撑起极高的并发吞吐。

**图解异步发送流程：**

```
业务线程                          Sender线程(IO)
    |                                  |
    |-- send(msg) --------------------->|
    |-- 序列化 + 分区路由                |-- 追加到 RecordAccumulator Batch
    |-- 立即返回(不等待网络响应)          |-- 定时/定量触发批量发送
    |                                  |-- NIO Selector 多路复用发送
    |                                  |-- 收到Broker响应
    |<-- Callback(msg, exception) -------|  触发异步回调通知结果
```

### 3.5 内存映射文件（mmap）

- Kafka 的索引文件（`.index` 和 `.timeindex`）采用 **mmap（内存映射文件）** 方式访问，将文件直接映射到内核虚拟地址空间。
- 读取索引时，OS 负责将文件页加载到 Page Cache，应用层无需显式 `read()` 系统调用，减少用户态与内核态的数据拷贝。
- 索引文件设计为稀疏且大小有限（默认最大 1GB），mmap 的内存开销可控。

### 3.6 其他内存优化

- **对象池化**：内部维护 ByteBuffer 对象池复用，减少对象创建和 GC 压力。
- **批量处理**：生产者按 `batch.size`（默认 16KB）和 `linger.ms`（默认 0ms）累积消息为批次发送，减少网络请求数和磁盘 I/O 次数，同时提升压缩比。
- **数据压缩**：支持 gzip、snappy、lz4、zstd 等算法，在批次级别压缩，降低网络带宽和磁盘存储开销。

---

## 4. 索引与查询

### 4.1 偏移量索引（Offset Index）

- **结构**：稀疏索引文件（`.index`），每个索引项包含两个字段：`relative offset`（相对于 Segment 起始 offset 的偏移）和 `position`（该消息在 .log 文件中的字节偏移量）。
- **查找流程**：
  1. 消费者请求从 Offset X 开始消费。
  2. 定位 X 所属的 Segment（通过 Segment 文件名二分查找）。
  3. 在该 Segment 的 .index 文件中**二分查找**最接近 X 的索引项。
  4. 从 .log 文件中该索引项对应的位置开始**顺序扫描**，找到第一条 Offset >= X 的消息。
- **稀疏性**：默认每隔 4KB 写入一个索引项，索引文件远小于 .log 文件，可完全 mmap 到内存。

### 4.2 时间戳索引（Timestamp Index）

- **结构**：稀疏索引文件（`.timeindex`），每个索引项包含 `timestamp` 和 `offset`。
- **用途**：支持按时间范围查询消息（如 `kafka-console-consumer --from-beginning --property timestamp=xxx`）。
- **查找流程**：二分查找最接近目标时间戳的索引项，然后从对应 offset 开始顺序扫描。

### 4.3 查询执行流程

Kafka 的「查询」本质上是**基于 offset 的顺序读取**，执行流程如下：

1. **Producer 写入**：消息追加到 Partition Leader 的当前 Segment 末尾，更新 LEO，异步刷盘。
2. **Consumer 拉取**：
   - Consumer 携带 `offset` 发送 Fetch 请求到 Leader。
   - Leader 校验 offset 合法性，定位对应 Segment 和索引文件。
   - 通过索引文件快速定位消息位置，从 .log 文件读取数据。
   - 通过 sendfile 零拷贝将数据发送给 Consumer。
3. **Follower 同步**：Follower 通过 ReplicaFetcherThread 持续向 Leader 发送 Fetch 请求拉取数据，写入本地日志并更新 LEO。

### 4.4 高效查询的关键设计

| 设计 | 效果 |
|------|------|
| 顺序追加写 | 避免磁盘寻道，机械盘顺序写可达 100MB/s+ |
| 稀疏索引 + mmap | 索引常驻内存，O(log n) 定位，无磁盘 I/O |
| 零拷贝传输 | 消除用户态拷贝，CPU 占用降低 30%+ |
| 页缓存预热 | 热点数据常驻内存，读操作几乎不碰磁盘 |
| 批量拉取 | 减少网络 RTT 和磁盘 I/O 次数 |

### 4.5 LEO（Log End Offset）详解

**LEO（Log End Offset）** 即日志末端位移，代表当前分区下一条待写入消息的 Offset。它是分区内消息连续写入位置的逻辑指针。

**LEO 的核心用途：**

1. **消费延迟计算**：Consumer 提交的 CURRENT-OFFSET 与 Leader 的 LEO 相减，即可得出消费延迟（LAG）。
   ```
   LAG = Leader_LEO - Consumer_Current_Offset
   ```
2. **副本同步监控**：Follower 的 LEO 与 Leader 的 LEO 之差反映了副本同步延迟。
3. **HW（高水位）计算**：HW = min(所有 ISR 副本的 LEO)，只有 Offset < HW 的消息才被视为"已提交"。

---

## 5. 并发与一致性

### 5.1 并发模型

- **分区级无锁设计**：单 Partition 同一时刻仅由一个生产者线程写入、一个消费者线程消费，天然避免锁竞争。不同 Partition 之间完全并行，实现水平扩展。
- **Reactor 网络模型**：Broker 采用多 Reactor 线程模型，IO 线程负责网络读写，业务线程负责消息处理，IO 与业务分离。
- **异步发送**：Producer 发送消息不阻塞，通过异步回调机制通知结果，配合批量发送提升吞吐。

### 5.2 副本机制与 ISR

- **Leader-Follower 架构**：每个 Partition 有一个 Leader 副本处理所有读写请求，多个 Follower 副本从 Leader 拉取数据同步。
- **ISR（In-Sync Replicas）**：与 Leader 保持同步的副本集合（包含 Leader 自身）。只有 ISR 中的副本才参与消息提交确认和 Leader 选举。
- **ISR 动态管理**：
  - **收缩**：Follower 在 `replica.lag.time.max.ms`（默认 30s）内未发送 Fetch 请求，或 LEO 落后过多，被移出 ISR。
  - **扩张**：被踢出的 Follower 持续拉取数据，当 LEO 追上 Leader 时重新加入 ISR。

### 5.3 高水位（HW）与消息可见性

- **LEO（Log End Offset）**：副本日志中下一条待写入消息的 offset。
- **HW（High Watermark）**：Leader 上 HW = min(所有 ISR 副本的 LEO)。只有 Offset < HW 的消息才被视为「已提交」并对消费者可见。
- **作用**：HW 确保消费者不会读到未同步到 Follower 的「未提交」消息，保证副本间数据一致性。

**HW 工作原理举例：**

```
假设 Partition 有 3 个副本 (Leader=A, Follower=B, Follower=C)

当前状态:
  Leader(A) LEO = 1000
  Follower(B) LEO = 980
  Follower(C) LEO = 950

HW = min(1000, 980, 950) = 950

含义:
  - Offset 0~949 的消息已同步到所有 ISR 副本，对消费者可见
  - Offset 950~999 的消息仅部分副本已写入，消费者不可见
  - 如果 Leader 在 Offset 999 处崩溃，Offset 950~999 的消息将永久丢失
```

### 5.4 生产者确认机制（acks）

| acks 值 | 行为 | 数据安全性 | 吞吐量 |
|---------|------|-----------|--------|
| `acks=0` | 生产者不等待任何确认，直接发送 | 可能丢失（网络故障/Broker 宕机） | 最高 |
| `acks=1` | 仅 Leader 写入成功即确认 | Leader 故障可能丢数据（Follower 未同步） | 高 |
| `acks=all`（或 `-1`） | 等待 ISR 中所有副本写入成功才确认 | 数据不丢失（需 `min.insync.replicas >= 2`） | 较低 |

- **min.insync.replicas**：配置 `acks=all` 时有效，定义写入成功所需的最小 ISR 副本数（生产建议设为 2）。若可用 ISR 副本数 < 此值，写入被拒绝。
- **unclean.leader.election.enable**：默认 `false`，禁止非 ISR 副本当选 Leader，防止数据丢失。设为 `true` 可在 Leader 全部失效时以牺牲一致性为代价换取可用性。

### 5.5 幂等性（Idempotent Producer）

- **原理**：通过 Producer ID（PID）+ 序列号（Sequence Number）实现单生产者单分区的精确一次语义。
- **Broker 端校验**：
  - 序列号 == last_sn + 1 → 正常接收
  - 序列号 <= last_sn → 重复消息，丢弃
  - 序列号 > last_sn + 1 → 拒绝（可能数据乱序）
- **启用条件**：`enable.idempotence=true`（Kafka 2.0+ 默认开启），需配合 `acks=all`、`retries > 0` 及 `max.in.flight.requests.per.connection <= 5`。

### 5.6 事务机制（Transaction）

- **解决跨分区原子性**：保证跨多个 Partition 的写入要么全部成功，要么全部回滚。
- **事务协调器**：通过 `transactional.id` 标识生产者，分配 PID 和 Epoch，保证跨会话幂等。
- **事务消息**：事务内消息写入后标记为未提交，消费者默认不可见；提交时写入 COMMIT Marker，回滚时写入 ABORT Marker。
- **流处理端到端一致性**：支持 `sendOffsetsToTransaction` 将消费位移纳入事务，实现「读-处理-写」的原子操作。
- **消费者隔离级别**：
  - `READ_UNCOMMITTED`（默认）：可读取所有消息（包括未提交的）。
  - `READ_COMMITTED`：仅读取已提交事务消息和非事务消息，过滤未提交及回滚消息。

### 5.7 消息投递语义总结

| 语义 | 配置组合 | 特点 |
|------|---------|------|
| At-Most-Once（至多一次） | `acks=0` | 允许丢失，不重复 |
| At-Least-Once（至少一次） | `acks=all` + 手动管理 offset | 保证不丢，可能重复 |
| Exactly-Once（精确一次） | 幂等性 + 事务 + `READ_COMMITTED` | 不丢不重 |

---

## 6. 分布式与高可用

### 6.1 元数据管理：从 ZooKeeper 到 KRaft

| 维度 | ZooKeeper 模式（传统） | KRaft 模式（Kafka 3.5+/4.0） |
|------|----------------------|----------------------------|
| 元数据存放 | 外部 ZooKeeper 集群的树状结构 | Kafka 内部 Topic `__cluster_metadata` |
| 共识协议 | ZAB（ZooKeeper Atomic Broadcast） | Raft 共识协议 |
| 选举机制 | ZK 临时有序节点 | Raft Leader 选举（含 Pre-Vote 预选举） |
| 变更通知 | Watcher 机制 | Raft 日志复制 + 状态机应用 |
| 性能 | 写操作延迟 5~15ms，规模受限 | 写操作延迟 1~3ms，支持百万级分区 |
| 部署复杂度 | 需额外维护 ZK 集群 | 自包含，无需外部依赖 |

- **KRaft 节点角色**：
  - **Controller（Voter）**：组成 Raft 共识集群，负责元数据写入、共识、存储及 Leader 选举，不处理业务流量。
  - **Broker（Observer）**：处理业务读写，从 Controller 拉取元数据日志更新本地缓存。
  - **Combined**：同时承担双重角色，仅推荐测试或超小型集群使用。
- **Kafka 4.0** 已完全移除 ZooKeeper 支持代码，KRaft 成为唯一原生架构。

### 6.2 分区副本与 Leader 选举

- **副本分配**：每个 Partition 的副本（AR, Assigned Replicas）可配置分布在不同的 Broker 上，实现故障隔离。
- **Leader 选举策略**：
  - **Clean Leader Election（默认）**：仅从 ISR 中选举新 Leader，保证数据一致性。
  - **Unclean Leader Election**：当 ISR 为空且 `unclean.leader.election.enable=true` 时，允许从非 ISR 副本选举，以牺牲一致性换取可用性。
  - **Preferred Leader Election**：自动/手动将 Leader 切换回优先副本，恢复集群负载均衡。

### 6.3 故障检测与恢复

1. **故障检测**：Broker 通过心跳机制向 Controller 发送心跳，若超过 `session.timeout.ms`（默认 18s）未收到心跳，Controller 判定该 Broker 失效。
2. **Leader 选举**：Controller 从受影响分区的 ISR 中选择新 Leader，广播 `LeaderAndIsrRequest` 更新元数据。
3. **客户端响应**：Producer 和 Consumer 自动感知变更并重连新 Leader。若 `acks=all` 且 ISR 副本数 < `min.insync.replicas`，写入被拒绝。
4. **节点恢复**：宕机 Broker 重启后作为 Follower 加入集群，从当前 Leader 拉取数据同步，追平后重新加入 ISR。

### 6.4 分片与扩容

- **自动分区分配**：Kafka 提供 `RangeAssignor`（默认）、`RoundRobinAssignor`、`StickyAssignor` 等分区分配策略。
- **动态扩容**：新增 Broker 后，通过 `kafka-reassign-partitions.sh` 工具重新分配分区副本，实现数据均衡迁移。
- **跨机房复制**：通过 MirrorMaker 2.0（基于 Streams API）或 Tiered Storage 实现跨数据中心/云的多活和灾备。

### 6.5 分层存储（Tiered Storage）

- **原理**：将冷数据自动迁移至对象存储（如 S3、OSS），Broker 变为无状态节点。
- **价值**：
  - 支持分钟级弹性扩缩容。
  - 读写路径隔离，冷读不拖垮写入。
  - 在合规审计场景（如 50TB 审计日志保留 7 年），数据保留成本可降低约 90%。

---

## 7. 持久化与恢复

### 7.1 持久化机制

Kafka 的持久化设计围绕「顺序写 + 副本复制 + 日志清理」三位一体：

1. **顺序追加写磁盘**：消息严格按 FIFO 顺序追加写入 Partition 日志文件（.log），避免随机 I/O。
2. **副本复制**：Leader 写入成功后立即通过 sendfile 零拷贝将数据发送给 Follower 同步，多副本保证数据不丢失。
3. **CRC32 校验**：消息写入时计算 CRC32 校验和，读取时校验以防止数据损坏。

### 7.2 Crash 后的恢复流程

当 Broker 崩溃重启后，恢复流程如下：

1. **加载元数据**：从 `__cluster_metadata`（KRaft）或 ZooKeeper 加载最新的集群元数据。
2. **重建 Partition 状态**：对每个 Partition 的本地日志文件：
   - 检查每个 Segment 的 `.log` 文件完整性（通过 CRC 校验）。
   - 重建偏移量索引（`.index`）和时间戳索引（`.timeindex`）—— 若索引文件缺失或损坏，从 `.log` 文件重新构建。
   - 截断（Truncate）不完整的 Segment（CRC 校验失败的尾部数据）。
3. **ISR 同步**：重启后的 Broker 作为 Follower，向 Leader 发送 Fetch 请求拉取缺失数据。
4. **加入 ISR**：当 LEO 追上 Leader 后，Leader 将其重新加入 ISR。
5. **恢复消费者位移**：从 `__consumer_offsets` Topic 加载各 Consumer Group 的提交位移。

### 7.3 `__consumer_offsets` Topic 的消息格式

`__consumer_offsets` 是 Kafka 内部用于保存消费者位移和组元数据的特殊 Topic。其消息采用标准的 Key-Value 结构，但格式由 Kafka 内部严格定义，用户不可篡改：

**Key 结构**：包含版本号和标识信息。

- 对于**位移提交消息**，Key 是一个三元组：`[Group ID, Topic Name, Partition Index]`，精确标识某个消费组在特定分区上的位移。
- 对于**组元数据消息**，Key 是一个二元组：`[Version, Group ID]`。

**Value 结构**：包含实际的位移数据或组状态。

- 对于**位移提交消息**，Value 包含：`Offset Value`（位移值）、`Metadata`（自定义元数据）、`Commit Timestamp`（提交时间戳）以及 `Expire Timestamp`（过期时间戳）。
- 对于**组元数据消息**，Value 包含：成员 ID、IP、超时时间、Generation、分配算法、协议类型等组状态信息。

**图解位移提交过程：**

```
Consumer Group "order-group" 消费 Topic "orders" Partition-0

步骤1: Consumer 处理完 Offset=999 的消息
步骤2: Consumer 调用 commitSync() 提交位移
步骤3: Consumer 向 Broker 发送 OffsetCommitRequest
        Key: ["order-group", "orders", 0]
        Value: {offset: 1000, metadata: "", commitTimestamp: 1690000000000, expireTimestamp: ...}
步骤4: Broker 将位移消息写入 __consumer_offsets Topic 的某个 Partition
步骤5: 下次 Consumer 重新消费时，Broker 从 __consumer_offsets 读取最新位移
```

### 7.4 关键恢复配置

| 配置参数 | 默认值 | 说明 |
|---------|--------|------|
| `replica.lag.time.max.ms` | 30000ms | Follower 同步超时阈值，超过此时间未同步将被移出 ISR |
| `unclean.leader.election.enable` | false | 是否允许非 ISR 副本当选 Leader（生产建议 false） |
| `min.insync.replicas` | 1 | acks=all 时写入成功所需最小 ISR 副本数（生产建议 2） |
| `log.retention.hours` | 168 (7天) | 消息保留时间，过期后按清理策略处理 |
| `log.segment.delete.delay.ms` | 60000ms | Segment 标记删除后等待时间，确保 Follower 已拉取 |
| `offsets.topic.replication.factor` | 1（旧版） | `__consumer_offsets` 副本数，生产环境建议设为 3 |

### 7.5 数据安全性保障体系

```
数据不丢失 = 顺序写磁盘 + 多副本复制(ISR) + acks=all + min.insync.replicas>=2 + unclean.leader.election=false
```

- **写入端**：`acks=all` + `retries>0` + `max.in.flight.requests.per.connection<=5` + 幂等性 → 保证消息不丢不重到达 Leader。
- **复制端**：ISR 机制 + HW 可见性 → 保证消息同步到至少一个 Follower 后才对消费者可见。
- **存储端**：CRC32 校验 + 顺序写 → 防止数据损坏。
- **选举端**：Clean Leader Election + `unclean.leader.election=false` → 保证新 Leader 数据不落后。
- **注意**：Producer 内存缓冲区（`buffer.memory`）中的数据在进程崩溃时**永久丢失**，需配合 `acks=all` 和适当的重试策略 mitigated。

---

## 8. 性能与场景

### 8.1 Kafka 为什么快？

| 优化手段 | 原理 | 效果 |
|---------|------|------|
| **顺序写磁盘** | 消息按分区追加写入，避免随机 I/O 寻道 | 机械盘顺序写可达 100MB/s+，SSD 可达 GB/s 级 |
| **页缓存（Page Cache）** | 利用 OS 内核页缓存，规避 JVM GC | 写入/读取速度接近内存级，避免 STW |
| **零拷贝（sendfile）** | 数据从 Page Cache 直接到网卡，不经用户态 | 消除 2 次 CPU 拷贝，CPU 占用降低 30%+ |
| **批量处理** | 生产者累积消息为批次发送，消费者批量拉取 | 减少网络 RTT 和磁盘 I/O 次数 |
| **分区并行** | 多 Partition 分布在不同 Broker 上 | 读写吞吐量随分区数线性扩展 |
| **分区级无锁** | 单 Partition 串行处理，避免锁竞争 | 高并发下低延迟，保证分区内顺序性 |
| **数据压缩** | 批次级压缩（lz4/zstd/snappy/gzip） | 降低网络带宽和磁盘存储开销 |
| **轻量级消息格式** | 紧凑二进制序列化，处理开销极小 | 单位时间内可处理更多消息 |

### 8.2 典型吞吐量

- **单机（单 Partition）**：顺序写可达 **数百 MB/s**，读可达 **网络带宽上限**。
- **集群（多 Partition 并行）**：吞吐量随分区数和 Broker 数线性扩展，生产集群轻松达到 **GB/s 级**。
- **延迟**：消息从发送到可消费的端到端延迟通常在 **毫秒级**（取决于网络、磁盘和 acks 配置）。

### 8.3 适合的场景

| 场景 | 说明 |
|------|------|
| **日志聚合** | 统一采集分布式系统的日志，持久化后供 ELK 等分析 |
| **事件驱动架构** | 微服务间异步通信，事件溯源（Event Sourcing） |
| **流式数据处理** | 与 Flink/Spark Streaming 集成，实时 ETL、实时告警 |
| **消息队列** | 系统间解耦、异步处理、流量削峰 |
| **指标监控** | 收集应用/基础设施指标，支持实时告警和历史回放 |
| **审计日志** | 所有操作事件持久化，支持合规审计和时间回放 |
| **数据湖摄入** | 作为 Lakehouse 的实时数据摄入层（Kafka → Iceberg/Hudi） |
| **AI 数据总线** | 为 AI Agent 提供实时上下文流，支持 MCP 协议集成 |

### 8.4 不适合的场景

| 场景 | 原因 |
|------|------|
| **低延迟 RPC 调用** | Kafka 是批处理/流式平台，非点对点 RPC，延迟高于 gRPC/HTTP |
| **复杂查询/事务** | 不支持 SQL 查询、ACID 事务（仅消息级别事务），需配合其他存储引擎 |
| **小消息高频场景（微消息）** | 每条消息有固定开销（网络帧、序列化等），百万级 QPS 的微小消息场景需特殊调优 |
| **需要随机读写** | Kafka 是顺序追加日志，不支持按 Key 随机读写（需配合 KTable/KStream 或外部存储） |
| **短生命周期任务** | 消息持久化在磁盘，短期任务（如请求转发）用 Redis/RabbitMQ 更合适 |
| **需要精细的消息路由** | Kafka 基于 Topic 订阅，不支持 RabbitMQ 式的 Exchange 灵活路由（需通过 Topic 设计模拟） |

### 8.5 与主流消息队列对比

| 维度 | Kafka | RabbitMQ | RocketMQ |
|------|-------|----------|----------|
| 定位 | 分布式流式平台 | 传统消息队列 | 分布式消息队列 |
| 吞吐量 | 极高（顺序写+零拷贝） | 中等 | 高 |
| 延迟 | 毫秒级 | 微秒级 | 毫秒级 |
| 消息保留 | 可配置持久保留（天/周/月） | 消费后自动删除 | 可配置保留 |
| 消息回溯 | 支持（通过 offset 任意回溯） | 不支持 | 支持 |
| 事务支持 | 跨分区事务（Kafka Transaction） | 不支持 | 支持 |
| 生态 | 流处理生态丰富（Streams/Flink/Spark） | 插件生态 | 阿里生态 |
| 适用规模 | 大规模数据管道 | 中小规模业务消息 | 中大规模业务消息 |

---

## 9. Kafka 常用面试问题

### 一、基础概念类

**Q1：Kafka 是什么？它解决了什么问题？**

Kafka 是一个分布式流式数据处理平台，核心解决系统间异步解耦、流量削峰填谷、日志/事件采集、实时流式计算、数据同步与备份等问题。与传统消息队列不同，Kafka 将消息持久化到磁盘并支持多消费者回放，适合大数据场景。

**Q2：Kafka 的核心数据模型有哪些？**

Topic（主题）、Partition（分区）、Message（消息）、Offset（偏移量）、Broker（节点）、Producer（生产者）、Consumer（消费者）、Consumer Group（消费者组）。其中 Partition 是并行度的核心，Consumer Group 是实现负载均衡的关键。

**Q3：Consumer Group 的工作原理是什么？**

同一 Consumer Group 中的消费者共享一个 Group ID，Topic 的每个 Partition 同一时刻只能被组内一个消费者消费。当组内消费者数量等于 Partition 数量时，达到最大并行度；消费者增多不会进一步提升吞吐（多余的消费者处于空闲状态）。

### 二、存储与性能类

**Q4：Kafka 为什么这么快？（高频面试题）**

核心原因包括：(1) 顺序写磁盘，避免随机 I/O 寻道；(2) 利用 OS Page Cache 缓存热点数据，规避 JVM GC；(3) sendfile 零拷贝传输，消除用户态数据拷贝；(4) 批量处理减少网络请求；(5) 分区并行实现水平扩展；(6) 分区级无锁设计；(7) 数据压缩降低带宽开销；(8) 轻量级二进制消息格式。

**Q5：请解释 Kafka 的 Log Segment 文件结构。**

每个 Partition 的日志被切分为多个 Log Segment，每个 Segment 包含三个文件：`.log`（存储消息实际数据，以 RecordBatch 为单位顺序追加）、`.index`（偏移量稀疏索引，记录 Relative Offset 到 .log 文件物理位置的映射，默认每 4KB 写入一条索引项）、`.timeindex`（时间戳索引，记录 Timestamp 到 Relative Offset 的映射，支持按时间范围查询）。三者通过相对偏移量串联，配合二分查找+顺序扫描实现高效检索。

**Q6：什么是 Page Cache？Kafka 如何利用它？**

Page Cache 是 Linux 内核的页缓存机制，将空闲物理内存用作磁盘缓存。Kafka 不维护自己的消息缓存，而是将数据写入 Page Cache 即返回（延迟写回），读取时优先命中 Page Cache 避免磁盘 I/O。这带来了写入极速化、零 GC 压力和读写缓存复用三大优势。写入调用 `pwrite()`，读取调用 `sendfile()` 零拷贝，索引检索调用 `mmap()` 内存映射。

**Q7：什么是零拷贝（Zero-Copy）？Kafka 中是如何实现的？**

零拷贝是指数据从磁盘到网卡的传输不经过用户态，减少 CPU 拷贝和上下文切换。Kafka 使用 Linux 的 `sendfile()` 系统调用，数据从 Page Cache 直接传输到 Socket Buffer（配合 DMA 控制器发往网卡），全程不经过用户态缓冲区。传统路径需 4 次拷贝（磁盘→Page Cache→用户态 Buffer→Socket Buffer→网卡），sendfile 路径仅需 2 次 DMA 拷贝。

**Q8：NIC Buffer 是什么？**

NIC Buffer 是物理网卡上的硬件内存。DMA 控制器将数据从 Socket Buffer 转移到 NIC Buffer 后，网卡从中读取并序列化为电信号/光信号发送到物理网络。在零拷贝场景下，从 Page Cache 到 NIC Buffer 的整段搬运由 DMA 硬件完成，无需 CPU 参与。

### 三、架构与一致性类

**Q9：Kafka 的副本机制（ISR）是如何工作的？**

每个 Partition 有一个 Leader 处理读写，多个 Follower 从 Leader 拉取数据同步。ISR（In-Sync Replicas）是与 Leader 保持同步的副本集合。Follower 在 `replica.lag.time.max.ms`（默认 30s）内未同步被移出 ISR，追上后重新加入。HW（高水位）= min(所有 ISR 副本的 LEO)，只有 Offset < HW 的消息才对消费者可见，确保不读到未同步的「脏数据」。

**Q10：acks=0/1/all 的区别是什么？生产环境怎么选？**

- `acks=0`：生产者不等待确认，可能丢消息，吞吐量最高。适用于日志采集等允许丢数据的场景。
- `acks=1`：仅 Leader 写入成功即确认，Leader 故障可能丢数据，吞吐量大。
- `acks=all`：等待 ISR 所有副本写入成功才确认，数据不丢失（需 `min.insync.replicas>=2`），吞吐量较低。生产环境推荐 `acks=all` + `min.insync.replicas=2`。

**Q11：什么是幂等性生产者？如何实现？**

幂等性生产者通过 Producer ID（PID）+ 序列号（Sequence Number）实现单生产者单分区的精确一次语义。Broker 端校验序列号：等于 last_sn+1 则正常接收，小于等于 last_sn 则是重复消息丢弃，大于 last_sn+1 则拒绝。需配合 `acks=all`、`retries>0` 及 `max.in.flight.requests.per.connection<=5`。

**Q12：Kafka 事务机制是如何工作的？**

事务机制解决跨分区原子性问题。通过 `transactional.id` 标识生产者，分配 PID 和 Epoch。事务内消息写入后标记为未提交（消费者 `READ_COMMITTED` 模式下不可见），提交时写入 COMMIT Marker，回滚时写入 ABORT Marker。支持 `sendOffsetsToTransaction` 将消费位移纳入事务，实现「读-处理-写」的端到端原子操作。

### 四、分布式与运维类

**Q13：Kafka 的 Leader 选举流程是怎样的？**

Broker 通过心跳向 Controller 汇报状态，超过 `session.timeout.ms`（默认 18s）未收到心跳则判定失效。Controller 从受影响分区的 ISR 中选择新 Leader，广播 `LeaderAndIsrRequest` 更新元数据。客户端自动感知变更并重连新 Leader。默认仅从 ISR 选举（Clean Leader Election），保证数据不丢失。

**Q14：KRaft 模式相比 ZooKeeper 有什么优势？**

KRaft 将元数据存储在 Kafka 内部的 `__cluster_metadata` Topic 中，使用 Raft 共识协议替代 ZAB。优势包括：写操作延迟更低（1~3ms vs 5~15ms）、支持百万级分区、无需维护外部 ZK 集群、部署更简单。Kafka 4.0 已完全移除 ZooKeeper 支持。

**Q15：Kafka 如何扩容？数据如何迁移？**

新增 Broker 后，通过 `kafka-reassign-partitions.sh` 工具重新分配分区副本，实现数据均衡迁移。Kafka 提供 `RangeAssignor`、`RoundRobinAssignor`、`StickyAssignor` 等分区分配策略。跨机房复制可通过 MirrorMaker 2.0 或分层存储（Tiered Storage）实现。

**Q16：什么是分层存储（Tiered Storage）？**

分层存储将冷数据自动迁移至对象存储（如 S3、OSS），Broker 变为无状态节点。支持分钟级弹性扩缩容，读写路径隔离（冷读不拖垮写入），在合规审计场景下数据保留成本可降低约 90%。

### 五、场景设计类

**Q17：Kafka 不适合哪些场景？**

(1) 低延迟 RPC 调用——Kafka 是批处理平台，延迟高于 gRPC；(2) 复杂查询/事务——不支持 SQL 查询和 ACID 事务；(3) 小消息高频场景——每条消息有固定开销；(4) 需要随机读写——Kafka 是顺序追加日志；(5) 短生命周期任务——用 Redis/RabbitMQ 更合适；(6) 需要精细消息路由——不支持 Exchange 灵活路由。

**Q18：如何保证 Kafka 消息不丢失？**

完整链路保障：(1) 生产者端 `acks=all` + `retries>0` + `max.in.flight.requests.per.connection<=5` + 幂等性，保证消息不丢不重到达 Leader；(2) 复制端 ISR 机制 + HW 可见性，保证消息同步到至少一个 Follower 后才对消费者可见；(3) 存储端 CRC32 校验 + 顺序写，防止数据损坏；(4) 选举端 Clean Leader Election + `unclean.leader.election=false`，保证新 Leader 数据不落后。

**Q19：Kafka 的消息积压如何处理？**

(1) 临时方案：增加消费者实例数量（不超过 Partition 数量），提升消费能力；(2) 扩容 Partition 数量，提升并行度；(3) 优化消费者处理逻辑，减少单条消息处理耗时；(4) 临时调大 `fetch.max.bytes` 和 `max.poll.records` 提升单次拉取量；(5) 对于历史数据积压，可新建 Topic 增加 Partition 数，通过消费者重平衡逐步消费。

**Q20：__consumer_offsets Topic 的作用是什么？**

`__consumer_offsets` 是 Kafka 内部 Topic，用于保存消费者组的位移（Offset）和组元数据。位移提交消息的 Key 为 `[Group ID, Topic Name, Partition Index]`，Value 包含位移值、元数据、提交时间戳和过期时间戳。Kafka 默认每 5 秒自动提交一次位移到该 Topic。
