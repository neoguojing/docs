
# ClickHouse 面试知识体系

> 按照「定位与数据模型 → 存储引擎 → 内存与数据结构 → 索引与查询 → 并发与一致性 → 分布式与高可用 → 持久化与恢复 → 性能与场景」整理。
>
> 核心目标：理解 ClickHouse **为什么快、怎么存、怎么查、怎么扩展、什么时候用**。

---

## 1. 定位与数据模型

### 1.1 ClickHouse 是什么？

ClickHouse 是面向 **OLAP（在线分析）** 的列式分布式数据库。

核心特点：

- **列式存储**：按列保存数据，分析查询只读取需要的列。
- **向量化执行**：一次处理一批数据，而不是逐行处理。
- **MPP / 分布式计算**：数据可以分布在多个节点并行查询。
- **高压缩率**：列中数据类型相同、相似度高，便于压缩。
- **面向分析场景优化**：聚合、扫描、统计、报表、日志分析等。
- **高吞吐写入 + 高吞吐查询**：尤其适合海量数据分析。

一句话：

> **ClickHouse = 面向海量数据分析的高性能列式 OLAP 数据库。**

---

### 1.2 OLTP vs OLAP

| 维度 | OLTP | OLAP / ClickHouse |
|---|---|---|
| 核心目标 | 事务处理 | 数据分析 |
| 数据量 | GB～TB | TB～PB+ |
| 查询特点 | 少量数据、点查/短查询 | 大量数据扫描、聚合 |
| 存储 | 行式常见 | 列式 |
| 更新 | 频繁 UPDATE/DELETE | 追加写为主 |
| 事务 | 强事务 | 弱于传统 OLTP |
| 典型数据库 | MySQL / PostgreSQL | ClickHouse / Doris |

---

### 1.3 数据模型

ClickHouse 典型模型：

```text
Database
  ↓
Table
  ↓
Columns
  ↓
Rows
```

但底层并不是传统数据库意义上的「一行一行存储」，而是：

```text
          Table
            ↓
       Data Parts
            ↓
       Columns
            ↓
       Granules
```

典型分析表：

```sql
CREATE TABLE events
(
    event_time DateTime,
    user_id UInt64,
    event_type String,
    amount Float64
)
ENGINE = MergeTree
ORDER BY (event_time, user_id);
```

---

### 1.4 ClickHouse 最重要的数据模型概念

#### Part

数据写入后形成独立的数据片段：

```text
INSERT
  ↓
Part
```

后台会不断进行 Part Merge。

#### Partition

用于将数据逻辑划分成更大的数据分区，例如：

```sql
PARTITION BY toYYYYMM(event_time)
```

得到：

```text
2026-01
2026-02
2026-03
...
```

主要作用：

- 数据管理
- 分区裁剪
- TTL
- 删除/归档

#### Primary Key / ORDER BY

ClickHouse 中非常重要。

```sql
ORDER BY (event_time, user_id)
```

它决定数据在 Part 内的排序方式，并影响：

- 主键索引
- 数据跳过
- 查询性能

> ClickHouse 的 Primary Key **不是传统 OLTP 数据库里的唯一约束索引**。

---

## 2. 存储引擎

### 2.1 MergeTree 是核心

ClickHouse 最重要的表引擎是：

```text
MergeTree
```

绝大多数 OLAP 场景首先考虑 MergeTree 家族。

核心思想：

```text
INSERT
  ↓
写入新 Part
  ↓
后台 Merge
  ↓
形成更大的 Part
```

---

### 2.2 为什么采用 Part？

避免每次写入都修改大量历史数据。

例如：

```text
INSERT 100万行
      ↓
   Part A

INSERT 50万行
      ↓
   Part B

后台：
A + B
 ↓
Merge
 ↓
Part C
```

这样写入路径简单，并且可以异步整理数据。

---

### 2.3 列式存储

传统行式：

```text
Row1: id time type amount
Row2: id time type amount
Row3: id time type amount
```

ClickHouse：

```text
id:
1
2
3

time:
...

type:
...

amount:
...
```

查询：

```sql
SELECT sum(amount)
FROM events;
```

只需要读取：

```text
amount
```

而不是整行。

---

### 2.4 数据压缩

列式存储非常适合压缩：

```text
同一列
↓
数据类型相同
↓
数据分布相似
↓
压缩效果好
```

ClickHouse 支持多种压缩算法，例如：

- LZ4
- ZSTD
- Delta
- DoubleDelta
- Gorilla

因此：

> **列式存储 + 数据类型优化 + 编码 + 压缩** 是 ClickHouse 高吞吐的重要基础。

