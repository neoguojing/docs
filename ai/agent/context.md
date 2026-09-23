# Agent 上下文工程（Context Engineering）技术架构全景

---

## 1. 核心概念与范式演进

### 1.1 从 Prompt Engineering 到 Context Engineering

传统的 Prompt Engineering 关注**单次静态表达的优化**（“如何通过词句引导模型给出更好回答”）；而面向生产环境的 Agent 上下文工程，关注的是**动态信息流编排与工作内存管理**（“在这一步决策中，模型究竟需要看见什么、忽略什么、引用什么”）。

```
[传统 Prompting]
System Prompt + User Prompt ──► LLM ──► Output

[Agent Context Engineering]
User Goal + State_t + Memory + JIT Retrieval + Tool Results + Recent History
                                │
                                ▼
                       [Context Assembler]
                                │
                                ▼
                       [Context Optimizer] (Trimming / Compaction / Ordering)
                                │
                                ▼
                               LLM
                                │
                                ▼
                              Action
                                │
                                ▼
                         [Environment]
                                │
                                ▼
                             State_{t+1} ──► 递归驱动下一轮 Context 组装

```

### 1.2 上下文工程的 8 个核心子领域

* **Selection（选择）**：确定候选信息中哪些与本轮决策直接相关。
* **Retrieval（获取）**：区分语义检索（RAG）与权威状态读取（Tool/JIT）。
* **Compression（压缩）**：对膨胀的历史轨迹进行高保真结构化状态提炼。
* **Externalization（外化）**：将大体量数据下沉至持久化系统，上下文仅留句柄（Pointer > Payload）。
* **Ordering（组织）**：依据模型注意力特性与 KV Cache 复用需求，分层排版上下文块。
* **Isolation（隔离）**：利用子 Agent 拓扑切分上下文边界，避免跨域污染。
* **Persistence（持久化）**：将不可再生经验和事实沉淀为跨会话的长期记忆。
* **Caching（复用）**：保持稳定前缀（Stable Prefix），适配底层推理引擎的 KV Cache 机制。

---

## 2. 核心原理与物理约束

### 2.1 Context Window 是工作内存（RAM），不是存储（Disk）

在操作系统体系中，CPU 寄存器和 L1/L2 缓存昂贵且易失，磁盘存储廉价且持久。将 Context Window 类比为工作内存，外部系统类比为外部持久化存储：

| 维度 | Context Window（工作内存） | 外部持久化存储（文件 / DB / Memory Store） |
| --- | --- | --- |
| **访问延迟** | 极快（直接参与注意力前向计算） | 较慢（依赖 Tool Call、I/O 调度或检索网络） |
| **容量成本** | 昂贵（线性/超线性增加计算与推理显存开销） | 极度廉价，理论上可无限水平扩展 |
| **状态持久性** | 易失（单次进程生命周期，随窗口重置而丢失） | 持久（跨会话、跨节点、可审计、可版本化） |
| **状态本质** | 局部计算工作区 | 扩展状态空间（Extended State Space） |

### 2.2 上下文腐烂（Context Rot）机理

上下文的盲目扩张不仅带来成本上涨，更会触发“质量劣变”：

1. **注意力竞争与稀释（Attention Dilution）**：有效 Token 与干扰 Token 争抢注意力权重，导致关键约束被忽略。
2. **状态漂移与冲突**：长程交互中出现前后矛盾的信息（如早期环境端口与后期重构端口并存），模型缺乏内置的冲突消除机制，极易采信失效旧状态。
3. **“中间迷失”（Lost in the Middle）的工程视角**：虽然 FlashAttention 和架构调优改善了注意力计算，但深层 Transformer 对序列两端（System Prefix 与 Recent Suffix）的寻址保真度在实测中依然显著高于大段漫长无序的中间日志。

### 2.3 上下文构造公式

任意时间步 $t$ 的有效输入上下文 $C_t$ 可抽象为：

 $$C_t = S + G_t + \text{State}_t + \text{Memory}_t + \text{Retrieval}_t + \text{Trajectory}_t$$ 

