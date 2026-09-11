# Elasticsearch 技术文档

## 1. 定位与数据模型

### 核心定位与解决的问题
Elasticsearch 是一个基于 Apache Lucene 构建的分布式、RESTful 风格的搜索与数据分析引擎。它的核心定位是提供近实时的全文检索、结构化搜索、日志分析以及指标监控能力。作为 ELK Stack（Elasticsearch, Logstash, Kibana）的核心组件，它主要解决以下问题：
- **全文检索**：支持复杂文本匹配、模糊查询、同义词、拼音搜索等。
- **日志与事件分析**：高吞吐写入海量日志，支持多维度聚合分析。
- **指标监控**：存储时序数据，支持实时统计与告警。
- **企业级搜索**：提供跨应用、跨数据源的统一搜索体验。

### 数据模型核心概念
Elasticsearch 采用面向文档的 NoSQL 数据模型，数据以 JSON 格式存储。核心概念包括：
- **索引 (Index)**：具有相似特征的文档集合，类似于关系型数据库中的"表"。
- **文档 (Document)**：索引中的最小数据单元，以 JSON 格式表示，类似于"行"。每个文档都有一个唯一的 `_id`。
- **字段 (Field)**：文档中的键值对，类似于"列"。
- **类型 (Type)**：在 7.x 版本前，一个索引下可以有多个类型；从 7.x 开始，一个索引只能有一个类型 `_doc`，8.x 中已彻底移除。

### 与关系型数据库的对比映射

| 关系型数据库 (RDBMS) | Elasticsearch | 说明 |
| :--- | :--- | :--- |
| Database | Index | 逻辑上的数据隔离边界 |
| Table | Type (已废弃) / Index | ES 推荐一个业务对应一个索引 |
| Row | Document | JSON 格式，无固定 Schema |
| Column | Field | 支持嵌套对象和数组 |
| Schema | Mapping | 定义字段类型、分词器等 |
| Index | Inverted Index | 底层数据结构完全不同 |
| SQL | Query DSL | 声明式 JSON 查询语言 |

### 映射 (Mapping) 机制
Mapping 定义了文档及其包含的字段如何被存储和索引。
- **动态映射 (Dynamic Mapping)**：ES 根据写入文档的字段值自动推断类型（如字符串推断为 `text` + `keyword`）。生产环境中**强烈建议关闭**（设置为 `false` 或 `strict`），以避免类型冲突和映射膨胀。
- **显式映射 (Explicit Mapping)**：在创建索引时明确定义字段类型、分词器、是否索引等。这是最佳实践。
- **核心数据类型**：
  - `text`：用于全文检索，写入时会被分词器拆分。
  - `keyword`：用于精确匹配、排序和聚合，不分词。
  - `date`：支持多种格式，内部存储为 UTC 毫秒时间戳。
  - `nested`：用于对象数组，保持对象内部字段的关联关系。

### 倒排索引的基本概念
倒排索引是 ES 实现全文检索的核心数据结构。它将文档内容分词后，建立"词项 (Term)"到"文档列表 (Posting List)"的映射。查询时，通过词项直接定位到包含该词的文档 ID，避免了全表扫描。

## 2. 存储引擎

### Lucene 的角色
Elasticsearch 本身不直接处理数据的底层存储，而是委托给底层的 Apache Lucene。Lucene 负责倒排索引的构建、段的合并、文件的读写等核心存储逻辑。ES 在其之上封装了分布式协调、REST API、集群管理等能力。

### 段 (Segment) 的概念
Lucene 的存储单元是段 (Segment)。每个段是一个独立的、**不可变**的倒排索引。
- **不可变性**：段一旦写入磁盘，就不会被修改。删除或更新操作实际上是标记删除或写入新段。这避免了锁竞争，极大提升了并发读性能。
- **按大小合并**：随着写入增加，小段会不断合并为大段，以回收删除标记的空间并提升查询效率。

### 写入流程
1. **请求进入**：文档写入主分片。
2. **内存缓冲区 (In-Memory Buffer)**：文档首先写入内存缓冲区，此时不可搜索。
3. **Translog 写入**：同时写入事务日志 (Translog) 并 `fsync` 到磁盘，保证崩溃恢复。
4. **刷新 (Refresh)**：默认每 1 秒，内存缓冲区的内容被写入文件系统缓存 (OS Page Cache) 并生成一个新的段（此时可搜索）。这是**近实时 (NRT)** 的代价。
5. **提交 (Flush / Commit)**：定期或 Translog 过大时，OS Page Cache 中的段被 `fsync` 到物理磁盘，同时清空 Translog。

