# Elasticsearch 技术文档（精简重构版）

> 学习主线：**是什么 → 为什么 → 怎么用 → 底层怎么实现 → 分布式与一致性 → 性能与场景**
>
> 核心记忆：**ES = 分布式搜索/分析引擎；Shard → Lucene Index → Segment；Term → DocID → 文档；Doc Values → 排序/聚合；Translog → 恢复；Refresh → 可搜索；Flush → 持久化；Merge → 回收空间。**

---

## 1. 定位与数据模型

### 1.1 Elasticsearch 是什么？为什么用？

Elasticsearch（ES）是基于 Apache Lucene 的**分布式搜索与数据分析引擎**，提供近实时全文检索、结构化查询、日志分析、指标分析等能力。

| 能力 | 解决什么问题 |
|---|---|
| 全文检索 | 分词、模糊查询、同义词等 |
| 结构化查询 | 精确匹配、范围过滤、排序 |
| 聚合分析 | 按字段统计、分组、指标计算 |
| 日志/事件分析 | 海量数据写入与多维查询 |
| 分布式 | 分片、复制、故障转移 |

**为什么不用 MySQL 直接搜索？**

ES 的核心优势是倒排索引、分布式分片和聚合能力，适合搜索与分析；复杂事务、强一致主数据仍更适合关系型数据库。

典型架构：

```text
MySQL / PostgreSQL
       │
       │ CDC / MQ
       ▼
Elasticsearch
       │
       ├── 搜索
       └── 分析

日志 → Logstash / Beats / Agent → ES → Kibana
```

---

### 1.2 核心数据模型

ES 是面向文档的 NoSQL 数据模型：

| ES | 类比 RDBMS | 含义 |
|---|---|---|
| Index | Table（更接近业务数据集合） | 文档集合 |
| Document | Row | 一条 JSON 文档 |
| Field | Column | 文档字段 |
| Mapping | Schema | 字段类型及索引方式 |
| Query DSL | SQL | 查询语言 |
| Shard | — | Index 的物理分片 |

`Type` 在 7.x 起被废弃，8.x 已移除。

---

### 1.3 Mapping：怎么定义字段？

Mapping 决定字段如何被索引和存储。

```json
PUT products
{
  "mappings": {
    "properties": {
      "product": {"type": "text"},
      "product.keyword": {"type": "keyword"},
      "price": {"type": "integer"},
      "create_time": {"type": "date"}
    }
  }
}
```

常用类型：

| 类型 | 用途 |
|---|---|
| `text` | 全文搜索，需要分词 |
| `keyword` | 精确匹配、排序、聚合 |
| `date` | 时间 |
| `nested` | 保持对象数组内部字段关联 |
| 数值类型 | 范围查询、排序、聚合 |

**为什么生产环境建议显式 Mapping？**

避免动态类型推断导致类型冲突和 Mapping 膨胀。

---

## 2. 存储引擎：数据到底怎么落盘？

### 2.1 核心层次

```text
Elasticsearch Index
└── Shard
    └── Lucene Index
        └── Segment
            ├── Term Index / FST
            ├── Term Dictionary
            ├── Postings
            ├── Stored Fields
            └── Doc Values
```

ES 本身不负责底层索引存储，核心由 Lucene 完成。

**Segment 是 Lucene 的基本索引单元：**

- 写入后基本不可修改
- 可独立参与查询
- 更新 = 新文档 + 旧文档删除标记
- 后台 Merge 合并 Segment，并最终清理删除数据

---

### 2.2 用同一组数据理解全部结构

统一使用：

```text
Doc1 = {"product":"MacBook Pro", "price":15000}
Doc2 = {"product":"MacBook Air", "price":8000}
Doc3 = {"product":"iPad Pro",    "price":6000}
```

分词：

```text
Doc1 → macbook, pro
Doc2 → macbook, air
Doc3 → ipad, pro
```

得到：

