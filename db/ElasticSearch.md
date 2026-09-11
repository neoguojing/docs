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

# Elasticsearch 底层磁盘存储

> 核心主线：**JSON → Memory Buffer + Translog → Refresh → Segment → Flush/fsync → 磁盘**
>
> Segment 内部按用途拆成多套结构：**FST/Term Dictionary 查词、Postings 找 DocID、Stored Fields 取原文、Doc Values 做排序/聚合**。

## 1. 整体结构

ES 的一个 Shard 底层对应一个 Lucene Index。Lucene 不直接以 JSON 文件保存数据，而是把数据组织成多个不可变的 Segment；每个 Segment 有自己的词典、倒排、Stored Fields、Doc Values 等文件。fileciteturn0file0L297-L325

```text
ES Index
└── Shard 0
    └── Lucene
        ├── Segment _0
        │   ├── .tip/.tim   Term Index + Term Dictionary
        │   ├── .doc/.pos   Postings
        │   ├── .fdx/.fdt   Stored Fields
        │   └── .dvm/.dvd   Doc Values
        └── Segment _1
            └── 同上
```

典型目录可概括为：

```text
<data>/nodes/0/indices/<index_uuid>/<shard>/
├── index/          # Lucene Segment 文件
│   ├── segments_N  # 当前 Commit 的 Segment 清单
│   ├── _0.tip
│   ├── _0.tim
│   ├── _0.doc
│   ├── _0.pos
│   ├── _0.fdx
│   ├── _0.fdt
│   ├── _0.dvm
│   └── _0.dvd
└── translog/       # ES 事务日志
    ├── translog-*.tlog
    └── translog.ckp
```

> **注意**：具体文件名、Codec、文件组合会随 Lucene 版本、字段类型和配置变化；`.cfs` 可能把小 Segment 的多个文件封装起来。

---

## 2. 用同一组数据理解所有结构

统一使用 3 个文档：

```text
DocID 1: {"product":"MacBook Pro", "price":15000}
DocID 2: {"product":"MacBook Air", "price":8000}
DocID 3: {"product":"iPad Pro",    "price":6000}
```

假设 `product` 分词后得到：

```text
Doc1 → macbook, pro
Doc2 → macbook, air
Doc3 → ipad, pro
```

于是：

```text
Term Dictionary
────────────────
air
ipad
macbook
pro

Postings
────────────────
air    → [Doc2]
ipad   → [Doc3]
macbook→ [Doc1, Doc2]
pro    → [Doc1, Doc3]
```

后面所有例子都基于这 3 个 DocID，不再更换数据。

---

## 3. Segment：不可变的小型索引

Segment 是 Lucene 的基本存储/检索单元。一个 Segment 可以独立完成查询，写入后基本不可修改。

```text
Segment _0
├── Term Dictionary : air, ipad, macbook, pro
├── Postings        : term → DocID
├── Stored Fields   : DocID → 原始字段
└── Doc Values      : field → values
```

如果更新 Doc1：

```text
旧 Segment _0
Doc1 price=15000  → 标记删除

新 Segment _1
Doc1 price=12000  → 新写入
```

后台 Merge：

```text
Segment _0 + Segment _1
          ↓
      Segment _2
          ↓
旧 Doc1 被物理清除
```

**为什么不可变？**

避免频繁修改大型倒排结构，降低读写竞争；同时便于利用 OS Page Cache。删除/更新通过新 Segment + 删除标记实现，最终由 Merge 回收。

---

## 4. Term Dictionary + FST：先找到“词”

### 4.1 Term Dictionary

`.tim` 保存 Segment 中有序的 Term，并组织成 Block。

```text
.tim

Block A:
  air
  ipad

Block B:
  macbook
  pro
```

查询 `macbook` 时，不应该从头扫描整个 `.tim`。

### 4.2 Term Index / FST

`.tip` 是 Term Index，使用 FST 等结构压缩保存“Term → `.tim` 中 Block 的定位信息”。

为了统一举例，假设：

```text
air     → .tim Block A
ipad    → .tim Block A
macbook → .tim Block B
pro     → .tim Block B
```

可以抽象成：

```text
                FST (.tip)
                   │
         ┌─────────┴─────────┐
         │                   │
       "a/i"               "m/p"
         │                   │
         ▼                   ▼
     Block A              Block B
         │                   │
         ▼                   ▼
      .tim:A              .tim:B
   air, ipad          macbook, pro
```

FST 的关键不是“保存完整词典”，而是**用紧凑状态机快速把查询词定位到 `.tim` 的相应范围**。

例如：

```text
查询 "macbook"
    ↓
FST (.tip)
    ↓
定位 Block B
    ↓
.tim Block B
    ↓
确认 macbook
    ↓
得到该 Term 对应的 Postings 信息
```

