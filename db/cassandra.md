# Cassandra 面试知识总结

> **定位：分布式、高可用、可水平扩展的 NoSQL 宽列数据库。**
>
> 核心目标：**海量数据 + 高吞吐 + 低延迟 + 高可用 + 水平扩展**。

Cassandra 的设计融合了：

- **Dynamo**：一致性 Hash、多主复制、Gossip、可调一致性
- **BigTable/HBase**：宽列数据模型、LSM 存储思想

---

# 1. 定位与数据模型

## 1.1 Cassandra 是什么？

| 维度 | 说明 |
|---|---|
| 类型 | 分布式 NoSQL / 宽列数据库 |
| 核心模型 | Partition-oriented Wide Column |
| 核心访问 | 根据 Partition Key 定位数据 |
| 一致性 | 最终一致性为主，可调一致性 |
| 架构 | 无中心、多主 |
| 扩展 | 水平扩展 |
| 目标 | 海量数据、高吞吐、高可用、低延迟 |

Cassandra 不追求关系型数据库的复杂查询和强事务，而是优先保证：

> **Availability + Scalability + Throughput**

## 1.2 数据模型

```text
Keyspace
    ↓
Table
    ↓
Partition
    ↓
Row
    ↓
Column
```

| 概念 | 类比 / 作用 |
|---|---|
| Keyspace | 类似 Database，同时定义复制策略 |
| Table | 数据表 |
| Partition | 数据分布和访问的基本单位 |
| Row | 行 |
| Column | 列 |

### Primary Key

```sql
PRIMARY KEY ((partition_key), clustering_key)
```

- **Partition Key**：决定数据分布到哪个节点
- **Clustering Key**：决定 Partition 内数据的排序和组织方式

> **Cassandra 表设计的核心：围绕查询模式设计 Partition Key。**

---

# 2. 存储引擎

Cassandra 使用 **LSM-Tree**。

核心写入路径：

```text
Client
  ↓
Commit Log
  ↓
Memtable
  ↓
Flush
  ↓
SSTable
```

## 2.1 Commit Log

类似 WAL。

作用：

> **保证崩溃后的数据恢复。**

Crash 后：

```text
Commit Log
     ↓
Replay
     ↓
恢复 Memtable
```

## 2.2 Memtable

内存中的写缓存。

- 接收新的写入
- 内存中组织数据
- 服务部分读取
- 达到阈值后 Flush

## 2.3 SSTable

Memtable Flush 后形成 SSTable。

特点：

- 磁盘文件
- 不可变
- 数据有序
- 支持压缩
- 包含索引、Bloom Filter 等辅助结构

## 2.4 Compaction

```text
SSTable 1
SSTable 2
SSTable 3
     ↓
 Compaction
     ↓
SSTable
```

作用：

- 合并 SSTable
- 清理旧版本
- 清理 Tombstone
- 优化读取

代价：

> **Write Amplification + 后台 I/O**

常见策略：

- STCS：Size-Tiered Compaction Strategy
- LCS：Leveled Compaction Strategy
- TWCS：Time-Window Compaction Strategy

---

# 3. 内存与数据结构

## 3.1 Memtable

```text
Write
  ↓
Memtable
```

保存最新写入的数据，减少直接访问磁盘。

## 3.2 Bloom Filter

查询 SSTable 时：

```text
Bloom Filter
     ↓
不存在 → 直接跳过
     ↓
可能存在
     ↓
Index
     ↓
Data
```

作用：

> **判断某个 Partition Key 是否肯定不在该 SSTable 中。**

## 3.3 SSTable Index

帮助从：

```text
Partition Key
      ↓
磁盘位置
```

快速定位数据。

典型读取过程：

```text
Partition Key
     ↓
定位 Token / Node
     ↓
Bloom Filter
     ↓
Index
     ↓
SSTable
```

---

# 4. 索引与查询

# Cassandra 索引、查询机制与一致性 Hash 深度解析

Apache Cassandra 是一款专为高吞吐写入和极致可用性设计的分布式 NoSQL 数据库。其底层架构深度依赖 LSM-Tree 存储模型与一致性 Hash 路由机制。理解其数据定位、查询限制及索引演进，是掌握 Cassandra 核心设计哲学的关键。

## 一、 数据定位：一致性 Hash 与 Token Ring

Cassandra 摒弃了传统哈希取模算法，采用一致性 Hash（Consistent Hashing）来解决集群动态扩缩容时的数据迁移风暴。

### 1. Token Ring（哈希环）与路由机制
Cassandra 默认使用 `Murmur3Partitioner`，将哈希空间映射为一个从 $-2^{63}$ 到 $2^{63}-1$ 的连续哈希环。
- **路由过程**：当数据写入时，系统对 `Partition Key` 进行 Hash 计算得出一个 Token 值。该 Token 落在环上后，系统顺时针寻找，遇到的第一个节点即为该数据的主副本（Primary Replica）。

