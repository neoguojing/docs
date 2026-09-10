# ClickHouse 面试知识体系

> 核心目标：理解 ClickHouse **为什么快、怎么存、怎么查、怎么扩展、什么时候用**。
>
> 主线：**定位与数据模型 → 存储引擎 → 内存与执行 → 索引与查询 → 并发与一致性 → 分布式与高可用 → 持久化与恢复 → 性能与场景 → 面试问题**

---

## 1. 定位与数据模型

### 1.1 ClickHouse 是什么？

ClickHouse 是面向 **OLAP（在线分析处理）** 的高性能列式数据库，适合海量数据的扫描、聚合、统计和实时分析。

核心特点：

- **列式存储**：只读取查询涉及的列。
- **高压缩率**：同列数据类型一致，便于编码和压缩。
- **向量化执行**：以 Block 为单位批量处理数据，充分利用 SIMD。
- **并行执行**：充分利用多核 CPU，并支持分布式计算。
- **稀疏索引 + Data Skipping**：尽量跳过无关数据。
- **高吞吐写入**：以批量、追加写为主要方式。

> **一句话：ClickHouse = 面向海量数据分析的高性能列式 OLAP 数据库。**

### 1.2 OLTP vs OLAP

| 维度 | OLTP | OLAP / ClickHouse |
|---|---|---|
| 核心目标 | 事务处理 | 数据分析 |
| 查询特点 | 点查、短查询 | 大量扫描、聚合 |
| 存储 | 行式常见 | 列式 |
| 写入 | 高频小事务 | 批量追加为主 |
| 更新 | 频繁 UPDATE/DELETE | 不适合高频行级修改 |
| 事务 | 强 ACID | 不以传统事务为核心 |
| 典型场景 | 订单、支付、库存 | 日志、BI、埋点、报表 |

### 1.3 数据组织

逻辑上：

```text
Database
   ↓
Table
   ↓
Columns / Rows
```

物理上更重要的是：

```text
Table
  ↓
Partition
  ↓
Data Part
  ↓
Column
  ↓
Granule
  ↓
Compressed Block
```

典型建表：

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

### 1.4 Part、Partition、ORDER BY

#### Part

一次批量写入会形成独立的 **Data Part**。

```text
INSERT
  ↓
Part
  ↓
Background Merge
  ↓
Larger Part
```

Part 是 MergeTree 的基本物理存储和合并单位。

#### Partition

Partition 是更大的数据管理边界，例如：

```sql
PARTITION BY toYYYYMM(event_time)
```

作用：

- 分区裁剪
- 数据生命周期管理
- TTL
- 删除 / 归档

#### ORDER BY / Primary Key

```sql
ORDER BY (tenant_id, event_time)
```

决定 Part 内的数据排序，并影响稀疏主键索引和查询数据跳过。

> ClickHouse 的 Primary Key **不是传统 OLTP 中保证唯一性的 B+Tree 索引**。

---

## 2. 存储引擎

### 2.1 MergeTree 核心思想

MergeTree 是 ClickHouse 最核心的表引擎家族。

核心机制：

1. **Append-only**：写入形成新的 Part，而不是原地修改旧数据。
2. **Immutable Part**：Part 写入后基本不可变。
3. **Background Merge**：后台不断把多个小 Part 合并成更大的 Part。
4. **ORDER BY**：每个 Part 内部按照排序键组织数据。

因此：

```text
批量 INSERT
    ↓
生成 Part
    ↓
多个 Part 并存
    ↓
后台 Merge
    ↓
更大的 Part
```

优势：

- 写入路径简单
- 减少随机写
- 降低读写锁竞争
- Merge 异步完成

> **面试重点：不要高频单条 INSERT。** 大量小 INSERT 会产生大量小 Part，增加文件、Merge 和协调开销；应尽量批量写入。

### 2.2 Merge 过程

多个 Part：

```text
Part A: [1,3,5]
Part B: [2,3,6]
       ↓ Merge
Part C: [1,2,3,3,5,6]
```

基本流程：

1. 选择同一 Partition 中需要合并的 Parts。
2. 利用各 Part 已按 ORDER BY 排序的特点进行归并。
3. 根据具体 MergeTree 引擎执行去重、聚合或折叠逻辑。
4. 生成新的 Part。
5. 旧 Part 标记为 inactive，之后清理。

### 2.3 MergeTree 家族

