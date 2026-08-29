# 数据库与数据系统设计指南 (Database & Data Systems Knowledge Map)

> 本文档基于《数据密集型应用系统设计》（DDIA）的核心思想，全面梳理了现代数据系统中的存储引擎、分布式架构、一致性保障与衍生数据处理的技术选型与底层机制。

---

## 1. 数据处理范式 (OLTP vs OLAP)

在 DDIA 体系中，数据系统按访问模式分为两类：

*   **OLTP (联机事务处理)**：面向业务事件记录（如订单、账户、ERP/OA），特点是高并发、低延迟、小批量读写，写放大与并发控制（锁/MVCC）是核心关注点。
*   **OLAP (联机分析处理)**：面向数据分析与报表（如数据仓库、BI），特点是低并发、大批量扫描与聚合计算，数据通常由 OLTP 系统通过 ETL/CDC 抽取而来，核心关注列式压缩与缓存/向量化执行。

---

## 2. 核心系统对照矩阵 (Systems Comparison Matrix)

| 系统 | 数据模型 | 内存与缓存机制 | 磁盘与存储结构 | 索引机制 | 并发控制与锁 | 高可用与复制 | 持久化与崩溃恢复 |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| **MySQL (InnoDB)** | 关系型 (Row) | Buffer Pool (LRU 算法) | 页结构 (16KB Page)，Change Buffer (非聚簇随机写转顺序写) | B+ Tree (聚簇与二级索引) | 行锁、表锁、意向锁、间隙锁 (Next-Key Lock) | Async/Semi-sync 复制 (基于 Binlog) | Redo Log (WAL 崩溃恢复) + Doublewrite |
| **LevelDB** | Key-Value | MemTable (SkipList) + Immutable MemTable | SSTable (LSM-Tree，分层 Compaction，不活跃数据下沉) | Sparse Index + Block Index + Bloom Filter | 单线程写 / 行级原子更新 (无显式事务锁) | 无分布式支持 (单机引擎) | WAL (顺序写日志文件) |
| **ElasticSearch**| 文档 (JSON) | OS PageCache + Node Query/Fielddata Cache | Segment 文件 (不可变，定期 Segment Merge) | Term Index (FST 变种) + Term Dict + Posting List | 乐观并发控制 (_version / primary_term) | Primary-Replica (ES Cluster 选举) | Translog (顺序写入追加，崩溃恢复) |
| **HDFS** | 分布式文件 | NameNode 内存 (保存文件 Block 与 DataNode 映射元数据) | 物理磁盘 DataBlock (默认 128M 块切分) | 文件路径映射树 (NameNode 维护) | 命名空间级读写锁 | 多副本机制 (Rack Awareness)，NameNode HA | EditLog (WAL) + FSImage (元数据快照) |
| **HBase** | 稀疏列族 | MemStore (SkipList 排序) + BlockCache (LRU 读缓存) | HFile (LSM-Tree，底层依赖 HDFS DataBlock) | HFile Block Index + Bloom Filter | 行级锁 + MVCC (Region 内原子写) | RegionServer 宕机自愈 (依赖 ZK/HDFS) | WAL (HLog，每个 RegionServer 共享或独立顺序追加) |
| **Cassandra** | 宽列族 (CQL)| MemTable (内存有序表) + Key/Row Cache | SSTable (LSM-Tree，基于 Compaction 清理) | Partition Key Hash Index + Clustering Key Index + Bloom Filter | 无全局锁，LWW (Last-Write-Wins) 或轻量级事务 (Paxos) | 无主复制 (Dynamo)，Gossip 协议，Hinted Handoff，Read Repair | CommitLog (追加写日志) |
| **MongoDB** | 文档 (BSON) | WiredTiger Cache (内存数据页缓存) | WiredTiger 数据块 (支持 B-Tree 或 LSM-Tree，默认 B-Tree) | B-Tree (支持单字段、复合、文本、地理等) | 意向锁与读写锁 (Global/DB/Collection/Document 粒度) | Replica Set (Primary-Secondary)，自动 Leader 选举 | Oplog (操作日志) + WiredTiger Journal (WAL) |
| **ClickHouse** | 列式 (OLAP) | Mark Cache (标记缓存) + Uncompressed Block Cache | MergeTree 数据片段 (Data Parts)，列式存储 + 压缩 (LZ4/ZSTD) | 主键稀疏索引 (Sparse Primary Index) + 跳数索引 | 无传统事务锁，数据追加写入 (Append-only Parts) | ClickHouse Keeper + ReplicatedMergeTree 异步复制 | 数据 Part 写完即持久化，支持 WAL 校验 |
| **Kafka** | 分布式消息流| OS PageCache (极度依赖 zero-copy / mmap) | Segment 文件 (.log 追加写) + 稀疏索引 (.index / .timeindex) | 稀疏偏移量索引 (Offset Index) | 分区级无锁追加 (Single-writer per partition) | Leader-Follower 复制，ISR 集合维护 | 磁盘追加日志 + ACK 机制 (0, 1, all) |
| **InfluxDB** | 时序数据 | Cache (内存未持久化点) + WAL Buffer | TSM (Time Structured Merge-Tree) 文件 | TSI (Time Series Index，倒排索引管理 Tag/Series) | 无事务锁，写追加优化 | 高可用集群 (Enterprise) / 单机开源版 | WAL (追加写时序日志) |
| **Redis** | 内存 KV | 纯内存存储 (jemalloc)，支持 LRU/LFU | RDB 快照文件 / AOF 日志文件 (非直接磁盘读写) | 哈希表 (Dict) + 跳表 (SkipList，用于 ZSet) | 单线程事件循环 (全内存无锁) / 6.0+ 多线程网络 IO | Redis Sentinel (哨兵) / Redis Cluster (分片集群) | AOF (写日志/重写机制) + RDB (fork CoW 快照) |