### 2. 虚拟节点（VNodes）的负载均衡
早期设计中，每个物理节点仅拥有一个 Token，极易导致数据倾斜。Cassandra 引入 VNodes 机制，将一个物理节点拆分为多个（通常为 256 个）虚拟节点均匀散布在环上。
- **扩容平滑**：新节点加入时，均匀地从全网所有节点接管一小段 Token 范围内的数据，无需全量迁移。
- **故障恢复**：节点宕机时，重建数据的负载被均匀分摊到集群的多个物理节点上，避免次生灾难。

### 3. 多副本拓扑与容灾
数据定位到首个节点后，Cassandra 会根据 `NetworkTopologyStrategy` 继续顺时针遍历哈希环，智能跳过同一机架（Rack）或数据中心（DC）的节点，将副本存放在物理隔离的节点上，确保跨可用区的容灾能力。

---

## 二、 查询引擎：LSM-Tree 下的读写路径

Cassandra 的查询限制源于其底层 LSM-Tree 架构，这决定了其“写入极快，读取依赖精确定位”的特性。

### 1. 极速的 Write Path
写入请求到达时，首先以 Append-Only 方式顺序追加到磁盘的 `CommitLog`，确保断电不丢数据；随后写入内存中的 `MemTable`。
- **特点**：整个过程仅涉及内存操作与磁盘顺序写，无随机 I/O，写入性能极高。
- **落盘**：MemTable 写满后刷入磁盘生成不可变的 `SSTable`，后台通过 Compaction 机制定期合并并清理 Tombstones（删除标记）。

### 2. 复杂的 Read Path
当执行 `WHERE partition_key = ?` 时，单节点内的检索流程高度优化：
1. **Bloom Filter**：首先检查目标 Partition 是否可能存在于当前 SSTable 中，若返回 False 则直接跳过，极大减少无效磁盘 I/O。
2. **Partition Key Cache & Index**：在内存 Key Cache 和磁盘 Partition Index 中查找目标 Partition Key 的字节偏移量。
3. **SSTable 扫描**：直接跳转到文件对应位置，读取并返回数据。

---

## 三、 索引与查询：从主键到 SAI 的演进

Cassandra 强制要求“查询驱动建模（Query-Driven Modeling）”，即一个查询语句对应一张表，严禁跨 Partition 的复杂查询。

### 1. Partition Key 与 Clustering Key 的协同
- **Partition Key**：决定数据去往哪台机器。查询必须包含完整的 Partition Key，否则 Coordinator 节点必须向全网所有节点广播查询（Scatter-Gather 模式），导致极高的网络开销和全表扫描。
- **Clustering Key**：决定数据在机器本地 SSTable 内部的排序存放。结合 Partition Key，系统可在磁盘上执行高效的顺序范围读取（Range Scan）。

### 2. 二级索引（2i）的局限
传统二级索引（2i）在每个节点本地维护独立的隐藏表。若查询不包含 Partition Key，Cassandra 必须向所有节点发起查询并合并结果。此外，每次写入都会触发索引文件的重新构建，导致严重的写放大（Write Amplification）和磁盘 I/O 饱和。因此，2i 仅推荐在已知 Partition Key 的前提下使用。

### 3. Storage-Attached Indexing (SAI) 的破局
Cassandra 5.0 引入的 SAI 彻底改变了非主键查询的格局。SAI 将索引信息直接附加到存储数据的 SSTable 上，而非独立的隐藏表。
- **性能提升**：SAI 在写入时将数据在内存中索引，随 MemTable 一起 Flush，避免了 2i 的写放大问题。
- **查询灵活性**：支持对非主键列进行等值查询、数值范围查询及 CONTAINS 语义，甚至支持向量搜索（Vector Search），大幅降低了数据模型反范式化的心智负担。

---

## 四、 核心设计哲学总结

| 维度 | 设计优良的查询 | 设计糟糕的查询 |
| :--- | :--- | :--- |
| **查询条件** | 包含完整的 Partition Key | 仅包含非 Key 字段，或跨 Partition 过滤 |
| **网络开销** | O(1) 路由，直接转发给目标节点 | Scatter-Gather，全网广播并内存合并 |
| **磁盘 I/O** | 命中 Bloom Filter，读取连续磁盘块 | 触发全表扫描，大量无效读取 |
| **节点负载** | 随机哈希保证请求均匀分散 | 产生热点（Hotspot），单节点流量过载 |

> **核心原则**：Cassandra 的强大在于其将复杂性下沉到了架构层。开发者必须顺应其“以查询定模型”的哲学，合理利用 VNodes 保证数据均匀，借助 Clustering Key 优化本地读取，并在 Cassandra 5.0+ 环境中积极拥抱 SAI 以获得更灵活的查询能力，同时坚决避免在生产环境使用 `ALLOW FILTERING`。
---

# 5. 并发、一致性与事务

# Apache Cassandra 架构与一致性机制深度解析

Cassandra 采用完全去中心化（Masterless）的对等网络架构，抛弃了传统关系型数据库的主从模型。它明确选择 **AP（高可用 + 分区容错）** 作为基础设计，通过牺牲部分强一致性来换取极致的高可用、线性横向扩展能力以及低延迟的读写性能。