| 引擎 | Merge 阶段主要行为 | 典型场景 |
|---|---|---|
| `MergeTree` | 保留数据，不做去重 | 日志、事件明细 |
| `ReplacingMergeTree` | 按规则保留版本 | 最新状态、去重 |
| `SummingMergeTree` | 相同排序键的数值聚合 | 预聚合指标 |
| `AggregatingMergeTree` | 合并聚合状态 | UV、分位数等复杂聚合 |
| `CollapsingMergeTree` | 通过 Sign 折叠记录 | 状态变化、异步更新 |
| `VersionedCollapsingMergeTree` | 在折叠基础上引入版本 | 乱序更新 |

### 2.4 Replicated 系列

例如：

```text
ReplicatedMergeTree
ReplicatedReplacingMergeTree
```

在 MergeTree 基础上增加副本协调与数据复制能力，通常配合 **ClickHouse Keeper / ZooKeeper**。

---

## 3. 内存与执行

### 3.1 ClickHouse 为什么快？

可以归纳为：

```text
列式存储
   ↓
少读取列
   ↓
压缩
   ↓
减少磁盘 IO
   ↓
ORDER BY + Sparse Index + Data Skipping
   ↓
减少扫描
   ↓
Block + Vectorization
   ↓
高效 CPU 计算
   ↓
Parallelism
   ↓
充分利用多核
```

### 3.2 Block：计算基本单位

ClickHouse 不是传统的逐行处理，而是以 **Block** 为主要计算单位。

```text
Block
├── Column A [a1,a2,a3,...]
├── Column B [b1,b2,b3,...]
└── Column C [c1,c2,c3,...]
```

列数组连续存放，有利于：

- CPU Cache
- 向量化执行
- SIMD
- 批量过滤和计算

### 3.3 向量化执行

传统标量方式：

```text
A0+B0
A1+B1
A2+B2
A3+B3
```

向量化：

```text
[A0,A1,A2,A3]
        +
[B0,B1,B2,B3]
        ↓
[C0,C1,C2,C3]
```

核心思想：

> **一次处理一批数据，而不是一次处理一行。**

### 3.4 编码与压缩

通常采用：

```text
原始数据
   ↓
Encoding
   ↓
Compression
   ↓
磁盘
```

常见编码：

- **Delta**：适合时间戳、递增 ID 等具有连续性的数值。
- **LowCardinality / 字典编码**：适合状态、类型等低基数字符串。

常见压缩：

- **LZ4**：解压速度快，适合性能优先场景。
- **ZSTD**：压缩率更高，适合更关注存储成本的场景。

### 3.5 GROUP BY 与 Hash Table

例如：

```sql
SELECT user_id, sum(amount)
FROM events
GROUP BY user_id;
```

执行思路：

```text
读取 Block
   ↓
计算 user_id Hash
   ↓
Hash Table
   ├── 不存在 → 创建聚合状态
   └── 已存在 → 更新聚合状态
```

主要风险：

> 高基数 GROUP BY 会导致 Hash Table 占用大量内存。

内存不足时可以进行 **External Aggregation / Spill**，把中间结果落盘后再归并，但会明显增加 IO。

### 3.6 JOIN 与内存

Hash Join 的典型过程：

```text
右表
 ↓
Build Hash Table
 ↓
内存

左表
 ↓
Block 读取
 ↓
Probe Hash Table
```

因此 Join 的主要风险是内存。

常见优化：

1. **小表放右边**。
2. 根据场景使用其他 Join Algorithm。
3. 高频维表关联可考虑 **Dictionaries**。
4. 大量关联场景优先考虑**宽表 / 反范式化**。
5. 分布式 JOIN 要特别关注数据分布和网络广播成本。

---

## 4. 索引与查询

### 4.1 ClickHouse 的索引思想

ClickHouse 不依赖传统 B+Tree 为每一行建立索引，而是：

```text
ORDER BY
   ↓
Sparse Primary Key Index
   ↓
Data Skipping
   ↓
减少需要读取的数据
```

核心目标不是“精准定位一行”，而是：

> **尽可能快速排除不需要扫描的数据范围。**

### 4.2 Granule 与稀疏主键索引

数据按照排序键组织后划分为 Granule。

默认情况下，一个 Granule 通常约为 **8192 行**（实际粒度可配置）。

索引记录 Granule 的关键排序值，用于快速判断目标可能落在哪些 Granule。