---

### 2.5 Merge

后台 Merge 是 MergeTree 的核心机制。

```text
Part A
Part B
Part C
 ↓
Background Merge
 ↓
Part D
```

Merge 的作用：

- 合并小 Part
- 减少 Part 数量
- 重新整理数据
- 提高查询效率
- 执行部分引擎相关的数据整理

因此 ClickHouse 的数据组织不是「写完就固定」，而是：

> **写入形成 Part，后台持续 Merge。**

---

### 2.6 常见 MergeTree 家族

| 引擎 | 主要用途 |
|---|---|
| MergeTree | 基础 OLAP |
| ReplacingMergeTree | 去重 / 最终保留版本 |
| SummingMergeTree | 数值聚合 |
| AggregatingMergeTree | 聚合状态 |
| CollapsingMergeTree | 状态折叠 |
| VersionedCollapsingMergeTree | 带版本的状态折叠 |
| ReplicatedMergeTree | 副本场景 |

面试重点：

> 不要死记所有引擎，重点理解 **MergeTree + ReplacingMergeTree + ReplicatedMergeTree**。

---

## 3. 内存与数据结构

### 3.1 ClickHouse 为什么能够高速分析？

核心不是单纯「全部放内存」。

而是：

```text
列式存储
+
压缩
+
Primary Key Index
+
Data Skipping
+
向量化执行
+
并行计算
+
高效聚合
```

---

### 3.2 内存主要用于什么？

查询过程中主要使用内存：

- 查询执行
- Block
- 聚合 Hash Table
- Sort
- Join
- 临时结果
- Buffer
- Cache

因此复杂 OLAP 查询可能消耗大量内存。

---

### 3.3 Block

ClickHouse 不倾向于逐行处理：

```text
Row
Row
Row
...
```

而是：

```text
Block
 ├── Column A
 ├── Column B
 ├── Column C
 └── ...
```

一次处理一批数据。

这就是向量化执行的重要基础。

---

### 3.4 向量化执行

传统：

```text
for each row:
    calculate()
```

ClickHouse：

```text
Block
 ↓
Vectorized Processing
 ↓
一次处理大量数据
```

减少：

- 函数调用
- 分支判断
- CPU cache miss
- 解释执行开销

并更容易利用 CPU SIMD。

---

### 3.5 聚合 Hash Table

例如：

```sql
SELECT
    user_id,
    sum(amount)
FROM events
GROUP BY user_id;
```

执行过程中通常需要维护：

```text
HashMap
user_id → aggregate state
```

数据量很大时：

```text
HashMap
 ↓
内存增长
 ↓
达到阈值
 ↓
可能进行外部聚合
```

因此：

> ClickHouse 的聚合性能很高，但大 GROUP BY 仍然可能受到内存限制。

---

## 4. 索引与查询

### 4.1 ClickHouse 的索引思想

ClickHouse 与 MySQL 最大区别之一：

> **不是依赖 B+Tree 索引进行大量点查，而是通过排序 + 稀疏主键索引 + Data Skipping 减少扫描数据。**

核心流程：

```text
SQL
 ↓
Partition Pruning
 ↓
Primary Key / Sparse Index
 ↓
Data Skipping
 ↓
读取需要的列
 ↓
向量化执行
 ↓
Aggregation / Join
 ↓
Result
```

---

### 4.2 Sparse Primary Key Index

假设：

```sql
ORDER BY (event_time, user_id)
```

数据已经按照这个顺序排序。

ClickHouse 使用稀疏索引定位可能的数据范围，而不是给每一行建立索引。

因此：

```text
Index
 ↓
定位可能相关的 Granule
 ↓
跳过大量无关数据
```

---

### 4.3 Granule

Part 内部进一步划分为 Granule。

可以理解为：

```text
Part
 ├── Granule 1
 ├── Granule 2
 ├── Granule 3
 └── ...
```

索引主要帮助判断：

> 某个 Granule 有没有可能包含目标数据？

如果不可能：

```text
Skip
```

---

### 4.4 Data Skipping Index

除了主键索引，还可以使用 Data Skipping Index。

常见类型：

- minmax
- set
- bloom_filter
- ngrambf_v1
- tokenbf_v1

例如：

```text
Granule
min(time) = 10:00
max(time) = 10:10
```

查询：

```sql
WHERE time > 11:00
```

则该 Granule 可以直接跳过。

---

### 4.5 ORDER BY 如何设计？

这是 ClickHouse 最重要的设计问题之一。

例如查询经常：

```sql
WHERE tenant_id = ?
  AND event_time BETWEEN ? AND ?
```