```text
Term Dictionary
air
ipad
macbook
pro

Postings
air      → [Doc2]
ipad     → [Doc3]
macbook  → [Doc1, Doc2]
pro      → [Doc1, Doc3]
```

---

### 2.3 Term Index / FST：怎么快速找到词？

**Term Dictionary** 保存 Segment 中有序的 Term。

**Term Index / FST** 是 Term Dictionary 的索引，用较小的内存快速定位 Term 所在范围。

```text
查询 "macbook"
      ↓
 FST / .tip
      ↓
定位 .tim 的 Block
      ↓
Term Dictionary
      ↓
macbook
```

**为什么需要 FST？**

Term Dictionary 可能非常大，不能完整放入内存；FST 通过共享路径压缩索引，减少内存并加速定位。

---

### 2.4 Postings：Term 怎么找到文档？

找到 `macbook` 后：

```text
macbook
   ↓
Postings
   ↓
[Doc1, Doc2]
```

Postings 不只保存 DocID，还可以保存：

- Term Frequency
- Position
- Offset / Payload（取决于索引能力）

DocID 通常采用 Delta + Block/Packed 等方式压缩：

```text
[100,103,104,109,110]
      ↓ Delta
[100,3,1,5,1]
      ↓
压缩存储
```

**一句话：**

> Postings 解决 **Term → DocID**。

---

### 2.5 Stored Fields：DocID 怎么取回文档？

搜索得到：

```text
macbook → [Doc1, Doc2]
```

如果需要返回 `_source`：

```text
DocID
 ↓
.fdx：定位
 ↓
.fdt：读取 Stored Fields
 ↓
返回文档字段
```

**一句话：**

> Stored Fields 解决 **DocID → 文档字段**。

它适合 Fetch 阶段，而不是排序/聚合。

---

### 2.6 Doc Values：为什么排序/聚合不读 `_source`？

例如：

```text
AVG(price)
```

如果逐个读取 JSON：

```text
Doc1 → JSON → price
Doc2 → JSON → price
Doc3 → JSON → price
```

效率较低。

Doc Values 按列组织：

```text
price
────────────
Doc1 → 15000
Doc2 →  8000
Doc3 →  6000
```

因此：

```text
AVG(price)
    ↓
Doc Values
    ↓
[15000,8000,6000]
    ↓
9666.67
```

**一句话：**

> Stored Fields 是 **按文档取**；Doc Values 是 **按字段取**，主要用于排序、聚合和脚本。

Doc Values 主要位于磁盘并依赖 OS Page Cache，避免把大量字段数据塞入 JVM Heap。

---

### 2.7 四种核心结构必须分清

| 结构 | 典型文件 | 方向 | 解决什么问题 |
|---|---|---|---|
| Term Index / FST | `.tip` | Term → Dictionary Block | 快速找词 |
| Term Dictionary | `.tim` | — | 保存 Term |
| Postings | `.doc/.pos` | Term → DocID | 找匹配文档 |
| Stored Fields | `.fdx/.fdt` | DocID → 字段 | Fetch 原文 |
| Doc Values | `.dvm/.dvd` | Field → Values | 排序/聚合 |

> 具体文件名和 Codec 会随 Lucene 版本、字段类型和配置变化；`.cfs` 还可能封装多个文件。

---

## 3. 写入、持久化与 Segment

### 3.1 写入主流程

```text
客户端写入
   ↓
Memory / Indexing Buffer
   │
   └──→ Translog
          ↓
       Refresh
          ↓
     新 Segment
          ↓
    OS Page Cache
          ↓
       Flush/fsync
          ↓
      Physical Disk
```

三个问题要分开：

| 机制 | 回答的问题 |
|---|---|
| Refresh | **什么时候可以搜索？** |
| Translog | **异常后怎么恢复？** |
| Flush/fsync | **什么时候完成持久化？** |
| Merge | **怎么减少 Segment 并回收删除数据？** |

---