### 4.3 FST 为什么省内存？

假设 Term Dictionary 有几十 GB：

```text
完整词典
几十 GB
   │
   └── 不可能全部常驻内存

FST
较小的压缩索引
   │
   └── 常驻/高效访问
          ↓
       定位 .tim
```

**面试关键词：**

> FST 是 Term Index 的核心结构，通过共享状态/路径并携带输出信息，以较小内存开销快速定位 Term Dictionary 中的 Block。

---

## 5. Postings：找到 Term 对应的 DocID

找到 `macbook` 后，还需要知道它在哪些文档：

```text
Term
macbook
   ↓
Postings
   ↓
[Doc1, Doc2]
```

完整关系：

```text
.tim
┌──────────┐
│ macbook  │
└────┬─────┘
     │
     ▼
.doc
┌──────────────────┐
│ DocID: 1         │
│ DocID: 2         │
└──────────────────┘
```

Postings 不只是 DocID，还可以包含：

```text
DocID
Term Frequency
Position
Offset / Payload（按索引能力）
```

例如：

```text
macbook
  ├── Doc1, TF=1
  └── Doc2, TF=1

pro
  ├── Doc1, TF=1
  └── Doc3, TF=1
```

### 为什么压缩？

假设 DocID：

```text
[100, 103, 104, 109, 110]
```

可以转成 Delta：

```text
[100, 3, 1, 5, 1]
```

再进行 Block/Packed 编码，减少磁盘空间和读取数据量。

因此：

> **Postings 解决的是：这个 Term 出现在哪些 DocID？**

---

## 6. Stored Fields：拿 DocID 找回原文

搜索 `macbook`：

```text
macbook
   ↓
Postings
   ↓
[Doc1, Doc2]
```

如果最终需要返回 JSON，还要：

```text
DocID
  ↓
.fdx
  ↓
定位 Stored Fields
  ↓
.fdt
  ↓
取出字段
```

统一例子：

```text
.fdx
┌─────────────────────┐
│ Doc1 → Chunk A      │
│ Doc2 → Chunk A      │
│ Doc3 → Chunk B      │
└─────────────────────┘

.fdt
┌─────────────────────────────────────┐
│ Chunk A                              │
│   Doc1 → MacBook Pro, 15000         │
│   Doc2 → MacBook Air, 8000          │
├─────────────────────────────────────┤
│ Chunk B                              │
│   Doc3 → iPad Pro, 6000             │
└─────────────────────────────────────┘
```

实际 Stored Fields 会采用二进制编码和分块压缩，不是简单的一行一个 JSON。

所以：

```text
查询 macbook
    ↓
Postings → [1,2]
    ↓
.fdx → 找到 Doc1/Doc2 所在 Chunk
    ↓
.fdt → 读取并解压 Chunk
    ↓
返回 _source / Stored Fields
```

**一句话：**

> Stored Fields 解决“已经找到 DocID，怎么把需要返回的字段取出来”。

---

## 7. Doc Values：按字段计算，而不是按文档取原文

现在执行：

```text
AVG(price)
```

如果从 `.fdt` 读取：

```text
Doc1 → JSON → 解析 price
Doc2 → JSON → 解析 price
Doc3 → JSON → 解析 price
```

效率低。

Doc Values 提前按列组织：

```text
.dvd

price
────────
Doc1 → 15000
Doc2 →  8000
Doc3 →  6000
```

另一个字段：

```text
product.keyword
────────────────
Doc1 → MacBook Pro
Doc2 → MacBook Air
Doc3 → iPad Pro
```

`.dvm` 提供字段的元数据、定位和编码信息，`.dvd` 保存实际 Doc Values 数据。

因此：

```text
AVG(price)
    ↓
.dvm
    ↓
定位 price 列
    ↓
.dvd
    ↓
[15000, 8000, 6000]
    ↓
AVG = 9666.67
```

**一句话：**

> Stored Fields 是“按文档取数据”，Doc Values 是“按字段取数据”。

---

## 8. 四种核心结构放在一起

仍然使用：

```text
Doc1 = MacBook Pro / 15000
Doc2 = MacBook Air /  8000
Doc3 = iPad Pro    /  6000
```

```text
                    Segment _0
                        │
       ┌────────────────┼────────────────┐
       │                │                │
       ▼                ▼                ▼
   Term/FST          Postings       Stored Fields
   .tip/.tim           .doc          .fdx/.fdt
       │                │                │
       │                │                │
       ▼                ▼                ▼
 "macbook"         [Doc1,Doc2]     Doc1 → JSON
                                      Doc2 → JSON
                                      Doc3 → JSON

                        +

                    Doc Values
                    .dvm/.dvd
                        │
                        ▼
                price → [15000,8000,6000]
```

对应查询：