可以考虑：

```sql
ORDER BY (tenant_id, event_time)
```

原则：

> **让高频过滤条件尽量出现在 ORDER BY 前部，并结合实际查询模式设计。**

不要简单按照：

```text
MySQL 索引思维
```

设计 ClickHouse ORDER BY。

---

### 4.6 查询执行

典型流程：

```text
SQL
 ↓
Parser
 ↓
Analyzer / Planner
 ↓
Query Plan
 ↓
Read From Storage
 ↓
Filter
 ↓
Aggregation / Join / Sort
 ↓
Distributed Merge
 ↓
Result
```

其中最重要的是：

> **尽可能减少需要读取和处理的数据量。**

---

### 4.7 为什么 SELECT * 很慢？

因为列式数据库的优势是：

```text
SELECT a
```

只读取：

```text
a
```

而：

```sql
SELECT *
```

需要读取大量列。

所以 ClickHouse 查询应该：

> **只查询需要的列。**

---

## 5. 并发与一致性

### 5.1 ClickHouse 是否支持事务？

ClickHouse 不是传统 OLTP 数据库。

因此：

- 不适合复杂事务
- 不以多行 ACID 事务为核心
- 更适合批量 INSERT
- 更适合追加写
- UPDATE / DELETE 成本相对高

核心定位：

> **分析优先，而不是事务优先。**

---

### 5.2 写入并发

多个 INSERT 可以并发执行：

```text
Client A → Part A
Client B → Part B
Client C → Part C
```

后台再：

```text
A + B + C
 ↓
Merge
```

这种模型避免大量写请求直接修改同一份历史数据。

---

### 5.3 UPDATE / DELETE

ClickHouse 支持 UPDATE / DELETE，但不要按照 MySQL 的思维大量使用。

因为列式存储 + Part 机制下，修改历史数据可能需要较大的数据重写成本。

因此典型设计是：

```text
业务数据
 ↓
追加写
 ↓
版本字段 / 状态字段
 ↓
查询时选择最终状态
```

例如：

```text
ReplacingMergeTree
```

用于某些去重/最终状态场景。

---

### 5.4 一致性

单机和分布式场景需要分别考虑。

ClickHouse 更强调：

```text
高吞吐
高可用
最终整理
```

而不是：

```text
强事务一致性
```

使用 ReplicatedMergeTree 等复制机制可以实现副本同步和故障恢复能力。

---

### 5.5 ReplacingMergeTree

典型：

```text
id=1 version=1
id=1 version=2
```

写入：

```text
两条记录都可能暂时存在
```

后台 Merge：

```text
Merge
 ↓
根据规则保留最终版本
```

因此：

> **ReplacingMergeTree 的“去重”不是传统唯一索引意义上的实时去重。**

必要时可以使用 `FINAL` 获取最终结果，但 `FINAL` 可能增加查询成本。

---

## 6. 分布式与高可用

### 6.1 ClickHouse 如何扩展？

典型架构：

```text
                Client
                  ↓
             Query Node
                  ↓
       ┌──────────┼──────────┐
       ↓          ↓          ↓
    Shard 1    Shard 2    Shard 3
       ↓          ↓          ↓
    Replica     Replica     Replica
```

两个核心概念：

- **Shard：分片**
- **Replica：副本**

---

### 6.2 Shard

Shard 用于：

> **水平切分数据，扩大存储和计算能力。**

例如：

```text
10亿数据

Shard 1 → 3亿
Shard 2 → 3亿
Shard 3 → 4亿
```

查询时：

```text
Query
 ↓
多个 Shard 并行执行
 ↓
Merge Result
```

---

### 6.3 Replica

Replica 用于：

> **数据冗余和高可用。**

例如：

```text
Shard 1
 ├── Replica 1
 └── Replica 2
```

一个节点故障：

```text
Replica 1 ❌
Replica 2 ✅
```

可以继续提供服务。

---

### 6.4 Distributed 表

常见架构：

```text
Distributed Table
      ↓
┌─────┼─────┐
↓     ↓     ↓
S1    S2    S3
```

Distributed 表主要负责：

- 分发查询
- 分发写入
- 汇总结果

真正的数据通常存储在本地 MergeTree 表中。

---

### 6.5 分布式查询

例如：

```sql
SELECT
    user_id,
    sum(amount)
FROM distributed_events
GROUP BY user_id;
```

大致：

```text
Coordinator
    ↓
Shard 1 → local aggregation
Shard 2 → local aggregation
Shard 3 → local aggregation
    ↓
Coordinator
    ↓
Merge aggregation
    ↓
Final Result
```

核心思想：