## 1. 核心架构与角色

集群由地位相等的节点组成一个逻辑环（Token Ring），节点间通过 **Gossip（八卦）协议** 每秒交换状态信息。这种设计消除了单点故障，也没有固定的主节点。

### Coordinator（协调节点）

Coordinator 是一个**逻辑角色**，而非物理存在的固定节点。客户端可以连接集群中的任意一个节点发起读写请求，该节点即刻成为本次请求的 Coordinator。它的职责是：

1. 根据一致性哈希算法计算出数据分布在哪些真实节点上。
2. 将请求并行派发给这些目标节点。
3. 等待足够数量的副本返回结果（根据一致性级别），再响应客户端。

## 2. 多副本与一致性哈希 (Consistent Hashing)

为了保证高可用，数据必须冗余存储（Replication）。Cassandra 利用一致性哈希将数据映射到环上，并通过虚拟节点（VNodes）技术实现数据的均匀分布与平滑扩缩容。

### RF = 3 时的哈希环与寻址过程

假设集群有 4 个节点，Token 范围划分如下：

```
                  [Node A: 负责 Token 01-25]
                 /                          \
                /                            \
[Node D: 负责 Token 76-100]      [Node B: 负责 Token 26-50]
                \                            /
                 \                          /
                  [Node C: 负责 Token 51-75]
```

**写入动作示例**：客户端请求写入 `Key = "user_123"`。

1. **哈希计算**：Coordinator 计算 `Hash("user_123") = 35`。
2. **顺时针落盘**：Token 35 落在 Node A (25) 和 Node B (50) 之间，顺时针遇到的第一个节点是 **Node B**，Node B 成为主副本。
3. **副本分布**：因 `RF = 3`，数据会自动保存在顺时针接下来的 3 个节点：**Node B**（第一副本）、**Node C**（第二副本）、**Node D**（第三副本）。
4. **并行写入**：Coordinator 会并行向 B、C、D 发送写入指令。

## 3. 存储引擎：LSM-Tree 写入与读取路径

Cassandra 的高性能不仅依赖于网络架构，更依赖于其底层的 LSM-Tree 存储引擎。

### 写入路径 (Write Path)

1. **Commit Log**：数据首先追加写入提交日志，确保数据的持久性（Durability）。
2. **Memtable**：数据写入内存表（Memtable），此时数据在内存中按主键排序。
3. **SSTable**：当 Memtable 达到阈值，会 Flush 到磁盘生成不可变的排序字符串表（SSTable）。
4. **响应客户端**：只要满足一致性级别的节点完成了 Commit Log 和 Memtable 的写入，即可向客户端返回成功。

### 读取路径 (Read Path)

1. **检查缓存**：优先检查 Row Cache（行缓存）。
2. **Bloom Filter**：通过布隆过滤器快速判断数据是否可能存在于某个 SSTable 中，避免无效的磁盘 I/O。
3. **多源合并**：如果缓存未命中，Coordinator 会同时查询 Memtable 和相关的 SSTables，根据时间戳合并出最新结果返回给客户端。

## 4. 可调一致性 (Tunable Consistency)

Consistency Level (CL) 是 Cassandra 的精髓：它定义了 Coordinator 必须等待多少个副本节点确认，才能向客户端返回"成功"。

| Consistency Level | 等待节点数 | 延迟与可用性 | 一致性保证 |
| :--- | :--- | :--- | :--- |
| **ONE** | 仅 1 个节点 | 延迟极低，可用性最高 | 最弱，极易读到旧数据 |
| **QUORUM** | 全局多数派：floor(RF / 2) + 1 | 延迟中等，容忍少数节点宕机 | 强（若配合 QUORUM 读） |
| **LOCAL_QUORUM** | 仅 Coordinator 所在 DC 的多数派 | 规避跨地域延迟，高可用 | 数据中心内部强一致 |
| **ALL** | 所有副本节点 | 延迟高，任意节点宕机即失败 | 绝对强一致 |

### QUORUM 与 LOCAL_QUORUM 跨机房示例

假设集群跨越北京机房（3个节点）和纽约机房（3个节点），总 `RF = 6`，北京机房 `RF = 3`。

- **使用 QUORUM（全局多数派）**：需要 `6 / 2 + 1 = 4` 个节点确认。即使北京 3 个节点秒回，也必须等待至少 1 个纽约节点跨洋确认。200ms 的跨国延迟会直接拖垮写入性能。
- **使用 LOCAL_QUORUM（本地多数派）**：只需本地 `3 / 2 + 1 = 2` 个节点确认。北京的 Coordinator 收到本地 2 个节点确认即可瞬间（可能只需 2ms）返回成功，系统会在后台异步将数据同步到纽约机房。

## 5. 最终一致性与三层修复机制

当使用 `LOCAL_QUORUM` 或 `ONE` 写入时，部分节点可能因网络抖动漏写数据。Cassandra 采用最终一致性模型，主动接纳不一致，并依赖三种机制修复数据：