```text
关键词搜索：
"macbook"
  → FST/.tip
  → Term Dictionary/.tim
  → Postings/.doc
  → [Doc1, Doc2]

返回原文：
[Doc1, Doc2]
  → .fdx
  → .fdt
  → JSON

排序/聚合：
price
  → .dvm
  → .dvd
  → [15000,8000,6000]
```

---

## 9. Translog：Segment 之外的“恢复日志”

写入 Doc3：

```text
                 Doc3
                  │
          ┌───────┴────────┐
          ▼                ▼
   Memory Buffer        Translog
          │                │
          │                └── 顺序追加
          ▼
       Refresh
          │
          ▼
     新 Segment
          │
          ▼
      OS Page Cache
```

Translog 的作用不是负责搜索，而是：

> **在 Lucene Segment 尚未完成持久化时，保证节点异常后可以恢复操作。**

因此：

```text
Refresh ≠ fsync 到物理盘
```

Refresh 主要解决：

```text
“什么时候可以搜索到？”
```

Translog + 持久化机制解决：

```text
“节点异常后怎么恢复？”
```

---

## 10. Refresh / Flush / Merge 一次讲清

### Refresh

```text
Memory Buffer
     ↓
  Refresh
     ↓
New Segment
     ↓
可搜索
```

核心：

> **让数据进入搜索视图。**

### Flush / Commit

```text
Segment / Page Cache
       ↓
     fsync
       ↓
 Physical Disk
       ↓
更新 Commit Point
       ↓
segments_N
```

核心：

> **推动 Lucene 索引持久化，并更新 Commit 状态；Translog 在满足条件后可以被清理。**

### Merge

```text
Segment _0 ─┐
Segment _1 ─┼──→ Merge → Segment _3
Segment _2 ─┘
```

作用：

- 减少 Segment 数量
- 合并索引文件
- 清理已标记删除的文档
- 回收磁盘空间

---

## 11. 一次完整写入 + 查询

### 写入

```text
POST /index/_doc/3
{"product":"iPad Pro","price":6000}
              │
              ▼
        Memory Buffer
              │
              ├────────→ Translog
              │
          Refresh
              │
              ▼
        Segment _1
              │
      ┌───────┼────────┐
      ▼       ▼        ▼
    .tim/.tip .doc   .fdt/.fdx
                      │
                    .dvd/.dvm
              │
              ▼
         OS Page Cache
              │
           Flush
              │
            fsync
              ▼
        Physical Disk
```

### 查询

```text
GET /index/_search
product:macbook
          │
          ▼
     FST / .tip
          │
          ▼
    Term Dictionary
       / .tim
          │
          ▼
    Postings / .doc
          │
          ▼
     [Doc1, Doc2]
          │
          ▼
    Query / Score
          │
          ▼
      Fetch阶段
          │
          ▼
     .fdx → .fdt
          │
          ▼
   Doc1 / Doc2 JSON
```

如果查询是：

```text
ORDER BY price
```

则主要利用：

```text
price
  ↓
Doc Values
  ↓
.dvm/.dvd
  ↓
排序
```

---

## 12. 最终记忆模型

把 ES/Lucene 磁盘存储压缩成下面这张图：

```text
                         Lucene Segment
                               │
          ┌────────────────────┼────────────────────┐
          │                    │                    │
          ▼                    ▼                    ▼
    Term Index/FST         Postings           Stored Fields
       .tip                  .doc              .fdx/.fdt
          │                    │                    │
          ▼                    ▼                    ▼
      定位词典              找 DocID              取原文
          │                    │                    │
          ▼                    │                    │
       .tim                   │                    │
    Term Dictionary           │                    │
          │                    │                    │
          └───────────────┬────┘                    │
                          ▼                         │
                       DocID ───────────────────────┘

                          +

                      Doc Values
                       .dvm/.dvd
                          │
                          ▼
                    排序 / 聚合 / 脚本
```

### 面试一句话

> **Elasticsearch 的一个 Shard 底层是 Lucene Index，数据以不可变 Segment 组织。Segment 内部不是简单保存 JSON，而是分别维护 Term Index/FST + Term Dictionary 用于定位词，Postings 用于从 Term 找 DocID，Stored Fields 用于从 DocID 找回文档字段，Doc Values 用于按字段进行排序和聚合。写入通过 Memory Buffer + Translog，Refresh 产生可搜索 Segment，Flush/fsync 完成持久化，后台 Merge 合并 Segment 并回收删除数据。**