### Translog 的作用
Translog 是 Elasticsearch 保证数据持久化和事务性的关键。由于 Refresh 只是将数据放入 OS Cache，如果此时节点宕机，内存中的数据会丢失。Translog 记录了所有未持久化到段文件的操作，节点重启时通过重放 Translog 恢复数据。

### 合并策略 (Segment Merge)
后台线程会异步触发段合并。合并过程会将多个小段合并为一个大段，同时物理删除被标记为删除的文档。合并策略通常基于段的大小和数量，可通过 `index.merge.policy` 调整。

### 近实时搜索 (NRT)
Elasticsearch 的搜索延迟通常在 1 秒左右，这取决于 `refresh_interval` 配置。如果业务允许，可以调大该值（如 30s）以提升写入吞吐；对于实时性要求极高的场景，可手动调用 `_refresh` API，但会严重影响性能。

## 3. 内存与数据结构

### Fielddata
Fielddata 是用于 `text` 字段排序和聚合的内存数据结构。由于 `text` 字段被分词，无法直接用于聚合。开启 Fielddata 会将所有词项加载到 JVM 堆内存中，极易导致 OOM。**生产环境应使用 `keyword` 类型或开启 `doc_values` 进行聚合**。

### 倒排索引的内存驻留
- **Term Dictionary**：存储词项及其在 Postings List 中的位置。由于数据量大，通常使用 FST (Finite State Transducer) 结构驻留内存，实现前缀快速查找。
- **Term Index**：Term Dictionary 的索引，完全驻留内存，用于快速定位 Term Dictionary 中的块。
- **Postings List**：文档 ID 列表，通常存储在磁盘，通过 OS Page Cache 缓存。

### 缓存机制
- **Request Cache**：缓存整个聚合查询的结果。如果查询条件完全一致且没有 `now` 等动态函数，直接返回缓存结果。
- **Query Cache**：缓存 Filter 上下文中的查询结果（即文档 ID 的 BitSet）。对于频繁使用的过滤条件，缓存命中率极高。
- **Field Data Cache**：缓存 Fielddata 和 Doc Values。

### 堆内存管理
Elasticsearch 严重依赖 JVM 堆内存。核心原则：
- **不超过 32GB**：由于 JVM 的指针压缩 (Compressed Oops) 机制，超过 32GB 后指针占用翻倍，实际可用内存反而下降。
- **预留 50% 给 OS Page Cache**：Lucene 依赖 OS Cache 加速段文件的读取，堆内存过大导致 OS Cache 不足，查询性能急剧下降。

### 内存优化策略
- 避免在 `text` 字段上聚合。
- 查询时禁用 `_source` 或仅获取必要字段。
- 合理设置 `index.cache.request.size`。
- 监控 JVM GC 频率和耗时，及时扩容或优化查询。

## 4. 索引与查询

### 索引创建流程
1. **映射解析**：校验 Mapping 合法性，构建内部数据结构。
2. **分片分配**：根据 `number_of_shards` 和 `number_of_replicas` 在集群节点上分配主分片和副本分片。
3. **初始化 Lucene**：每个分片初始化独立的 Lucene IndexWriter。

### 查询执行流程 (Query Then Fetch)
1. **Query Phase**：协调节点将查询分发到所有相关分片。每个分片在本地执行查询，返回匹配的文档 ID 和排序值（不返回文档内容）。
2. **Fetch Phase**：协调节点根据汇总的文档 ID，再次向相关分片请求完整的文档内容 (`_source`)。

### 查询解析与执行树
Query DSL 被解析为内部的 `Query` 对象树。Lucene 会进行查询重写 (Query Rewrite)，例如将 `term` 查询优化为常量评分，或将多个 `must` 子句合并。

### 评分机制：BM25
ES 7.x 默认使用 BM25 算法替代 TF-IDF。BM25 引入了文档长度归一化和词频饱和机制，避免了长文档或高频词带来的评分偏差。公式核心参数：
- `k1`：控制词频饱和度（默认 1.2）。
- `b`：控制文档长度归一化程度（默认 0.75）。