> **尽可能让计算靠近数据，先局部聚合，再汇总。**

---

### 6.6 ZooKeeper / ClickHouse Keeper

复制、DDL 等分布式协调场景需要协调服务。

现代 ClickHouse 常见：

```text
ClickHouse Keeper
```

用于协调：

- Replica
- Replication Log
- 分布式元数据
- 部分 DDL 协调

---

### 6.7 扩容

ClickHouse 横向扩展主要：

```text
增加 Shard
```

但是扩容并不意味着历史数据自动完美均衡。

需要考虑：

- 数据重新分布
- Sharding Key
- 写入路由
- 查询路由
- 历史数据迁移

所以：

> **Sharding Key 是 ClickHouse 分布式设计中的关键。**

---

## 7. 持久化与恢复

### 7.1 数据如何落盘？

典型流程：

```text
INSERT
 ↓
写入 Part
 ↓
磁盘
 ↓
后台 Merge
 ↓
生成新的 Part
```

数据最终存储在磁盘上的 Part 中。

---

### 7.2 Crash 后怎么办？

因为数据不是只存在内存中，而是持久化到磁盘。

正常情况下：

```text
Memory
  ↓
Write
  ↓
Disk Part
```

Crash：

```text
Process Crash
 ↓
重新启动
 ↓
扫描已有 Parts
 ↓
恢复可用数据
```

对于已经成功持久化的数据，可以从磁盘恢复。

---

### 7.3 WAL / 写入一致性

ClickHouse 的持久化机制不能简单理解成传统 OLTP 数据库的：

```text
Transaction
 ↓
WAL
 ↓
Page
```

它更接近：

```text
INSERT
 ↓
构建 Part
 ↓
写入磁盘
 ↓
Part 成为可见数据
```

不同版本和配置下具体写入路径会有所差异。

面试时重点理解：

> **ClickHouse 通过 Part + 磁盘持久化 + 副本机制保证数据可靠性，而不是依赖传统 OLTP 事务模型。**

---

### 7.4 副本恢复

ReplicatedMergeTree：

```text
Replica A
Replica B
```

其中一个故障：

```text
A ❌
B ✅
```

A 恢复：

```text
A restart
 ↓
发现缺失 Part
 ↓
从其他 Replica 获取
 ↓
恢复数据
```

---

### 7.5 Backup

生产环境仍然需要：

- Backup
- Object Storage
- Snapshot
- 跨机房容灾

> **副本 ≠ 备份。**

副本主要解决：

```text
节点故障
```

备份主要解决：

```text
误删除
数据损坏
灾难恢复
```

---

## 8. 性能与场景

### 8.1 ClickHouse 为什么快？

可以总结成 7 点：

| 原因 | 作用 |
|---|---|
| 列式存储 | 少读数据 |
| 数据压缩 | 少 IO |
| ORDER BY | 快速定位数据范围 |
| Sparse Index | 跳过无关数据 |
| 向量化 | 提高 CPU 利用率 |
| 并行执行 | 利用多核 |
| 分布式 | 多节点并行 |

核心公式：

```text
ClickHouse 快
=
少读
+
少算
+
并行算
+
高效算
```

---

### 8.2 适合什么场景？

非常适合：

- 日志分析
- 埋点分析
- 用户行为分析
- BI 报表
- 实时 OLAP
- 广告分析
- 风控分析
- 监控指标
- IoT 数据
- 海量事件分析
- 时间序列分析

典型数据：

```text
用户
 ↓
行为事件
 ↓
数十亿 / 数百亿记录
 ↓
按时间、用户、地区、业务维度统计
```

---

### 8.3 不适合什么？

不适合：

- 强事务业务
- 高频单行 UPDATE
- 高频单行 DELETE
- 强一致性 OLTP
- 复杂事务
- 大量随机点查
- 强依赖外键约束的业务

例如：

```text
订单创建
支付
库存扣减
账户余额
```

通常更适合：

```text
MySQL / PostgreSQL
```

而不是直接用 ClickHouse。

---

## 9. ClickHouse 核心知识串联

面试可以用下面这条链路理解整个 ClickHouse：

```text
                    ClickHouse
                         │
              ┌──────────┴──────────┐
              ↓                     ↓
          OLAP 分析              分布式
              │                     │
              ↓                     ↓
        Columnar Storage        Shard + Replica
              │                     │
              ↓                     ↓
          MergeTree             Distributed
              │
              ↓
        Part + Merge
              │
              ↓
      ORDER BY / Primary Key
              │
              ↓
       Sparse Index / Granule
              │
              ↓
        Data Skipping
              │
              ↓
        少量数据读取
              │
              ↓
         Block / Vector
              │
              ↓
        Parallel Execution
              │
              ↓
        Aggregation / Join
              │
              ↓
           Result
```

