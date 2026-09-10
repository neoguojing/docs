# Apache Cassandra 存储引擎（LSM-Tree）完整技术文档

## 一、核心架构哲学

Apache Cassandra 的存储引擎基于 **LSM-Tree（Log-Structured Merge-Tree）** 架构。其核心哲学是**"绝不就地修改（No in-place updates）"**，通过将所有的随机写转化为顺序追加（Append-only），最大化磁盘吞吐量。

| 维度 | LSM-Tree（Cassandra, RocksDB） | B+ Tree（MySQL, PostgreSQL） |
|------|-------------------------------|------------------------------|
| 存储哲学 | 异地追加（Append-Only） | 原地更新（In-Place Update） |
| I/O 模式 | 全顺序 I/O | 大量随机 I/O |
| 写入性能 | 极高（内存操作 + 顺序刷盘） | 较低（需寻找页分裂、随机写磁盘） |
| 读取性能 | 较慢（需多文件归并，存在读放大） | 极快（通常只需几次磁盘寻道） |
| 空间放大 | 显著（历史版本和墓碑长期共存） | 较小（直接覆盖旧数据） |

## 二、全局架构与写入流（Write Path）

Cassandra 的写入路径是一个高吞吐、低延迟的流水线，数据必须同时写入磁盘日志和内存缓冲才算成功，且写前无需读（No read-before-write）。

### 2.1 写入流程总览

```
[ Client ]
    | (Write Request: Key, Value)
    v
+---------------------------------------------------+
|  Cassandra Node (存储引擎)                          |
|                                                   |
|  1. 追加到尾部                  2. 插入有序结构     |
|  [ Commit Log (磁盘) ]        [ Memtable (内存) ]  |
|  (WAL, 顺序写防丢失)          (跳表结构, 按Key排序) |
|            |                          |            |
|  (节点崩溃时用于恢复)      (满阈值触发 Flush)       |
|                                     |             |
+-------------------------------------|-------------+
                                      v
                             3. 刷写为不可变文件
                      +-------------------------------+
                      | [ SSTable 1 ]   [ SSTable 2 ] | (磁盘)
                      +-------------------------------+
```

### 2.2 Commit Log（提交日志 / WAL）

- **物理位置**：磁盘。通常建议配置在独立的物理磁盘上（默认路径如 /var/lib/cassandra/commitlog/），以避免与数据读写产生 I/O 竞争。
- **核心概念**：系统崩溃后的"后悔药"。所有写操作在进入 Memtable 前，必须先追加到 Commit Log。它不关心数据属于哪个表，只负责按时间顺序记录所有的修改（Mutation）。
- **数据结构**：循环使用的固定大小段文件（Segment Files，默认 32MB）。文件内部是连续的字节流，包含一系列的 Mutation。

```
[ Commit Log Segment 文件内部结构 ]
+---------------+-------------------+-------------------+-------------------+
| File Header   | Mutation 1        | Mutation 2        | Mutation 3 ...    |
| (版本、压缩等)| [表A, Key1, 修改] | [表B, Key2, 删除] | [表A, Key3, 修改] |
|               | + CRC 校验码      | + CRC 校验码      | + CRC 校验码      |
+---------------+-------------------+-------------------+-------------------+
```

### 2.3 Memtable（内存表 / C0 Tree）

- **物理位置**：内存。存在于 JVM 堆内存（Heap）或堆外内存（Off-heap，用于减少 GC 压力）。
- **核心概念**：写缓存与排序器。它接收来自 Commit Log 的随机写入，并在内存中将其整理成按 Partition Key 排序的有序结构。当达到容量阈值时，整个 Memtable 会被冻结并作为一个整体 Flush 到磁盘形成 SSTable。
- **数据结构**：底层实现通常是 ConcurrentSkipListMap（并发跳表）。跳表通过多层链表实现了 O(log n) 的插入和查找复杂度，既保证了并发写入性能，又维护了数据的有序性。

```
[ Memtable - 跳表 (Skip List) 结构抽象 ]

Level 3: [ Head ] ----------------------------------------> [ Key: "Zack" ] -> Null
Level 2: [ Head ] ----------------> [ Key: "Lily" ] ------> [ Key: "Zack" ] -> Null
Level 1: [ Head ] -> [ Key: "Bob" ] -> [ Key: "Lily" ] -> [ Key: "Tom" ] -> [ Key: "Zack" ] -> Null
                     |               |                 |                 |
(底层数据)          (Row Data)      (Row Data)        (Row Data)        (Row Data)
```

