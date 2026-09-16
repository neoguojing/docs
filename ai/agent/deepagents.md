# LangChain / LangGraph / DeepAgents：核心能力与架构

> 目标：从工程架构角度理解 LangChain、LangGraph、DeepAgents 三者的定位、核心能力、内部关系以及如何组合使用。
>
> 核心结论：
>
> **LangChain = Agent Framework（框架）**
>
> **LangGraph = Agent Runtime / Orchestration（运行时与编排）**
>
> **DeepAgents = Agent Harness（复杂 Agent 能力套件）**
>
> 三者不是互斥产品，而是可以组合使用的三个层次。DeepAgents 构建在 LangChain 核心能力之上，并使用 LangGraph Runtime 执行。

---

## 1. 三者整体关系

```text
                         Application / Business
                                  │
                                  ▼
                    ┌─────────────────────────┐
                    │      DeepAgents         │
                    │     Agent Harness       │
                    │                         │
                    │ Planning / Todo         │
                    │ Subagents               │
                    │ Filesystem              │
                    │ Skills                  │
                    │ Memory                  │
                    │ Context Management      │
                    └────────────┬────────────┘
                                 │
                                 ▼
                    ┌─────────────────────────┐
                    │       LangChain         │
                    │     Agent Framework     │
                    │                         │
                    │ Model / Tool / Prompt   │
                    │ Agent / Middleware      │
                    │ Structured Output       │
                    │ Integrations            │
                    └────────────┬────────────┘
                                 │
                                 ▼
                    ┌─────────────────────────┐
                    │       LangGraph         │
                    │    Agent Runtime        │
                    │                         │
                    │ State / Graph / Node    │
                    │ Checkpoint / Persistence│
                    │ Durable Execution       │
                    │ Streaming / HITL        │
                    └────────────┬────────────┘
                                 │
                    ┌────────────┴────────────┐
                    ▼                         ▼
                  Model                      Tools
             OpenAI/Claude/...          DB / API / MCP
```

官方目前将三者分别定位为：

| 层次 | 产品 | 定位 | 核心问题 |
|---|---|---|---|
| Harness | DeepAgents | 复杂 Agent 能力套件 | Agent 如何高质量完成长期复杂任务？ |
| Framework | LangChain | Agent 开发框架 | 如何快速组装 Model、Tool、Prompt、Agent？ |
| Runtime | LangGraph | Agent 编排运行时 | 如何可靠地运行有状态、长时间、多步骤 Agent？ |

因此可以简单记忆：

```text
DeepAgents：给 Agent 装上“复杂任务能力”
LangChain：提供“Agent 开发积木”
LangGraph：提供“Agent 运行引擎”
```

---

# 2. LangChain 是什么？

## 2.1 定位

LangChain 是一个 **Agent Framework**。

核心职责不是提供一个“完整 Agent”，而是提供构建 Agent 所需要的：

- Model 抽象
- Tool 抽象
- Prompt
- Message
- Structured Output
- Middleware
- Agent abstraction
- 大量模型 / 工具 / 数据源集成

LangChain 官方当前将其定位为 **abstractions and integrations layer**。

可以理解为：

```text
LangChain
    =
Model + Tool + Prompt + Agent + Middleware
+ Integrations
```

---

# 3. LangChain 架构

```text
                         LangChain
                             │
       ┌─────────────────────┼─────────────────────┐
       │                     │                     │
       ▼                     ▼                     ▼
    Models                 Tools              Messages
       │                     │                     │
       │                     │                     │
       └─────────────────────┼─────────────────────┘
                             ▼
                         Middleware
                             │
                             ▼
                      Agent Abstraction
                             │
                             ▼
                    LangGraph Runtime
```

一个最基本的 Agent 可以抽象成：

```text
User
 │
 ▼
Agent
 │
 ├── LLM
 │
 ├── Tool
 │
 ├── LLM
 │
 ├── Tool
 │
 └── Final Answer
```

也就是：

```text
while not finished:

    response = model(context)

    if response requests tool:
        result = tool(...)
        context += result
    else:
        return response
```

这个循环本身非常简单。

LangChain 的价值主要在于把：

```text
Model
Tool
Message
Middleware
Agent
```

统一抽象起来。

---

# 4. LangChain Middleware

Middleware 是 LangChain Agent 架构中非常重要的扩展点。