---

## 10. 面试重点：必须理解的 10 个问题

### ① ClickHouse 为什么快？

> 列式存储减少 IO，压缩降低数据量，ORDER BY + 稀疏索引 + Data Skipping 减少扫描，向量化和并行执行提高 CPU 利用率，分布式架构进一步扩大吞吐。

### ② MergeTree 是什么？

> ClickHouse 最核心的表引擎。数据写入形成 Part，后台通过 Merge 合并 Part，并基于 ORDER BY 建立稀疏索引。

### ③ Part 是什么？

> 数据写入产生的独立数据片段，是 MergeTree 数据组织和后台 Merge 的基本单位。

### ④ Partition 和 ORDER BY 有什么区别？

> Partition 是更大的数据管理/裁剪边界；ORDER BY 决定 Part 内的数据排序和主键索引组织。两者作用不同，不能互相替代。

### ⑤ ClickHouse 为什么不需要像 MySQL 一样大量 B+Tree？

> OLAP 查询通常扫描大量数据。ClickHouse 通过列式存储、排序、稀疏索引和 Data Skipping 减少扫描范围，更适合分析型查询。

### ⑥ ReplacingMergeTree 是实时去重吗？

> 不是。它主要在 Merge 阶段按照版本等规则最终整理数据，数据写入后短时间内可能存在重复记录。

### ⑦ Shard 和 Replica 的区别？

> Shard 是分片，解决数据和计算规模；Replica 是副本，解决冗余和高可用。

### ⑧ Distributed 表做什么？

> 提供分布式查询/写入入口，将请求分发到多个 Shard，并汇总结果；真正的数据通常存储在本地 MergeTree 表中。

### ⑨ ClickHouse 支持事务吗？

> 支持程度和模型不同于传统 OLTP 数据库，不应把它当成强事务数据库使用。ClickHouse 更适合批量、追加式分析数据。

### ⑩ ClickHouse 和 MySQL 怎么选？

```text
业务事务 / 点查 / 高频更新
        ↓
      MySQL

海量数据 / 聚合 / 报表 / 日志分析
        ↓
    ClickHouse
```

---

## 11. 一张图理解 ClickHouse

```text
                    ┌──────────────┐
                    │   Client     │
                    └──────┬───────┘
                           ↓
                    ┌──────────────┐
                    │ SQL / Query  │
                    └──────┬───────┘
                           ↓
                ┌─────────────────────┐
                │ Query Planner        │
                └──────────┬──────────┘
                           ↓
             Partition Pruning / Index
                           ↓
                ┌─────────────────────┐
                │ MergeTree Parts      │
                │                      │
                │ Part                 │
                │  ├─ Column           │
                │  ├─ Granule          │
                │  └─ Sparse Index     │
                └──────────┬──────────┘
                           ↓
                    Data Skipping
                           ↓
                   Read Required Columns
                           ↓
                     Block / Vector
                           ↓
                  Parallel Processing
                           ↓
                 Aggregation / Join
                           ↓
                       Result


写入：

INSERT
  ↓
Part
  ↓
Disk
  ↓
Background Merge
  ↓
Larger Part


分布式：

                Distributed Table
                       ↓
          ┌────────────┼────────────┐
          ↓            ↓            ↓
       Shard 1      Shard 2      Shard 3
          ↓            ↓            ↓
      Replica       Replica       Replica
```

---

## 12. 最终记忆模型

如果只记住 ClickHouse 的核心，可以记住：

```text
1. 定位
   OLAP / 海量数据分析

2. 存储
   Columnar + MergeTree + Part

3. 数据组织
   Partition + ORDER BY + Granule

4. 查询优化
   Sparse Index + Data Skipping

5. 执行
   Block + Vectorization + Parallelism

6. 分布式
   Shard + Replica + Distributed

7. 一致性
   弱于 OLTP，追加写/最终整理思路

8. 持久化
   Part 落盘 + Replica + Backup

9. 性能
   少读 + 少算 + 并行 + 压缩

10. 场景
   日志 / BI / 埋点 / 实时分析 / 海量聚合
```

> **一句话总结：**
>
> ClickHouse 通过 **列式存储减少读取量，ORDER BY + 稀疏索引 + Data Skipping 减少扫描量，向量化 + 并行执行提高计算效率，再通过 Shard + Replica 实现分布式扩展和高可用**，因此特别适合海量数据 OLAP，而不适合传统强事务 OLTP。