1. **Hinted Handoff（提示移交）**：写入时，若目标节点短暂不可达，Coordinator 会在本地暂存写请求元数据（Hint），待该节点恢复后主动推送补写。
2. **Read Repair（读时修复）**：在 Quorum 读过程中，若发现参与响应的副本间数据版本不一致，Coordinator 在返回最新结果的同时，异步发起后台修复，将最新值同步至陈旧副本。
3. **Anti-Entropy（反熵修复）**：基于 **Merkle Tree（哈希树）** 的全量/增量校验。通常通过 `nodetool repair` 手动触发，作为兜底手段保障跨数据中心数据收敛。

### Merkle Tree 结构与比对示例

Merkle Tree 是一种倒置树，叶子节点是真实数据块的 Hash，父节点是子节点 Hash 组合后的 Hash。

```
       [Node 1 的树 (含最新 C)]               [Node 2 的树 (含旧 C')]
              Root_1                              Root_2
             /      \                            /      \
        Hash_AB    Hash_CD                  Hash_AB    Hash_C'D
        /    \      /    \                  /    \      /    \
      H(A)  H(B)  H(C)  H(D)              H(A)  H(B)  H(C') H(D)
```

- **对比根节点**：`Root_1 != Root_2`，说明数据有差异。
- **向下对比**：左分支 `Hash_AB` 相同，说明 A、B 完全一致，无需再查。
- **精准定位**：右分支不同，继续向下查，发现 `H(D)` 一致，但 `H(C) != H(C')`。
- **精准修复**：Coordinator 精确锁定只有数据块 C 不一致，只需在网络中传输几十字节的 C 的真实数据覆盖掉 C' 即可，避免了全表扫描和海量数据传输。

## 6. 冲突解决：Last Write Wins (LWW)

在无主架构中，没有全局锁。当并发更新同一条数据时，Cassandra 采用 **Last Write Wins (LWW)** 策略：

- 每次写入必须携带一个精确到微秒的 **Timestamp**。
- 无论网络传输的先后顺序如何，节点在合并数据时，时间戳最大的数据永远覆盖较小的数据。
- **删除操作（Tombstone）**：删除实际上是插入一个带有最新时间戳的"墓碑"标记。读取时若墓碑时间戳大于旧数据，则判定为已删除。

## 7. 事务与条件更新：Paxos 与 LWT

Cassandra 追求高吞吐，不支持传统的 `BEGIN ... COMMIT` 跨行/跨表 ACID 事务。但对于必须依赖旧状态的条件更新（Compare-And-Set），提供 **Lightweight Transactions (LWT)**。

LWT 基于 **Paxos 共识算法** 保证线性一致性，确保并发安全。这需要 4 次网络往返（Round-trips），延迟通常是普通写入的 4 倍以上，仅限极其严苛的并发修改场景（如库存扣减、计数器）使用。

**LWT 执行过程示例**：

账户 `balance = 50`。客户端 X 想在 `balance == 50` 时更新为 `100`；客户端 Y 想更新为 `80`。

1. **Prepare（准备与承诺）**：Coordinator X 生成提议号 T1 发给所有副本。副本承诺"不再接受比 T1 更老的提议"。
2. **Read（读取现状）**：Coordinator X 读出当前真实状态 `balance = 50`。（此时若 Coordinator Y 带着 T2 > T1 杀入，副本会接受 Y，导致 X 的后续操作被拒绝）。
3. **Propose（提出提议）**：Coordinator X 在本地比对 `balance == 50` 成立，向副本发起更新指令：设为 100。
4. **Commit（正式提交）**：多数派副本接受提议后，Coordinator X 发送 Commit 落盘。客户端 X 收到成功响应。客户端 Y 的条件更新因不满足前置条件或提议号被拦截，返回 `[applied] = false`。

## 8. 批处理 (Batch)

Cassandra 的 Batch **不能替代传统业务的事务**，它不具备隔离性（Isolation），执行中途其他客户端可能读到部分已更新的数据。

- **Unlogged Batch（纯网络优化）**：强制要求放入同一个 Batch 的所有操作必须属于**同一个 Partition Key**。这保证所有操作发往同一个物理节点，将多次网络请求打包为一次，极大提升性能。
- **Logged Batch（保障原子性）**：适用于必须同时更新多张表（如主表和物化视图）的场景。Coordinator 先将整个批次写入系统的 `batchlog` 表。即使 Coordinator 崩溃，集群其他节点也会扫描 `batchlog` 继续重试未完成的写入，确保这批数据最终都被成功写入（Atomicity），但不保证同时生效。

---

# 6. 分布式与高可用

这份文档主要描述了以 Apache Cassandra 为代表的分布式 NoSQL 数据库在**分布式架构与高可用性**方面的核心设计。

## 6.1 一致性 Hash (Consistent Hashing)

