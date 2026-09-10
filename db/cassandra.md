# Cassandra 面试知识总结

> **定位：分布式、高可用、可水平扩展的 NoSQL 宽列数据库。**  
> **核心目标：海量数据 + 高吞吐 + 低延迟 + 高可用 + 水平扩展。**

Cassandra 的设计主要融合：
- **Dynamo**：一致性 Hash、多副本、Gossip、可调一致性
- **BigTable / HBase**：宽列数据模型、LSM 存储思想

---

# 1. 定位与数据模型

## 1.1 Cassandra 是什么？

| 维度 | 说明 |
|---|---|
| 类型 | 分布式 NoSQL / 宽列数据库 |
| 核心模型 | Partition-oriented Wide Column |
| 核心访问 | 根据 Partition Key 定位数据 |
| 一致性 | 最终一致性为主，可调一致性 |
| 架构 | 无中心、对等节点 |
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

| 概念 | 作用 |
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

- **Partition Key**：决定数据分布到哪个节点。
- **Clustering Key**：决定 Partition 内数据的排序和组织方式。

> **Cassandra 表设计的核心：围绕查询模式设计 Partition Key。**

---

# 2. 分布式架构与数据分布

## 2.1 一致性 Hash 与 Token Ring

Cassandra 使用一致性 Hash 将 `Partition Key` 映射到 Token Ring。

默认 `Murmur3Partitioner` 的 Token 范围为：

```text
-2^63 ～ 2^63 - 1
```

基本路由流程：

```text
Partition Key
      ↓
Hash
      ↓
Token
      ↓
Token Ring
      ↓
确定 Replica
```

与传统：

```text
hash(key) % N
```

相比，一致性 Hash 在节点增删时只需要迁移部分 Token 范围的数据，避免大量数据重新映射。

## 2.2 VNodes

VNodes（Virtual Nodes）将一个物理节点拆分成多个 Token 范围，分散在整个 Token Ring 上。

作用：

- 改善数据倾斜
- 扩容时平滑接管 Token
- 节点故障时降低恢复压力
- 改善集群负载均衡

> **注意：VNodes 的核心价值是把一个物理节点的责任拆散到多个 Token 范围，而不是简单理解成“一个节点拥有一个连续区间”。**

## 2.3 Masterless / Multi-Master

Cassandra 是完全去中心化的 P2P 架构：

- 没有固定 Master
- 所有节点地位基本平等
- 任意节点都可以接收客户端请求
- 请求接收节点临时充当 Coordinator

```text
Client
   ↓
Node A
(Coordinator)
   ├──→ Node B
   ├──→ Node C
   └──→ Node D
       Replica
```

### Coordinator

Coordinator 是**逻辑角色**，不是固定节点。

职责：

1. 根据 Partition Key 计算数据位置。
2. 找到目标副本。
3. 并行发送读写请求。
4. 根据 Consistency Level 等待足够副本响应。
5. 汇总结果并返回客户端。

## 2.4 Replication Factor

`RF` 表示一个 Partition 在集群中的副本数量。

例如：

```text
RF = 3

Partition
 ├── Replica 1
 ├── Replica 2
 └── Replica 3
```

生产环境常见 `RF = 3`，并结合拓扑策略将副本分散到不同故障域。

## 2.5 Replication Strategy

### SimpleStrategy

- 顺时针放置副本
- 不考虑机架 / 数据中心拓扑
- 适合开发测试或简单单 DC 场景

### NetworkTopologyStrategy

生产环境常用：

- 支持多 Data Center
- 考虑 Rack 拓扑
- 可分别配置各 DC 的 RF
- 尽量避免多个副本落在同一故障域

示意：

```text
DC1                         DC2
RF = 2                      RF = 1

Rack1       Rack2           Rack1
Node A      Node C          Node E
Node B      Node D          Node F

副本：
A → C → E
```

---

# 3. Gossip 与故障处理

## 3.1 Gossip

Cassandra 没有中心节点，因此节点需要通过 Gossip 交换：

- 集群拓扑
- 节点状态
- 节点版本信息

Gossip 状态使用：

- `Generation`：节点启动代次
- `Version`：状态版本

典型通信过程：

```text
Node A                         Node B

   SYN  ─────────────────────→
       携带节点状态摘要

        ←──────────────────── ACK
          状态差异 / 请求数据

   ACK2 ─────────────────────→
          最新状态数据
```