| 结构 | 典型文件 | 统一例子 | 解决什么问题 |
|---|---|---|---|
| Term Index / FST | `.tip` | `macbook → Block B` | 快速定位词典 |
| Term Dictionary | `.tim` | `air, ipad, macbook, pro` | 保存有哪些 Term |
| Postings | `.doc` | `macbook → [1,2]` | Term → DocID |
| Stored Fields | `.fdx/.fdt` | `1 → MacBook Pro/15000` | DocID → 原文/字段 |
| Doc Values | `.dvm/.dvd` | `price → [15000,8000,6000]` | 字段 → 值，排序/聚合 |
| Segment | 一组上述文件 | `_0` | 独立、不可变的索引单元 |
| Translog | `.tlog` | `Doc3 写入操作` | 崩溃恢复 |
| `segments_N` | `segments_N` | `_0,_1` 有效 | Commit 后的 Segment 清单 |

## 3. 内存与数据结构
# Elasticsearch 内存布局与底层数据结构详解

## 1. 宏观内存布局：JVM 与 OS Page Cache 的平衡

Elasticsearch 的内存并非全部由 JVM 掌控，而是遵循“一半留给 JVM，一半留给操作系统”的设计哲学。

*   **JVM 堆内存（Heap）：** 管理集群元数据、节点/请求缓存（Query/Request Cache）、写入缓冲（Indexing Buffer），以及常驻内存的 Lucene 词项索引（Term Index，基于 FST）。
    *   **配置红线：** 最大不超过物理内存的 50%，且绝对**不超过 32GB**。超过 32GB 会导致 JVM 指针压缩（Compressed Oops）失效，内存指针占用翻倍，实际可用内存不增反降。
*   **OS Page Cache（堆外/系统缓存）：** Lucene 严重依赖操作系统的文件系统缓存。段文件（Segments）、词典（Term Dictionary）、倒排表（Postings List）和列式存储（Doc Values）均存储于磁盘，通过 `mmap` 映射入内存。Page Cache 越大，磁盘 I/O 越少，查询越快。

---

## 2. 核心缓存机制

| 缓存名称 | 管理者 | 作用与特点 |
| :--- | :--- | :--- |
| **Node Query Cache** | JVM | 节点级缓存，基于 LRU 淘汰。**仅缓存 Filter 上下文**（不计算算分）。底层使用 BitSet（0/1 数组）存储文档匹配状态，位运算极速加速高频过滤条件。 |
| **Shard Request Cache**| JVM | 分片级缓存。直接缓存复杂聚合（Aggregation）的最终结果。若底层 Segment 无更新，相同查询直接返回结果。注意：带有 `now` 等时间函数的查询会导致此缓存失效。 |
| **Field Data Cache** | JVM | **高危！** 仅在对 `text` 字段强制聚合或排序时触发。将倒排索引实时反转并加载到堆内存，极易导致 OOM。生产环境应严格禁止在 `text` 字段上聚合。 |

---

## 3. 核心数据结构与查询加速算法

### FST (有限状态转换器)：内存字典的极限压缩
**作用：** 作为 Term Index，将海量词条的磁盘偏移量驻留内存。
**原理与构建示例：** FST 是一种结合了前缀树（Trie）并实现**前缀和后缀双向共享**的自动机，数值被拆分并分布在路径的边上。数据必须按**字典序**插入。

**示例：插入 `mop` (10), `moth` (20), `pop` (15)**
1.  **插入 `mop` (10)：** 路径为 `m(10) -> o(0) -> p(0)`。累加得 10。
2.  **插入 `moth` (20)：** 与 `mop` 共享前缀 `mo`。`mo` 只能共享最小值 10。在 `o` 之后分岔，`t` 边上放剩下的 10，`h` 边放 0。
    *   查询 `moth`：走 `m(10) -> o(0) -> t(10) -> h(0)`，累加得 20。
3.  **插入 `pop` (15)：** 独立拉出前缀 `p(15)`。算法发现后缀 `op` 与之前的 `mop` 后缀相同，直接将 `p` 的线连入已有的 `o` 节点。
    *   查询 `pop`：走新线 `p(15)`，顺着老线 `o(0) -> p(0)`，累加得 15。

### Skip List (跳表)：加速多条件倒排交集
**作用：** 快速合并多个查询条件的倒排表（如 `A AND B`）。
**原理：** 为有序的倒排文档 ID 块建立分层索引。比对时，若目标 ID 大于当前块的最大值，则通过跳表指针直接跳过整个块，将时间复杂度从 $O(N)$ 降至接近 $O(\log N)$。

### FOR (Frame of Reference)：倒排表的位级压缩
**作用：** 极致压缩 Postings List 占用的磁盘和缓存空间。
**原理示例：** 压缩 DocID 列表 `[73, 300, 302, 332, 343, 372]`。
1.  **Delta 编码：** 记录差值转化为 `[73, 227, 2, 30, 11, 29]`。
2.  **Bit Packing：** 找寻差值中最大值 227，二进制需 8 bits。Lucene 统一分配 8 bits 容量存放这 6 个数字，总计 48 bits（常规 32位整型需 192 bits），空间节省 75%。