**概念详细描述：**
传统的哈希取模算法（如 `hash(key) % N`）在节点增删时，会导致大量数据重新映射，造成缓存雪崩或极大的数据迁移开销。一致性哈希通过构建一个首尾相连的**哈希环（Token Ring）**，将数据和节点映射到同一个闭环的数值空间（在 Cassandra 默认的 `Murmur3Partitioner` 中，范围是 -2^63 到 2^63-1）。
*   **Virtual Nodes (VNodes 虚拟节点)：** 现代 Cassandra 引入了虚拟节点技术。物理节点不再只负责环上的一个连续区间，而是负责多个随机分布的 Token 区间。这解决了数据倾斜问题，并在节点扩容/缩容时实现了极速的数据负载均衡。

**核心算法流程：**
1.  **路由计算：** 客户端发起请求，提供 Partition Key。
2.  **哈希映射：** 数据库对 Partition Key 使用 Hash 函数计算得到一个哈希值（Token）。
3.  **顺时针寻址：** 将该 Token 放到哈希环上，**顺时针**寻找遇到的第一个节点，该节点即为该数据的 Primary Replica（主副本）。

**文本图表：一致性 Hash 环与路由**
```text
                  [Node A] (Token: 0)
                 /                   \
               /                       \
   (Token: 75)                           (Token: 25)
   [Node D]                               [Node B]
      \                                     /
        \                                 /
         \                               /
                  [Node C] (Token: 50)

数据路由过程：
Partition Key: "user_123" 
  -> Hash("user_123") = Token 15
  -> 在环上顺时针找，落入 0~25 之间
  -> 数据存储在 [Node B]
```

## 6.2 多主架构 (Masterless / Multi-Master Architecture)

**概念详细描述：**
受 Amazon Dynamo 论文启发，Cassandra 采用了完全无中心（Decentralized）、去中心化的 P2P 架构。没有主从（Master-Slave）之分，集群中的所有节点（Node）地位完全平等。

*   **Coordinator（协调者）：** 这是一个**动态角色**。任何节点都可以接收客户端的读写请求。当节点 A 接收到客户端的请求时，针对该次请求，节点 A 就充当 Coordinator。它负责根据一致性哈希环，找出数据所在的真正副本节点，将请求并发分发给它们，等待足够数量（满足 Consistency Level）的响应后，再将结果返回给客户端。

**文本图表：多主架构与协调者机制**
```text
                 +-------------+
                 |   Client    |
                 +------+------+
                        | (请求 Key="user_123")
                        v
               +-----------------+
               | Node A (接收方) |  <-- 此时作为 Coordinator
               +-----------------+
              /         |         \
 (转发写请求)/          |          \(转发写请求)
           v            v            v
      +--------+   +--------+   +--------+
      | Node B |   | Node C |   | Node D |
      +--------+   +--------+   +--------+
       (副本1)      (副本2)      (副本3)
```

## 6.3 Replication Factor (复制因子)

**概念详细描述：**
Replication Factor (RF) 定义了集群中同一个数据分片（Partition）在整个集群中拥有的副本总数。
*   **RF = 1：** 数据只有一份，节点宕机则数据不可用（单点故障）。
*   **RF = 3：** 生产环境的标准配置。数据被复制到 3 个不同的节点上。结合可调一致性（Tunable Consistency），例如读写一致性级别设置为 `QUORUM` (满足 quorum = floor(RF/2) + 1 = 2)，允许集群中任意 1 个节点宕机而不影响读写。

**副本分布原则：** 系统会尽可能避免将同一个数据的多个副本放在同一个故障域（如同一台机架、同一个机房）。

## 6.4 Replication Strategy (复制策略)

**概念详细描述：**
副本应该放在环上的哪些节点？这由复制策略决定。

**1. SimpleStrategy（简单策略）：**
*   **流程：** 将第一个副本放在由 Partition Key 哈希计算出的 Token 所在的节点上，然后顺时针沿着哈希环放置后续的副本，**不考虑任何网络或机架拓扑结构**。
*   **适用场景：** 单个数据中心、开发测试环境。

**2. NetworkTopologyStrategy（网络拓扑策略）：**
*   **概念：** 生产环境必备。它允许跨多个数据中心（Data Center）配置不同的 RF，并具备**机架感知（Rack-Aware）**能力。
*   **算法流程：** 
    1. 确定第一个副本的位置（顺时针第一个节点）。
    2. 继续顺时针遍历环，对于后续的副本，算法会强制跳过与已有副本处于**同一个 Rack（机架）**的节点。
    3. 直到在不同的 Rack 上集齐了所需数量的副本。这保证了如果整个机架断电或交换机故障，数据依然可用。

**文本图表：NetworkTopologyStrategy 分布**
```text
Data Center 1 (DC1)                  Data Center 2 (DC2)
RF = 2                               RF = 1

Rack 1        Rack 2                 Rack 1
[Node A]      [Node C]               [Node E]
[Node B]      [Node D]               [Node F]

假设数据落点起于 Node A：
- 副本 1：Node A (DC1, Rack1)
- 副本 2：跳过 Node B (同Rack)，选择 Node C (DC1, Rack2)
- 副本 3：分配给 DC2 的首个节点 Node E
```