## 3.2 故障检测

Cassandra 使用 **Phi Accrual Failure Detector**。

核心思想：

```text
历史心跳间隔
      ↓
统计延迟分布
      ↓
计算 Phi
      ↓
判断节点是否疑似故障
```

相比简单的固定超时，Phi 能更好地适应网络抖动。

## 3.3 Hinted Handoff

用于处理**短暂节点不可用**。

```text
Client
  ↓
Coordinator
  ├──→ Node C  写成功
  └──→ Node B  不可用
             ↓
           Hint
             ↓
       Node B 恢复
             ↓
         重放 Hint
```

目的：避免因为临时节点故障导致副本长期缺失。

## 3.4 Repair / Anti-Entropy

Hinted Handoff 并不能覆盖所有长期不一致，因此 Cassandra 还需要 Repair。

核心思想：

```text
Replica
   ↓
构建 Merkle Tree
   ↓
比较 Hash
   ↓
定位不同分支
   ↓
同步差异数据
```

Merkle Tree：

```text
             Root
           /      \
       Hash AB   Hash CD
       /   \      /   \
     H(A) H(B)  H(C)  H(D)
```

如果两个副本：

```text
Root 相同 → 数据一致

Root 不同
   ↓
继续比较子节点
   ↓
定位差异数据
   ↓
只同步差异部分
```

> **Repair 是长期运行下保证副本最终收敛的重要机制。**

---

# 4. 存储引擎：LSM-Tree

Cassandra 使用 **LSM-Tree**，核心目标是把随机写转换为顺序写。

## 4.1 Write Path

```text
Client
  ↓
Coordinator
  ↓
Replica
  ↓
Commit Log
  ↓
Memtable
  ↓
Flush
  ↓
SSTable
  ↓
Compaction
```

### Commit Log

类似 WAL。

作用：

> **保证节点崩溃后的数据恢复。**

### Memtable

内存中的写缓存：

- 接收写入
- 组织数据
- 提供部分读取
- 达到阈值后 Flush

### SSTable

Memtable Flush 后生成：

- 磁盘文件
- 不可变
- 有序
- 支持压缩
- 包含 Index、Bloom Filter 等辅助结构

### Compaction

```text
SSTable 1
SSTable 2
SSTable 3
    ↓
Compaction
    ↓
新的 SSTable
```

作用：

- 合并 SSTable
- 清理旧版本
- 清理 Tombstone
- 优化读取

代价：

> **Write Amplification + 后台 I/O**

常见策略：

| 策略 | 典型特点 |
|---|---|
| STCS | Size-Tiered，适合较常规写入场景 |
| LCS | Leveled，强调读取性能和 SSTable 层级管理 |
| TWCS | Time-Window，适合时间序列 / TTL 数据 |

---

# 5. 查询与索引

## 5.1 Query-Driven Modeling

Cassandra 的核心设计思想：

> **先确定查询模式，再设计表。**

关系型数据库强调减少冗余，而 Cassandra 更倾向：

> **空间换时间 + 数据冗余 + 避免 JOIN**

常见原则：

1. 一张表服务一个主要查询模式。
2. 根据 Partition Key 设计数据分布。
3. 尽量让一次查询命中一个 Partition。
4. 可以通过多张冗余表支持不同查询路径。

例如：

```text
users_by_phone
users_by_email
orders_by_user
```

而不是依赖复杂 JOIN。

## 5.2 Partition Key

Partition Key 决定：

- 数据落在哪个 Token
- 数据由哪些节点承担
- 查询首先定位哪些副本

因此必须避免：

### Hot Partition

```text
大量请求
    ↓
同一个 Partition
    ↓
同一个/少数节点
    ↓
热点
```

例如使用低基数字段作为 Partition Key，可能造成大量数据集中。

## 5.3 Clustering Key

Clustering Key 决定 Partition 内部的数据排序。

适合：

- 时间排序
- 范围查询
- Partition 内有序读取

例如：

```sql
PRIMARY KEY ((sensor_id), recorded_at)
```

可以支持：

```sql
WHERE sensor_id = ?
  AND recorded_at >= ?
  AND recorded_at < ?
```

## 5.4 Read Path

典型读取过程：

```text
Partition Key
     ↓
Token / Replica
     ↓
Cache
     ↓
Bloom Filter
     ↓
Partition Index
     ↓
SSTable
     ↓
多版本合并
     ↓
返回最新结果
```