可以理解为：

```text
                 Agent Loop
                     │
        ┌────────────┼────────────┐
        ▼            ▼            ▼
   Before Model   After Model   Tool Call
        │            │            │
        └────────────┼────────────┘
                     ▼
                 Middleware
```

Middleware 可以实现：

- Context 管理
- 动态 Prompt
- Tool 拦截
- Human-in-the-loop
- Guardrails
- 自动总结
- 重试
- 日志
- 自定义 Agent 行为

因此 LangChain 的 Agent 并不是一个完全固定的黑盒。

可以通过 Middleware 修改 Agent Loop。

---

# 5. LangChain Agent 的架构特点

LangChain 更适合：

```text
模型 + 工具 + 少量业务逻辑
```

例如：

```text
User
 ↓
LangChain Agent
 ↓
LLM
 ↓
Search Tool
 ↓
LLM
 ↓
Database Tool
 ↓
LLM
 ↓
Answer
```

当任务主要是：

- Chatbot
- RAG
- Tool Calling
- 简单 Agent
- Structured Output
- API / Database Agent

LangChain 通常已经足够。

---

# 6. LangGraph 是什么？

## 6.1 定位

LangGraph 是 **Agent Runtime / Orchestration Framework**。

它解决的问题不是：

> “如何调用一个 LLM？”

而是：

> “如何让一个复杂 Agent Workflow 长时间、可靠、有状态地运行？”

核心能力包括：

- State
- Graph
- Node
- Edge
- Checkpoint
- Persistence
- Durable Execution
- Streaming
- Human-in-the-loop
- Retry / Recovery
- Deterministic + Agentic 混合流程

---

# 7. 为什么需要 LangGraph？

简单 Agent：

```text
LLM
 ↓
Tool
 ↓
LLM
 ↓
Tool
 ↓
Answer
```

复杂 Agent：

```text
                 Planner
                    │
                    ▼
                 Task DAG
                    │
       ┌────────────┼────────────┐
       ▼            ▼            ▼
   Research       Coding       Search
       │            │            │
       └────────────┼────────────┘
                    ▼
                 Verifier
                    │
              ┌─────┴─────┐
              │           │
             Fail        Pass
              │           │
              ▼           ▼
            Retry        Final
```

这已经不是简单的：

```text
LLM → Tool → LLM
```

而是一个**有状态的工作流**。

LangGraph 就是用 Graph + State 描述和执行这种系统。

---

# 8. LangGraph 核心模型

LangGraph 最重要的三个概念：

```text
State
Node
Edge
```

## 8.1 State

State 是整个 Agent Workflow 的共享状态。

例如：

```python
class State(TypedDict):
    messages: list
    todos: list
    research_results: list
    code_result: str
    verification_result: str
```

---

## 8.2 Node

Node 是执行逻辑。

```text
Planner Node
Research Node
Coding Node
Verifier Node
```

例如：

```text
Planner
   ↓
Research
   ↓
Verifier
```

Node 可以：

- 调用 LLM
- 调用 Tool
- 执行业务代码
- 调用 Subgraph
- 修改 State

---

## 8.3 Edge

Edge 决定下一步执行什么。

例如：

```text
Planner
   │
   ├── research ──→ Research
   │
   └── coding ────→ Coding
```

也可以是条件 Edge：

```text
Verifier
   │
   ├── PASS → Final
   │
   └── FAIL → Retry
```

---

# 9. LangGraph Runtime 架构

```text
                         LangGraph Runtime
                                │
                 ┌──────────────┼──────────────┐
                 │              │              │
                 ▼              ▼              ▼
              Graph           State       Execution
                 │              │              │
          ┌──────┴──────┐       │        ┌─────┴─────┐
          ▼             ▼       │        ▼           ▼
        Node           Edge     │      Retry       Resume
          │             │       │
          └──────┬──────┘       │
                 ▼              ▼
              Workflow      Checkpoint
                                │
                                ▼
                            Persistence
```

LangGraph 的关键不是 Graph 这个数据结构本身，而是它提供了一个可以：

```text
执行
暂停
恢复
持久化
重试
流式输出
人工介入
```

的 Runtime。

---

# 10. LangGraph 最重要的能力：Durable Execution

例如：

```text
Task
 ↓
Research
 ↓
调用外部 API
 ↓
执行中断
```

普通程序：