### 3.2 Refresh：为什么是“近实时”？

Refresh 将内存中的索引数据生成新的 Segment，使其进入搜索视图：

```text
Memory Buffer
    ↓ Refresh
New Segment
    ↓
可搜索
```

因此：

> **Refresh ≠ fsync 到物理盘。**

默认情况下通常约每秒 Refresh 一次，因此 ES 是 **Near Real-Time（NRT）**，而不是严格实时。

---

### 3.3 Translog：为什么还需要它？

Refresh 后 Segment 可搜索，但仍不能简单理解成“已经完成所有持久化”。

Translog 记录写操作，用于节点异常后的恢复：

```text
写入
 ├── Memory Buffer
 └── Translog
        ↓
      异常
        ↓
  重放 Translog
        ↓
      恢复
```

---

### 3.4 Flush / Commit：解决什么问题？

```text
Segment / Page Cache
        ↓
      fsync
        ↓
Physical Disk
        ↓
Commit Point
```

Commit Point 中维护当前有效的 Segment 信息，例如：

```text
segments_N
    ↓
当前有效 Segment
_0
_1
_2
```

满足条件后，旧 Translog 可以清理。

---

### 3.5 Merge：为什么必须合并 Segment？

频繁 Refresh 会产生很多小 Segment：

```text
_0   _1   _2   _3
 \    |    |   /
      Merge
        ↓
       _4
```

作用：

1. 减少 Segment 数量
2. 提升查询效率
3. 合并索引文件
4. 清理 Tombstone
5. 回收磁盘空间

---

## 4. 内存与底层数据结构

### 4.1 JVM Heap vs OS Page Cache

ES 的性能不能只看 JVM Heap：

| 区域 | 主要内容 | 为什么 |
|---|---|---|
| JVM Heap | 集群元数据、缓存、Indexing Buffer 等 | ES/JVM 管理 |
| OS Page Cache | Lucene Segment 文件 | 大量磁盘读取依赖系统缓存 |

核心思想：

> **Heap 负责 ES/JVM 管理的数据；OS Page Cache 负责大量 Lucene 文件的高速访问。**

Heap 通常不应超过物理内存的 50%；同时需要考虑 JVM Compressed Oops 等限制。实际生产配置应结合 ES/JDK 版本与官方建议。

---

### 4.2 常见缓存

| 缓存 | 作用 | 重点 |
|---|---|---|
| Node Query Cache | Filter 查询结果 | 不参与评分 |
| Shard Request Cache | 分片级请求/聚合结果 | Segment 无更新时更有效 |
| Fielddata Cache | `text` 字段排序/聚合 | 容易大量占用 Heap |

**为什么避免对 `text` 做聚合？**

`text` 主要面向全文搜索；强制使用 Fielddata 会将数据加载到 Heap，容易造成 GC/OOM。需要聚合时通常使用 `keyword`。

---

### 4.3 关键数据结构

| 结构 | 作用 | 为什么 |
|---|---|---|
| FST | Term → Term Dictionary 定位 | 节省内存、快速查词 |
| Skip List | 加速倒排表交集 | 减少无效 DocID 比较 |
| Delta + FOR/Packed | 压缩 DocID | 减少磁盘/I/O |
| BKD Tree | 数值、日期、Geo 查询 | 高效范围检索 |
| Doc Values | 字段列式访问 | 高效排序/聚合 |

---

## 5. 索引与查询：怎么用？

### 5.1 创建索引

```http
PUT /products
{
  "settings": {
    "number_of_shards": 3,
    "number_of_replicas": 1
  },
  "mappings": {
    "properties": {
      "product": {"type": "text"},
      "price": {"type": "integer"}
    }
  }
}
```

创建时核心动作：

```text
Mapping 校验
    ↓
确定 Primary Shard / Replica
    ↓
初始化 Lucene Index
```

---

### 5.2 写入文档

```http
POST /products/_doc/1
{
  "product": "MacBook Pro",
  "price": 15000
}
```

