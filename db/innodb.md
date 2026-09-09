# MySQL 面试核心知识框架

> **核心主线**：定位(OLTP) → 存储(B+Tree/Page) → 内存(Buffer Pool) → 并发(MVCC+Lock) → 日志(Redo/Undo/Binlog) → 高可用(Replication)。

---

## 1. 定位与场景
*   **核心特性**：关系模型 (Schema 强约束) + SQL + ACID 事务 + 丰富索引。
*   **适用场景**：高并发 OLTP 业务（订单、支付、用户、库存等强一致性场景）。
*   **不适场景**：超大规模 OLAP 分析（如全表 JOIN/聚合）、全文搜索、极低延迟 KV 缓存、海量时序/物联网数据写入。

## 2. 存储引擎 (InnoDB 核心)
*   **存储结构**：数据组织核心单位是 **Page** (默认 16KB)。
*   **B+Tree 索引**：树高度低，叶子节点有序相连，支持 $O(\log N)$ 范围查询与精确查找，极其适配磁盘/SSD 存储。
*   **写入机制 (异步落盘)**：`UPDATE` → 修改 Buffer Pool 中的 Page → 产生 Undo + Redo → `Commit` → Redo 刷盘持久化 (WAL) → 数据页后续异步刷盘。

## 3. 内存与数据结构
*   **Buffer Pool**：最重要内存区域，缓存数据页和索引页，减少磁盘 I/O。
*   **LRU 淘汰优化**：分为 New List (5/8, 热数据) 和 Old List (3/8, 冷数据)。新页先入 Old 区，被再次访问才移入 New 区，防止全表扫描导致热数据被污染洗出。
*   **Change Buffer**：暂存对“不在内存中的二级索引”的修改，待该索引页后续被读入时再 Merge，将随机 I/O 转化为顺序 I/O。
*   **Adaptive Hash Index (AHI)**：根据热点访问模式自动建立 Hash 映射，将 $O(\log N)$ 树检索降维至 $O(1)$ 内存直达。仅限等值查询，不支持范围及 `LIKE`。
*   **Log Buffer**：缓存 Redo Log，按批次刷盘减少 I/O 频率。

## 4. 索引与查询
*   **聚簇索引**：主键索引的叶子节点直接保存**完整数据行**。
*   **二级索引**：叶子节点保存**主键值**。若查询非索引覆盖字段，需拿到主键后回到聚簇索引查找，称为**回表**。
*   **覆盖索引**：查询所需字段已全部在二级索引中，无需回表，直接返回。

## 5. 并发与一致性 (事务)
*   **ACID 保障**：Atomicity (Undo)、Consistency (约束+事务)、Isolation (MVCC+Lock)、Durability (Redo Log)。
*   **隔离级别**：InnoDB 默认隔离级别为 **RR (Repeatable Read)**。普通 `SELECT` 是一致性非锁定读；`SELECT ... FOR UPDATE` 是锁定读。
*   **MVCC (多版本并发控制)**：解决读写并发相互阻塞问题。
    *   **机制**：当前数据行隐藏字段 (`DB_TRX_ID`, `DB_ROLL_PTR`) → 顺指针找 Undo Log 中的历史版本快照 → 结合 Read View 判断该版本对当前事务是否可见。
*   **锁机制**：用于写写冲突或锁定读。含 Record Lock (行锁)、Gap Lock (间隙锁)、Next-Key Lock、Intention Lock (意向锁)。

## 6. 持久化与日志
*   **Redo Log (引擎层)**：实现 **WAL (预写日志)**。记录物理修改，循环写入。核心用于 **Crash Recovery (崩溃恢复)**，保证已提交且未刷盘的数据不丢失。
*   **Undo Log (引擎层)**：记录逻辑修改。用于 **事务回滚** 和提供 **MVCC 历史版本快照**。
*   **Binlog (Server 层)**：记录逻辑/语句变更。持续追加写入，主要用于 **主从复制** 和 **数据恢复 (PITR)**。
*   **两阶段提交**：协调 Redo 和 Binlog 的写入 (`Prepare` → 写 Binlog → `Commit`)，确保引擎层和 Server 层状态绝对一致。
*   **Doublewrite Buffer**：脏页刷盘前先顺序写入该区域，防止宕机引发部分页写入 (Partial Page Write) 导致数据页不可逆损坏。

## 7. 分布式与高可用
*   **主从复制 (Replication)**：Primary 写 Binlog → Replica 拉取并重放 (Replay)。用于读写分离、高可用、容灾。
*   **分库分表**：应对单机存储/并发瓶颈，方案包括 Hash、Range、按业务拆分等（需引入分布式事务和 Sharding 路由）。
*   **崩溃恢复流程**：Crash → Redo 前滚重做 → Undo 回滚未提交事务 → 恢复到一致性状态。

## 8. 一句话总结回答模板
“InnoDB 是面向 OLTP 的关系型引擎。它底层以 **B+Tree** 和 **Page** 组织数据；依靠 **Buffer Pool** 拦截磁盘 IO；在并发控制上通过 **MVCC + 锁** 保障事务隔离；写入时遵循 **WAL**，依靠 **Redo Log** 实现崩溃自愈，依靠 **Undo Log** 支持回滚，并结合 Server 层的 **Binlog** 进行数据复制与高可用架构。”
