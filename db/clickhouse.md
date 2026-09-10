
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

ClickHouse 的核心存储机制建立在 MergeTree (合并树) 引擎之上，采用**列式存储**与**稀疏索引**。数据以追加（Append）方式写入磁盘形成不可变的 Data Part，并在后台异步合并（Merge）。

## 一、 宏观逻辑与磁盘文件布局

表（Table）的数据首先按照分区键（Partition Key）被划分为多个 Partition。随着数据的持续写入，每个 Partition 目录下会生成多个不可变的物理文件夹，称为 **Data Part**。

### 1. 宏观逻辑结构
```text
[ Table: events ]
  │
  ├── [ Partition: 202310 ] (逻辑概念，按月分区)
  │    │
  │    ├── [ Data Part 目录: 202310_1_5_1 ] (物理目录，代表一次合并后的数据块)
  │    │    ├── primary.idx          (稀疏主键索引文件，整个Part共享)
  │    │    ├── count.txt            (记录当前Part的总行数)
  │    │    │
  │    │    ├── user_id.bin          (【Columns】: user_id 列的压缩数据文件)
  │    │    ├── user_id.mrk2         (【Marks】: user_id 列的标记文件)
  │    │    │
  │    │    ├── event_type.bin       (【Columns】: event_type 列的压缩数据)
  │    │    └── event_type.mrk2      (【Marks】: event_type 列的标记文件)
  │    │
  │    └── [ Data Part 目录: 202310_6_6_0 ] (新写入的、尚未合并的数据)
  │
  └── [ Partition: 202311 ]
```

### 2. 微观寻址结构 (Index -> Granules -> Blocks)
ClickHouse 底层没有“行”的概念，只有**颗粒（Granule）**和**压缩块（Compressed Block）**。

```text
                      【逻辑划分】默认每 8192 行划分一个 Granule (颗粒)
Row 0~8191             Row 8192~16383         Row 16384~24575       ...
[ Granule 0 ]          [ Granule 1 ]          [ Granule 2 ]         ...
      │                      │                      │
======│======================│======================│========================
      │                      │                      │
      ▼                      ▼                      ▼
[ primary.idx 稀疏索引 ] (常驻内存，仅保存每个Granule第一行的主键)
+----------------+----------------+----------------+
| 主键值 A       | 主键值 B       | 主键值 C       |
+----------------+----------------+----------------+
  (下标: 0)          (下标: 1)          (下标: 2)
      │                      │                      │
      │ 内存二分查找匹配后，通过数组下标 1:1 映射到 Mark 标记
      ▼                      ▼                      ▼
[ user_id.mrk2 标记文件 ] (定长数组，加载到内存)
+------------------------------------+------------------------------------+
| Mark 0                             | Mark 1                             |
| BlockOffset: 0  | GranuleOffset: 0 | BlockOffset: 0  | GranOFS: 32768   |
+------------------------------------+------------------------------------+
      │                      │
      │ 拿着 BlockOffset (物理字节偏移) 去 .bin 文件寻址
      ▼                      ▼
[ user_id.bin 压缩数据文件 ] (磁盘流，LZ4 / ZSTD)
=============================================================================
| [ Compressed Block 0 ] (物理大小: 85000 字节)                                 |
| ┌─────────────────────────────────────────────────────────────────────────┐ |
| │ 【解压后的内存块】(Uncompressed Block)                                    │ |
| │ -> 从 0 字节开始是 Granule 0 的数据 (由 Mark 0 的 GranOFS 决定)           │ |
| │ -> 从 32768 字节开始是 Granule 1 的数据 (由 Mark 1 的 GranOFS 决定)       │ |
| └─────────────────────────────────────────────────────────────────────────┘ |
=============================================================================
```

*   **Granules (颗粒):** 逻辑上的数据分割单元，默认 8192 行，是稀疏索引的最小指向目标。
*   **Columns (`.bin`):** 每一列的数据被单独抽离，切分成 Compressed Block（通常 64KB-1MB）。一个物理压缩块可能包含一个或多个 Granules 的数据。
*   **Marks (`.mrk2`):** 导航图，记录 Granule 在 `.bin` 中的绝对物理偏移量（BlockOffset）以及解压后的相对内存偏移量（GranuleOffset）。

---

## 二、 三大核心文件物理结构拆解

### 1. `primary.idx` (稀疏主键索引文件)
**定位：** 常驻内存的全局“目录”。连续的、定长的一维数组，严格按 `ORDER BY` 字段序列化。

