
可以，下面把 **A2A 权限安全**作为 Multi-Agent 文档的一部分合进去，并保持之前的紧凑结构。已去掉 LangGraph 和 A2A 历史。

# Multi-Agent 与 A2A 协议

## 1. 为什么需要 Multi-Agent

单 Agent 可以完成大量任务，但复杂系统容易出现 Context 膨胀、能力耦合、权限过大、任务无法并行等问题。

Multi-Agent 的核心是：

> **将复杂任务拆分为多个具有独立能力、Context、权限和运行环境的 Agent，并通过标准化机制进行协作。**

| 问题         | Multi-Agent 解决方式                   |
| ---------- | ---------------------------------- |
| Context 过大 | Agent 独立 Context                   |
| 能力过度集中     | 按职责拆分 Agent                        |
| 权限过大       | Agent 独立权限                         |
| 复杂任务耗时     | 多 Agent 并行                         |
| 专业能力不同     | Coding / Research / Test 等专用 Agent |
| Runtime 不同 | Agent 可以使用不同模型、框架和运行环境             |
| 独立部署       | Agent 可以独立部署和扩缩容                   |

典型结构：

```text
                         User
                          │
                          ▼
                 ┌─────────────────┐
                 │ Agent Console   │
                 │ / Supervisor    │
                 └────────┬────────┘
                          │
                    Task / Routing
                          │
          ┌───────────────┼───────────────┐
          ▼               ▼               ▼
    Coding Agent    Research Agent    Test Agent
          │
     Codex / Claude Code
```

---

## 2. A2A 是什么

**A2A（Agent2Agent）是一套 Agent ↔ Agent 的通信与协作协议。**

它解决：

> 一个 Agent 如何发现另一个 Agent、提交任务、获取任务状态以及获取任务结果。

例如：

```text
Supervisor
    │
    │ A2A
    ▼
Coding Agent
    │
    ├── Codex
    └── Claude Code
```

Supervisor 不需要知道 Coding Agent 内部使用什么模型、Prompt、Memory 或 Agent Framework。

---

## 3. A2A 核心对象

### Agent Card

描述 Agent 的身份和能力：

```text
Agent Card
├── Identity
├── Description
├── Capabilities
├── Skills
├── Endpoint
└── Authentication
```

用于 Agent Discovery 和能力描述。

### Message

Agent 之间的通信消息：

```text
Supervisor
    │
    │ "实现用户登录功能"
    ▼
Coding Agent
```

### Task

表示一个需要执行的任务：

```text
Task
├── id
├── status
├── input
└── output
```

任务可能经历：

```text
submitted → working → completed
                    ├→ failed
                    ├→ canceled
                    └→ input-required
```

### Artifact

表示任务产生的正式结果，例如：

```text
Artifact
├── source code
├── patch
├── test report
└── build result
```

简单区分：

```text
Message  = Agent 之间说什么
Task     = 要做什么 + 当前状态
Artifact = 最终产生什么
```

---

## 4. Agent 如何返回结果

### 同步

适合短任务：

```text
Supervisor
    │ request
    ▼
Agent
    │ response
    ▼
Supervisor
```

### Streaming

适合持续产生输出的任务：

```text
Supervisor
    │
    ▼
Agent
 ├── event
 ├── event
 ├── event
 └── completed
```

### Push Notification

适合长任务：

```text
Supervisor
    │ create task
    ▼
Agent
    │
    │ 长时间执行
    │
    └──────► Notification
                  │
                  ▼
             Supervisor
                  │
             Query Task
                  │
             Get Artifact
```

Notification 可以只表示：

```text
task_id = 123
status = completed
```

Supervisor 再获取完整 Task / Artifact。

---

## 5. A2A 权限与敏感信息

A2A 不会自动赋予 Agent 权限。核心是：

```text
A2A 权限安全
│
├── 谁？
│   └── Authentication / Identity
│
├── 能干什么？
│   └── Authorization / OAuth Scope
│
├── 能访问什么？
│   └── Task / Artifact Access Control
│
└── Secret？
    └── Out-of-band / Agent-bound Credential
```

### 5.1 谁？—— Authentication / Identity

首先确认：

> 谁在调用 Agent？

例如：

```text
Supervisor
   │
   │ Bearer Token
   ▼
Coding Agent
   │
   └── Authentication
          ↓
       Identity
```