```text
Sparse Index
  ↓
定位可能的数据范围
  ↓
Mark
  ↓
定位数据文件中的位置
  ↓
读取 Granule
```

### 4.3 Part 文件结构

一个 Part 中可以看到类似：

```text
Part/
├── primary.idx
├── count.txt
├── user_id.bin
├── user_id.mrk2
├── event_type.bin
└── event_type.mrk2
```

核心关系：

```text
primary.idx
    ↓
Mark
    ↓
.bin
```

- `primary.idx`：稀疏主键索引。
- `.mrk/.mrk2`：记录数据在列文件中的定位信息。
- `.bin`：实际列数据。

因此可以理解为：

```text
Sparse Index
    ↓
找到 Granule
    ↓
Mark 定位
    ↓
读取对应 Compressed Block
    ↓
解压
    ↓
Block 计算
```

### 4.4 Data Skipping Index

当过滤条件不是 ORDER BY 的有效前缀时，可以使用 Data Skipping Index。

常见类型：

| 类型 | 适用场景 |
|---|---|
| `minmax` | 范围查询、具有局部连续性的数值/时间 |
| `set` | 低基数枚举值、IN 查询 |
| `bloom_filter` | 高基数离散值、点匹配 |

例如：

```sql
INDEX amt_idx total_amount
TYPE minmax
GRANULARITY 2
```

如果一个数据范围：

```text
max(total_amount) = 5000
```

而查询：

```sql
WHERE total_amount > 10000
```

则该范围可以直接跳过。

### 4.5 ORDER BY 如何设计？

例如高频查询：

```sql
WHERE tenant_id = ?
  AND event_time BETWEEN ? AND ?
```

可以考虑：

```sql
ORDER BY (tenant_id, event_time)
```

基本原则：

- 高频过滤字段尽量靠前。
- 充分考虑查询模式，而不是照搬 MySQL 索引设计。
- 利用排序让相关数据尽可能集中，从而提高数据跳过效率。

### 4.6 查询执行流程

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
Partition Pruning
 ↓
Sparse Index / Data Skipping
 ↓
读取需要的列
 ↓
Filter
 ↓
Aggregation / Join / Sort
 ↓
Distributed Merge
 ↓
Result
```

### 4.7 为什么不建议 SELECT *？

列式存储最大的优势是：

```text
SELECT amount
      ↓
只读取 amount 列
```

而：

```sql
SELECT *
```

会读取大量列，增加：

- 磁盘 IO
- 解压成本
- 内存占用
- CPU 计算

> **OLAP 查询应尽量只读取真正需要的列。**

---

## 5. 并发与一致性

### 5.1 事务定位

ClickHouse 的设计重点是：

```text
批量写入
+
高吞吐分析
```

而不是：

```text
复杂事务
+
高频行级更新
```

不适合：

- 多行复杂事务
- 强事务一致性
- 高频单行 UPDATE
- 高频单行 DELETE

### 5.2 并发写入

多个客户端可以并发写入：

```text
Client A → Part A
Client B → Part B
Client C → Part C
              ↓
        Background Merge
```

写入之间减少了传统行级修改带来的锁竞争。

### 5.3 UPDATE / DELETE

ClickHouse 更倾向：

> **追加新版本，而不是频繁修改历史数据。**

例如状态更新：

```text
旧版本：id=1,status=pending
新版本：id=1,status=paid
```

配合 `ReplacingMergeTree` 在 Merge 阶段进行整理。

### 5.4 ReplacingMergeTree 是否实时去重？

**不是。**

数据写入后，短时间内可能存在多个版本。

因此：

```text
INSERT
 ↓
多个版本并存
 ↓
Background Merge
 ↓
最终整理
```

如果使用 `FINAL`，可以在查询阶段执行更严格的去重，但可能带来明显性能开销。

### 5.5 获取最新状态的常见方式

#### `argMax`

```sql
SELECT
    id,
    argMax(status, update_time) AS current_status,
    argMax(amount, update_time) AS final_amount,
    max(update_time) AS latest_time
FROM orders
GROUP BY id;
```

适合只需要部分字段最新值的场景。

#### `LIMIT 1 BY`

```sql
SELECT *
FROM orders
ORDER BY update_time DESC
LIMIT 1 BY id;
```

适合需要保留整行最新状态的场景。

### 5.6 删除方式

可以按成本和场景理解：

| 方式 | 特点 | 场景 |
|---|---|---|
| `DROP PARTITION` | 分区级物理删除，效率高 | 批量清理历史数据 |
| Replacing / Collapsing | 通过追加新版本处理 | 状态更新 |
| Lightweight Delete | 逻辑删除，后台再清理 | 常规删除 |
| Mutation | 异步重写数据 | 低频清洗、合规删除 |

---

## 6. 分布式与高可用

### 6.1 Shard：解决规模问题

Shard 是数据分片。

```text
完整数据
   ↓