```text
[ primary.idx 物理结构 ] (纯二进制序列化)
  +-------------------------------------------------------+
  | 主键字段 1 的值 | 主键字段 2 的值 | ... (属于 Mark 0) | ----> Granule 0 头部
  +-------------------------------------------------------+
  | 主键字段 1 的值 | 主键字段 2 的值 | ... (属于 Mark 1) | ----> Granule 1 头部
  +-------------------------------------------------------+
```

### 2. `.mrk2` (标记文件)
**定位：** 内存与磁盘之间的“寻址桥梁”。定长的二维元组数组，支持自适应索引粒度。

```text
[ <column_name>.mrk2 物理结构 ] (定长 24 Bytes)
  +======================+=======================+======================+
  | 压缩块的物理偏移量   | 解压后的内存偏移量    | 该 Granule 的行数    |
  | (UInt64, 8 Bytes)    | (UInt64, 8 Bytes)     | (UInt64, 8 Bytes)    |
  +======================+=======================+======================+ <--- Mark 0
  | Compressed Offset    | Uncompressed Offset   | Row Count            |
  +======================+=======================+======================+ <--- Mark 1
```

### 3. `.bin` (数据文件)
**定位：** 存储实际一列数据的压缩流文件。

```text
[ <column_name>.bin 物理结构 ]
+========================================================================+
| Compressed Block 0 (64KB ~ 1MB)                                        |
|   [ Header ]                                                           |
|   | Checksum (16B) | Comp Alg (1B) | Comp Size (4B) | Uncomp Size (4B) |
|   [ Payload ]                                                          |
|   | LZ4 / ZSTD 压缩的二进制字节流 (包含多个 Granule 数据)              |
+========================================================================+
| Compressed Block 1 ...                                                 |
+========================================================================+
```

---

## 三、 写入与读取的流转流程（超级快递仓库模型）

三大文件通过**严格的数组下标对齐**进行协作。我们将 10 万条玩家充值记录的存储和查询，具象化为一个“超级快递仓库”。

### 1. 写入流程：货物入库
*   **装箱 (划分 Granule):** 引擎将记录按 `UserID` 排序，**每 8192 条记录装进一个标准纸箱**（Granule）。10 万条记录装了 13 个箱子（0号箱：User_100 到 User_800；1号箱：User_805 到 User_1500...）。
*   **写大屏目录 (生成 `.idx`):** 仓库管理员在大屏幕上登记每个箱子的**第一名玩家**（`[100, 805, 1502 ...]`）。这块屏幕就是常驻内存的 `primary.idx`。
*   **抽真空打包 (生成 `.bin`):** 为了节省空间，管理员用“真空压缩袋”把 0号箱和 1号箱装在一起抽真空，放在仓库的**第 0 米处**。这就是 `.bin` 里的 Compressed Block。
*   **记私人小本本 (生成 `.mrk2`):** 管理员在小本本记录暗号：
    *   第 0 行 (0号箱)：去 0米处的压缩袋，剪开后，它在最顶上（相对位置 0）。
    *   第 1 行 (1号箱)：去 0米处的压缩袋，剪开后，往下翻 50 厘米（相对位置 50）。

### 2. 读取流程：大海捞针
当需要查询 `User_1000` 的充值记录时，引擎执行以下极限寻址：
*   **看大屏幕找箱号 (内存二分查找):** 管理员看大屏幕 `[100, 805, 1502 ...]`。利用二分查找快速判断出 `1000` 介于 `805` 和 `1502` 之间，目标必定在 **1号箱**。
*   **翻小本本找位置 (读取标记):** 查阅 `.mrk2` 第 1 行，得知目标在“**第 0 米处**的压缩袋，解压后往下翻 **50 厘米**”。
*   **进仓库搬货解压 (精细 I/O):** 直奔磁盘 `.bin` 文件第 0 字节处，读取该 Compressed Block 进内存并解压。
*   **精准提取 (向量化过滤):** 在解压后的内存块中，向下偏移 50 厘米（Uncompressed Offset），拽出 1号箱的 8192 条记录。通过 SIMD 指令快速过滤出 `User_1000`，完成查询。


## 2. 存储引擎

# ClickHouse 存储引擎与核心机制深度解析

ClickHouse 之所以能在 OLAP（联机分析处理）领域大放异彩，其极速查询能力的底层密码就藏在它的存储引擎设计中。其中，**MergeTree（合并树）**不仅是 ClickHouse 最核心的表引擎，更是整个 ClickHouse 架构设计的灵魂。