### 2.4 Flush（异步刷盘）

当 Memtable 达到配置阈值（如堆内存的 1/4），它会被冻结，不再接受新写入。后台线程将其按顺序遍历，直接以连续的 I/O 流顺序写入磁盘，生成一套完整的全新 SSTable（Data.db, Index.db, Summary.db, Filter.db）。随后，清理对应的 Commit Log 段。

```
[ 客户端请求 ] --> Mutation (bob, age:25, T1)
                     |
         +-----------+-----------+
         v 1. 顺序写磁盘           v 2. 写入内存跳表 (按Key排序)
  [ Commit Log Segment ]      [ Memtable (SkipList) ]
  |...|bob,age:25,T1|...|      Level 1: -> [alice] -> [bob, T1] -> [zack] ->
         |                               |
         | (灾难恢复用)                    | 3. 异步 Flush (顺序转储)
         v                               v
  [ 重放日志恢复内存 ]               [ 新的 SSTable (不可变磁盘文件) ]
```

## 三、局部数据结构：SSTable 核心文件簇

当 Memtable Flush 到磁盘后，并不会只生成一个文件，而是生成一组文件（统称为一个 SSTable，默认路径 /var/lib/cassandra/data/<keyspace>/<table>/）。

### 3.1 Data.db（数据文件）

- **物理位置**：磁盘。
- **数据结构**：按 Partition Key 严格排序的真实数据块。

```
[ Data.db 内部结构 ]
+---------------------+---------------------+---------------------+
| Partition Key: "A"  | Partition Key: "B"  | Partition Key: "C"  |
| - Row 1 (Clustering)| - Row 1             | - Row 1             |
| - Row 2 (Clustering)| - Row 2             |                     |
+---------------------+---------------------+---------------------+

[ Data.db 磁盘文件内部的物理布局 ]

+-------------------------------------------------------+
| Partition 1 (例如: Partition Key = "user_100")         |
+-------------------------------------------------------+
| 1. Partition Header (分区头)                           |
|    - Partition Key 字节数据                            |
|    - Partition Tombstone (如果整个分区被删，记录时间和标记)  |
+-------------------------------------------------------+
| 2. Static Row (静态行 - 可选)                          |
|    - 整个分区共享的静态列数据 (如用户的注册时间)             |
+-------------------------------------------------------+
| 3. Row 1 (逻辑行 1: 例如 Clustering Key = "2026-09-01")|
|    - Row Header: 标志位(是否存在, 是否有TTL等)           |
|    - Clustering Values: "2026-09-01"                  |
|    - Cell A (列名/ID, Delta时间戳, 字节长度, 真实Value)  |
|    - Cell B (列名/ID, Delta时间戳, 字节长度, 真实Value)  |
+-------------------------------------------------------+
| 4. Row 2 (逻辑行 2: 例如 Clustering Key = "2026-09-02")|
|    - Row Header: 标志位                               |
|    - Clustering Values: "2026-09-02"                  |
|    - Cell A (列名/ID, Delta时间戳, 字节长度, 真实Value)  |
|    - Cell C (列名/ID, Delta时间戳, 字节长度, 真实Value)  | 
+-------------------------------------------------------+
| Partition 2 (例如: Partition Key = "user_101")         |
+-------------------------------------------------------+
```

### 3.2 Index.db（分区索引 - Partition Index）

- **物理位置**：磁盘。
- **核心概念**：因为 Data.db 很大，直接全表扫描太慢。Index.db 记录了每个 Partition Key 在 Data.db 中的精准字节偏移量（Offset）。
- **数据结构**：稠密索引（Dense Index），包含该 SSTable 中所有的 Partition Key 及其指向 Data.db 的指针。

```
[ Index.db 内部结构 ]
[ Key: "A" ] -> Offset: 0x0000
[ Key: "B" ] -> Offset: 0x0A2F
[ Key: "C" ] -> Offset: 0x1B50
...
```

### 3.3 Summary.db（分区摘要 - Partition Summary）

