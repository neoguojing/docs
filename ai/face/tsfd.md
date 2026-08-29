# TSFD（时空特征数据库）源码深度解析与 DDIA 理论对照整理

## 一、 系统整体架构与组件分工

TSFD 是一个针对海量人脸/人体特征向量以及抓拍时空元数据设计的高并发、低延迟分布式检索与存储系统。其架构将计算与存储分离、在线近实时检索与离线聚类分析解耦，并采用了典型的异构存储与分层索引架构。

```text
                  ┌────────────────────────────────────────────────────────┐
                  │                      IPS / VPS                         │
                  └───────────────────────────┬────────────────────────────┘
                                              │ OBJECT_INFO
                                              ▼
┌──────────────────────────────────────────────────────────────────────────────────────────┐
│ Ingestion Layer                                                                          │
│                                       ┌──────────┐                                       │
│                                       │  Kafka   │ (Feature Buffer / Oplog)              │
│                                       └────┬─────┘                                       │
└────────────────────────────────────────────┼─────────────────────────────────────────────┘
                                             │
┌────────────────────────────────────────────▼─────────────────────────────────────────────┐
│ Routing & Batching Layer                                                                 │
│                                   ┌────────────────┐                                     │
│                                   │  Coordinator   │ (Region Routing, Shard Assignment)  │
│                                   └────────┬───────┘                                     │
└────────────────────────────────────────────┼─────────────────────────────────────────────┘
                                             │ gRPC (BatchAddFeature)
┌────────────────────────────────────────────▼─────────────────────────────────────────────┐
│ Compute & Vector Search Layer                                                            │
│                                 ┌───────────────────┐                                    │
│                                 │   Worker Cluster  │ (GPU/CPU Faiss IVF ANN Index)      │
│                                 └──────────┬────────┘                                    │
└────────────────────────────────────────────┼─────────────────────────────────────────────┘
                                             │
┌────────────────────────────────────────────▼─────────────────────────────────────────────┐
│ Persistence & Storage Layer                                                              │
│      ┌─────────────────────────┐                        ┌───────────────────────────┐    │
│      │   Cassandra DB Cluster  │ (Raw Features/Metadata)│   MinIO Object Storage    │    │
│      └─────────────────────────┘                        └───────────────────────────┘    │
└──────────────────────────────────────────────────────────────────────────────────────────┘
```

### 1. 核心组件职责
*   **Coordinator（数据接入与路由节点）**：消费 Kafka 中的流式抓拍数据，按 `region_id` 进行空间路由与分片重排，批量转发给对应的 Worker 节点，支持多 Coordinator 并行消费与动态负载均衡。
*   **Worker（存储与向量计算节点）**：管理 GPU 显存与 CPU 内存中的 Faiss IVF 近似最近邻（ANN）向量索引，负责特征增删、局部分片（Shard）生命周期管理、GPU/CPU 粗搜，并将原始特征与分片索引持久化。
*   **Reducer（网关与聚合节点）**：提供外部查询 API（ Search / Cluster Search 等），利用 Leader 选举机制执行定时清理与索引加载，在查询时向所有 Worker 广播请求（Scatter-Gather），并完成精确重排（Exact Search）与分值归一化。
*   **Kafka（消息中间件 / 预写日志 WAL）**：作为特征流量缓冲池与操作日志（OpLog）的传输通道，实现系统间解耦与异步写入。
*   **Cassandra（分布式列式持久化存储）**：提供持久化的 LSM-Tree 存储，保存 Raw Feature、索引元数据、Clusters 和 Metadata。
*   **MinIO（对象存储）**：存储向量索引快照（Snapshot）和离线聚类文件，用于冷启动加载和灾备。

### 2. DDIA 理论对照：异构分层存储 (Heterogeneous Storage Hierarchy)
*   **存储与计算分离 (Decoupled Compute & Storage)**：TSFD 将高维向量索引置于 GPU 显存/内存（用于高速 ANN 近似计算），而将原始特征与属性持久化于 Cassandra（用于精确回查与重排序）。这符合 DDIA 第 3 章中关于 **内存数据库（In-Memory DB）与磁盘 LSM-Tree 结合** 的设计思路，用 GPU 内存换取极低延迟的向量匹配，用 Cassandra LSM-Tree 换取高吞吐写入能力。

---

## 二、 数据分片、编码与 Cassandra 表结构设计

### 1. ID 编码机制（Snowflake 变体）
TSFD 自定义了包含时空上下文信息的 64-bit（8字节）整数 ID 结构：