### 过滤器缓存 (Filter Cache)
在 `bool` 查询的 `filter` 上下文中执行的查询不参与评分，且结果会被缓存到 Query Cache 中。对于枚举值、时间范围、状态码等过滤条件，**必须使用 filter**。

### 深度分页问题
`from + size` 超过 10000 时会被拒绝。因为每个分片需要在内存中排序 `from + size` 条数据，协调节点需汇总 `shards * (from + size)` 条数据。
- **Scroll API**：适用于全量导出，不适用于实时搜索。
- **search_after**：适用于实时深度分页，利用上一页最后一条记录的排序值作为游标，避免全局排序。

### 聚合执行原理
聚合在 Query Phase 执行。每个分片在本地构建聚合树（如 Hash 表、树形结构），返回局部结果。协调节点合并局部结果，生成全局结果。对于高基数聚合，可能触发全局序数 (Global Ordinals) 构建，耗时较长。

## 5. 并发与一致性

### 版本控制：乐观并发控制 (OCC)
ES 不支持传统数据库的事务和锁，而是通过版本号实现 OCC。
- **internal**：默认版本控制，每次更新版本号 +1。可通过 `?version=1` 防止并发覆盖。
- **external / external_gte**：使用外部系统的版本号，适用于数据同步场景。

### 序列号与生成号
- **Sequence Number**：每个文档操作在分片上的全局递增序列号，用于追踪操作顺序。
- **Primary Term**：主分片选举的代数。每次主分片变更，Primary Term 递增。结合 Sequence Number 可唯一确定一个操作。

### 写操作的并发控制
在主分片上，写操作是**串行化**的。Lucene 的 `IndexWriter` 内部有锁，保证同一时刻只有一个线程在写入段文件。副本分片的写入也是串行的，但多个副本之间是并行的。

### 读操作的并发
读操作可以在主分片或副本分片上执行（通过 `?preference` 或 `?routing` 控制）。副本分片可以分散读压力，但可能读到未同步的最新数据（最终一致性）。

### 一致性级别
- **Quorum 机制**：写操作需要 `quorum` 个分片（主 + 副本）确认成功才算成功。公式：`(primary + replicas) / 2 + 1`。
- **wait_for_active_shards**：可配置写入前需等待多少个分片处于活跃状态，防止脑裂时数据丢失。

### 批量操作的原子性
`_bulk` API **不是原子操作**。部分请求失败不影响其他请求。客户端必须检查响应中的 `errors` 字段和每个 item 的 `status`，对失败项进行重试。

### 线程池模型
ES 内部维护多个线程池：
- **write**：处理索引、更新、删除。
- **search**：处理查询和聚合。
- **management**：处理集群状态变更。
当队列满时，触发**拒绝策略**（Reject），客户端收到 `EsRejectedExecutionException`。此时应实施**背压机制**，降低写入速率。

## 6. 分布式与高可用

### 节点角色
- **Master**：管理集群元数据、分片分配、节点加入/离开。建议 3 个专用节点。
- **Data**：存储数据，执行增删改查。
- **Coordinating**：接收请求，分发查询，合并结果。所有节点默认都是协调节点。
- **Ingest**：预处理管道，写入前对数据进行转换。

### 分片分配策略
- **主分片**：创建时固定，不可更改。
- **副本分片**：不会分配到与主分片相同的节点，保证节点故障时数据不丢失。
- **Shard Allocation Awareness**：感知机架、可用区，将副本分散到不同物理位置。

### 路由机制
默认路由公式：`shard = hash(routing) % number_of_primary_shards`。
- **自定义路由**：通过 `?routing=user_123`，将同一用户的数据路由到同一分片，避免跨分片查询。
- **注意**：修改主分片数或路由键会导致数据重新分布。

### 集群状态管理
Master 节点通过 **Zen Discovery** (7.x+) 或 **Voting Configuration** (8.x+) 进行选举。采用类 Raft 协议，保证多数派同意。集群状态（索引列表、Mapping、分片位置）仅在 Master 节点更新，并增量同步到所有节点。

### 故障转移与恢复
- **故障检测**：节点间通过 Ping 机制检测。Master 发现节点失联后，将分片标记为 `UNASSIGNED`。
- **重新分配**：Master 重新计算分片分配，提升副本为主分片，并在新节点上创建副本。
- **延迟分配**：`index.unassigned.node_left.delayed_timeout` (默认 1m)，避免短暂网络抖动导致大量数据迁移。