- **物理位置**：启动时加载到内存，也有对应的磁盘文件用于持久化。
- **核心概念**：Index.db 依然很大，无法全部放入内存。Summary 是 Index.db 的"目录"。它是一个稀疏索引（Sparse Index），默认每隔 128 个 Key 采样一个。
- **数据结构**：数组。

```
[ Partition Summary 内存结构 ]

(通过二分查找，快速定位目标 Key 在 Index.db 中的大致范围)

索引项 0:   [ Key: "A" ]   -> 指向 Index.db Offset 0
索引项 128: [ Key: "H" ]   -> 指向 Index.db Offset 8192
索引项 256: [ Key: "M" ]   -> 指向 Index.db Offset 16384

查找 "K": 发现 "K" 在 "H" 和 "M" 之间，因此只需要去磁盘读取 Index.db 中 8192 到 16384 这段数据，避免了全量读取 Index 文件。
```

### 3.4 Filter.db（布隆过滤器 - Bloom Filter）

- **物理位置**：启动时加载到内存，持久化在磁盘。
- **核心概念**：在查询任何索引之前，先通过它快速判断一个 Key 是否绝对不存在于当前 SSTable 中。极大减少了不必要的磁盘 I/O。
- **数据结构**：位数组（Bit Array）+ 多个哈希函数。

```
[ Bloom Filter 内存结构 ]

Bit Array: [ 0 | 1 | 0 | 0 | 1 | 1 | 0 | 0 ]
Hash("Bob") -> [1, 4, 5] -> 全部为 1 -> "Bob" 可能存在，继续查 Summary。
Hash("Tom") -> [0, 4, 7] -> 存在 0 -> "Tom" 绝对不存在，直接跳过此 SSTable。
```

### 3.5 SSTable 文件簇全局结构图

```
[ SSTable 文件簇全局结构 ]

磁盘目录: /var/lib/cassandra/data/<keyspace>/<table>/

+------------------+
|  Data.db         |  --> 按 Partition Key 排序的真实数据块
|  (磁盘数据文件)    |
+------------------+
        |
        | 通过 Index.db 定位
        v
+------------------+
|  Index.db        |  --> 稠密索引: Key -> Data.db 的 Offset
|  (分区索引文件)    |
+------------------+
        |
        | 通过 Summary.db 加速定位
        v
+------------------+
|  Summary.db      |  --> 稀疏索引: 每128个Key采样一个, 内存中二分查找
|  (分区摘要文件)    |
+------------------+
        |
        | 快速排除不存在
        v
+------------------+
|  Filter.db       |  --> 布隆过滤器: 位数组+多哈希, 判断Key是否绝对不存在
|  (过滤器文件)      |
+------------------+

注: 每个 SSTable 对应一组上述文件。随着 Compaction 进行，
    多个 SSTable 合并为新的 SSTable，旧文件被删除。
```

## 四、读取流（Read Path）：多级过滤与多路归并

由于 SSTable 是不可变的，同一个 Key 的不同版本可能散落在多个 SSTable 中。为了在海量文件中高效读取，Cassandra 设计了多级过滤与索引算法。读取本质上是一个**多路查找并按时间戳合并（Merge on Read）**的过程。

### 4.1 读取流程总览

```
(Read Request: Get Key "Alice")
      |
      v
1. [ Memtable ] (内存中有最新修改吗？) --有--> 记录并继续
      | (向下查找 SSTables)
      v
2. [ Bloom Filter ] (内存: Filter.db)
      | 算法: 快速判断 "Alice" 是否绝对不在当前 SSTable。
      | 若大概率存在，则进入下一步。
      v
3. [ Key Cache ] (内存)
      | 缓存了热点数据在 Index.db 中的精准位置。
      v (未命中)
4. [ Partition Summary ] (内存)
      | 算法: 稀疏索引。抽取 Index.db 中的部分主键建立区间。
      | 作用: 告诉系统 "Alice" 的索引在磁盘的 Offset X 到 Y 之间。
      v
5. [ Partition Index ] (磁盘: Index.db)
      | 根据 Summary 给出的区间，在磁盘中进行二分查找，定位精确数据地址。
      v
6. [ Data ] (磁盘: Data.db)
      | 读取真实数据，并与 Memtable 及其他 SSTable 的数据进行时间戳合并(Merge)。
```

### 4.2 查询场景走查

**场景设定**：