```text
Process died
     ↓
全部重新执行
```

LangGraph：

```text
Task
 ↓
Research
 ↓
Checkpoint
 ↓
Process died
 ↓
Restart
 ↓
Resume
```

因此 LangGraph 非常适合：

- 长时间 Agent
- 多步骤任务
- Human approval
- 外部 API
- 大型工作流
- 异步任务
- 需要恢复的任务

---

# 11. LangGraph 的确定性 + Agentic 混合

这是 LangGraph 非常重要的设计思想。

传统 Workflow：

```text
A → B → C → D
```

全部确定。

Agent：

```text
LLM → Tool → LLM → Tool
```

大量概率性决策。

LangGraph 可以混合：

```text
                 Planner
                    │
                 LLM决策
                    │
                    ▼
             ┌──────────────┐
             │ Deterministic│
             │   Executor   │
             └──────┬───────┘
                    │
                    ▼
                  LLM
                    │
                    ▼
                Verifier
```

因此：

> **确定性的部分交给代码，非确定性的部分交给 LLM。**

这是构建生产级 Agent 非常重要的工程思想。

---

# 12. DeepAgents 是什么？

DeepAgents 是一个 **Agent Harness**。

它不是重新发明 Agent Runtime。

它建立在：

```text
LangChain
    ↓
LangGraph
```

之上。

官方当前实现可以理解成：

```text
DeepAgents
    =
LangChain Agent
+
Middleware
+
Planning
+
Subagents
+
Filesystem
+
Context Management
+
Skills
+
Memory
```

因此 DeepAgents 本质上是：

> **把复杂 Agent 常用的最佳实践预先组装起来。**

---

# 13. DeepAgents 核心能力

## 13.1 Planning

提供 Todo / Planning 能力。

```text
用户任务
   ↓
Planner
   ↓
┌──────────────────┐
│ TODO             │
│                  │
│ 1. Research      │
│ 2. Analyze       │
│ 3. Implement     │
│ 4. Verify        │
│ 5. Report        │
└──────────────────┘
```

Planning 的价值不仅是“管理任务”，更重要的是：

> 把长期任务的目标和当前进度外化，减少 Agent 在长任务中偏离目标。

---

# 14. Subagents

DeepAgents 可以创建子 Agent：

```text
                 Main Agent
                     │
       ┌─────────────┼─────────────┐
       ▼             ▼             ▼
 Research Agent   Coding Agent   Review Agent
       │             │             │
       ▼             ▼             ▼
   Research       Implement       Verify
       │             │             │
       └─────────────┼─────────────┘
                     ▼
                 Main Agent
```

最重要的价值不是“多几个 Agent”。

而是：

> **Context Isolation。**

例如：

```text
Main Context
     │
     ├── Research Context
     │       └── 大量搜索结果
     │
     ├── Coding Context
     │       └── 大量代码
     │
     └── Review Context
             └── 验证信息
```

最终 Main Agent 只拿到：

```text
Research Summary
Coding Result
Review Result
```

避免主 Context 被所有细节污染。

---

# 15. Filesystem

DeepAgents 提供虚拟文件系统。

```text
Agent
 │
 ├── read_file
 ├── write_file
 ├── edit_file
 └── ls
```

例如：

```text
/workspace/

research/
├── langchain.md
├── langgraph.md
└── deepagents.md

analysis/
└── comparison.md

output/
└── final.md
```

核心思想：

```text
Context Window
      │
      │ 不放所有信息
      ▼
Filesystem
      │
      │ 按需读取
      ▼
Context
```

也就是：

> **Externalized Context / Context 外化。**

这是 DeepAgents 和 Context Engineering 结合最紧密的地方。

---

# 16. Context Management

复杂 Agent 最大的问题之一：

```text
任务越来越长
       ↓
Context 越来越大
       ↓
Token 成本越来越高
       ↓
噪声越来越多
       ↓
模型注意力下降
       ↓
任务质量下降
```

DeepAgents 的解决思路：

```text
                 Context Management
                         │
          ┌──────────────┼──────────────┐
          ▼              ▼              ▼
      Compression     Filesystem      Subagents
          │              │              │
        Summary       Externalize      Isolation
          │              │              │
          └──────────────┼──────────────┘
                         ▼
                  High-Signal Context
```

因此 DeepAgents 的核心其实可以概括为：