### 跨集群复制 (CCR)
允许将一个集群的索引实时复制到另一个集群，用于异地多活、读写分离或合规备份。

### 动态扩缩容
新增节点后，Master 自动触发 **Shard Rebalance**，将分片从负载高的节点迁移到新节点，平衡磁盘和 CPU 使用率。可通过 `cluster.routing.rebalance.enable` 控制。

## 7. 持久化与恢复

### 写入持久化链路
`Index Request` → `Translog (fsync)` → `In-Memory Buffer` → `Refresh (OS Cache)` → `Flush (Disk)`。
只有 Translog 的 `fsync` 是强持久化保证。

### Refresh 与 Flush 的区别
- **Refresh**：轻量级，将内存数据转为可搜索的段（在 OS Cache），默认 1s。
- **Flush**：重量级，将 OS Cache 中的段 `fsync` 到磁盘，清空 Translog，触发 Lucene Commit。

### Checkpoint 机制
Translog 文件头部记录了 `Checkpoint`，表示已安全持久化到段文件的操作位置。恢复时只需重放 Checkpoint 之后的操作。

### 崩溃恢复流程
1. 节点重启，加载本地 Lucene 段文件。
2. 读取 Translog，校验 Checkpoint。
3. 重放 Checkpoint 之后的 Translog 操作，重建内存状态。
4. 向 Master 注册，等待分片分配。

### Translog 刷新策略
- **request**：每次写入都 `fsync` Translog（默认，最安全，性能较低）。
- **async**：每 5s 或 512MB `fsync` 一次（高性能，可能丢失 5s 数据）。

### 快照与恢复 (Snapshot & Restore)
基于共享仓库（S3, HDFS, NFS）的增量备份。
- **快照**：仅备份未备份的段文件，不阻塞读写。
- **恢复**：优先从本地恢复，不足部分从仓库拉取。

### 索引生命周期管理 (ILM)
自动化管理数据生命周期：
- **Hot**：高频写入，使用 SSD。
- **Warm**：只读，使用 HDD，减少副本。
- **Cold**：低频访问，冻结索引，使用廉价存储。
- **Delete**：过期删除。

## 8. 性能与场景

### 优势场景
- **全文检索**：电商搜索、站内搜索。
- **日志分析**：ELK Stack，海量日志的实时聚合。
- **指标监控**：替代部分 Prometheus 场景，支持高基数标签。
- **安全分析 (SIEM)**：海量安全事件的关联分析。

### 性能调优方向
- **写入调优**：使用 `_bulk` API（5MB-15MB/批）；写入时关闭副本和 Refresh，完成后开启；使用自动生成 ID（跳过版本查找）；调整 `index.translog.durability=async`。
- **查询调优**：尽量使用 `filter`；避免深度分页；使用 `search_after`；只返回必要字段 (`_source` filtering)；利用 Routing 减少参与分片数。
- **集群调优**：合理设置分片大小（10GB-50GB）；预留 50% 内存给 OS Cache；分离 Master 和 Data 节点。

### 不适合的场景
- **复杂事务**：不支持 ACID，不支持跨文档事务。
- **强一致性读**：默认最终一致性，实时读可能读到旧数据。
- **关系型关联**：不支持 JOIN，反规范化设计导致写入放大。
- **高频点查**：作为主数据库，KV 查询性能不如 Redis/DynamoDB。

### 选型指南
- **MySQL + ES**：MySQL 作为主库保证事务，ES 作为从库提供搜索和分析。通过 CDC (Canal/Debezium) 同步数据。
- **纯 ES**：日志、监控、内容管理等无事务要求的场景。

### 常见性能瓶颈与排查
- **写入拒绝**：检查 `write` 线程池队列，降低批量大小或增加节点。
- **查询超时**：检查慢查询日志，优化 DSL，检查 Fielddata 内存占用。
- **GC 频繁**：检查堆内存使用，减少缓存大小，或升级硬件。
- **分片不均衡**：检查 Routing 是否倾斜，手动调整 Shard Allocation。

### 生态协同
- **Kibana**：可视化、Dashboard、DevTools、安全管理。
- **Logstash**：ETL 管道，支持丰富的 Input/Filter/Output 插件。
- **Beats**：轻量级数据采集器（Filebeat, Metricbeat）。
- **Elastic Agent**：统一代理，简化部署和管理。