## 1. 核心基石：MergeTree 与 Part 机制

### 1.1 核心思想
绝大多数 OLAP 场景首先考虑 MergeTree 家族。其核心思想基于 LSM-Tree（Log-Structured Merge-Tree），将随机写转化为顺序写：
1. **INSERT 写入**：数据写入时，形成一个不可变的临时目录（称为 **Part**）。
2. **后台 Merge**：后台任务定期将多个小的 Part 合并（Merge）成更大的 Part。

### 1.2 为什么采用 Part 机制？
传统关系型数据库在写入时往往会修改 B+ 树的节点（随机写），这在海量吞吐下会导致严重的 I/O 瓶颈。
ClickHouse 采用追加写（Append-only）：
* **避免锁竞争**：由于 Part 写入后不可变（Immutable），读取操作不需要加锁，读写互不干扰。
* **异步整理**：写入路径极简，数据规整和优化由后台异步的 Merge 操作完成。

> **面试避坑**：绝不要在 ClickHouse 中高频地进行单条 INSERT。例如每秒插入一条，会瞬间产生海量极小的 Part 文件夹，耗尽文件描述符并拖垮 ZooKeeper。**最佳实践是攒批写入（Batch Insert）**，如每批 5万-10万行。

---

## 2. 极致性能的秘密：列式存储与 SIMD

### 2.1 列式存储的优势
* **传统行式**：`[id, time, type, amount], [id, time, type, amount]...`
* **ClickHouse 列式**：
  * `id` 文件：`1, 2, 3...`
  * `amount` 文件：`10.5, 20.0, 15.2...`

当执行 `SELECT sum(amount) FROM events;` 时，存储层只需读取 `amount` 这一列的文件，**I/O 开销通常可以直接降低 90% 以上**。

### 2.2 向量化执行 (SIMD) 深度解析与举例
列式存储使得数据在内存中也是连续存放的同类型数组，这为 **SIMD (Single Instruction, Multiple Data - 单指令多数据流)** 提供了完美舞台。SIMD 允许 CPU 在同一个时钟周期内，对一组数据执行相同的操作。

**举例说明：**
假设我们要计算两个数组的相加：`A = [1, 2, 3, 4]`, `B = [5, 6, 7, 8]`，结果存入 `C`。

* **传统标量执行（Scalar）**：需要循环 4 次。
  1. 周期1：`C[0] = 1 + 5`
  2. 周期2：`C[1] = 2 + 6`
  3. 周期3：`C[2] = 3 + 7`
  4. 周期4：`C[3] = 4 + 8`

* **SIMD 向量化执行**：
  利用现代 CPU 的超宽寄存器（如 256位的 AVX2 寄存器，可以一次性容纳 8 个 32位整数）。CPU 发出**一条**加法指令，在一个时钟周期内，直接将整个向量对齐相加完成运算。
  1. 周期1：`[C0, C1, C2, C3] = [1+5, 2+6, 3+7, 4+8]`

**在 ClickHouse 中的体现**：
当进行 `WHERE amount > 100` 过滤时，ClickHouse 底层并非通过 `for` 循环逐行判断（这会引发大量 CPU 分支预测失败），而是利用 SIMD 指令将一批连续的 `amount` 数据加载到 CPU 寄存器中，一次性比较，返回一个 Bitmap（如 `[1, 0, 0, 1]`），速度快上几个数量级。

---

## 3. 数据瘦身术：编码 (Encoding) 与 压缩 (Compression)

ClickHouse 采用“先编码，后压缩”的两步走策略，将磁盘 I/O 压榨到极致。

### 3.1 第一步：数据编码 (Encoding) 举例
编码是根据数据的**业务特征**，在进行通用压缩前进行的特殊转换。

* **案例 A：Delta 编码（适用于时间戳、自增ID）**
  * **原始数据**：`[1700000000, 1700000010, 1700000020, 1700000030]` (都是占 4 字节的大整数)
  * **Delta 编码后**：存储第一个基准值，后续存差值 -> `[1700000000, 10, 10, 10]`
  * **效果**：大数字变成了极小的数字 `10`，原本需要 4 字节，现在甚至不到 1 字节即可表示。

* **案例 B：字典编码 / LowCardinality（适用于状态、类型等低基数文本）**
  * **原始数据**：`["Apple", "Banana", "Apple", "Apple", "Banana"]`
  * **字典编码后**：生成字典 `{0:"Apple", 1:"Banana"}`，数据变为 `[0, 1, 0, 0, 1]`。
  * **效果**：将长字符串比较转换为了极快的整数比较，同时大幅缩减体积。