`feature_id` / `object_id` 编码格式：
```text
ID = | 0 (1b) | region_id (14b) | camera_idx (7b) | timestamp_in_sec (32b) | subsequence_id (10b) |
```
**设计目的**：将空间属性（`region_id`, `camera_idx`）和时间属性（`timestamp`）直接嵌入 64 位整型中。

### 2. 二维混合分片策略 (Spatial + Temporal Hybrid Partitioning)
TSFD 采用了空间（Region）+ 时间（Time Window）的复合分片模式：
*   **空间分片**：`region_id` 映射到 Kafka Partition 及具体的 Worker 节点。
*   **时间分片**：同 Region 下的连续时间数据聚集为一个 Shard 分片（定义起始时间 `first_time` 和结束时间 `last_time`）。
*   **关闭分片规则**（满足其一即可）：
    1. 时间跨度达到 1 周。
    2. 分片内特征数达到 200 万。
    3. 特征数大于 26 万，且持续 `Unmodify Days` 天未写入新数据。

### 3. Cassandra 数据表建模
```sql
-- 1. indexes 表：记录分片元信息
CREATE TABLE face_feature_version.indexes (
    region_id int,
    index_id uuid,
    first_time timestamp,
    last_time timestamp,
    update_time timestamp,
    shard_size int,
    status int,            -- 0: open (可写), 1: close (只读/待训练)
    worker_id text,
    PRIMARY KEY (region_id, index_id)
);

-- 2. features 表：存储原始特征与抓拍元数据
CREATE TABLE face_feature_version.features (
    region_id int,
    captured_date int,
    captured_time timestamp,
    camera_idx int,
    sequence int,
    annotation blob,
    cluster_id bigint,
    extra_info text,
    feature blob,          -- 原始高维特征向量
    object_id text,
    index_id uuid,
    PRIMARY KEY ((region_id, captured_date), captured_time, camera_idx, sequence)
);

-- 3. clusters 表：存储聚类中心与关联特征
CREATE TABLE face_feature_version.clusters (
    cluster_id bigint PRIMARY KEY,
    centroid blob,
    feature_ids list<bigint>,
    created_at timestamp,
    modified_at timestamp,
    user_key text,
    extra_info text
);
```

### 4. DDIA 理论对照：分区与二级索引 (Partitioning & Secondary Indexes)
*   **主键与聚集列选择（DDIA 第 6 章）**：在 `features` 表中，以 `(region_id, captured_date)` 作为 Partition Key，将单个区域一天内的数据聚簇在同一个存储节点；以 `(captured_time, camera_idx, sequence)` 作为 Clustering Key，在磁盘物理排序上按抓拍时间严格递增排列。这使得范围查询变为连续磁盘 I/O 读，极大提升检索效率。
*   **文档分区索引 (Document-Partitioned / Local Index)**：Worker 节点上的 Faiss IVF 索引属于本地二级索引（Local Index）。每个 Shard 仅对其管理的本地特征向量构建 IVF 索引。查询时必须采用 **Scatter-Gather（分散-聚集）** 模式，向所有 Shard 发起并行查询，然后在 Reducer 侧进行全局 Top-K 合并。

---

## 三、 核心业务流程与分布式算法

### 1. 数据写入流（Ingestion Pipeline）
```text
[Kafka Target Topic]
         │
         ▼
[Coordinator: ConsumerImpl]
         │
         ▼
[DbRouter -> Batcher] (按 RegionId 分散至 1024 个队列，攒 Batch)
         │
         ▼
[ShardAssignment] (寻找或创建 ShardSlave / RemoteShard)
         │
         ▼ gRPC: BatchAddFeature
[Worker: ShardController]
         ├──> 1. 写入本地 Faiss 内存索引 (粗搜准备)
         ├──> 2. 异步/同步写入 Cassandra `features` 表 (持久化)
         └──> 3. (可选) 投递 OpLog 到 Kafka (同步给 OLAP 聚类集群)
```
*   **Sequence ID 生成算法**：Coordinator 通过 `nsSubsequences` 维系摄像头级递增序列，算法保证在同一毫秒内的并发抓拍通过 NodeID + Offset 分散，防止 ID 冲突。
*   **批处理与缓冲（Batching）**：Coordinator 维护 Batcher 队列，按大小（如 16/32）或超时阈值将单个数据拼接为大 Batch，降低 RPC 频次与数据库 Write Amplification（写入放大）。

### 2. 分布式检索 path：两阶段搜索 (Two-Phase Search / Rescoring Pattern)
TSFD 针对高维向量搜索的精度与速度矛盾，设计了粗搜（Coarse Search）与精搜（Exact Search/Re-ranking）两阶段架构：