---

## 3. 核心机制专题拆解 (DDIA 理论落地)

### 3.1 存储引擎设计 (B-Tree vs LSM-Tree)

*   **B-Tree 体系 (如 MySQL InnoDB)**：
    *   **更新模式**：原地更新 (In-place Update)。读取效率高 (读放大低)，但产生磁盘随机写 (写放大高)。
    *   **写优化**：使用 **Change Buffer** 缓存非聚簇索引页的修改，将随机写在后台合并转换为顺序写。
*   **LSM-Tree 体系 (如 LevelDB, HBase, Cassandra)**：
    *   **更新模式**：追加写 (Append-only)。写吞吐极高 (写放大低)，但存在数据重叠与多版本，导致读延迟升高 (读放大高) 与无效数据占用 (空间放大高)。
    *   **优化手段**：
        1. **MemTable + WAL**：内存 SkipList 保持写入有序，WAL 保证持久性。
        2. **SSTable + Bloom Filter**：磁盘文件按 Key 排序，借助布隆过滤器迅速判断 Key 是否不存在，避免不必要的磁盘 IO。
        3. **Compaction (合并重写)**：后台定期执行 Compaction，消除旧版本与删除标记 (Tombstone)。

### 3.2 索引与数据检索机制

*   **B+ Tree 索引**：非叶子节点存储 Key，叶子节点存储 Key+Data/指针并双向单链表相连。适合范围扫描与点查。
*   **全文/倒排索引 (ElasticSearch)**：
    *   **Term Index**：基于 FST (Finite State Transducer) 将前缀树压入内存，定位 Term 在 Dict 中的位置。
    *   **Term Dict**：包含关键字，按字典序排列，使用二分查找。
    *   **Posting List**：记录文档 ID 列表，采用 FOR 与 Roaring Bitmaps 压缩。
*   **列式稀疏索引 (ClickHouse)**：按主键列每隔固定行 (如 8192 行) 建立一个索引标记，配合列数据独立压缩存储，实现海量数据高速过滤。

### 3.3 分布式数据分片与路由 (Partitioning & Sharding)

*   **哈希/一致性哈希 (Consistent Hashing)**：
    *   **代表系统**：Cassandra、Redis Cluster。
    *   **特点**：缓解节点扩缩容时的大规模数据迁移，Cassandra 结合虚拟节点 (Virtual Nodes) 进一步均衡负载分布。
*   **范围/Region 分片 (Range-based Partitions)**：
    *   **代表系统**：HBase (Region)、MongoDB (Chunk)。
    *   **特点**：支持高效范围查询，容易因递增 Key 造成热点写入。HBase 通过 Region 动态分裂 (Split) 与合并 (Merge) 弹性扩展。

### 3.4 高可用、数据持久化与崩溃恢复

*   **写预留日志 (WAL / Redo Log / AOF)**：
    *   **MySQL Redo Log**：物理/逻辑混合日志，环形写入，结合 Doublewrite Buffer 解决半页刷盘失败。
    *   **Redis AOF**：文本命令追加。通过 `fork()` 利用操作系统 **Copy-on-Write (CoW)** 生成新 AOF，父进程用 AOF Rewrite Buffer 暂存增量命令。
    *   **LevelDB / HBase WAL**：先顺序写磁盘 WAL，再更新内存，保证崩溃时数据可从 Log 重放恢复。
*   **数据快照 (Snapshots / Binlog)**：
    *   **MySQL Binlog**：逻辑日志，用于主从复制与 PITR。
    *   **Redis RDB**：全量二进制快照，基于 CoW 极快导出，恢复极快。

### 3.5 事务与并发控制 (Transactions & Isolation)

*   **并发控制三大技术**：
    *   **封锁协议 (Locking)**：行锁、表锁、意向锁 (IS/IX)。用于解决脏写，实现排他性。
    *   **多版本并发控制 (MVCC)**：结合 Undo Log 与 Read View，实现“读不加锁，读写不冲突” (Snapshot Isolation 核心)。
    *   **间隙锁与 Next-Key Lock**：MySQL 在 RR (Repeatable Read) 级别下，锁住记录间的空隙，防止幻读 (Phantom Read)。
*   **分布式事务与最终一致性**：
    *   **强一致提交**：两阶段提交 (2PC)，存在单点与阻塞等待开销。
    *   **Dynamo 风格最终一致性**：通过 Quorum 配置 ($W + R > N$)，结合 Read Repair (读修复) 与 Anti-Entropy (基于 Merkle Tree 的后台反熵校准) 达成最终一致。