* $S$：固定前缀系统指令与规则（Stable System Context）
* $G_t$：当前拆解的目标/子目标（Active Goal）
* $\text{State}_t$：任务当前进度与环境快照（Structured State）
* $\text{Memory}_t$：经过冲突消解后注入的相关长期记忆（Injected Memory）
* $\text{Retrieval}_t$：即时拉取的文件局部或精准切片（JIT Context）
* $\text{Trajectory}_t$：最近 $k$ 轮的纯净交互足迹（Sliding Working Window）

---

## 3. 六大核心策略与生命周期管理

### 3.1 结构化状态压缩（Structured State Compaction）

放弃“对上一段话写个总结”的自由文本摘要思路，采用强 Schema 模式重构上下文，确保提取的信息具备“冷启动恢复能力”：

```json
{
  "goal": "重构网关认证中间件并修复跨域漏洞",
  "completed_milestones": ["编写 JWT 校验管道", "单元测试拦截无效 Token"],
  "current_in_progress": "修复 OPTIONS 预检请求的跨域头丢失问题",
  "active_constraints": ["Go 1.22 原生路由规范", "严禁使用第三方外部 CGO 库"],
  "active_blockers": ["OPTIONS 请求未走 Auth 中间件但直接被 403 拦截"],
  "modified_artifacts": ["internal/middleware/cors.go", "internal/middleware/auth.go"],
  "pending_tasks": ["补齐 OPTIONS 单元测试用例", "执行集成回归测试"],
  "next_action": "在 cors.go 中补充针对 OPTIONS 方法的直接放行并附加 Header 逻辑"
}

```

### 3.2 动态信息效用评分（Utility Model）

当 Token 水位触碰警戒线（如 75%）时，通过效用比对信息进行分类分流：

 $$\text{Utility}(i) = \frac{\text{Relevance}(i) \times \text{Importance}(i) \times \text{Freshness}(i) \times \text{Uniqueness}(i)}{\text{Token\_Count}(i)}$$ 

* **高 Utility / 低 Token** $\to$ **保留（Keep）**：直接保留在活动上下文中。
* **高 Utility / 高 Token** $\to$ **压缩（Compress）**：抽取成结构化状态字段。
* **低 Utility / 可重现** $\to$ **外化（Externalize）**：写入外部存储系统（保存为 `/tmp/log_step_4.txt`），上下文仅保留引用句柄。
* **低 Utility / 不可重现** $\to$ **丢弃（Drop）**：过期临时的格式报错与重试尝试直接清除。

### 3.3 外化与 JIT 延迟加载（Just-in-Time Context）

* **Pointer > Payload 原则**：遇到代码库、5000 行错误日志、大型 DOM 树时，严禁原样注入。上下文只呈现资源句柄：`[FileRef: src/server.py | Size: 18KB | SHA: e2b4c1]`。
* **JIT 与 RAG 的分工边界**：
* **RAG（语义驱动）**：解决“不知道确切位置，寻找语义相关知识”（如公司报销政策、通用文档匹配）。
* **JIT Tool Call（确定性驱动）**：解决“已知确切目标，拉取权威事实”（如 `read_lines(path, start, end)`、`get_db_schema(table)`）。



### 3.4 记忆生命周期与读写分离架构

1. **分类沉淀**：
* **Profile**：静态事实（用户开发环境、硬件架构）。
* **Project**：核心决策的背景（Why）。
* **Feedback**：用户显式校正（权重最高，具有覆写权）。


2. **持久化评判黄金律**：**“如果未来能以低延迟 Tool Call 原样获取，就绝不持久化到长期记忆中。”**
3. **读写解耦（Observer Pattern）**：
* 主 Agent 专职执行任务，严禁自省分析并自写记忆。
* 后台异步 Memory Worker 监听执行事件总线（Event Bus），按固定步长拉取多轮历史，执行实体抽取、冲突检测、消重去冲突后落库。



---

## 4. 全链路系统架构