```text
Client Query ──> [Reducer: Search]
                     │
                     ├── Phase 1: 粗搜 (Coarse Search / Broad Search)
                     │    ├── 广播 gRPC 到所有 Worker 节点
                     │    ├── Worker 调用 GPU Faiss IVF 索引执行 ANN 近似距离计算
                     │    └── 各 Shard 返回 Top-K 候选集 (ObjID + Cosine/IP Distance)
                     │
                     ├── Phase 2: 精搜 (Exact Re-ranking / Fine Search)
                     │    ├── Reducer 根据候选 ObjID，批量并发读取 Cassandra `features` 表
                     │    ├── 计算 100% 准确的点积/余弦相似度 (Inner Product)
                     │    └── 分段分值归一化 (Piecewise Score Normalization)
                     │
                     └──> 最终全局 Top-K 排序并返回 Client
```
**分值归一化公式（Piecewise Linear Mapping）**
为了消除不同模型或特征版本下的对比偏差，Reducer 采用基于分段点（`SrcPoints` -> `DstPoints`）的线性插值归一化算法：
$$ \text{NormScore} = \frac{\text{score} - \text{SrcPoints}[i-1]}{\text{SrcPoints}[i] - \text{SrcPoints}[i-1]} \times (\text{DstPoints}[i] - \text{DstPoints}[i-1]) + \text{DstPoints}[i-1] $$

### 3. DDIA 理论对照：两阶段读取与近似算法 (Approximate Algorithms & Two-Phase Reads)
*   **ANN 近似算法与准确率平衡（DDIA 第 1 章 & 3 章）**：在高维空间（如 256/512 维向量）中，精确 K 近邻搜索（KNN）需要全量扫描，无法满足毫秒级响应。TSFD 在阶段一采用 Faiss IVF（倒排文件）进行空间划分与向量量化（ANN），牺牲少许 Recall 换取数量级的耗时下降；在阶段二通过 Cassandra 提取 Raw Feature 进行重新计算（Re-ranking），弥补精度损失。
*   **Scatter-Gather 放大效应**：查询延迟取决于最慢的节点（Tail Latency / 长尾延迟问题，DDIA 第 1 章）。TSFD 通过在 Worker 端将多 Shard 检索并行化及 GPU 批处理计算来缓解此问题。

---

## 四、 高可用、选主与分片负载均衡

### 1. Reducer 选主机制 (Leader Election)
*   **机制**：通过 `leaderelect.LeaderElector` 借助 Cassandra 的 LWT（Lightweight Transactions，基于 Paxos 协议）或心跳锁表实现 Leader 选举。只有 `isLeader == true` 的 Reducer 节点才会启动 Reloader 和 DBDelScan 事件循环。

### 2. Coordinator 动态重平衡 (Rebalancing & Shard Assignment)
*   **策略选择 (WorkerPickStrategy)**：支持 RoundRobin（轮询）、LoadScore（基于内存/GPU负载得分）、LoadScorePeriod 及 Random。
*   **Rebalance 循环**：Crontab 定期触发 `DbRouter.loadBalancer`。当检测到 Worker 节点写入量过大或分片大小超过 `RebalanceMinShardSize` 时，重新分派 `region_id -> worker_id` 的映射，防止单节点热点。

### 3. Worker 分片生命周期 (Shard Lifecycle)
```text
[新建 Shard] (Status: Open) ──> [持续写入数据/更新边界]
                                     │
                                     ▼ (符合关闭条件: Size>200万 或 1周 或 无新数据)
                             [关闭 Shard] (Status: Close / Writing=true)
                                     │
                                     ▼ (DumpShards 触发)
                             [Dump 索引至磁盘 / MinIO]
                                     │
                                     ▼ (Trainer 触发)
                             [训练 Faiss IVF 索引] (样本数 > MinReClusterSize)
                                     │
                                     ▼ (超过 ShardExpiredDays)
                             [清理释放] (删除 GPU/内存索引，清理 Cassandra 记录)
```

### 4. DDIA 理论对照：分布式共识与动态再平衡 (Consensus & Dynamic Rebalancing)
*   **Leader 选举与共识（DDIA 第 9 章）**：Reducer 利用 Cassandra 实现分布式锁/共识，避免多个节点同时执行清理或加载任务，保证了分布式任务执行的单点控制与幂等性。
*   **分区重平衡策略（DDIA 第 6 章）**：TSFD 避免了简单的 `hash(key) % N` 重新分区，而是采用了类似**动态分区（Dynamic Partitioning）**的模式。以 Shard 为基本迁移单元，按 `region_id` 动态调整分片到 Worker 的指派关系（ShardAssignment），实现了低成本的负载均衡。