---

### 5.3 查询：Query → Fetch

```text
客户端
  ↓
协调节点
  ↓
Query Phase
  ↓
各 Shard 查询
  ↓
DocID + Score / Sort Value
  ↓
协调节点汇总
  ↓
Fetch Phase
  ↓
读取 _source
  ↓
返回结果
```

**为什么分 Query / Fetch？**

Query 阶段只需要找到候选文档及排序信息，不必立即传输完整文档，可以减少网络和数据传输。

---

### 5.4 全文搜索：倒排索引怎么工作？

```text
"MacBook Pro"
      ↓
Analyzer
      ↓
macbook / pro
      ↓
FST
      ↓
Term Dictionary
      ↓
Postings
      ↓
Doc1
```

倒排索引本质：

> **Term → Posting List → DocID**

避免逐文档扫描。

---

### 5.5 BM25：为什么能排序？

ES 7.x 默认使用 BM25。

它综合考虑：

- Term Frequency：词出现次数
- Inverse Document Frequency：词的稀有程度
- Document Length：文档长度

核心参数：

- `k1`：控制词频饱和
- `b`：控制文档长度归一化

因此全文搜索通常不仅回答“匹不匹配”，还会回答“哪个更相关”。

---

### 5.6 Filter：为什么应该使用 filter？

例如：

```json
{
  "query": {
    "bool": {
      "must": {
        "match": {"product": "macbook"}
      },
      "filter": [
        {"term": {"status": "online"}},
        {"range": {"price": {"lt": 10000}}}
      ]
    }
  }
}
```

区别：

| Query | Filter |
|---|---|
| 关注相关性 | 关注是否匹配 |
| 参与评分 | 不参与评分 |
| 适合全文搜索 | 适合状态、时间、枚举值等条件 |

---

### 5.7 深度分页：为什么不能一直 `from + size`？

假设：

```text
from = 10000
size = 20
```

每个 Shard 都需要准备前 `10020` 个结果，再由协调节点汇总。

分片越多，内存和网络成本越高。

常见方案：

| API | 用途 |
|---|---|
| `from + size` | 普通浅分页 |
| `search_after` | 实时深分页 |
| `scroll` | 全量导出/批处理 |

---

### 5.8 聚合：怎么统计？

例如：

```text
按 product.keyword 分组
        ↓
各 Shard 本地聚合
        ↓
返回局部结果
        ↓
协调节点合并
        ↓
全局结果
```

高基数字段可能涉及 Global Ordinals，构建本身可能产生额外开销。

---

## 6. `_bulk`：怎么批量写？

### 6.1 为什么使用 Bulk？

单条写入：

```text
1000 条数据
→ 1000 次 HTTP 请求
```

Bulk：

```text
1000 条数据
→ 1 批请求
→ ES 按 Shard 分发
```

主要减少：

- 网络请求开销
- HTTP 解析开销
- 底层频繁写入开销

---

### 6.2 NDJSON 格式

Bulk 不是 JSON 数组，而是 NDJSON：

```http
POST /_bulk
{"index":{"_index":"users","_id":"1"}}
{"name":"张三","age":25}
{"create":{"_index":"users","_id":"2"}}
{"name":"李四","age":30}
{"update":{"_index":"users","_id":"1"}}
{"doc":{"age":26}}
{"delete":{"_index":"users","_id":"3"}}
```

四种操作：

| 操作 | 含义 |
|---|---|
| `index` | 创建或全量替换 |
| `create` | 仅创建，存在则 409 |
| `update` | 局部更新 |
| `delete` | 删除 |

---

### 6.3 Bulk 为什么不是事务？

Bulk 中每个 item 独立执行：

```text
item1 ✓
item2 ✓
item3 ✗
item4 ✓
```

不会因为 item3 失败而整体回滚。

因此必须：

1. 检查 `errors`
2. 遍历 `items`
3. 对失败项单独处理
4. 对 429 等情况进行重试

