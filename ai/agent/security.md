可以。核心问题是上一版把 **Identity / Authorization / Capability / Resource / Sandbox / Audit** 平铺展开了，导致概念很多但主线不明显。

建议压缩成一条主线：

> **Agent 要安全，本质就是：谁（Identity）通过什么能力（Tool/MCP）对什么资源（Resource）执行什么操作（Action），由 Policy 决定是否允许，再由 Sandbox 限制最大影响范围。**

下面按这个逻辑重构，FilesystemPermission 只作为一个小例子。

# Agent 安全与权限体系设计

## 1. 核心抽象

Agent 安全可以统一抽象为：

```text
Principal
   ↓
Capability
   ↓
Action
   ↓
Resource
   ↓
Policy → Allow / Deny / Interrupt
   ↓
Sandbox / Security Boundary
   ↓
Audit
```

其中：

| 概念             | 含义          | 示例                              |
| -------------- | ----------- | ------------------------------- |
| **Principal**  | 谁在操作        | User / Agent / Run              |
| **Capability** | 通过什么能力访问    | Tool / MCP / API / Shell        |
| **Action**     | 做什么         | read / write / delete / execute |
| **Resource**   | 操作什么        | File / DB / API / Secret        |
| **Context**    | 在什么条件下      | Project / Env / Time            |
| **Policy**     | 是否允许        | allow / deny / interrupt        |
| **Sandbox**    | 即使执行，最多影响什么 | FS / Network / Process          |

核心授权模型：

```text
Authorize(
    Principal,
    Capability,
    Action,
    Resource,
    Context
) → Allow / Deny / Interrupt
```

**关键点：Tool / MCP 不是 Resource，而是 Capability / Access Path。**

> Resource = 操作对象
> Capability = 访问资源的能力/路径

---

# 2. Agent 安全主要解决什么

实际上可以归纳成 4 个问题：

### 2.1 Identity：谁在操作？

需要区分：

```text
User
  ↓
Agent
  ↓
Run
  ↓
Tool / MCP / API
```

例如：

```text
User = neo
Agent = coding-agent
Run = run-123
```

工具调用必须携带身份上下文，方便后续：

* 权限判断
* 凭证隔离
* 审计追踪

---

### 2.2 Least Privilege：允许做什么？

权限不能简单定义为：

```text
Agent = 可以访问文件
```

而应该定义为：

```text
谁
对什么 Resource
执行什么 Action
在什么 Context
可以做什么
```

例如 Coding Agent：

```text
/workspace/**       read + write
repo-A              git write
database-A          query
production          deny
shell                sandbox only
```

原则：

```text
Effective Permission
= User Permission
  ∩ Agent Policy
  ∩ Resource Policy
  ∩ Environment Boundary
```

---

### 2.3 Permission Bypass：能不能绕过权限？

这是 Agent 安全中特别重要的问题。

假设：

```text
File Tool
    ↓
Policy
    ↓
禁止写 /etc/*
```

但 Agent 还有：

```text
Shell
Python
MCP
Custom Tool
```

于是：

```text
write_file("/etc/a") → Deny

shell("echo x > /etc/a") → Allow
```

这就是**权限穿透 / Permission Bypass**。

本质：

> **同一个 Resource 存在多个 Access Path，但这些路径没有遵守同一套安全策略。**

所以不能只保护某一个 Tool。

---

# 3. 如何解决权限穿透

核心方案只有两个层次。

## 3.1 统一 Policy Gateway

所有高风险能力进入统一授权层：

```text
                ┌─ Tool
Agent ─ Policy ─┼─ MCP
                ├─ API
                └─ Code / Shell
```

统一判断：

```text
Principal
+ Capability
+ Action
+ Resource
+ Context
        ↓
     Policy
        ↓
 Allow / Deny / Interrupt
```

这样可以避免：

```text
File Tool 有权限控制
MCP 没有
Shell 没有
Custom Tool 没有
```

---

## 3.2 Resource 侧再次限制