### 3.2 第二步：通用压缩 (Compression) 过程举例
经过编码后，数据变得高度同质化，此时再交由通用的压缩算法处理。ClickHouse 主要使用 **LZ4（默认，重速度）** 和 **ZSTD（重压缩率）**。

#### A. LZ4 压缩举例（寻找重复的字节模式）
* **特点**：解压速度极快（可达数 GB/s），几乎不占用 CPU，是温热数据的首选。
* **过程**：
  * **待压缩块**：`abcde_xyz_abcde`
  * **LZ4 处理后**：`abcde_xyz_[距离=10, 长度=5]`
  * **原理**：列式存储中相邻数据相似，LZ4 通过指针（向前寻找多远，复制多长）替代冗余数据。

#### B. ZSTD (Zstandard) 压缩过程与举例（剧烈压缩）
* **特点**：由 Facebook 开源，压缩率极高。解压速度虽不如 LZ4，但在高压缩比算法中性能顶尖。常用于 ClickHouse 的**冷数据层**，大幅节省磁盘成本。
* **过程**：ZSTD 是两阶段压缩算法，结合了 **字典匹配 (类似 LZ77)** 和 **有限状态熵编码 (FSE/Huffman)**。
* **深度举例**：
  * **原始文本**：`ClickHouse is fast. ClickHouse is powerful.`
  * **阶段1：字典匹配（找重复）**
    扫描发现重复段落。被替换为：`ClickHouse is fast. [Distance=20, Length=15]powerful.`
  * **阶段2：熵编码（高频词用短码，低频词用长码）**
    统计上述结果中字母出现的频率。假设字母 `e` 出现极多，`Z` 出现极少。
    在传统的 ASCII 编码中，`e` 和 `Z` 都占 8 个 bit。
    ZSTD 通过 FSE（Finite State Entropy）重新编码：将最高频的 `e` 编码为仅占 2 个 bit（如 `01`），将低频的 `Z` 编码为 12 个 bit。
  * **综合效果**：经过这“剔除重复段落 + 高频位缩减”的两套组合拳，经过列式同质化排列的数据，体积通常能被压缩到原来的 1/5 甚至 1/10。

---

## 4. 后台数据重组：Merge 过程与举例

后台 Merge 是 MergeTree 引擎赖以生存的心脏跳动。数据“写完就固定”只是微观表现，宏观上数据处于不断的生命周期演进中。

### 4.1 Merge 机制核心过程
1. **挑选触发**：后台进程定期扫描，挑选出多个属于同一分区（Partition）且大小相近的 Part 文件夹（如 Part A 和 Part B）。
2. **多路归并排序**：将 A 和 B 中的数据读入内存。由于每个 Part 内部已经按照 `ORDER BY` 排好序了，ClickHouse 只需执行非常高效的**归并排序（Merge Sort）**。
3. **引擎逻辑执行**：在此阶段，ClickHouse 会根据具体的表引擎类型，执行数据的去重、聚合等物理操作。
4. **生成新 Part**：将合并后的结果写入全新的 Part C。
5. **废弃与清理**：将 Part A 和 B 标记为 Inactive。一段时间后，由清理线程物理删除。

### 4.2 Merge 过程举例
假设表有两列 `(id, val)`，以 `id` 作为 `ORDER BY` 主键。

* **Part A (已排序)**：`[1, 'a'], [3, 'b'], [5, 'c']`
* **Part B (已排序)**：`[2, 'd'], [3, 'x'], [6, 'e']`

**执行 Merge**：
* 如果是**基础 MergeTree**：只做合并排序。结果为 `[1, 'a'], [2, 'd'], [3, 'b'], [3, 'x'], [5, 'c'], [6, 'e']`。
* 如果是**ReplacingMergeTree**：遇到主键相同的 `id=3`，进行去重（默认保留后写入的）。结果为 `[1, 'a'], [2, 'd'], [3, 'x'], [5, 'c'], [6, 'e']`。

---

## 5. MergeTree 引擎家族对比总结

不同的 MergeTree 引擎，它们的**相同点**在于底层机制，而**差异点**完全体现在 **Merge 阶段对同一主键（ORDER BY）数据的处理逻辑上**。