批次应控制在合理大小，原文建议约 **5MB～15MB / 批**，实际应根据文档大小、节点资源和吞吐测试调整。

---

## 7. 并发与一致性

### 7.1 OCC：为什么不用悲观锁？

ES 主要采用**乐观并发控制（OCC）**：

```text
读取版本
   ↓
修改
   ↓
提交时检查版本
   ↓
版本一致 → 成功
版本变化 → 409 Conflict
```

现代 ES 主要使用：

```text
if_seq_no
+
if_primary_term
```

避免并发更新互相覆盖。

---

### 7.2 SeqNo + Primary Term 是什么？

| 概念 | 含义 |
|---|---|
| Sequence Number | 主分片上的写操作序号 |
| Primary Term | 主分片发生重新选举后的代数 |

```text
Primary Term = 5
SeqNo = 100
```

可以用来判断操作的新旧以及主分片代际，避免旧 Primary 恢复后继续写入。

---

### 7.3 主副本如何处理读写？

```text
写：
Client
 ↓
Primary
 ↓
Replica

读：
Client
 ↓
Primary / Replica
```

副本可以分担查询压力。

但主副同步存在延迟，因此部分读取可能暂时看到旧数据。

---

## 8. 分布式与高可用

### 8.1 节点角色

| 角色 | 主要职责 |
|---|---|
| Master | 集群元数据、节点和分片管理 |
| Data | 存储数据、执行查询和写入 |
| Coordinating | 接收请求、分发、汇总 |
| Ingest | 写入前数据预处理 |

所有节点默认都可以承担协调角色。

生产环境常见：

```text
3 Master
  +
N Data
  +
Coordinating（按需）
```

---

### 8.2 Shard：为什么需要分片？

一个 Index 可以拆成多个 Primary Shard：

```text
Index
├── Shard 0
├── Shard 1
└── Shard 2
```

每个 Shard 本质上是一个独立 Lucene Index。

好处：

- 数据水平分布
- 查询并行
- 写入并行
- 节点故障后可通过 Replica 恢复

Primary 数量在 Index 创建时确定；Replica 数量可以动态调整。

---

### 8.3 Replica：为什么需要副本？

```text
Shard 0 Primary
        ↓
Shard 0 Replica
```

副本作用：

1. 高可用
2. 故障转移
3. 分担读压力

Primary 与 Replica 不应部署在同一节点；还可以通过 Allocation Awareness 分散到不同机架/可用区。

---

### 8.4 Routing：数据怎么找到 Shard？

核心公式：

```text
shard = hash(routing) % number_of_primary_shards
```

默认：

```text
routing = _id
```

例如：

```text
hash("user_456") % 3 = 1
```

因此写入：

```text
user_456
   ↓
Shard 1
```

**为什么使用自定义 Routing？**

如果经常查询同一用户的数据：

```text
routing=user_123
```

可以让该用户的数据集中在同一 Shard，减少广播查询。

代价是可能产生数据倾斜。

如果 Primary Shard 数量或 Routing 规则发生变化，通常需要 Reindex 重新分布数据。

---

### 8.5 写入流程

```text
Client
  ↓
Coordinating Node
  ↓
hash(routing) % primary_shards
  ↓
Primary Shard
  ├── Memory Buffer
  ├── Translog
  └── Replica
        ↓
      返回成功
```

---

### 8.6 Master 故障后怎么办？

```text
Master 集群
M1  M2  M3
    ↓
M1 故障
    ↓
剩余节点重新选举
    ↓
新 Master
```

ES 7.x+ 使用 Voting Configuration 等机制，通过多数票选举 Master。

---

### 8.7 节点故障如何恢复？

```text
D1 宕机
  ↓
Master 检测到节点失联
  ↓
Shard 0(P) → UNASSIGNED
  ↓
Shard 0(R) 晋升为 Primary
  ↓
重新创建 Replica
```