- 查询语句：SELECT email, age FROM users WHERE user_id = 'alice';
- Memtable（内存）：包含 Alice 最近的一次修改（年龄改为 31岁，时间戳 T3）
- SSTable 1（较新）：包含 Alice 的邮箱修改记录（email: alice@new.com，时间戳 T2）
- SSTable 2（旧）：完全没有 Alice 的数据
- SSTable 3（最旧）：包含 Alice 的初始注册数据（age: 30, email: alice@old.com，时间戳 T1）

**第一阶段：内存探查（Memory Phase）**

步骤1: 检索 Memtable (内存)
  动作: 在 ConcurrentSkipListMap 中查找 user_id = 'alice'
  结果: 命中！找到 age = 31 (T3)
  决策: 因为查询还需要 email 字段，且 Cassandra 无法确定 Memtable 里的数据
        是否是完整的行（可能有其他字段在磁盘上），系统必须继续向下查找磁盘文件。

步骤2: 探测 SSTable 1 (较新文件)
  Bloom Filter (Filter.db - 内存):
    将 "alice" 经过多个 Hash 函数计算，检查位数组对应的比特位。
    发现全部为 1。结论：大概率存在，继续。
  Key Cache (内存):
    检查 Key Cache 是否缓存了 "alice" 在 SSTable 1 中的索引位置。
    结果：未命中 (Cache Miss)。
  Partition Summary (Summary.db - 内存):
    在内存中的稀疏索引数组执行二分查找。
    算法发现："alice" 在采样点 "aaron" (指向 Index.db 偏移量 1000)
    和 "andy" (指向偏移量 2000) 之间。
    输出：磁盘寻址区间 [1000, 2000]。

**第二阶段：磁盘寻址（Disk Phase）**

步骤3: 检索 SSTable 1 的磁盘文件
  Partition Index (Index.db - 磁盘):
    动作: 磁头 Seek 到文件偏移量 1000 处，开始顺序扫描，直到遇到 "alice"。
    结果: 在偏移量 1350 处找到 "alice"，读取其对应的数据指针
          （假设为 Data.db 偏移量 8888）。
  Data (Data.db - 磁盘):
    动作: 磁头 Seek 到 Data.db 偏移量 8888 处读取数据块。
    结果: 读取到 email = alice@new.com (T2)。

**第三阶段：布隆过滤器立功**

步骤4: 探测 SSTable 2 (旧文件)
  Bloom Filter (内存):
    对 "alice" 计算 Hash，发现有一个映射的比特位是 0。
    决策：绝对不存在！
    效果：直接跳过 SSTable 2。完美避免了一次昂贵的磁盘 I/O (读 Index 和 Data)。

**第四阶段：缓存命中**

步骤5: 探测 SSTable 3 (最旧文件)
  Bloom Filter (内存): 所有特征位均为 1。结论：大概率存在。
  Key Cache (内存):
    动作: 查找 "alice" 在 SSTable 3 中的位置。
    结果: 命中！(之前有人查过)。
    缓存直接返回了指向 Data.db 偏移量 5555 的指针。
    决策: 直接跳过 Partition Summary (内存) 和 Partition Index (磁盘)。
  Data (Data.db - 磁盘):
    Seek 到 5555，读取到 age = 30 (T1), email = alice@old.com (T1)。

### 4.3 最终合并（Merge）

```
[ 时间戳合并 (Merge) ]

数据源              获取到的字段          时间戳(Timestamp)    胜出者判断逻辑
Memtable            age: 31              T3                  T3 > T1，age 取 31
SSTable 1           email: alice@new.com T2                  T2 > T1，email 取 alice@new.com
SSTable 2           (无数据)             --                  --
SSTable 3           age: 30,             T1                  全部被更高时间戳的版本覆盖
                    email: alice@old.com                        (被淘汰)

最终返回给客户端的结果:
{ user_id: 'alice', age: 31, email: 'alice@new.com' }
```

### 4.4 寻址路径全局摘要图