## 6.5 Gossip (Gossip 协议 / 流行病协议)

**概念详细描述：**
Gossip 是一种分布式的点对点通信协议。因为集群没有 Master 节点来集中管理元数据，所有节点需要一种机制来了解集群的“拓扑结构”和“其他节点的健康状态”。Gossip 就像人类社会中的“八卦”一样，信息通过节点间的随机通信，呈指数级在整个集群中扩散。

**算法与通信流程详细描述：**
Cassandra 的 Gossip 状态由 `Generation`（节点启动时间戳，重启改变）和 `Version`（内部状态版本，递增）标识，以此来判断谁的数据最新。
1.  **节点选择：** 每秒钟，每个节点会随机选择 1~3 个其他节点进行 Gossip 通信。
2.  **三次握手协议 (SYN-ACK-ACK2)：**
    *   **SYN：** 节点 A 向节点 B 发送 `GossipDigestSyn` 消息，包含自己知道的所有集群节点列表、Generation 和最大的 Version 号。
    *   **ACK：** 节点 B 收到后与自己的状态进行比对。对于 A 比 B 旧的数据，B 准备好新数据；对于 A 比 B 新的数据，B 请求 A 发送。B 将这些封装成 `GossipDigestAck` 发给 A。
    *   **ACK2：** 节点 A 收到 ACK 后，更新自己的落后状态，并将 B 请求的新状态封装成 `GossipDigestAck2` 发给 B。至此，A 和 B 的状态达成一致。

**文本图表：Gossip 三次握手流程**
```text
   Node A (Initiator)                                Node B (Peer)
         |                                                 |
         | -------- (1) SYN: 我知道的(节点, Gen, 最大的Version) ----> |
         |                                                 | (比对状态)
         | <------- (2) ACK: 你落后的数据实体 + 我需要你发的数据 ----- |
 (更新自己的状态)                                            |
         |                                                 |
         | -------- (3) ACK2: 你请求的最新数据实体 ----------------> |
         |                                                 | (更新自己的状态)
```

## 6.6 故障处理 (Fault Handling)

**概念详细描述：**
在分布式系统中，节点宕机是常态（硬盘损坏、网络分区等）。Cassandra 利用 Gossip 结合多种修复机制来保证极高的高可用性和最终一致性。

**核心算法与流程：**

**1. 故障检测 (Failure Detection - Phi Accrual Failure Detector)**
*   **算法思想：** 传统的故障检测是二元（Ping-Pong 超时判定），在网络抖动时极易误判。Cassandra 使用 Phi 累积故障检测器。它记录并统计来自某节点心跳（Gossip）到达的时间间隔历史，计算出一个正态分布。
*   **计算：** 计算出当前时间与上次心跳时间的差值，如果心跳延迟的概率极低（用 Phi 值表示，通常 Phi > 8 表示故障概率超过 99.9%），则将其标记为 DOWN。

**2. Hinted Handoff (提示移交)**
*   **目的：** 处理短暂的节点不可用。
*   **流程：** 
    1. 客户端发起写请求。
    2. Coordinator 发现需要写入的一个副本节点（如 Node B）宕机。
    3. Coordinator 不会报错，而是将写入数据（Hint）保存在本地的一个特殊目录中。
    4. Gossip 发现 Node B 重新上线，Coordinator 会将保存的 Hint 重新发送给 Node B（重放）。

**3. Anti-Entropy Repair (反熵数据修复 / 节点同步)**
*   **目的：** 处理长期的节点宕机（Hinted Handoff 通常只保存 3 小时，超时后不再重放），或者彻底的数据损坏，确保数据的最终一致性。
*   **核心算法 (Merkle Tree 默克尔树对比)：**
    1. 修复过程触发时，需要对比的两个副本节点会在本地为特定的 Token 范围生成 **Merkle Tree（哈希树）**。
    2. Merkle Tree 的叶子节点是具体数据的哈希，父节点是子节点哈希的组合哈希。
    3. 两台机器交换 Merkle Tree 的**根哈希（Root Hash）**。
    4. 如果根哈希一致，说明数据完全一致，流程结束（极低的带宽消耗）。
    5. 如果不一致，则自顶向下逐层对比，最终精确定位到不一致的那个数据块所在的叶子节点，然后只针对这小部分差异数据进行网络传输和同步。

**文本图表：Hinted Handoff 流程**
```text
           [Client]
              | 写请求
              v
       +--------------+
       | Coordinator  |
       |  (Node A)    |
       +--------------+
         /          \
  (写成功)           (写失败/超时)
       v              v
 +--------+      +--------+
 | Node C |      | Node B | (处于DOWN状态)
 +--------+      +--------+
   (副本)         /
               /
(本地持久化)  / 
[Hint: "属于 NodeB 的未写数据"]

(一段时间后，Node B 恢复，Gossip 传播状态 UP)
Coordinator A 读取 Hint -> 发送给 Node B -> Node B 追平数据
```
---

# 7. 持久化与恢复