### Bloom Filter

Bloom Filter 用于判断：

> **某个 Partition Key 是否肯定不存在于某个 SSTable。**

```text
Bloom Filter
   ├── 不存在 → 跳过 SSTable
   └── 可能存在 → 继续 Index 查询
```

注意：

> Bloom Filter 存在误判，但不会把“存在”误判成“不存在”。

### SSTable Index

帮助从：

```text
Partition Key
      ↓
磁盘位置 / 数据范围
```

快速定位目标数据。

## 5.5 查询限制

Cassandra 不适合：

- 复杂 JOIN
- 大量跨 Partition 查询
- 不确定的查询模式
- 复杂动态聚合

如果查询缺少有效 Partition Key，可能产生：

```text
Coordinator
    ↓
Scatter-Gather
    ↓
多个节点
    ↓
结果汇总
```

这会带来明显的网络和计算开销。

## 5.6 二级索引与 SAI

### 传统 2i

传统二级索引存在一些问题：
Q
- 分布式查询成本高
- 跨节点查询可能产生 Scatter-Gather
- 高写入场景可能增加索引维护开销

### SAI

**Storage-Attached Indexing（SAI）** 将索引与 SSTable 存储结构结合。

相比传统 2i：

- 更贴近存储层
- 减少独立索引表带来的开销
- 支持更灵活的非主键查询
- Cassandra 5.0 中进一步增强

> SAI 并不意味着 Cassandra 变成了传统关系型数据库；查询驱动建模仍然是核心原则。

### ALLOW FILTERING

`ALLOW FILTERING` 可以允许 Cassandra 执行代价较高的过滤查询。

生产环境应谨慎使用，因为可能导致：

```text
大量数据读取
    ↓
过滤
    ↓
高 CPU / I/O / 网络开销
```

---

# 6. 并发、一致性与事务

## 6.1 CAP 定位

Cassandra 通常被理解为：

> **偏 AP 的分布式数据库。**

其设计重点：

```text
Partition Tolerance
        +
Availability
        ↓
弱化传统强一致性
```

但 Cassandra 并不是“只能最终一致”，它提供：

- 可调一致性
- LWT / Paxos

## 6.2 Tunable Consistency

Consistency Level 决定 Coordinator 需要等待多少副本确认。

| CL | 含义 | 特点 |
|---|---|---|
| ONE | 一个副本响应 | 低延迟、高可用、一致性较弱 |
| QUORUM | 多数副本响应 | 在合适读写配置下提供较强一致性 |
| LOCAL_QUORUM | 本 DC 多数副本响应 | 避免跨 DC 延迟，适合多 DC |
| ALL | 所有副本响应 | 一致性强，但可用性和延迟较差 |

对于 RF = 3：

```text
QUORUM = floor(3 / 2) + 1 = 2
```

### 多 DC

假设：

```text
北京 DC：RF = 3
纽约 DC：RF = 3
总 RF = 6
```

`QUORUM` 需要全局多数派：

```text
6 / 2 + 1 = 4
```

因此可能需要跨地域等待。

`LOCAL_QUORUM` 只需要本 DC 多数派：

```text
3 / 2 + 1 = 2
```

因此可以显著降低跨地域延迟。

## 6.3 最终一致性与副本修复

副本可能暂时存在版本差异，Cassandra 通过多种机制促进最终收敛：

1. **Hinted Handoff**：短暂故障后的补写。
2. **Read Repair**：读取过程中发现副本差异并修复。
3. **Anti-Entropy Repair**：通过 Repair 长期校正副本。

## 6.4 LWW

并发更新同一数据时，Cassandra 使用时间戳解决冲突：

> **Last Write Wins**

基本逻辑：

```text
写入 A timestamp = T1
写入 B timestamp = T2

T2 > T1
   ↓
B 覆盖 A
```

因此分布式系统中“谁先到达”并不一定决定最终值，时间戳会参与冲突解决。

## 6.5 Tombstone

删除通常不是立即物理删除，而是写入 Tombstone：

```text
DELETE
  ↓
Tombstone
  ↓
Compaction
  ↓
最终清理旧数据
```

大量 Tombstone 会导致：

- 查询扫描更多数据
- Compaction 压力增加
- I/O 增加
- 读取性能下降

因此 TTL 和频繁删除场景需要重点关注 Tombstone。