> **Context Engineering + Agent Execution。**

---

# 17. Skills

Skills 是按需加载的专业能力。

例如：

```text
skills/
├── database/
│   └── SKILL.md
├── coding/
│   └── SKILL.md
├── research/
│   └── SKILL.md
└── finance/
    └── SKILL.md
```

Agent：

```text
任务
 ↓
判断需要什么 Skill
 ↓
加载 Skill
 ↓
执行
```

而不是：

```text
所有 Skill
 ↓
全部塞进 System Prompt
```

因此 Skills 同样属于：

> **Progressive Disclosure / 渐进式 Context Loading。**

---

# 18. Memory

DeepAgents 可以利用 LangGraph Store 等机制保存跨 Session 信息。

```text
Session A
   ↓
Agent
   ↓
Memory Store
   │
   │
Session B
   ↓
Agent
   ↓
Retrieve Memory
```

Memory 与 Context 不应该混淆：

```text
Context
= 当前任务为了完成任务需要看到的信息

Memory
= 跨任务 / 跨 Session 可以再次使用的信息
```

---

# 19. DeepAgents 架构

可以把 DeepAgents 简化成：

```text
                        Deep Agent
                            │
          ┌─────────────────┼─────────────────┐
          │                 │                 │
          ▼                 ▼                 ▼
       Planning          Context           Execution
          │              Management            │
          │                 │                  │
       Todo            ┌────┼────┐             │
                       │    │    │             │
                       ▼    ▼    ▼             ▼
                    Files  Sub  Skills       Tools
                          Agents
                       │
                       ▼
                    Memory
                       │
                       ▼
                LangChain Agent
                       │
                       ▼
                 LangGraph Runtime
```

---

# 20. 三者最核心的架构对比

| | LangChain | LangGraph | DeepAgents |
|---|---|---|---|
| 定位 | Framework | Runtime | Harness |
| 抽象层次 | 中 | 低 | 高 |
| 主要解决 | Agent 开发 | Agent 编排与运行 | 复杂 Agent |
| Model | ✓ | 可用 | ✓ |
| Tool | ✓ | 可用 | ✓ |
| Agent Loop | ✓ | 可自己实现 | ✓ |
| Middleware | ✓ | 可组合 | 大量使用 |
| State | 基础 Agent 状态 | 核心能力 | 使用 |
| Graph | 底层使用 | 核心抽象 | 使用 |
| Checkpoint | Runtime 提供 | ✓ | 使用 |
| Durable Execution | Runtime 提供 | ✓ | 使用 |
| Streaming | ✓ | ✓ | ✓ |
| HITL | ✓ / Middleware | ✓ | ✓ |
| Planning | 可自己实现 | 可自己实现 | 内置 |
| Subagents | 可自己实现 | 可自己实现 | 内置 |
| Filesystem | 可自己实现 | Backend 可扩展 | 内置 |
| Skills | 可自己实现 | 可自己实现 | 内置 |
| Context Management | Middleware | 自己设计 | 内置 |
| Memory | 可集成 | Store | 集成 |
| 控制力 | 中 | **最高** | 较低 |
| 开发速度 | 高 | 中 | **最高** |

---

# 21. 从工程视角理解三者

可以用三个问题记忆：

### LangChain

> **我需要哪些 Agent 开发积木？**

```text
Model
Tool
Prompt
Middleware
Agent
Integration
```

---

### LangGraph

> **这个 Agent Workflow 如何可靠运行？**

```text
State
Node
Edge
Checkpoint
Persistence
Retry
Resume
HITL
Streaming
```

---

### DeepAgents

> **怎样让 Agent 真正完成复杂、长期任务？**

```text
Planning
+
Subagents
+
Filesystem
+
Context Management
+
Skills
+
Memory
```

---

# 22. 三者组合后的完整架构

一个典型复杂 Agent：