## 一、 完整写入流程：速度与安全的平衡

分布式数据库将每一次写入都设计为“内存追加 + 顺序落盘”，避免了随机 I/O 带来的性能损耗，实现了极高的吞吐量和极低的写入延迟。

### 核心写入路径
`Client` → `Coordinator` → `Commit Log` → `Memtable` → `SSTable`

1. **Coordinator (协调者) 路由**
   - 客户端（Client）发起写请求到集群中的任意一个节点，该节点自动成为本次请求的 Coordinator。
   - Coordinator 根据数据的 Partition Key 计算 Hash，判断数据应该落在集群中的哪几个目标节点（Replica）上，并将请求转发给它们。
2. **Commit Log (提交日志) 持久化**
   - 目标节点收到数据后，首先**顺序追加**写入磁盘上的 Commit Log。
   - **意义**：这是保证数据不丢失的物理底线。顺序写的速度极快，最大程度减少了磁盘寻道时间。
3. **Memtable (内存表) 缓存**
   - 写入 Commit Log 的同时，数据也会被更新到内存中的数据结构（Memtable）。
   - **意义**：只要 Memtable 更新成功，节点即可向 Coordinator 汇报“写入成功”。这使得数据库的写入操作基本在内存级别完成，极大地提升了速度。
4. **SSTable (排序字符串表) 刷盘**
   - 随着数据不断写入，Memtable 最终会被写满（达到配置的阈值）。
   - 此时，系统会分配一个新的 Memtable 接收新写入，而旧的 Memtable 会变成不可变状态（Immutable），并在后台被 Flush（刷入）到磁盘，生成不可变的 SSTable 文件。

---

## 二、 节点宕机 (Crash) 的恢复逻辑

由于 Memtable 完全驻留在内存中，如果节点在 Memtable 刷盘变为 SSTable 之前发生突然断电或系统 Crash，内存中的数据将会丢失。此时，**Commit Log** 就会发挥“救生圈”的作用。

### 恢复流程 (Crash Recovery)
`Node Crash` → `Commit Log Replay` → `恢复 Memtable` → `继续工作`

1. **节点重启**：系统启动时，引擎检测到上次是非正常关闭或发现有未处理完的 Commit Log。
2. **Replay (重放)**：引擎会读取磁盘上尚未被标记为已完全刷盘的 Commit Log 记录。
3. **重建内存**：将日志中的写操作在内存中重新执行一遍，完整且精准地恢复出宕机前的 Memtable 状态。
4. **恢复服务**：数据找回后，节点继续正常的刷盘逻辑，并重新加入集群接受客户端的读写请求。

---

## 三、 数据可靠性保障矩阵

Cassandra 保证数据“不丢”且“高可用”，并不是单一机制的结果，而是多个子系统的组合：
* **Commit Log**：保证单节点宕机不丢近期数据（单机持久性）。
* **多副本 (Replication)**：保证单节点永久损坏时，集群其他节点仍有数据（可用性和容灾）。
* **Repair (反熵修复)**：保证多副本之间在长期运行中能够达到最终一致。
* **Snapshot / Backup**：防止人为误删或大规模机房灾难。

---

## 四、 多副本与 Repair (修复) 机制

在分布式理论中，“多副本”并不意味着“所有副本每一刻都完全一致”。网络抖动、节点瞬断、甚至高并发写入时的 Hinted Handoff 失败，都会导致不同节点上的数据版本产生分歧（例如节点 N1 拥有最新的 V2 数据，而节点 N2 错过了更新，仍然是 V1）。

为了保证数据的长期收敛（最终一致性），系统引入了 **Anti-Entropy (反熵) Repair** 机制。

### 为什么需要 Merkle Tree (默克尔树)？
如果节点之间直接全量对比几十 GB 甚至上 TB 的数据，会极其消耗 CPU 和网络带宽。Repair 的核心诉求是**“极小代价发现差异，精准同步缺失数据”**。为此，系统采用了 Merkle Tree。

`Replica` → `构建 Merkle Tree` → `比较 Hash (自顶向下)` → `发现差异分支` → `精准同步数据`

### Merkle Tree 的比对与同步流程
1. **数据分块与 Hash 计算**：系统将底层的真实数据按范围切分成多个小块（对应树的 Leaf Node），并为每个小块计算一个 Hash 值。
2. **向上聚合构建树**：相邻节点的 Hash 值两两组合，计算出父节点的 Hash，一层层向树顶聚合，最终在顶部生成唯一的“根哈希”（Root Hash）。
3. **快速比对 (Top-Down)**：
   - 节点 A 和节点 B 首先交换各自的 **Root Hash**。如果两者相同，说明树下面管理的所有数据完全一致，比对瞬间结束，无需网络传输。
   - 如果 Root Hash 不同，说明数据有差异。双方继续往下走一层，交换子节点的 Hash。
   - **顺藤摸瓜**：每次只需顺着 Hash 不同的分支往下找，直接跳过那些 Hash 相同的庞大分支。