## 6.6 LWT / Paxos

Cassandra 不提供传统关系型数据库那种：

```text
BEGIN
  UPDATE ...
  UPDATE ...
COMMIT
```

式跨行 / 跨表 ACID 事务。

但对于：

> **Compare-And-Set（CAS）条件更新**

提供 **Lightweight Transactions（LWT）**。

LWT 基于 Paxos，用于实现线性一致性的条件操作。

例如：

```sql
UPDATE account
SET balance = 100
WHERE id = 1
IF balance = 50;
```

只有当：

```text
balance == 50
```

成立时更新才会成功。

典型过程：

```text
Prepare
   ↓
Read / Promise
   ↓
Propose
   ↓
Commit
```

代价：

> **LWT 比普通读写延迟高，因此只应该用于确实需要 CAS / 线性一致性的场景。**

## 6.7 Batch

Cassandra Batch **不能简单等同于传统事务**。

### Unlogged Batch

适合：

- 同一 Partition
- 减少多次网络请求

### Logged Batch

通过 Batchlog 提高批量操作的可靠性。

但需要区分：

> **Batch 的原子性语义 ≠ 传统关系数据库完整 ACID 事务。**

不要为了追求“事务”而把大量跨 Partition 操作塞进 Batch。

---

# 7. 持久化、崩溃恢复与备份

## 7.1 数据可靠性体系

```text
Commit Log
    ↓
单节点崩溃恢复

Replication
    ↓
节点级容灾

Repair
    ↓
副本长期收敛

Snapshot / Backup
    ↓
误删 / 大规模灾难恢复
```

## 7.2 Crash Recovery

如果节点在 Memtable Flush 前崩溃：

```text
Node Crash
    ↓
Commit Log Replay
    ↓
恢复 Memtable
    ↓
继续 Flush
    ↓
恢复服务
```

Commit Log 的意义：

> **Memtable 丢失后，可以通过日志重新构建近期写入。**

## 7.3 Snapshot / Backup

Replication 解决的是节点故障和副本容灾；它不能替代备份。

例如：

```text
误删除
    ↓
删除操作也会被复制
    ↓
所有副本都可能删除
```

因此仍需要 Snapshot / Backup 应对人为误操作和大规模灾难。

---

# 8. 性能与适用场景

## 8.1 为什么 Cassandra 写入性能高？

核心原因：

| 机制 | 作用 |
|---|---|
| LSM-Tree | 将随机写转换为顺序写 |
| Commit Log | 顺序追加，保证持久性 |
| Memtable | 内存写入 |
| SSTable | 批量顺序刷盘 |
| Masterless | 消除单点和 Master 瓶颈 |
| 水平扩展 | 节点增加即可扩展容量和吞吐 |
| Bloom Filter | 减少无效 SSTable 查询 |

核心链路：

```text
Write
 ↓
Commit Log + Memtable
 ↓
SSTable
 ↓
Compaction
```

## 8.2 适合场景

适合：

- 海量数据
- 高并发、高吞吐
- 写密集型业务
- KV 查询
- 日志
- 时间序列
- 用户行为数据
- 消息 / 事件数据
- IoT
- 多机房、多活
- 对高可用要求高的系统

典型 IoT：

```text
sensor_id
     ↓
Partition Key

recorded_at
     ↓
Clustering Key
```

可以高效查询：

> 某传感器过去 24 小时的数据，并按时间排序。

## 8.3 不适合场景

| 场景 | 原因 |
|---|---|
| 复杂 JOIN | 非关系型设计 |
| 强 ACID 事务 | 不适合跨行 / 跨表事务 |
| 大量跨 Partition 查询 | Scatter-Gather 成本高 |
| 复杂报表 | 不适合临时分析查询 |
| 查询模式高度不确定 | Query-Driven Modeling 难以满足 |
| 强关系约束 | 没有传统外键 / 关系约束模型 |

---

# 9. Cassandra 容易遗漏的关键知识点

## 9.1 Hot Partition

```text
低基数 Partition Key
        ↓
大量数据 / 请求
        ↓
少数 Partition
        ↓
少数节点
        ↓
Hotspot
```

> **Partition Key 是 Cassandra 数据建模最重要的设计点之一。**

## 9.2 Partition Size

除了避免 Hot Partition，还要控制单个 Partition 的大小。

理想上：