### 5.1 相同点（家族基因）
1. **物理存储**：均采用列式存储、Part 目录结构。
2. **索引机制**：均依赖 `ORDER BY` 生成**稀疏索引（Sparse Index）**，支持通过标记文件（Mark）快速跳过无效数据块。
3. **分区机制**：均支持 `PARTITION BY` 进行数据分区生命周期管理。
4. **后台行为**：均依赖后台异步的 Merge 动作来整理数据。

### 5.2 差异点与应用场景对比

| 引擎名称 | 核心差异（Merge 时的特殊行为） | 典型应用场景 |
| :--- | :--- | :--- |
| **MergeTree** | **不干涉数据**。遇到主键相同的数据，直接保留多条，仅按顺序合并。 | 基础日志流、事件明细记录、不需要修改/去重的流水数据。 |
| **ReplacingMergeTree** | **去重**。Merge 时，删除主键相同的数据，默认保留最后写入的版本（或指定版本字段最大的一条）。 | 需要更新/去重的场景（如用户最新状态）。*注意：去重只在 Merge 时发生，查询时需配合 `FINAL` 关键字保证强一致性。* |
| **SummingMergeTree** | **相加**。Merge 时，将主键相同的行的数值类型列（指定列）进行累加合并为一行。 | 预聚合场景。例如统计每个用户每日的广告点击量、消耗金额。可以大幅降低存储与查询开销。 |
| **AggregatingMergeTree** | **状态合并**。比 Summing 更高级，使用 `AggregateFunction`（如 `uniqState`），Merge 时合并聚合的中间状态数据。 | 复杂的预聚合（如计算精准去重 UV、分位数等），常配合物化视图 (Materialized View) 使用。 |
| **CollapsingMergeTree** | **状态折叠**。通过特殊的 `Sign` 列（1 为写入，-1 为取消），Merge 时将主键相同且 Sign 相反的行相互抵消（物理删除）。 | 异步更新/删除数据的场景。比 Replacing 性能更好，但对写入顺序要求极高（必须先写 1，后写 -1）。 |
| **VersionedCollapsingMergeTree**| 在 Collapsing 的基础上加入了版本号 `Version` 列。 | 解决乱序写入导致的无法折叠问题。只要 Version 对应，即使 -1 先到达，1 后到达，Merge 时也能正确抵消。 |

**补充说明：Replicated 系列**
ClickHouse 提供了上述所有引擎的 **Replicated** 版本（如 `ReplicatedMergeTree`, `ReplicatedReplacingMergeTree`）。
* **作用**：它们在单机引擎的基础上，引入了 ZooKeeper 协调机制。
* **差异**：除了执行常规 Merge 动作外，当节点产生新 Part 时，还会将元数据注册到 ZK，其他副本节点会异步拉取（Fetch）物理文件，从而实现数据的高可用副本同步。
---

## 3. 内存与数据结构

ClickHouse 能够实现极致的 OLAP（联机分析处理）查询速度，绝不仅靠“把数据塞进内存”这种暴力手段，而是通过一套从底层存储到上层计算的精妙架构。以下是对其核心机制的深度拆解。

## 3.1 ClickHouse 为什么能够高速分析？

ClickHouse 的“快”，建立在以下核心机制的协同工作上：

* **列式存储 (Columnar Storage)**：数据按列而非按行连续存储。如果查询只涉及 100 列中的 3 列，系统只需读取这 3 列的数据文件，I/O 开销直接呈指数级下降。
* **压缩 (Compression)**：因为同一列的数据类型完全相同（例如全是日期或全是金额），ClickHouse 可以使用极其高效的压缩算法（如 LZ4、ZSTD）。这不仅节省磁盘空间，更大幅减少了将数据从磁盘读入内存的耗时。
* **Primary Key Index (稀疏主键索引)**：与 MySQL 的 B+ 树（精准到行）不同，ClickHouse 的索引是“稀疏”的。它按一定间隔（默认 8192 行）记录主键值。这使得索引极小，可以常驻内存，在查询时能迅速锁定数据所在的粗略范围。
* **Data Skipping (数据跳过)**：通过二级索引（如 MinMax 索引），系统在读取数据块前会先判断该块是否包含目标数据。如果不包含，直接跳过整个数据块，避免无效 I/O。
* **向量化执行 (Vectorized Execution)**：放弃传统的“逐行处理”逻辑，采用“按块（Block）”并行处理，充分利用现代 CPU 的 SIMD 指令集。
* **并行计算 (Parallel Computing)**：ClickHouse 天生是多线程的。一个查询会被拆分成多个任务，分配给多个 CPU 核心甚至多个节点同时执行，最大化榨干硬件算力。
* **高效聚合 (Efficient Aggregation)**：针对 `GROUP BY` 等操作，底层重写了多种高度优化的 Hash Table，根据不同的数据类型和基数，自动选择最快的哈希表实现。