```
查询 Key: "alice"
      |
      v
[ Memtable ] ──(命中)──> 保存版本，由于需确保数据完整性，必须向下探查
      |
      v (对每一层 SSTable)
[ Bloom Filter ] ──(有0位)──> 绝对不存在！跳过整个 SSTable。(省下巨量磁盘I/O)
      |
      v (全为1)
[ Key Cache ] ──(命中)──> 【直达 Data.db！】(省下读取 Index 磁盘 I/O)
      |
      v (未命中)
[ Partition Summary ] ──> 在内存中二分查找，圈定 Index.db 的磁盘扫描范围。
      |
      v
[ Partition Index (磁盘) ] ──> 在限定范围内顺序扫描磁盘，找到目标 Key 的精确 Offset。
      |
      v
[ Data (磁盘) ] ──> 根据精确 Offset 读取真实数据，参与最终合并。
```

## 五、删除机制：墓碑（Tombstone）

由于 SSTable 不可变，系统无法深入旧文件抹除数据，删除实际上是一次**特殊的插入**。

### 5.1 删除执行流程

```
[ 删除操作走查: DELETE age FROM users WHERE user_id = 'alice' (时间戳 T4) ]

步骤1: 写入墓碑 (Tombstone)
  引擎并不去查找 'alice' 在哪里。
  而是直接向 Commit Log 和 Memtable 插入一条带有特殊标记的记录，
  称为 Tombstone，并附带当前时间戳 T4。

步骤2: 查询时的遮蔽 (Masking)
  当读取 'alice' 时，查询流程会同时扫出旧 SSTable 中的
  age:30 (T1) 和新生成的 Tombstone (T4)。
  在最后的 Merge 阶段，算法比较时间戳：T4 > T1。
  系统发现高版本是一个墓碑，于是判定该数据已被删除，向客户端返回 null。

步骤3: 墓碑的副作用 (Read Amplification)
  删除数据反而会占据更多磁盘空间（多写入了一个墓碑）。
  如果大量删除，查询时必须读取大量实际上已经被删除的数据和墓碑进行比对，
  导致查询极度缓慢（著名的 Tombstone Overwhelming 异常）。
```

### 5.2 墓碑遮蔽效应图示

```
[ 磁盘合并读取时的遮蔽效应 (Merge Engine) ]

SSTable 3 (旧): [ alice | age:30 | T1 ] ──┐
                                            │ 比较时间戳
SSTable 1 (新): [ alice | age (Tombstone) | T4 ] ──┴--> T4 > T1, 返回 Null

* Tombstone 就像一张写着"此数据已死"的便利贴，贴在旧数据之上。
```

## 六、后台清理：Compaction（压实）

Compaction 是后台的清道夫，负责合并文件、回收空间和整理碎片，其核心算法是 **K路归并排序**。

### 6.1 Compaction 执行流程

```
[ K路归并排序 与 冲突解决 (Compaction) ]

  SSTable A (旧):  [ alice, age:30(T1) ]       [ bob, age:25(T1) ]
  SSTable B (中):  [ alice, email:x@x(T2) ]    [ bob, TOMBSTONE(T3) ]
  SSTable C (新):  [ alice, age:31(T4) ]       [ clark, age:40(T4) ]
         |                |                            |
         └────────────────┼────────────────────────────┘
                          v K-Way Stream Merge

[ 内存中的 Merge 逻辑 ]
Key: alice -> age:31(T4) 胜出 (丢弃T1), email:x@x(T2) 保留
Key: bob   -> TOMBSTONE(T3) 胜出 (丢弃T1). 假设已过 10 天宽限期，直接彻底销毁 bob。
Key: clark -> age:40(T4) 保留

                          v 顺序流式写出 (无锁、全速 I/O)

  SSTable D (全新): [ alice, age:31(T4), email:x@x(T2) ]  [ clark, age:40(T4) ]

(合并完成后，彻底删除物理文件 SSTable A, B, C)
```

### 6.2 Compaction 核心步骤

1. **并行读取**：引擎同时打开 A, B, C 三个文件的流。由于这三个文件内部都是按 Partition Key 严格排序的，合并只需从头顺序拉取，不需要全量加载到内存。
2. **版本冲突与丢弃（LWW）**：当三个流同时读取到同一个 Key 时，按列对比时间戳。保留时间戳最大的版本，直接丢弃旧版本。
3. **墓碑回收（GC Grace Seconds）**：检查 Tombstone 的生成时间。如果距离现在已经超过了配置的 gc_grace_seconds（默认 10 天），并且确信更老的记录已经被清理，这个 Tombstone 就会被彻底丢弃（真正释放空间）。
4. **生成新文件并原位替换**：合并后的纯净数据以顺序写的方式生成一个新的大型 SSTable (SSTable D)。SSTable D 构建完成后，系统修改元数据指向 D，然后异步删除旧的 A, B, C 文件。