不能只相信 Agent 层的 Policy。

还应该让 Resource 自己具备安全边界：

```text
Agent Policy
      ↓
Tool Gateway
      ↓
Sandbox / Resource ACL
      ↓
Resource
```

例如：

* DB：数据库账号权限
* API：OAuth Scope / API Scope
* 文件：Filesystem ACL
* Secret：Secret Scope
* Shell：Sandbox
* Production：网络隔离

这样即使 Agent 层策略出现 Bug，底层 Resource 仍然有保护。

---

# 4. Policy：权限规则如何表达

Policy 可以采用：

* RBAC
* ABAC
* ACL
* Capability-based Authorization
* Declarative Policy

不需要绑定某一种实现。

例如一个通用规则：

```text
{
    principal: "coding-agent",
    action: "write",
    resource: "/workspace/**",
    mode: "allow"
}
```

或者：

```text
{
    principal: "coding-agent",
    action: "write",
    resource: "/production/**",
    mode: "deny"
}
```

Policy Engine 最终解决：

```text
Can this Principal
perform this Action
on this Resource
under this Context?
```

---

# 5. FilesystemPermission：一个具体例子

`FilesystemPermission` 可以看作上述 Policy 在**文件系统 Resource**上的一个具体实现。

例如：

```text
operations:
    read / write

paths:
    /workspace/**
    /etc/**

mode:
    allow / deny / interrupt
```

例如：

```text
read  /workspace/** → allow
write /workspace/** → allow

write /etc/**       → deny

write /important/** → interrupt
```

规则通常按照声明顺序匹配：

```text
Rule 1
  ↓
匹配 → 立即决定
  ↓
不匹配
  ↓
Rule 2
```

即：

> **first-match-wins**

但这里需要注意：

**FilesystemPermission 只解决 File Tool 这一类访问路径的问题。**

如果 Agent 还有：

```text
Shell
Python
MCP
Custom Tool
```

这些路径仍然可能访问文件。

所以它不是完整的 Agent Security，而只是：

```text
Policy
  └── Filesystem Policy
        └── FilesystemPermission
```

---

# 6. Policy 与 Sandbox 的区别

两者解决不同问题。

### Policy

回答：

> **“允不允许做？”**

例如：

```text
write /production/db → deny
```

### Sandbox

回答：

> **“即使执行了，最多能造成多大影响？”**

例如：

```text
Filesystem → /workspace
Network    → deny production
Process    → restricted
CPU        → 2 cores
Memory     → 4 GB
```

因此：

```text
Policy = 逻辑安全边界
Sandbox = 运行时安全边界
```

二者应该同时存在。

---

# 7. 最终架构

完整 Agent Security 可以压缩成：

```text
User
 ↓
Identity
 ↓
Agent / Run
 ↓
Policy Engine
 ↓
Tool Gateway
 ├── Tool
 ├── MCP
 ├── API
 └── Code / Shell
 ↓
Sandbox / Resource ACL
 ↓
Resource
 ↓
Audit
```

对应关系：

```text
Identity   → 谁
Policy     → 能不能做
Capability → 通过什么做
Action     → 做什么
Resource   → 对什么做
Sandbox    → 最多影响什么
Audit      → 谁做过什么
```

## 8. 面试时一句话总结

> **Agent 安全的核心不是给 Agent 配一组简单的文件权限，而是建立统一的 Identity、Policy 和 Capability 模型：明确谁通过什么访问路径，对什么 Resource 执行什么 Action；Policy 决定是否允许，Sandbox 限制最大影响范围，同时在 Resource 侧再次做权限控制，避免通过 Shell、MCP、API 等其他路径绕过权限。**

其中：

```text
FilesystemPermission
        ↓
只是一个具体的 Policy 实现
        ↓
不是 Agent Security 的整体模型
```

这一版可以作为后续 **Agent Harness / Security** 文档的主干，基本只保留一条逻辑链：**Identity → Policy → Capability → Resource → Sandbox**，其他概念都挂在这条线上。