---

## 3.2 内存主要用于什么？

复杂 OLAP 查询会消耗大量内存，因为 ClickHouse 的计算高度依赖内存来换取速度。查询过程中的内存消耗大户包括：

| 内存消耗项 | 运作机制与场景 |
| :--- | :--- |
| **查询执行 Block** | 数据从磁盘读出后，会以 Block（列数组）的形式在内存中流转和计算。并发越高，同时存在于内存中的 Block 越多。 |
| **聚合 Hash Table** | 执行 `GROUP BY` 时，系统必须在内存中维护一个巨大的哈希表来存储分组键和对应的聚合状态。 |
| **Sort (排序)** | 执行 `ORDER BY` 时，如果没有可以利用的索引，全量数据或大量中间结果需要在内存中进行快速排序。 |
| **Join (表连接)** | 默认的 `Hash Join` 通常需要将右表完整加载到内存中构建哈希表，极度吃内存。 |
| **Buffer & Cache** | 字典数据（Dictionaries）、标记缓存（Mark Cache）以及异步插入时的缓冲区，均需要占用常驻内存。 |

---

## 3.3 数据处理的基本单元：Block

传统关系型数据库（如 MySQL、PostgreSQL）通常采用基于火山模型（Volcano Model）的**逐行处理 (Row-by-Row)**：拉取一行，计算一行，再拉取下一行。这在分析海量数据时效率极低。

ClickHouse 的核心数据结构是 **Block**。一个 Block 包含了一批数据（通常是几千到几万行），但在 Block 内部，数据是按列（Column）组织的数组。

**逻辑对比：**
* **传统模型：** `Row 1 (A, B, C) -> Row 2 (A, B, C) -> Row 3...`
* **ClickHouse 模型：**
  > Block
  >  ├── Column A [a1, a2, a3, ... a8192]
  >  ├── Column B [b1, b2, b3, ... b8192]
  >  └── Column C [c1, c2, c3, ... c8192]

**为什么 Block 很重要？** 
因为数组在内存中是物理连续的。这不仅极其契合 CPU 的缓存预取机制（避免 CPU Cache Miss），更是实现“向量化执行”的物理基础。

---

## 3.4 向量化执行 (Vectorized Execution)

向量化执行是 ClickHouse 计算极快的绝对主力。它将循环中的单次操作转化为批量操作。

**算法与概念对比：**

1. **传统标量执行 (Scalar Processing)**：
   为了计算 `A + B = C`，CPU 每读取一行，就需要执行一次加法指令。处理 1000 行，需要进行 1000 次函数调用，伴随着大量的虚函数开销和分支预测失败。
2. **向量化执行与 SIMD**：
   SIMD（单指令多数据流）是现代 CPU 的硬件级特性。借助 Block 提供的连续列数组，ClickHouse 可以在**一个 CPU 时钟周期内，用一条指令同时完成多个数据（例如 8 个 32 位浮点数）的加法运算**。此时，计算逻辑变成了针对整个数组的紧凑 `for` 循环，极大地减少了方法调用的开销和底层的条件分支判断。

---

## 3.5 聚合 Hash Table 的运作与瓶颈

在 OLAP 分析中，`GROUP BY` 是最常见的操作。

**执行示例：**
```sql
SELECT user_id, sum(amount) FROM events GROUP BY user_id;
```

**底层算法执行步骤：**
1. 系统会在内存中初始化一个 **Hash Map**。
2. 遍历数据 Block。对于每一个 `user_id`，计算其 Hash 值。
3. 如果 `user_id` 不在 Hash Map 中，则插入新 Key，并初始化聚合状态（例如 `amount` 设为当前值）。
4. 如果 `user_id` 已存在，则取出对应的聚合状态，将新的 `amount` 累加进去（`state += amount`）。

**内存瓶颈 (Memory Bottleneck)：**
当 `user_id` 的基数（Cardinality，即不同 ID 的数量）极高时，Hash Map 会在内存中疯狂膨胀。触碰内存阈值时，系统会尝试**外部聚合 (External Aggregation)**：将当前 Hash Map 刷新（Spill）到磁盘临时文件，清空内存继续处理，最后进行多路归并（Merge）。但这会导致 I/O 剧增，性能骤降。