### BKD Tree：数值与空间的降维打击
**作用：** 替代倒排索引，专门处理数值（Integer, Date）和空间地理（Geo）的精准与范围检索。
**原理：** 在多维度空间中基于中位数不断切分数据块。遇到范围查询（如 `age > 30`）时，直接在树的分支上剪枝，无需遍历和枚举具体 Term。

### Doc Values：聚合排序的列式救星
**作用：** 解决 JVM OOM，用于非 text 字段的排序、聚合。
**原理：** 构建倒排索引时，同步在磁盘上生成按列连续排列的文件（正向映射：文档 ID -> 具体值）。计算聚合时，依靠 OS Page Cache 将紧凑的列数组喂给 CPU，完美契合 CPU 预读机制（Prefetch）。

---

## 4. 写入加速与近实时 (Near Real-Time) 机制

Elasticsearch 的毫秒级搜索基于以下内存与磁盘的流转机制：

1.  **Indexing Buffer：** 写入的文档首先进入堆内存的 Buffer，并记录 Translog (WAL) 防丢失。**此时不可搜索**。
2.  **Refresh (近实时核心)：** 默认每秒执行一次。将 Buffer 数据刷入操作系统的 Page Cache，生成不可变的 Lucene Segment。**数据一旦进入 OS Cache 即可被搜索**。
3.  **Flush：** 当 Translog 满或定时触发。执行物理 `fsync`，将 OS Cache 中的 Segment 强制落盘，并清空旧 Translog。
4.  **Merge (段合并)：** 后台异步将大量 Refresh 产生的小 Segment 合并为大 Segment，并在此刻物理删除 Tombstone（被标记删除）的文档数据。


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

Elasticsearch 处理并发的核心哲学是：**不加锁，靠版本；要吞吐，舍强一致。** 作为分布式系统，传统的悲观锁（Pessimistic Locking）会严重拖垮性能，因此 ES 全面拥抱乐观并发控制（OCC）和最终一致性来保障数据正确性。

### 1. 版本控制：乐观并发控制 (OCC)
ES 假设并发冲突发生的概率极低。写入时它不锁数据，而是带着版本凭证比对，凭证对得上才允许修改。
- **Internal（内部控制）**：默认机制。较新的 ES 版本依赖 `if_seq_no` 和 `if_primary_term` 联合校验，取代了老的 `_version`。若提交时的序号低于 ES 当前记录，请求将被拒绝（409 Conflict）。
- **External（外部控制）**：专为数据同步（如 MySQL -> ES）设计。以 MySQL 的时间戳或事务 ID 作为外部版本号。规则是：提交的外部版本号必须**严格大于** ES 当前记录的版本号才允许覆盖，有效解决网络延迟导致的数据乱序覆盖。

### 2. 序列号与生成号
为了在多节点间准确回放日志和比对数据新旧，ES 设计了双重坐标：
- **Primary Term (主分片代数)**：每次主分片重新选举，Term 就会 `+1`。彻底防止网络分区恢复后，旧主分片“诈尸”引发脑裂。
- **Sequence Number (序列号)**：主分片上发生的每一次写操作，都会获得一个分片级全局递增的编号。
*(通过 `Term + SeqNo`，ES 就能在全球集群中精确定位任何一次修改的绝对顺序。)*

### 3. 读写并发的底层机制
- **写入的局部串行**：在主分片上，写操作是串行化的。底层 Lucene 的 `IndexWriter` 持有独占锁，同一时刻只有一个线程在往段文件（Segment）里写数据。
- **读取的全局并发**：读操作可以在主分片或副本分片上执行。副本分片极大分散了读压力，但由于主副同步的毫秒级延迟，可能读到未更新的数据（**最终一致性**）。

### 4. 一致性级别与防脑裂
- **Quorum 机制**：写操作需要过半数的分片确认才算成功。公式：`(primary + replicas) / 2 + 1`。
- **wait_for_active_shards**：决定“写入前必须有多少个分片存活”。配置过半机制能有效防止网络分区（脑裂）时将数据写入到脱离大部队的假主分片中。

---

## 第二部分：批量操作 _bulk API 深度解析

`_bulk` 是 Elasticsearch 中用于**批量执行 CRUD**操作的核心 API。

### 1. 为什么必须使用 `_bulk`？
- **消除网络开销**：将上千次 HTTP 请求的 TCP 握手和 Header 解析成本压缩为 1 次。
- **提升集群吞吐量**：ES 接收后将大包拆分，按路由并发分发给各分片节点，最大化利用集群算力。
- **优化底层写入**：底层的 Lucene 引擎可以更高效地利用内存缓冲区，减少段文件的频繁刷新和磁盘 I/O。