核心是：

> **Replica + 自动故障转移 = 单节点故障下的高可用。**

短暂网络抖动时可以使用 Delayed Allocation，避免立即发生大量无效分片迁移。

---

### 8.8 跨机架 / 跨可用区

```text
AZ1                AZ2
Shard 0(P)         Shard 0(R)
```

通过 Allocation Awareness 让副本分散到不同物理故障域。

这样可以进一步容忍：

```text
单机故障
   ↓
机架故障
   ↓
可用区故障
```

---

### 8.9 CCR：机房级容灾

```text
Leader Cluster
      │
      │ Async Replication
      ↓
Follower Cluster
```

用途：

- 异地容灾
- 就近查询
- 读写隔离

Leader 接受写入，Follower 异步复制；主集群发生机房级故障时，可将备集群用于灾备切换。

---

### 8.10 扩容与 Rebalance

新节点加入：

```text
New Node
   ↓
Master 重新评估分配
   ↓
Shard Rebalance
   ↓
数据迁移
```

目标：

- 均衡磁盘
- 均衡 CPU
- 分散查询/写入压力

高峰期需要控制 Rebalance，避免大量迁移影响业务。

---

## 9. 持久化与恢复

把存储与高可用合在一起理解：

```text
                写入
                 ↓
        ┌────────┴────────┐
        ↓                 ↓
 Memory Buffer         Translog
        ↓                 ↓
     Refresh            恢复
        ↓
     Segment
        ↓
   Page Cache
        ↓
   Flush/fsync
        ↓
   Physical Disk

        +
     Replica
        ↓
   节点故障恢复
```

所以 ES 的可靠性来自两层：

| 层次 | 机制 | 解决问题 |
|---|---|---|
| 节点内 | Translog + 持久化 | 节点重启/崩溃恢复 |
| 节点间 | Replica | 节点故障 |
| 故障域 | Allocation Awareness | 机架/AZ 故障 |
| 集群间 | CCR | 异地灾备 |

---

## 10. 性能与场景

### 10.1 适合什么？

| 场景 | 原因 |
|---|---|
| 电商/站内搜索 | 全文检索 + 排序 |
| 日志分析 | 高吞吐写入 + 聚合 |
| 指标分析 | 时间/标签查询 + 聚合 |
| 安全分析 SIEM | 海量事件搜索与关联 |
| 内容搜索 | 分词、相关性排序 |

---

### 10.2 不适合什么？

| 场景 | 原因 |
|---|---|
| 复杂事务 | 不适合作为关系型事务数据库 |
| 强一致主库 | 默认存在副本同步延迟 |
| 大量 JOIN | 不擅长关系型关联 |
| 核心 KV 主库 | 通常不如专用 KV 数据库直接 |

典型架构：

```text
MySQL
  ↓ CDC
Elasticsearch
  ↓
搜索 / 分析
```

即：

> **MySQL 做事实主库，ES 做搜索与分析副本。**

---

### 10.3 写入优化

```text
1. 使用 _bulk
2. 控制批次大小
3. 合理调整 Refresh
4. 批量导入时可降低 Refresh/Replica 开销
5. 根据业务评估 Translog durability
6. 避免不必要的更新
```

批量导入完成后再恢复正常 Refresh/Replica 配置。

---

### 10.4 查询优化

```text
1. 精确条件优先使用 filter
2. 避免深度 from + size
3. 深分页使用 search_after
4. 全量导出使用 scroll
5. 只返回需要的 _source 字段
6. 合理使用 Routing
7. 避免 text + Fielddata
8. 控制高基数聚合
```

---

### 10.5 分片优化

原文建议：

> 单分片约 **10GB～50GB**，实际应根据数据量、查询模式、恢复时间和节点资源综合确定。

原则：

```text
分片太少
→ 并行度不足

分片太多
→ 查询广播、元数据、Merge、恢复成本增加
```

---

### 10.6 常见问题排查