4. **精准数据同步**：排查到最底层时，系统就能精确定位到究竟是哪几个小块（Leaf Node）的数据不一致。最终，节点之间只通过网络传输这几个不一致的数据块，以极小的网络开销完成了整个节点级别的数据修复。
---

# 8. 性能与场景

## 8.1 为什么 Cassandra 快？

### ① LSM

写入主要采用顺序追加：

```text
Commit Log
    ↓
Memtable
    ↓
SSTable
```

### ② 分布式

数据分散到多个节点，请求可以并行处理。

### ③ 水平扩展

增加节点可以扩展：

- 存储容量
- 处理能力
- 吞吐量

### ④ Bloom Filter + Index

减少无效 SSTable 查询。

---

## 8.2 适合场景

- 海量数据
- 高并发
- 高吞吐
- KV 查询
- 时间序列
- 日志
- IoT
- 用户行为数据
- 消息 / 事件数据
- 全球多机房部署
- 对高可用要求高的系统

## 8.3 不适合场景

- 复杂 JOIN
- 强依赖 ACID 事务
- 大量跨 Partition 操作
- 复杂报表查询
- 查询模式高度不确定
- 强依赖关系约束的业务

---

# 9. 容易遗漏的知识点

## 9.1 CAP

Cassandra 的设计重点是：

```text
Partition Tolerance
        +
Availability
        ↓
弱化传统强一致性
```

通常将 Cassandra 理解为：

> **偏 AP 的分布式数据库。**

但 Cassandra 又提供：

> **可调一致性 + LWT**

所以不能简单理解成“只能最终一致”。

## 9.2 Tombstone

Cassandra 删除数据通常不是立即从 SSTable 中物理删除，而是产生：

> **Tombstone（墓碑）**

之后在 Compaction 中清理。

大量 Tombstone 会导致：

```text
大量 Tombstone
      ↓
查询扫描更多数据
      ↓
性能下降
```

## 9.3 Hot Partition

Partition Key 设计不好：

```text
大量请求
    ↓
同一个 Partition
    ↓
同一个/少数节点
    ↓
热点
```

因此：

> **Partition Key 设计是 Cassandra 最重要的数据建模问题之一。**

## 9.4 Compaction Strategy

需要根据：

- 写入模式
- 读取模式
- TTL
- 时间序列特征

选择合适的 Compaction Strategy。

常见：

- STCS
- LCS
- TWCS

---

# 10. Cassandra 核心知识地图

```text
Cassandra
│
├── 1. 定位
│   └── 分布式 NoSQL / 宽列数据库
│
├── 2. 数据模型
│   ├── Keyspace
│   ├── Table
│   ├── Partition
│   ├── Row
│   └── Column
│
├── 3. 存储
│   ├── Commit Log
│   ├── Memtable
│   ├── SSTable
│   └── Compaction
│
├── 4. 查询
│   ├── Partition Key
│   ├── Clustering Key
│   ├── Bloom Filter
│   └── Index
│
├── 5. 并发 / 一致性 / 事务
│   ├── 多副本
│   ├── Tunable Consistency
│   ├── Eventual Consistency
│   ├── LWW
│   ├── LWT / Paxos
│   └── Batch
│
├── 6. 分布式
│   ├── 一致性 Hash
│   ├── Multi-Master
│   ├── Replication Factor
│   ├── Gossip
│   └── 故障检测
│
├── 7. 持久化 / 恢复
│   ├── Commit Log
│   ├── Replica
│   ├── Repair
│   ├── Snapshot
│   └── Backup
│
├── 8. 性能
│   ├── LSM
│   ├── 顺序写
│   ├── 水平扩展
│   ├── Bloom Filter
│   └── Compaction
│
└── 9. 常见问题
    ├── CAP
    ├── Tombstone
    ├── Hot Partition
    └── Compaction Strategy
```

---

# 11. 面试一句话总结

> **Cassandra 是一个无中心的分布式宽列数据库，通过一致性 Hash 将 Partition 分布到多个节点，并通过多副本实现高可用；底层采用 LSM-Tree，通过 Commit Log、Memtable、SSTable 实现高吞吐写入，通过 Bloom Filter 和 Index 优化读取；一致性方面默认以最终一致性为主，可以通过 Consistency Level 调节读写一致性，并通过 LWT/Paxos 支持特定场景下的线性一致性操作。它最大的优势是海量数据下的高吞吐、低延迟和水平扩展能力，但不适合复杂查询和跨 Partition 事务。**

## 面试最重要的一条主线

```text
数据怎么分？
    ↓
Partition Key
    ↓
怎么写？
    ↓
Commit Log → Memtable → SSTable
    ↓
怎么读？
    ↓
Bloom Filter → Index → SSTable
    ↓
怎么保证高可用？
    ↓
Replication + Gossip
    ↓
副本不一致怎么办？
    ↓
Consistency Level + Repair
    ↓
需要强一致操作怎么办？
    ↓
LWT / Paxos
    ↓
为什么能扩展？
    ↓
一致性 Hash + 多节点并行 + 水平扩展
```