Server 根据身份决定是否接受请求。

---

### 5.2 能干什么？—— Authorization / OAuth Scope

身份认证之后，还要判断：

> 这个 Agent 被允许执行什么操作？

例如：

```text
Coding Agent

Scope:
repo:read
repo:write
```

允许：

```text
✓ 读取代码
✓ 修改代码
```

不代表允许：

```text
✗ 删除仓库
✗ 访问 production-db
```

即：

```text
Identity
   ↓
OAuth Scope / Authorization
   ↓
Allow / Deny
```

---

### 5.3 能访问什么？—— Task / Artifact Access Control

Agent 不能因为知道 `task_id` 就自动获得 Task。

例如：

```text
Task #123
owner = Supervisor-A

Artifact #456
task = #123
```

Agent-B 请求：

```text
Get Task #123
       │
       ▼
Authorization
       │
       ├── No  → Permission Denied
       │
       └── Yes → 返回 Task / Artifact
```

因此 Task、Artifact 都需要访问控制。

---

### 5.4 Secret？—— Credential Isolation

例如 Coding Agent 需要 GitHub Token。

不要：

```text
Supervisor
    │
    │ A2A
    │
    ├── github_token=xxxx
    ▼
Coding Agent
```

而是通过安全的带外机制获取 Credential：

```text
Supervisor
    │
    │ A2A
    ▼
Coding Agent
    │
    │ Secure Credential
    ▼
GitHub
```

如果 Credential 必须经过 Agent 链传递，应限制 Credential 的使用范围，并避免其他 Agent 获得可读取的 Secret。

核心原则：

> **Agent 获取的是被授权的能力，而不是任意 Secret。**

---

## 6. A2A 与 MCP

两者解决不同问题：

```text
                    Agent
                   /     \
                  /       \
                A2A       MCP
                 │          │
                 ▼          ▼
              Agent      Tool / Resource
```

**A2A：**

```text
Agent ↔ Agent
```

例如：

```text
Supervisor
    ↓ A2A
Coding Agent
```

**MCP：**

```text
Agent ↔ Tool / Resource
```

例如：

```text
Coding Agent
    ↓ MCP
GitHub / Browser / Database / Filesystem
```

因此：

```text
A2A = Agent 网络
MCP = Tool / Resource 网络
```

---

## 7. Supervisor + Codex / Claude Code

实际的 Agent Console 可以设计成：

```text
                         User
                          │
                          ▼
                 ┌─────────────────┐
                 │ Agent Console   │
                 │ Supervisor      │
                 └────────┬────────┘
                          │
                    Task Management
                          │
              ┌───────────┼───────────┐
              │           │           │
             A2A         A2A         A2A
              │           │           │
              ▼           ▼           ▼
        Coding Agent  Research    Test Agent
              │
        ┌─────┴─────┐
        ▼           ▼
      Codex     Claude Code
        │           │
        └─────┬─────┘
              ▼
             MCP
              │
       ┌──────┼──────┐
       ▼      ▼      ▼
    GitHub  Browser  Filesystem
```

Codex / Claude Code 不一定直接实现 A2A。

可以通过 Coding Agent 进行封装：

```text
A2A Server
    │
    ▼
Coding Agent
    │
    ├── Codex Adapter
    └── Claude Code Adapter
```

这样 Supervisor 只需要面对统一的 Coding Agent。

---

## 8. 整体模型

最终可以把 Multi-Agent 系统理解成：

```text
                         Supervisor
                             │
                    ┌────────┴────────┐
                    │                 │
                   A2A               MCP
                    │                 │
                    ▼                 ▼
              Agent Network     Tool / Resource
                    │                 │
          ┌─────────┼─────────┐       │
          ▼         ▼         ▼       │
       Coding    Research    Test     │
          │                           │
          └───────────────────────────┘
```

其中安全边界贯穿 Agent 间和 Agent 与资源之间：

```text
Authentication → 谁
Authorization  → 能干什么
Access Control → 能访问什么
Credential     → Secret 怎么保护
```

**核心关系：**

> **Multi-Agent 解决能力与任务拆分；A2A 解决 Agent 间标准化协作；MCP 解决 Agent 与 Tool / Resource 的连接；Authentication / Authorization / Access Control / Credential Isolation 负责安全边界。**