| 现象 | 优先检查 |
|---|---|
| 写入拒绝 | Write 线程池、批量大小、节点资源 |
| 查询慢 | Slow Log、DSL、分片数量、聚合 |
| GC 频繁 | Heap、Fielddata、缓存 |
| OOM | Fielddata、聚合、请求大小 |
| 分片不均 | Routing 倾斜、Shard Allocation |
| 深分页慢 | `from + size` 是否过深 |
| 聚合慢 | 高基数字段、Global Ordinals、Doc Values |

---

# 11. 最终记忆图

```text
                    Elasticsearch
                         │
                 Distributed Index
                         │
                    ┌────┴────┐
                    │  Shard  │
                    └────┬────┘
                         │
                   Lucene Index
                         │
                    ┌────┴────┐
                    │ Segment │
                    └────┬────┘
          ┌──────────────┼──────────────┐
          ↓              ↓              ↓
     FST/Term         Postings      Stored Fields
     找 Term           找 DocID       取文档
          │              │              │
          └──────┬───────┘              │
                 ↓                      │
               DocID ───────────────────┘

                    +
               Doc Values
                    ↓
              排序 / 聚合

写入：
Memory Buffer → Translog → Refresh → Segment
                                   ↓
                              Page Cache
                                   ↓
                              Flush/fsync

后台：
Segment + Segment → Merge → 更大 Segment + 清理删除数据

分布式：
Routing → Primary Shard → Replica
                         ↓
                  故障转移 / 高可用
```

---

# 12. 面试一句话

> **Elasticsearch 是基于 Lucene 的分布式搜索与分析引擎。一个 Index 被划分为多个 Shard，每个 Shard 底层是 Lucene Index，并以不可变 Segment 组织数据。全文搜索通过 FST/Term Dictionary 找 Term，再通过 Postings 找 DocID，Fetch 阶段通过 Stored Fields 取文档；排序和聚合主要依赖 Doc Values。写入经过 Memory Buffer + Translog，Refresh 让数据可搜索，Flush/fsync 完成持久化，Merge 合并 Segment 并清理删除数据。分布式层通过 Primary/Replica、Routing、Master 选举和自动故障转移实现水平扩展与高可用。**

---

# 13. 面试高频追问速答

**Q：为什么 ES 搜索快？**

> 倒排索引直接从 Term 定位 DocID，FST 加速词典定位，Postings 使用压缩和跳跃结构减少扫描；查询还可以在多个 Shard 并行执行。

**Q：为什么 Segment 不直接修改？**

> 不可变 Segment 简化并发读写，避免频繁修改大型倒排结构；更新通过新 Segment + 删除标记实现，后台 Merge 再清理。

**Q：Stored Fields 和 Doc Values 有什么区别？**

> Stored Fields 面向 `DocID → 文档字段`，主要服务 Fetch；Doc Values 面向 `Field → Values`，主要服务排序、聚合和脚本。

**Q：Refresh 和 Flush 有什么区别？**

> Refresh 解决“能不能搜索到”；Flush/fsync 解决“数据是否完成持久化”。

**Q：Translog 是干什么的？**

> 记录写操作，用于节点异常后的恢复，不是搜索索引本身。

**Q：为什么不能对 text 聚合？**

> text 面向全文搜索；强制使用 Fielddata 会把数据加载到 JVM Heap，容易造成高内存甚至 OOM。聚合通常使用 keyword。

**Q：为什么需要 Replica？**

> Replica 同时承担容灾和读压力分担；Primary 故障后 Replica 可以晋升。

**Q：Routing 有什么用？**

> 根据 `hash(routing) % primary_shards` 定位 Shard。合理的自定义 Routing 可以减少查询涉及的 Shard，但要防止数据倾斜。

**Q：ES 为什么不是传统数据库？**

> ES 优先解决搜索、分析和水平扩展，不以复杂事务、JOIN 和强一致主数据为核心。