### 2. `_bulk` 的数据格式 (NDJSON)
它**不是**一个标准的 JSON 数组，而是 **NDJSON (Newline Delimited JSON)** 格式。每一行都是一个独立的 JSON 对象，且必须以换行符 `\n` 结尾。

**结构说明：**
请求由“操作元数据行”和紧随其后的“数据正文行”交替组成（`delete` 操作只有元数据行）：
```json
{ "action": { "metadata" } }
{ "document_data" }
```

**支持的四种操作 (Action)：**
- `index`：索引文档（ID 存在则全量替换，不存在则创建）。
- `create`：创建文档（严格模式，ID 已存在则报错 409）。
- `update`：局部更新文档。
- `delete`：删除文档。

### 3. 实战代码示例
```http
POST /_bulk
{"index":{"_index":"users","_id":"1"}}
{"name":"张三","age":25,"role":"admin"}
{"create":{"_index":"users","_id":"2"}}
{"name":"李四","age":30,"role":"user"}
{"update":{"_index":"users","_id":"1"}}
{"doc":{"age":26}}
{"delete":{"_index":"users","_id":"3"}}
<此处必须有一个换行符>
```

### 4. `_bulk` 的三大避坑指南
1. **非原子操作（无事务）**：`_bulk` 里的指令是相互独立的。部分失败不会回滚整体操作。**必须**解析返回体检查 `errors: true`，遍历 `items` 数组找出失败项（如 429限流 或 409冲突）并进行重试。
2. **严格控制批次大小**：包过大会打满节点内存，引发 GC 停顿或 OOM。建议将每个 Bulk 请求控制在 **5MB 到 15MB 之间**（约 1000 - 5000 条数据）。
3. **严禁内部换行**：单条数据 JSON（例如数据正文行）**内部绝对不能包含换行符 `\n`**。必须序列化为紧凑的一行，否则会导致解析崩溃。
   
## 6. 分布式与高可用

# Elasticsearch 分布式与高可用架构详解

## 一、概述

Elasticsearch（简称 ES）的高可用与分布式架构是一套由"分布式容错 + 数据冗余 + 自动故障转移"构成的完整体系，其核心目标是实现数据不丢失、服务不中断以及故障自动恢复。

---

## 二、分布式架构与节点角色

ES 采用去中心化架构，对外表现为单一整体，内部通过不同角色的节点协同工作。

### 2.1 节点角色详解

| 节点角色 | 职责说明 | 建议 |
|---------|---------|------|
| **Master** | 管理集群元数据、分片分配、节点加入/离开 | 建议 3 个专用节点 |
| **Data** | 存储数据，执行增删改查 | 生产环境独立部署 |
| **Coordinating** | 接收请求，分发查询，合并结果 | 所有节点默认都是协调节点 |
| **Ingest** | 预处理管道，写入前对数据进行转换 | 减轻主节点和数据节点负担 |

### 2.2 生产级节点角色分离架构

在一个生产级的 ES 集群中，为了保证高可用和性能隔离，通常会将节点角色分离。以下是一个典型的 **"3 Master + N Data + 协调节点"** 架构：

```
                        [ Client / 负载均衡器 ]
                                |
                                v
+-----------------------------------------------------------------------+
|                      Elasticsearch Cluster                            |
|                                                                       |
|  [ Coordinating Nodes ]  <-- 接收请求、分发查询、合并结果 (可复用)      |
|       (Node C1)   (Node C2)                                           |
|           |              |                                            |
|           +--------------+-------------+                             |
|                          |             |                              |
|           [ Master Nodes ] (专用主节点，保证集群稳定)                   |
|           (Node M1)   (Node M2)   (Node M3)                           |
|              |             |             |                             |
|              +-------------+-------------+                             |
|                            |                                           |
|           [ Data Nodes ] (负责实际数据存储与计算)                        |
|      +-------------------+-------------------+                         |
|      |                   |                   |                         |
|  (Node D1)           (Node D2)           (Node D3)                     |
|  +-----------+       +-----------+       +-----------+                 |
|  | Shard 0(P)|       | Shard 1(P)|       | Shard 2(P)|                 |
|  | Shard 1(R)|       | Shard 2(R)|       | Shard 0(R)|                 |
|  +-----------+       +-----------+       +-----------+                 |
+-----------------------------------------------------------------------+
```

**核心解析：**

- **Master 节点（M1-M3）**：作为集群的"大脑"，只负责管理元数据（如创建索引、分配分片）。采用 3 个专用节点可以防止脑裂，保证高可用。
- **Data 节点（D1-D3）**：作为"苦力"，负责存储数据和执行增删改查。
- **分片分布（P=主分片，R=副本分片）**：注意观察，**主分片和它的副本绝不会在同一个节点上**。例如 Shard 0 的主分片在 D1，副本在 D3。如果 D1 宕机，D3 上的副本会瞬间接管，实现高可用。