Shard Key
   ↓
Shard 1 / Shard 2 / Shard 3
```

解决：

- 单机存储容量
- 单机 CPU
- 单机查询吞吐

### 6.2 Replica：解决高可用

Replica 是同一 Shard 的副本。

```text
Shard 1
├── Replica A
└── Replica B
```

主要解决：

- 数据冗余
- 节点故障
- 服务可用性
- 查询负载分担

### 6.3 Distributed Table

Distributed 表通常不直接保存业务数据，而是作为分布式访问入口。

```text
Client
  ↓
Distributed Table
  ↓
┌───────┬───────┬───────┐
Shard 1  Shard 2  Shard 3
  ↓        ↓        ↓
Local    Local    Local
Table    Table    Table
```

查询过程：

```text
Coordinator
    ↓
Scatter
    ↓
各 Shard 本地执行
    ↓
局部结果
    ↓
Gather / Merge
    ↓
最终结果
```

### 6.4 ClickHouse Keeper

Keeper / ZooKeeper 主要负责：

- 复制日志
- 副本协调
- 元数据状态
- 分布式 DDL 协调
- 副本状态管理

> Keeper **不存储业务数据**；真正的数据 Part 仍在 ClickHouse 节点本地磁盘。

### 6.5 副本同步

简化流程：

```text
Replica A INSERT
      ↓
生成 Part
      ↓
记录复制日志
      ↓
Keeper
      ↓
Replica B 发现任务
      ↓
从健康副本 Fetch Part
      ↓
Checksum 校验
      ↓
本地提交
```

---

## 7. 持久化与恢复

### 7.1 Part 如何落盘？

核心思想：

> **数据以完整 Part 为单位写入和生效。**

简化流程：

```text
INSERT
 ↓
内存排序
 ↓
生成临时目录
 ↓
写 .bin / index / mark
 ↓
完成写入
 ↓
rename 为正式 Part
 ↓
查询可见
```

Part 形成后，后台再通过 Merge 生成新的 Part。

### 7.2 Crash 后如何恢复？

已完成并生效的 Part 已经在磁盘上。

因此重启后：

```text
扫描合法 Part
    ↓
加载元数据 / 索引
    ↓
恢复表状态
```

如果写入过程中只存在未完成的临时目录，则清理未完成数据；客户端应对未确认成功的批次具备重试能力。

### 7.3 WAL 与事务

ClickHouse 的持久化模型不同于典型 OLTP：

| 维度 | OLTP | ClickHouse MergeTree |
|---|---|---|
| 核心机制 | WAL + Buffer Pool | Part 文件 |
| 写入方式 | 修改已有数据页 | 生成新的 Part |
| 事务 | 强 ACID | 不以传统事务为核心 |
| 小批量写入 | 友好 | 容易产生大量 Part |
| 数据整理 | 脏页 / 日志 | Background Merge |

> 新版本也存在 WAL 等机制，但不要因此把 ClickHouse 理解成传统 OLTP 数据库；**批量写入 + Part + Merge** 仍是理解 MergeTree 的主线。

### 7.4 Replica ≠ Backup

这是面试中的重要区别：

```text
Replica
→ 防硬件故障 / 节点故障

Backup
→ 防误删 / 脏数据 / 逻辑灾难
```

副本会同步状态，因此错误操作可能同步到所有副本。

备份需要独立的历史快照，并可使用 ClickHouse `BACKUP / RESTORE` 等方案。

---

## 8. 性能与场景

### 8.1 ClickHouse 为什么快？

可以浓缩成：

| 机制 | 作用 |
|---|---|
| 列式存储 | 少读列 |
| 编码 / 压缩 | 少 IO |
| ORDER BY | 让相关数据集中 |
| Sparse Index | 快速缩小范围 |
| Data Skipping | 跳过无关数据 |
| Block | 批量处理 |
| Vectorization / SIMD | 提高 CPU 效率 |
| Parallelism | 利用多核 |
| Shard | 水平扩展 |

### 8.2 核心公式

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

### 8.3 适合什么场景？

适合：

- 日志分析
- 埋点分析
- 用户行为分析
- BI / 报表
- 实时 OLAP
- 广告分析
- 风控分析
- 监控指标
- IoT
- 海量事件分析
- 时间序列分析

典型特征：

```text
数十亿 / 数百亿数据
        ↓