```
                              User Input
                                   │
                                   ▼
                     ┌───────────────────────────┐
                     │     Agent Controller      │
                     └─────────────┬─────────────┘
                                   │
                                   ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                    上下文装配引擎 (Context Assembler)                         │
├─────────────────────────────────────────────────────────────────────────────┤
│  [Layer 1: 稳定前缀 (KV Cache 命中区)]                                        │
│  - System Persona / Core Rules / Tool Schemas (静态声明)                      │
├─────────────────────────────────────────────────────────────────────────────┤
│  [Layer 2: 动态上下文装配区]                                                  │
│  - User Active Goal (当前子目标)                                             │
│  - Compacted State (压缩后的状态快照)                                        │
│  - Memory Top-K (经冲突消解的长期事实)                                       │
│  - JIT Slices (原子工具实时取回的局部文本)                                    │
├─────────────────────────────────────────────────────────────────────────────┤
│  [Layer 3: 滑动工作窗口]                                                     │
│  - Recent Raw Turns (最近 N 轮未经压缩的精准交互轨迹)                          │
└─────────────────────────────────────────────────────────────────────────────┘
                                   │
                                   ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                    上下文优化器 (Context Optimizer)                          │
│     Token 预算校验  ──►  Tool 输出清洗截断  ──►  高危信息剔除                  │
└─────────────────────────────────────────────────────────────────────────────┘
                                   │
                                   ▼
                        LLM 推理引擎 (Inference)
                                   │
                   ┌───────────────┴───────────────┐
                   ▼                               ▼
                 Text Output                     Action Output
                                                   │
                                                   ▼
                                       权限与安全护栏 (Guardrails)
                                                   │
                                                   ▼
                                       工具运行时 (Tool Runtime)
                                                   │
                                                   ▼
                                       真实物理环境 (OS/DB/Web)
                                                   │
                                                   ▼
                                          状态更新与事件日志
                                                   │
                            ┌──────────────────────┴──────────────────────┐
                            ▼                                             ▼
                     更新内部 State 快照                            异步 Memory Worker
                                                                          │
                                                                          ▼
                                                                     写入持久记忆库

```

---

## 5. 推理优化与协同设计

### 5.1 Cache-Friendly 设计规范

在结合现代高并发推理引擎（如 vLLM、SGLang）时，上下文的物理组织直接决定了前缀树缓存（Radix Attention / Prefix Cache）的命中率：

1. **Stable Prefix 模式**：不变的系统指令、全局规则、工具声明必须严格排布在前缀位置，避免在序列开头动态插入时间戳、随机数或频繁变动的状态变量导致整段缓存击穿。
2. **Append-Only 轨迹增长**：中间交互过程严格顺序向后追加。如需纠正策略，采用生成“反思/修正动作”追加到序列尾部，严禁回溯修改历史已缓存的消息块。

---

## 6. 生产落地的权衡考量（Trade-offs）

1. **Compaction 频率与信息丢失的平衡**：
* 频繁压缩：Token 占用低、延迟稳定，但容易在摘要迭代中累积损失细节（“传话筒效应”）。
* 迟滞压缩：细节保留充分，但逼近长文本极限，引发中间迷失且显著推高推理成本。
* *最佳实践*：基于 Token 水位（如 70%）与核心状态变更（如进入新阶段）双触发。


2. **JIT 检索延迟与上下文纯净度的平衡**：
* 每次决策都调用原子工具读取：上下文信噪比极高，但增加了网络 I/O 与 Tool Loop 轮数。
* 一次性批量预加载：执行单轮完成，但可能引入 80% 的无效信息。
* *最佳实践*：粗粒度索引（目录、Schema）预加载，细粒度详情（代码实现、文件正文）走 JIT 按需拉取。


3. **KV Cache 命中率与动态更新的冲突**：
* 为了极致 Cache 命中，倾向于完全不动 System Prompt。
* 当动态状态或环境发生改变时，将其统一封装为“系统通知事件（System Event）”作为最新的 User/Tool 消息追加在末尾，严禁回改静态前缀。



---