---

## 三、数据分片与副本机制（高可用的基石）

分片（Shard）是 ES 分布式存储的最小物理单元，本质是一个独立的 Lucene 索引实例。

### 3.1 主分片（Primary Shard）

- 数据写入的唯一入口
- 数量在索引创建时固定且不可更改
- 建议单分片大小控制在 10GB~50GB 之间，以平衡恢复速度与元数据压力

### 3.2 副本分片（Replica Shard）

- 主分片的完整拷贝
- 副本不仅用于容灾，还能分担读请求压力，提升查询并发能力
- 副本数量可动态调整，生产环境至少配置 1 个副本

### 3.3 分片分配策略

| 策略 | 说明 |
|------|------|
| **主分片** | 创建时固定，不可更改 |
| **副本分片** | 不会分配到与主分片相同的节点，保证节点故障时数据不丢失 |
| **Shard Allocation Awareness** | 感知机架、可用区，将副本分散到不同物理位置 |

**严格的分配策略：** ES 的分片分配器会强制保证主分片与其副本绝不部署在同一节点上。结合 Shard Allocation Awareness（机架/可用区感知），ES 还能将副本分散到不同的物理机架或云可用区，从而容忍机房级别的故障。

---

## 四、智能路由与数据定位

ES 通过固定算法确定文档的物理存储位置：

> **路由公式：shard = hash(routing) % number_of_primary_shards**

### 4.1 默认路由

默认使用文档 ID（_id）作为 routing 值，确保数据均匀分布，且查询时能直接定位分片，无需广播。

### 4.2 自定义路由

通过指定 `?routing=user_123`，可以将同一用户的数据集中在同一分片，极大提升针对该用户的跨文档查询性能，避免跨分片查询带来的网络开销。

### 4.3 注意事项

- 修改主分片数或路由键会导致已有数据无法按新规则定位
- 必须通过 Reindex（重建索引）重新分布数据

---

## 五、数据写入与路由机制流程

当客户端写入一条数据时，ES 是如何知道该把它存到哪个节点、哪个分片的？

```
  [ Client ]
      |
      | 1. POST /index/_doc/123?routing=user_456 {"name": "Alice"}
      v
  [ Coordinating Node ]
      |
      | 2. 计算路由: shard = hash("user_456") % 3 = 1
      |    (目标: 主分片 Shard 1)
      v
  [ Data Node 2 ]  <-- 定位到 Shard 1 (Primary)
      |
      | 3. 写入内存 Buffer + 写入 Translog (防丢失)
      |
      | 4. 并行转发给副本
      +-------------------------+
      |                         |
      v                         v
  [ Data Node 3 ]           [ Data Node 1 ]
  (Shard 1 Replica)         (Shard 1 Replica)
      |                         |
      | 5. 副本写入成功           | 5. 副本写入成功
      +------------+------------+
                   |
                   v
          [ 返回 Client: 成功 ]
```

**核心解析：**

- **路由公式**：shard = hash(routing) % 主分片数。如果不指定 routing，默认用文档 _id。
- **Translog 机制**：这是 ES 保证数据不丢失的核心。数据写入内存后，会先写一份预写日志（Translog）到磁盘。即使节点突然断电，重启后也能通过 Translog 恢复数据。
- **同步复制**：写入请求必须在主分片和至少一个副本分片都写成功后，才会向客户端返回成功。

---

## 六、集群状态管理与高可用选举

### 6.1 选举机制

ES 7.x+ 版本采用基于类 Raft 协议的 Voting Configuration 机制。只要超过半数的候选主节点存活，集群就能选出新 Master，维持正常运行。

### 6.2 状态同步

集群状态（包括索引列表、Mapping、分片位置等）仅在 Master 节点更新，并增量同步到所有节点，确保整个集群对拓扑结构的认知一致。

---

## 七、故障检测、转移与自愈

当集群发生异常时，ES 的自动故障转移机制会在毫秒到秒级内完成服务切换：

### 7.1 故障检测

节点间通过心跳（Ping）机制检测状态。Master 发现节点失联后，会将该节点上的分片标记为 UNASSIGNED。

### 7.2 自动恢复流程