按时间 / 用户 / 地区 / 业务维度
        ↓
扫描 + 聚合 + 统计
```

### 8.4 不适合什么？

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

这类业务通常更适合 MySQL / PostgreSQL 等 OLTP 数据库。

---

## 9. ClickHouse 核心知识串联

```text
                         ClickHouse
                              │
              ┌───────────────┴───────────────┐
              ↓                               ↓
          OLAP 分析                         分布式
              │                               │
              ↓                               ↓
       Columnar Storage                 Shard + Replica
              │                               │
              ↓                               ↓
          MergeTree                     Distributed
              │
              ↓
         Part + Merge
              │
              ↓
      Partition + ORDER BY
              │
              ↓
     Sparse Index + Granule
              │
              ↓
        Data Skipping
              │
              ↓
        少量数据读取
              │
              ↓
        Block + Vector
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

> 列式存储减少读取量，压缩减少 IO，ORDER BY + 稀疏索引 + Data Skipping 减少扫描，Block + 向量化提高 CPU 利用率，并行执行充分利用多核，分布式进一步扩展吞吐。

### ② MergeTree 是什么？

> ClickHouse 最核心的表引擎家族。数据以 Part 形式追加写入，后台通过 Merge 整理 Part，并利用 ORDER BY 建立稀疏索引。

### ③ Part 是什么？

> Part 是一次批量写入形成的独立数据片段，也是 MergeTree 的基本物理存储和 Merge 单位。

### ④ Partition 和 ORDER BY 有什么区别？

> Partition 是更大的数据管理和裁剪边界；ORDER BY 决定 Part 内数据排序以及稀疏索引组织。两者不能互相替代。

### ⑤ ClickHouse 为什么不需要大量 B+Tree？

> OLAP 更关注大量数据的扫描和聚合。ClickHouse 通过列式存储、排序、稀疏索引和 Data Skipping 来减少需要扫描的数据。

### ⑥ ReplacingMergeTree 是实时去重吗？

> 不是。它主要在 Merge 阶段完成数据整理，因此写入后短时间内可能存在多个版本。

### ⑦ Shard 和 Replica 的区别？

> **Shard 解决规模问题，Replica 解决高可用和冗余问题。**

### ⑧ Distributed 表做什么？

> 提供分布式查询 / 写入入口，把请求分发到多个 Shard，并汇总结果；真正的数据通常存储在各节点的 Local MergeTree 表中。

### ⑨ ClickHouse 支持事务吗？

> ClickHouse 不应被当成传统强事务 OLTP 数据库使用。它的设计重点是批量、追加式写入和高性能分析。

### ⑩ ClickHouse 和 MySQL 怎么选？

```text
事务 / 点查 / 高频更新
        ↓
      MySQL

海量数据 / 聚合 / 报表 / 日志分析
        ↓
   ClickHouse
```

---

## 11. 一张图理解 ClickHouse

### 查询

```text
Client
  ↓
SQL
  ↓
Query Planner
  ↓
Partition Pruning
  ↓
Sparse Index / Data Skipping
  ↓
MergeTree Parts
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
```

### 写入

```text
INSERT
  ↓
排序
  ↓
Part
  ↓
Disk
  ↓
Background Merge
  ↓
Larger Part
```

### 分布式

```text
Distributed Table
       ↓
┌──────┼──────┐
↓      ↓      ↓
Shard1 Shard2 Shard3
↓      ↓      ↓
Replica Replica Replica
```

---

## 12. 最终记忆模型

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
   批量追加 + Merge 最终整理

8. 持久化
   Part 落盘 + Replica + Backup

9. 性能
   少读 + 少算 + 并行 + 压缩

10. 场景
    日志 / BI / 埋点 / 实时分析 / 海量聚合
```

> **一句话总结：**
>
> ClickHouse 通过 **列式存储减少读取量，ORDER BY + 稀疏索引 + Data Skipping 减少扫描量，Block + 向量化 + 并行执行提高计算效率，再通过 Shard + Replica 实现分布式扩展和高可用**，因此特别适合海量数据 OLAP，而不适合传统强事务 OLTP。