---

## 3.6 Join (表连接) 执行机制与优化

在 ClickHouse 中，JOIN 往往是性能瓶颈和内存消耗的“重灾区”。ClickHouse 的底层架构是为“宽表（Wide Table）”设计的，对 JOIN 的处理逻辑与传统数据库有显著不同。

### 1. 默认执行算法：Hash Join
当执行 `SELECT * FROM table_A JOIN table_B ON ...` 时，ClickHouse 的默认行为如下：

* **构建阶段 (Build Phase)**：ClickHouse 会将 **右表 (table_B)** 的数据**全量**读取到内存中，并在内存中为其构建一个 Hash Table。
* **探测阶段 (Probe Phase)**：系统以流式的方式，按 Block 读取 **左表 (table_A)** 的数据，用左表的连接键去内存中的 Hash Table 里探测匹配项。

**核心痛点：OOM（内存溢出）**
因为右表必须全部放入内存，如果右表是一张几百 GB 的大表，内存会瞬间被撑爆并报错。

### 2. 分布式表 Join 的坑：GLOBAL JOIN
在分布式 ClickHouse 集群中，普通 `JOIN` 只会在每个节点上进行本地匹配，若右表数据分散，会导致匹配不全。
此时需使用 `GLOBAL JOIN`：发起查询的节点会把右表数据全捞出来，聚合成一张完整的内存表，然后**通过网络广播（Broadcast）**发送给集群所有节点。这会带来极大的网络传输开销和各节点的内存压力。

### 3. 应对 Join 的核心优化策略

* **策略一：小表永远在右边**
  因为加载到内存的是右表，写 SQL 时务必遵循：`大表 JOIN 小表`。
* **策略二：使用 Merge Join 算法**
  如果右表实在太大，可设置 `SET join_algorithm = 'partial_merge'`。不再把右表全量放内存，而是对左右表按连接键**排序**，然后利用双指针归并匹配。内存不够可落盘，但速度比 Hash Join 慢。
* **策略三：使用字典表 (Dictionaries)**
  对于常被关联的维度表（如用户信息、商品分类），将其配置为 **外部字典 (External Dictionaries)**。字典数据常驻内存，通过 `dictGet()` 函数调用，速度比 JOIN 快几个数量级。
* **策略四：终极杀器 —— 宽表模式 (Denormalization)**
  这是最推荐的最佳实践。**在数据导入 ClickHouse 之前**（如 Flink/Spark 阶段），提前将多张表打平生成一张**大宽表**。直接对单张宽表查询，彻底抛弃 JOIN，全面释放 ClickHouse 的列存和向量化性能。
---

## 4. 索引与查询

## 1. ClickHouse 的索引思想
ClickHouse 与 MySQL 最大区别之一，在于它**不是依赖 B+Tree 索引进行大量点查**，而是通过**排序 + 稀疏主键索引 + Data Skipping** 极大程度减少扫描数据。

可以把它想象成一个极度强调“批量排除”的巨型物流仓库，直接把不相关的货架推开，保留可能的范围。

**典型查询核心流程：**
SQL -> Partition Pruning (分区裁剪) -> Primary Key / Sparse Index (主键/稀疏索引) -> Data Skipping (数据跳跃过滤) -> 读取需要的列 -> 向量化执行 -> Aggregation / Join -> Result

---

## 2. 稀疏主键索引 (Sparse Primary Key Index) 与 Granule

ClickHouse 绝对不会为每一行数据创建索引（海量数据下会撑爆内存）。它使用的是**稀疏主键索引**。
* **排序是基石：** 数据在写入时，必须先按照 `ORDER BY` 排好队。
* **Granule (颗粒)：** 排好序的数据被切分成一个个固定大小的包裹（默认 8192 行一个 Granule）。
* **稀疏路标：** 索引只记录每个包裹的“第一行数据”。查询时，系统通过这些路标判断目标数据**有没有可能**在这个包裹里。

```text
[ 稀疏主键索引 (内存中的路标) ]       [ 物理数据文件 (按 ORDER BY 排序) ]
                                 
(路标 1) ID=1, Time=08:00  -----> [ Granule 1 ] 包含 ID: 1~100 的连续数据
(路标 2) ID=101, Time=09:00 ----> [ Granule 2 ] 包含 ID: 101~200 的连续数据
(路标 3) ID=201, Time=10:00 ----> [ Granule 3 ] 包含 ID: 201~300 的连续数据
```