```
  [ 正常状态 ]                          [ 故障发生 ]
  +-----------+       +-----------+     +-----------+       +-----------+
  | Node D1   |       | Node D2   |     | Node D1   |       | Node D2   |
  | Shard 0(P)|       | Shard 1(P)|     |  宕机   |       | Shard 1(P)|
  | Shard 1(R)|       | Shard 2(P)|     |           |       | Shard 2(P)|
  +-----------+       +-----------+     +-----------+       +-----------+

  [ 自动恢复过程 (Master 节点接管) ]

  1. Master 节点通过 Ping 发现 Node D1 失联。
  2. Master 将 Node D1 上的 Shard 0(P) 标记为 UNASSIGNED。
  3. Master 发现 Shard 0 在其他节点有副本 (假设在 Node D3)。
  4. Master 将 Node D3 上的 Shard 0(R) 提升为新的主分片 Shard 0(P)。
  5. Master 在 Node D2 或新加入的节点上，为新的 Shard 0(P) 创建副本。

  [ 恢复后状态 ]
  +-----------+       +-----------+       +-----------+
  | Node D2   |       | Node D3   |       | Node D4   |
  | Shard 1(P)|       | Shard 0(P)| <-- 新主分片 |       | Shard 0(R)| <-- 新副本
  | Shard 2(P)|       | Shard 2(R)|       |           |
  +-----------+       +-----------+       +-----------+
```

### 7.3 延迟分配（Delayed Allocation）

为防止短暂的网络抖动或节点重启导致大量数据在集群内无效迁移，ES 默认设置了 `index.unassigned.node_left.delayed_timeout`（默认 1 分钟）。在此期间，ES 会等待原节点恢复，若超时未恢复才触发数据重分配。

### 7.4 副本晋升

只要集群中还有副本存在，主分片的故障就能在秒级被接管，对外部服务几乎无感知。

---

## 八、跨集群复制 (CCR) 与异地容灾

对于地域级容灾或读写分离需求，ES 提供了跨集群复制（CCR）功能。

### 8.1 CCR 架构图

```
  [ 主集群 (Leader) - 北京机房 ]             [ 备集群 (Follower) - 上海机房 ]
  +-------------------------+               +-------------------------+
  | Shard 0(P)  Shard 1(P)  |               | Shard 0(R)  Shard 1(R)  |
  | Shard 2(P)  Shard 3(P)  |               | Shard 2(R)  Shard 3(R)  |
  +-------------------------+               +-------------------------+
              |                                           ^
              |  实时异步复制 (Async Replication)          |
              +-------------------------------------------+
```

### 8.2 核心解析

- **Leader/Follower 架构**：Leader 集群接受写入，并将数据实时异步复制到 Follower 集群。Follower 集群默认只读，可用于就近查询或作为灾备集群。
- **读写分离与容灾**：主集群负责写入，数据实时同步到备集群。备集群可以提供只读查询服务。
- **异地多活**：当主集群所在机房整体故障时，可以将备集群提升为 Leader 集群，继续接受写入，实现机房级别的容灾。

---

## 九、动态扩缩容与负载均衡

### 9.1 自动 Rebalance

当新节点加入集群时，Master 会自动触发分片重平衡（Shard Rebalance），将部分分片从负载较高的节点迁移到新节点，以平衡磁盘和 CPU 使用率。

### 9.2 流量控制

可通过 `cluster.routing.rebalance.enable` 控制重平衡的时机（如仅在节点加入/离开时触发），避免在业务高峰期进行大量数据迁移影响性能。

### 9.3 客户端负载均衡

为避免单点故障，客户端请求应通过外部负载均衡器或轮询机制分发到多个协调节点，而非直连单一节点。

---

## 十、补充：近实时搜索与数据持久性保障

除了架构层面的高可用，ES 在存储引擎层面也保障了数据的可靠性：

### 10.1 近实时搜索（NRT）

数据写入后先存入内存缓冲区，默认每 1 秒执行一次 Refresh 操作生成 Segment 供搜索，因此存在最多 1 秒的搜索延迟。

### 10.2 Translog 机制

为了防止内存数据在 Refresh 前因宕机丢失，ES 采用预写日志（Translog）。数据写入内存的同时会追加写入 Translog 并 fsync 落盘。即使节点崩溃，重启后也能通过重放 Translog 恢复未持久化的数据，确保数据不丢失。

---

## 十一、总结：ES 高可用的三大防线

| 防线 | 机制 | 容忍故障范围 |
|------|------|-------------|
| **第一道防线** | 副本机制 | 单台机器宕机 |
| **第二道防线** | 机架/可用区感知 | 交换机故障或机房断电 |
| **第三道防线** | 跨集群复制 (CCR) | 城市级灾难 |

---

## 十二、关键配置参数速查

| 参数名 | 默认值 | 说明 |
|--------|--------|------|
| `index.unassigned.node_left.delayed_timeout` | 1m | 节点离开后延迟分配超时时间 |
| `cluster.routing.rebalance.enable` | all | 控制分片重平衡的时机 |
| `discovery.zen.ping.unicast.hosts` | - | 初始节点发现列表（7.x） |
| `cluster.initial_master_nodes` | - | 初始主节点列表（7.x+） |
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