```text
合理 Partition Key
      +
合理 Partition Size
      ↓
均衡的数据分布
      +
可控的读取成本
```

不要让一个 Partition 无限增长。

## 9.3 Compaction

选择 Compaction Strategy 时需要考虑：

- 写入模式
- 读取模式
- TTL
- 数据是否具有明显时间窗口
- Tombstone 特征

## 9.4 Counter

Cassandra 提供 Counter 类型用于特定计数场景，但其语义和普通字段不同，不能简单当成传统数据库里的：

```sql
UPDATE count = count + 1
```

来理解。

使用前应明确 Counter 的并发、复制和业务语义。

## 9.5 Time Series 建模

时间序列数据通常不能简单使用：

```text
sensor_id
```

作为唯一 Partition Key 无限累积。

更合理的思路是：

```text
(sensor_id, time_bucket)
        ↓
Partition Key
```

例如按：

```text
sensor_id + day
sensor_id + hour
```

进行时间分桶，避免 Partition 无限增长。

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
│   ├── Column
│   ├── Partition Key
│   └── Clustering Key
│
├── 3. 分布式
│   ├── Consistent Hash
│   ├── Token Ring
│   ├── VNodes
│   ├── Masterless
│   ├── Replication Factor
│   └── Replication Strategy
│
├── 4. 节点通信 / 故障
│   ├── Gossip
│   ├── Failure Detection
│   ├── Hinted Handoff
│   └── Repair / Merkle Tree
│
├── 5. 存储
│   ├── Commit Log
│   ├── Memtable
│   ├── SSTable
│   └── Compaction
│
├── 6. 查询
│   ├── Query-Driven Modeling
│   ├── Partition Key
│   ├── Clustering Key
│   ├── Bloom Filter
│   ├── SSTable Index
│   ├── 2i
│   ├── SAI
│   └── ALLOW FILTERING
│
├── 7. 一致性 / 并发 / 事务
│   ├── CAP
│   ├── Consistency Level
│   ├── Eventual Consistency
│   ├── LWW
│   ├── Tombstone
│   ├── LWT / Paxos
│   └── Batch
│
├── 8. 可靠性
│   ├── Crash Recovery
│   ├── Repair
│   ├── Snapshot
│   └── Backup
│
└── 9. 性能 / 场景
    ├── 高吞吐
    ├── 低延迟
    ├── 水平扩展
    ├── Hot Partition
    ├── Partition Size
    ├── Time Series
    └── Compaction Strategy
```

---

# 11. 面试主线

面试时可以围绕下面这条主线展开：

```text
Cassandra 是什么？
        ↓
分布式 NoSQL / 宽列数据库
        ↓
数据怎么分？
        ↓
Partition Key
        ↓
怎么路由？
        ↓
Consistent Hash + Token Ring + VNodes
        ↓
怎么保证高可用？
        ↓
Replication + Masterless
        ↓
节点怎么发现 / 故障怎么处理？
        ↓
Gossip + Failure Detection + Hinted Handoff + Repair
        ↓
怎么写？
        ↓
Commit Log → Memtable → SSTable
        ↓
怎么读？
        ↓
Bloom Filter → Index → SSTable
        ↓
怎么保证一致性？
        ↓
Consistency Level + Replica Repair
        ↓
并发更新冲突怎么办？
        ↓
LWW
        ↓
需要条件更新 / 强一致怎么办？
        ↓
LWT / Paxos
        ↓
为什么能扩展？
        ↓
Masterless + Consistent Hash + 水平扩展
        ↓
为什么快？
        ↓
顺序写 + 内存 + 并行 + 分布式
        ↓
什么时候不用？
        ↓
复杂 JOIN / 强事务 / 大量跨 Partition 查询
```

---

# 12. 面试一句话总结

> **Cassandra 是一个无中心的分布式宽列数据库，通过一致性 Hash 将 Partition 分布到多个节点，并通过多副本实现高可用；底层采用 LSM-Tree，通过 Commit Log、Memtable、SSTable 实现高吞吐写入，通过 Bloom Filter 和 Index 优化读取；一致性方面以最终一致性为主要设计取向，并提供可调 Consistency Level 以及 LWT/Paxos 等机制支持特定场景下的更强一致性操作。它最大的优势是海量数据下的高吞吐、低延迟和水平扩展能力，但不适合复杂查询、频繁跨 Partition 操作和传统跨行/跨表 ACID 事务。**