---

## 五、 OPOD 兼容方案与异步解耦 (OpLog Architecture)

TSFD 引入了 OpLog（操作日志）机制，用于实现在线时空库（TSFD Default Mode）与离线聚类/分析库（TSFD OLAP Mode）之间的异步数据同步。

```text
                        ┌───────────────────────────────┐
                        │      IPS/VPS (抓拍数据源)      │
                        └───────────────┬───────────────┘
                                        │
                                        ▼
                        ┌───────────────────────────────┐
                        │  tsfd 默认模式 (Online OLTP)   │
                        └───────┬───────────────┬───────┘
                                │               │
                  特征写入       │               │ 投递 Feature OpLog
                                ▼               ▼
                        ┌──────────────┐ ┌──────────────┐
                        │  Cassandra   │ │    Kafka     │ (oplog-feature)
                        └──────────────┘ └──────┬───────┘
                                                │
                                                ▼
                                 ┌─────────────────────────────┐
                                 │  OPOD / pyutil (聚类引擎)   │
                                 └──────────────┬──────────────┘
                                                │
                                                │ 投递 Cluster OpLog
                                                ▼
                        ┌──────────────┐ ┌──────────────┐
                        │  tsfd-olap   │ │    Kafka     │ (oplog-cluster)
                        │  (聚类模式)  │◄┴──────────────┘
                        └──────┬───────┘
                               │
                               ▼
                        [查询聚类信息 (Actor)]
```

### 1. OpLog 分类
*   **Feature 相关 OpLog**：`OpAddFeature`, `OpDeleteFeature`, `OpAddIndex`。由在线 TSFD 产生，推送到 Kafka Topic `oplog-feature`。
*   **Cluster 相关 OpLog**：`OpAddCluster`, `OpUpdateCluster`, `OpMergeCluster`, `OpDeleteCluster` 等。由离线聚类服务（OPOD）计算完成后产生，发送至 `oplog-cluster`，最终由 `tsfd-olap` 消费并写入 Cassandra。

### 2. DDIA 理论对照：变更数据捕获与 Lambda/Kappa 架构 (CDC & Stream Decoupling)
*   **变更数据捕获 (Change Data Capture, CDC，DDIA 第 11 章)**：OpLog 本质上是系统的 WAL / Change Feed。在线系统（OLTP）将数据变更以流的形式发布，离线聚类引擎（OLAP）作为订阅者消费变更。避免了在同一个数据库上的资源竞争，实现了 OLTP 与 OLAP 系统的彻底解耦。
*   **最终一致性 (Eventual Consistency，DDIA 第 5 章)**：聚类结果的更新通过异步日志广播回写，在线库与聚类库之间存在短暂的复制延迟（Replication Lag），系统在设计上接受这种最终一致性模型。

---

## 六、 架构总结与对比对照表

| 维度 | TSFD 实现机制 | DDIA 对应概念 / 理论原则 |
| :--- | :--- | :--- |
| **存储引擎** | GPU 显存 (Faiss IVF) + Local Disk (idx) + Cassandra (LSM) | 异构分层存储、内存数据库与 LSM-Tree 持久化结合 |
| **数据分区** | Region 空间 Hash + Time Window 动态时间分片 | 基于范围与 Hash 的混合分区 (Hybrid Partitioning) |
| **索引架构** | 每 Shard 独立构建 Faiss 索引，搜索时广播 | 本地二级索引 (Local Index / Document-partitioned) |
| **读路径优化** | 两阶段检索：GPU ANN 粗搜 -> Raw Vector 精确重排与归一化 | 近似算法 (ANN) 与两阶段重排序 (Rescoring) |
| **组件通信** | Kafka 流式缓冲 + OpLog 操作日志解耦 | 变更数据捕获 (CDC) 与 Event-driven Stream Processing |
| **高可用共识** | Reducer 通过 Cassandra LWT/锁进行 Leader 选举 | 分布式共识 (Consensus) 与单主节点任务调度 |
| **负载均衡** | Coordinator 根据 Worker LoadScore 动态重指派 Region | 动态分区重平衡 (Dynamic Partition Rebalancing) |

### 总结
TSFD 源码展现了一个典型的针对特定领域（高维向量 + 海量抓拍时空元数据）深度优化的分布式系统。它并没有盲目依赖单一的通用数据库，而是将 Kafka 的流式吞吐、Cassandra 的高并发 LSM 写入、GPU 的高并行向量计算以及 Faiss 的近似检索算法 有机结合。在设计上严格遵循了 DDIA 中关于分区、复制、异步数据流处理以及两阶段读取的核心设计范式。