```text
                           User
                            │
                            ▼
                    ┌───────────────┐
                    │  DeepAgents   │
                    │               │
                    │   Planner     │
                    │      │        │
                    │      ▼        │
                    │   Subagents   │
                    │      │        │
                    │      ▼        │
                    │  Context Mgmt │
                    │      │        │
                    │      ▼        │
                    │ Files / Memory│
                    └───────┬───────┘
                            │
                            ▼
                    ┌───────────────┐
                    │   LangChain   │
                    │               │
                    │ Model         │
                    │ Tools         │
                    │ Middleware    │
                    │ Agent         │
                    └───────┬───────┘
                            │
                            ▼
                    ┌───────────────┐
                    │   LangGraph   │
                    │               │
                    │ State         │
                    │ Graph         │
                    │ Checkpoint    │
                    │ Persistence   │
                    │ Durable Exec  │
                    │ Streaming     │
                    │ HITL          │
                    └───────┬───────┘
                            │
             ┌──────────────┼──────────────┐
             ▼              ▼              ▼
           LLM            Tools          MCP
```

---

# 23. 和你自己的 Agent 架构如何对应

如果设计：

```text
Planner
   ↓
Task DAG
   ↓
Task Center
   ↓
Executor
   ↓
Verifier
```

推荐理解为：

```text
                 Your Agent System
                        │
              ┌─────────┴─────────┐
              │                   │
       Orchestration         Agent Harness
              │                   │
          LangGraph            DeepAgents
              │                   │
       Task DAG / State       Planning
       Task Center            Context
       Executor               Subagents
       Verifier               Filesystem
       Retry                  Skills
       Scheduling             Memory
```

更进一步：

```text
                 System Level
                      │
                 LangGraph
                      │
       ┌──────────────┼──────────────┐
       ▼              ▼              ▼
    Planner       Task Center      Executor
                                      │
                                      ▼
                              DeepAgents Agent
                                      │
                       ┌──────────────┼──────────────┐
                       ▼              ▼              ▼
                    Subagent       Tools        Filesystem
                       │
                       ▼
                    Context
```

这样划分比较合理：

### LangGraph 负责

- 系统级任务编排
- State
- Task 生命周期
- Checkpoint
- Retry / Resume
- 长时间运行
- 人工介入
- Executor 调度

### DeepAgents 负责

- 单个复杂 Agent 如何工作
- Planning
- Context Engineering
- Subagent
- Filesystem
- Skills
- Memory
- Tool 使用

---

# 24. 为什么不直接全部使用 DeepAgents？

因为两者关注的问题不同。

例如你的系统：

```text
User
 ↓
Planner
 ↓
DAG
 ↓
Task Center
 ↓
Executor
 ↓
Verifier
```

这是一个**系统级 Orchestration**。

如果直接让 DeepAgents 管理所有东西：

```text
DeepAgents
    ↓
Everything
```

会失去：

- 任务中心的明确边界
- DAG 的可视化
- 自定义调度策略
- Executor 生命周期控制
- 系统级 Retry
- Temporal 迁移空间

因此更合理的是：

```text
LangGraph
    ↓
控制整个 Agent System

DeepAgents
    ↓
控制一个复杂 Agent 的内部执行
```

---

# 25. 最终记忆模型

建议直接记下面这张图：

```text
             ┌─────────────────────────┐
             │       DeepAgents        │
             │      Agent Harness      │
             │                         │
             │ Planning               │
             │ Subagents              │
             │ Filesystem             │
             │ Skills                 │
             │ Context Management     │
             │ Memory                 │
             └───────────┬─────────────┘
                         │
                         ▼
             ┌─────────────────────────┐
             │       LangChain         │
             │     Agent Framework     │
             │                         │
             │ Model / Tool / Prompt   │
             │ Middleware / Agent      │
             └───────────┬─────────────┘
                         │
                         ▼
             ┌─────────────────────────┐
             │       LangGraph         │
             │      Agent Runtime      │
             │                         │
             │ State / Graph           │
             │ Checkpoint              │
             │ Persistence             │
             │ Durable Execution       │
             │ Streaming / HITL        │
             └─────────────────────────┘
```

一句话：

> **LangChain 提供 Agent 的“开发抽象”，LangGraph 提供 Agent 的“运行时”，DeepAgents 提供复杂 Agent 的“能力与上下文工程”。**

---

# 26. 官方资料

- LangChain / LangGraph / DeepAgents 架构定位：
  https://www.langchain.com/blog/deep-agents-vs-langchain-vs-langgraph
- LangGraph Overview：
  https://docs.langchain.com/oss/python/langgraph/overview
- DeepAgents Overview：
  https://docs.langchain.com/oss/python/deepagents/overview
- DeepAgents Architecture：
  https://github.com/langchain-ai/deepagents/blob/main/libs/ARCHITECTURE.md
