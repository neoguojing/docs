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

## 4.1 数据如何定位？

```text
Partition Key
      ↓
Hash
      ↓
Token
      ↓
Token Ring
      ↓
Replica Nodes
```

Cassandra 使用一致性 Hash 将数据分布到集群节点。

## 4.2 为什么 Partition Key 很重要？

Partition Key 决定：

1. 数据存储在哪些节点
2. 查询需要访问哪些节点
3. 数据是否均匀分布
4. 是否产生热点 Partition

因此：

> **设计 Cassandra 表时首先考虑业务查询模式，而不是追求一个通用的数据模型。**

## 4.3 查询特点

适合：

```text
WHERE partition_key = ?
```

以及 Partition 内基于 Clustering Key 的范围查询。

不擅长：

- JOIN
- 多表关联
- 任意条件查询
- 大量跨 Partition 查询
- 复杂事务

---

# 5. 并发、一致性与事务

## 5.1 多副本

假设：

```text
Replication Factor = 3

        Partition
        /   |   \
       N1   N2   N3
```

同一份数据保存多个副本。

目的：

- 高可用
- 容灾
- 数据冗余

## 5.2 可调一致性

常见 Consistency Level：

```text
ONE
QUORUM
ALL
LOCAL_QUORUM
EACH_QUORUM
...
```

例如 RF=3：

```text
ONE       → 1 个副本响应
QUORUM    → 2 个副本响应
ALL       → 3 个副本响应
```

注意：

> **Consistency Level 不是说只向这么多个副本发送请求，而是 Coordinator 等待多少个副本确认后向客户端返回。**

## 5.3 最终一致性

普通 Cassandra 数据采用最终一致性模型。

不同副本短时间内可能不一致：

```text
N1 → V2
N2 → V1
N3 → V2
```

之后通过副本同步机制趋于一致。

## 5.4 冲突解决

Cassandra 使用：

> **Last Write Wins（LWW）**

基于 timestamp 判断新旧：

```text
V1 @ T1
V2 @ T2

T2 > T1
 ↓
V2 wins
```

## 5.5 事务

Cassandra **不是传统关系数据库式的 ACID 事务数据库**。

不适合：

```text
BEGIN
  UPDATE A
  UPDATE B
COMMIT
```

这种跨 Partition 的传统事务。

### LWT

需要条件更新时，可以使用 Lightweight Transaction：

```sql
UPDATE users
SET balance = 100
WHERE id = 1
IF balance = 50;
```

LWT 基于 **Paxos**，提供线性一致性的 Compare-And-Set 能力。

特点：

```text
普通写
  ↓
高吞吐

LWT
  ↓
Paxos
  ↓
更强一致性
  ↓
更高延迟 / 更低吞吐
```

> **LWT 是特殊场景下的强一致操作，不应该把 Cassandra 当成传统事务数据库使用。**

## 5.6 Batch

Cassandra 支持 Batch，但：

> **Batch ≠ 跨 Partition ACID Transaction。**

Batch 更适合将相关写操作作为一个批次提交，而不是替代复杂业务事务。

---

# 6. 分布式与高可用

## 6.1 一致性 Hash

```text
Partition Key
      ↓
    Hash
      ↓
    Token
      ↓
  Token Ring
      ↓
Replica
```

数据根据 Token 分布到不同节点。

## 6.2 多主架构

Cassandra 没有传统意义上的 Master。

```text
Node A ←→ Node B
  ↕          ↕
Node C ←→ Node D
```

任何节点都可以接收请求，并作为：

> **Coordinator**

协调本次读写。

## 6.3 Replication Factor

例如：

```text
RF = 3
```

表示一个 Partition 有 3 个副本。

副本可以分布在：

- 不同节点
- 不同 Rack
- 不同 Data Center

## 6.4 Replication Strategy

### SimpleStrategy

简单 Token Ring 场景。

不适合复杂生产环境。

### NetworkTopologyStrategy

生产环境常用。

可以按照：

- Data Center
- Rack

等拓扑选择副本。

## 6.5 Gossip

节点之间通过 Gossip：

- 发现节点
- 传播节点状态
- 进行故障检测
- 维护集群 Membership

## 6.6 故障处理

```text
Gossip
  ↓
发现节点异常
  ↓
其他副本继续提供服务
  ↓
Hint / Repair 等机制
  ↓
副本最终同步
```

因此 Cassandra 可以做到：

> **节点故障时仍然保持较高的可用性。**

---

# 7. 持久化与恢复

完整写入流程：

```text
Client
  ↓
Coordinator
  ↓
Commit Log
  ↓
Memtable
  ↓
SSTable
```

Crash：

```text
Node Crash
    ↓
Commit Log Replay
    ↓
恢复 Memtable
    ↓
继续工作
```

数据可靠性主要来自：

```text
Commit Log
    +
多副本
    +
Repair
    +
Snapshot / Backup
```

## Repair

副本可能出现：

```text
N1 → V2
N2 → V1
N3 → V2
```

Repair 可以通过 **Merkle Tree** 等机制比较副本差异：

```text
Replica
   ↓
Merkle Tree
   ↓
比较 Hash
   ↓
发现差异
   ↓
同步数据
```

> **多副本 ≠ 副本永远一致，Repair 是保证副本长期收敛的重要机制。**

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