### 6.3 Compaction 策略对比

| 策略 | 全称 | 核心算法 | 触发条件 | 最佳适用场景 | 缺点 |
|------|------|----------|----------|--------------|------|
| STCS | Size-Tiered Compaction Strategy | 将大小相近的多个 SSTable 合并为一个更大的 | 写入触发 | 写入密集型（默认策略） | 会引发空间放大（临时需要双倍磁盘） |
| LCS | Leveled Compaction Strategy | 数据分层管理，每层容量呈 10 倍递增，层内文件互不重叠 | 层级阈值触发 | 读取密集型、更新频繁的场景 | 写放大较大 |
| TWCS | Time-Window Compaction Strategy | 按时间窗口（如一天）将 SSTable 分桶，过期整个桶直接删除 | 时间窗口到期 | 时序数据（IoT、监控日志）、TTL 过期数据 | 不适合随机读写场景 |

### 6.4 Compaction 的代价

这种极端的空间优化和读取优化，代价是**写放大（Write Amplification）**。同样一条数据，在其生命周期内会被反复从磁盘读出来、合并、再写回磁盘多次，大量消耗后台的 I/O 和 CPU 资源。

## 七、LSM-Tree 全局架构总览

```
[ LSM-Tree 全局架构与 Cassandra 映射图 ]

[ Client Write: (Key: "Bob", Age: 25) ]
       |
       ├──> 1. 追加日志 (防数据丢失)
       |    [ Commit Log / WAL ] (磁盘 - 纯顺序写)
       |    └─> "Bob"|25|T1 --> "Alice"|30|T2
       |
       └──> 2. 写入内存树 (C0 Tree)
            [ Memtable ] (内存 - 跳表或红黑树, 按Key排序)
            └─> [Alice:30|T2] --> [Bob:25|T1] --> [Charlie:22|T0]
                   |
                   | (内存满，触发 Flush)
                   v
                   3. 刷写为不可变磁盘树 (C1 Tree)
             [ Level 0 SSTables ] (磁盘 - 顺序写出，文件内数据有序)
             [SSTable 1: Alice-Charlie]   [SSTable 2: David-Eve]
                   |
                   | 4. 触发归并 (Compaction)
                   v
                   5. 合并形成更大的磁盘树 (C2 Tree, C3 Tree...)
             [ Level 1 SSTables ] (合并清理过期数据和墓碑)
             [SSTable 3: Alice-Eve (数据更加紧凑)]
```

## 八、LSM-Tree 与 B+Tree 本质对比

| 对比维度 | LSM-Tree（Cassandra, RocksDB） | B+ Tree（MySQL, PostgreSQL） |
|----------|-------------------------------|------------------------------|
| 存储哲学 | 异地追加（Append-Only） | 原地更新（In-Place Update） |
| I/O 模式 | 全顺序 I/O | 大量随机 I/O |
| 写入性能 | 极高（内存操作 + 顺序刷盘） | 较低（需寻找页分裂、随机写磁盘） |
| 读取性能 | 较慢（需多文件归并，存在读放大） | 极快（通常只需几次磁盘寻道） |
| 空间放大 | 显著（历史版本和墓碑长期共存） | 较小（直接覆盖旧数据） |
| 适用场景 | 写密集型、大数据分析、时序数据 | 读密集型、事务型 OLTP 场景 |
| 崩溃恢复 | Commit Log 重放，恢复速度快 | Redo Log 重放，恢复速度取决于日志量 |
| 数据更新 | 追加新记录，旧记录标记为墓碑 | 原地覆盖旧页 |
| 数据删除 | 写入 Tombstone，延迟释放空间 | 直接标记页内记录为删除，快速释放 |

## 九、总结

Cassandra 的存储引擎设计哲学可以概括为：

- **写的时候拼命往后追加**（顺序写，快）
- **读的时候靠各种内存索引和过滤器去猜位置**（省 I/O）
- **闲下来的时候在后台慢慢整理合并**（Compaction）

这种设计以写放大和空间放大为代价，换取了极致的写入吞吐量和可线性扩展的架构能力，使其成为大规模分布式数据存储的理想选择。