**查询执行过程 (例: `WHERE ID = 150`)：**
1. 查阅路标：150 大于 101，且小于 201。
2. 锁定范围：目标只可能存在于 Granule 2 中。
3. 数据跳跃：直接跳过 Granule 1 和 Granule 3 (Skip)，只从磁盘加载 Granule 2。

---

## 3. Data Skipping Index (数据跳跃索引)

一张表只能有一个物理排序规则 (`ORDER BY`)。当我们经常需要用**非主键列**进行过滤时，为了避免全表扫描，可以使用 Data Skipping Index 为它们贴上“防拆标签”。

### 3.1 minmax (极值索引)
* **适用场景：** 数据分布有一定局部连续性，或者经常需要按范围查询的非主键字段（如时间、价格、年龄）。
* **示例：**
  ```sql
  CREATE TABLE orders (
      tenant_id String,
      event_time DateTime,
      total_amount Float64,
      -- 为 total_amount 建立极值索引，每 2 个 Granule 生成一个极值区间
      INDEX amt_idx total_amount TYPE minmax GRANULARITY 2
  ) ENGINE = MergeTree()
  ORDER BY (tenant_id, event_time); 
  ```
* **效果：** 当查询 `WHERE total_amount > 10000` 时，若某个块记录的 `max(total_amount)` 仅为 5000，则直接跳过该数据块。

### 3.2 set (集合索引)
* **适用场景：** 枚举值较少、常用于 `IN` 查询的字段（如状态码、商品类目）。
* **示例：**
  ```sql
  INDEX status_idx status TYPE set(100) GRANULARITY 1
  ```
* **效果：** 查询 `WHERE status = 'FAILED'` 时，若索引中的集合里没有该值，则跳过该 Granule。

### 3.3 bloom_filter (布隆过滤器)
* **适用场景：** 基数极大（不重复值非常多）的离散型字段，如 `user_id`、`订单号`，常用于点对点匹配。
* **示例：**
  ```sql
  INDEX user_idx user_id TYPE bloom_filter GRANULARITY 1
  ```
* **效果：** 查询 `WHERE user_id = 'U123456'` 时，布隆过滤器快速验证。如果过滤器判定“绝对没有”，则直接跳过整个 Granule。

---

## 4. ORDER BY 如何设计？

这是 ClickHouse 性能的生死线，**切忌照搬 MySQL 的索引思维**。

假设查询通常是：`WHERE tenant_id = ? AND event_time BETWEEN ? AND ?`
建议设计：`ORDER BY (tenant_id, event_time)`

**设计原则：**让高频过滤条件尽量出现在 ORDER BY 前部。
就像整理衣柜，先按“主人 (tenant_id)”分，再按“季节 (event_time)”分。因为查询通常针对特定主人，这样 ClickHouse 能瞬间排除掉其他主人的所有数据，让跳跃效率最大化。

---

## 5. 查询执行流程

典型流程如下，其核心思想是**尽可能减少需要读取和处理的数据量**：

1. **SQL**
2. **Parser** (解析)
3. **Analyzer / Planner** (分析/计划)
4. **Query Plan** (查询计划)
5. **Read From Storage** (底层存储读取 - 应用稀疏索引与Skipping Index)
6. **Filter** (精确过滤)
7. **Aggregation / Join / Sort** (聚合计算 / 连接 / 排序)
8. **Distributed Merge** (分布式节点结果合并)
9. **Result** (返回结果)

---

## 6. 为什么 SELECT * 是绝对的性能杀手？

```text
【 行式存储 (如 MySQL InnoDB) 】      【 列式存储 (如 ClickHouse) 】
文件按“行”打包。查 1 个字段也要读全行。    文件按“列”切分。查哪个字段就只读哪个文件。

数据块 1: [ID=1, Name=A, Age=20]      ID文件  : [1, 2, 3] -> SELECT ID 极快
数据块 2: [ID=2, Name=B, Age=25]      Name文件: [A, B, C] 
数据块 3: [ID=3, Name=C, Age=30]      Age文件 : [20, 25, 30] 
```

* **SELECT a：** 在列式存储中，系统直接去读取单列数据文件，按需读取，数据量极小。
* **SELECT *：** 系统被迫去分别读取表里的几十甚至几百个列文件，产生巨大的磁盘 I/O。随后在内存中把它们一行一行地“拼接”对齐，疯狂消耗 CPU 和内存，彻底抵消了列式存储的设计红利。
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
