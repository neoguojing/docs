Search for `^(class |    @step|    async def |    def |@dataclass|[A-Z_]+ = )`

Read , lines 1 to 180
Completed with input: {
  "path": "/workspaces/mobilerun/mobilerun/agent/droid/droid_agent.py",
  "view_range": [
    1,
    180
  ]
}

Search for `from llama_index|Context|StartEvent|StopEvent|Workflow|WorkflowHandler|@step|...`

Ran terminal command: git status --short

Read , lines 261 to 520
Completed with input: {
  "path": "/workspaces/mobilerun/mobilerun/agent/droid/droid_agent.py",
  "view_range": [
    261,
    520
  ]
}

Read , lines 521 to 665
Completed with input: {
  "path": "/workspaces/mobilerun/mobilerun/agent/droid/droid_agent.py",
  "view_range": [
    521,
    665
  ]
}

Read , lines 665 to 910
Completed with input: {
  "path": "/workspaces/mobilerun/mobilerun/agent/droid/droid_agent.py",
  "view_range": [
    665,
    910
  ]
}

Read , lines 923 to 1140
Completed with input: {
  "path": "/workspaces/mobilerun/mobilerun/agent/droid/droid_agent.py",
  "view_range": [
    923,
    1140
  ]
}

Read , lines 1140 to 1365
Completed with input: {
  "path": "/workspaces/mobilerun/mobilerun/agent/droid/droid_agent.py",
  "view_range": [
    1140,
    1365
  ]
}

Read , lines 1110 to 1140
Completed with input: {
  "path": "/workspaces/mobilerun/mobilerun/agent/droid/droid_agent.py",
  "view_range": [
    1110,
    1140
  ]
}

Read , lines 1340 to 1365
Completed with input: {
  "path": "/workspaces/mobilerun/mobilerun/agent/droid/droid_agent.py",
  "view_range": [
    1340,
    1365
  ]
}

Read , lines 465 to 523
Completed with input: {
  "path": "/workspaces/mobilerun/mobilerun/agent/droid/droid_agent.py",
  "view_range": [
    465,
    523
  ]
}

Read , lines 820 to 910
Completed with input: {
  "path": "/workspaces/mobilerun/mobilerun/agent/droid/droid_agent.py",
  "view_range": [
    820,
    910
  ]
}

Search for `^class (FastAgentExecuteEvent|FastAgentResultEvent|ManagerInputEvent|ManagerP...`

## 总览

`MobileAgent` 是整个任务的**顶层编排 Workflow**：它组装设备、状态读取器和工具，然后根据配置选择执行路线：

- `reasoning=False`：由 `FastAgent` 自己完成观察、推理、调用工具和循环。
- `reasoning=True`：由 Manager 规划子目标，Executor 执行单个动作，结果再交回 Manager。
- 配置使用外部 Agent 时：`start_handler` 走外部 Agent 分支，不走内置的 FastAgent 或 Manager/Executor 流程。

因此，`droid_agent.py` 的重点不是实现某一个具体动作，而是**搭起运行环境、路由工作流事件、协调子 Agent 并处理最终收尾**。

## 关键类：是什么、为什么需要

| 类 | 是什么 | 为什么需要 |
|---|---|---|
| `MobileAgent` | 继承 LlamaIndex `Workflow` 的顶层任务编排器。 | 把配置、设备连接、工具、子 Agent 和任务生命周期放在统一入口管理。 |
| `MobileAgentState` | 父子 Agent 共享的 Pydantic 状态对象，保存目标、设备状态、计划、动作历史、记忆和完成状态。 | 让 Manager 和 Executor 能在不同工作流步骤间共享任务进度。 |
| `FastAgent` | 直接执行模式的子 Workflow。 | 适合不需要单独高层规划器的任务，由一个 Agent 完成工具调用循环。 |
| `ManagerAgent` / `StatelessManagerAgent` | 推理模式的规划器。 | 分析任务进度，决定下一子目标，或判断是否可以结束。 |
| `ExecutorAgent` | 推理模式下执行具体动作的子 Workflow。 | 把 Manager 的子目标转换为一次工具动作，避免将高层规划和底层操作混在一起。 |
| `ActionContext` | 动作函数需要的依赖集合，如已连接的 driver、当前 UI、共享状态和 StateProvider。 | 工具函数通过统一接口取得设备能力，不需要各自重新连接或查找设备。 |
| `StateProvider` / `UIState` | 分别负责读取、规范化设备观察，以及表示一次 UI 快照。 | Agent 使用统一的 UI 数据结构，不必直接处理各平台的原始界面数据。 |
| `ToolRegistry` | 工具的注册、过滤、签名生成和执行分发器。 | 让内置、自定义和 MCP 工具共用一套调用入口。 |
| `droid/events.py` 等事件类 | 描述任务流程的输入、计划、动作结果和结束信号。 | 告诉 Workflow 当前发生了什么，以及下一步应路由到哪里。 |

## 关键流程

### 1. 初始化和启动：`__init__` → `run` → `start_handler`

构造函数创建 `MobileAgentState`，解析配置、凭据、prompt 和 LLM，并按配置选择创建 Manager/Executor 子 Agent。此时部分设备依赖还没建立；之后由 `start_handler` 连接并注入。

`run()` 先应用 tracing session context，再调用父类 `Workflow.run()`。入口见 `MobileAgent.run`。

`start_handler` 是启动步骤，主要做以下工作：

1. 初始化本次运行的 workflow deadline 和可选 trajectory。
2. 如果配置了外部 Agent，加载并运行它。
3. 根据设备配置选择并连接 driver；Android 未指定序列号时解析设备 serial。
4. 按平台和运行模式创建 StateProvider。
5. 创建内置工具注册表，加入自定义/MCP 工具，并按设备能力和配置过滤。
6. 创建 `ActionContext`，将 driver、provider、共享状态等依赖注入 Manager/Executor。
7. 记录初始化遥测，然后发出 `FastAgentExecuteEvent` 或 `ManagerInputEvent`，决定运行模式。

对应入口是 `MobileAgent.start_handler`。这是“依赖装配和模式分流”集中的地方。

### 2. 直接执行模式：FastAgent Loop

`start_handler` 发出 `FastAgentExecuteEvent` 后，`execute_task` 创建 FastAgent 并运行它。FastAgent 内部负责逐轮观察、请求 LLM、解析工具调用、执行并回传结果。

MobileAgent 会转发 FastAgent 的子流程事件。收到 `FastAgentResultEvent` 后，`handle_fast_agent_result` 转成 `FinalizeEvent`，进入收尾。

### 3. 规划模式：Manager → Executor → Manager

这是与用户选中的 `run_manager` 直接相关的主循环：

1. **Manager 阶段：** `run_manager` 检查 `max_steps`，增加步数，运行 Manager 并收集它的事件和结果。
2. **路由判断：** `handle_manager_plan` 检查 Manager 输出：有最终答案则进入结束流程；有子目标则发送 `ExecutorInputEvent`。
3. **动作执行：** `run_executor` 运行 Executor，接收动作结果，并将动作、成功状态、摘要和错误写进共享状态。
4. **回到规划：** `handle_executor_result` 检查近期连续失败情况，并返回新的 `ManagerInputEvent`，让 Manager 根据最新状态重新规划。

可简化为：

```text
ManagerInputEvent
    ↓
run_manager → Manager 输出计划/答案
    ↓
handle_manager_plan
    ├─ 有最终答案 → FinalizeEvent
    └─ 有子目标 → ExecutorInputEvent
                       ↓
                run_executor → 更新共享动作历史
                       ↓
                handle_executor_result
                       └─ ManagerInputEvent（开始下一轮）
```

为什么拆成两层：Manager 关注任务级计划和整体进展，Executor 关注本步动作及参数；父级 Workflow 负责控制二者的交接和停止条件。

### 4. 最终收尾：`finalize`

`finalize` 创建最终 `ResultEvent`，并在总时间预算内尽力执行收尾操作：

- 将共享状态标记为已完成，并发布最终事件。
- 若配置了结构化输出，运行 `StructuredOutputAgent` 提取结果。
- 刷新遥测；按条件抓取最终截图和 UI 状态。
- 保存 trajectory，停止写入器，并断开 MCP 连接。
- 将最终结果返回调用方。

这里的预算控制是为了避免截图、遥测或清理操作拖住任务结束。

## 利用了 LlamaIndex Workflow 的哪些功能？

| 功能 | 在该文件中的用法 | 实际作用 |
|---|---|---|
| `Workflow` | `MobileAgent(Workflow)` | 将任务组织为事件驱动的工作流，而不是一段单体顺序函数。 |
| `@step` | 装饰 `start_handler`、`run_manager`、`run_executor`、`finalize` 等方法 | 将方法声明为工作流步骤；步骤接收事件，并返回事件供后续步骤路由。 |
| `StartEvent`、`Event` 子类、`StopEvent` | 启动步骤接收 `StartEvent`，步骤之间传递 `ManagerInputEvent`、`ExecutorInputEvent` 等，结果以 `ResultEvent(StopEvent)` 表示结束。 | 用明确的数据类型表达启动、分支、结果和终止。事件的定义可见 `droid/events.py`。 |
| `Context` 和 `ctx.store` | `start_handler` 将 workflow deadline 存进 `Context.store`；`finalize` 再读取。 | 保存本次 Workflow 运行期间的上下文数据，不需要把所有控制数据塞进共享 Agent 状态。 |
| `ctx.write_event_to_stream` | 发布启动、截图、UI 状态、工具结果等事件。 | 允许调用方或 CLI 观察正在发生的事情。 |
| `WorkflowHandler`、`stream_events()` | 用 `agent.run()` 取得 handler，迭代 `handler.stream_events()`，最后 `await handler` 取得结果。 | 同时支持子 Agent 的过程事件流和最终结果。 |
| 子 Workflow | 运行 FastAgent、Manager、Executor，并转发其事件。 | 复用各 Agent 自己的步骤流程，同时由 MobileAgent 统一协调父子工作流。 |

注意：`MobileAgentState` 是任务业务状态，`Context.store` 更偏 Workflow 单次运行中的辅助数据，而 events 负责步骤路由和事件流。三者目的不同，不能互相替代。

## 如何理解这个文件的边界

`droid_agent.py` 负责**组织和协调**，但不是所有逻辑的实现位置：

- prompt 怎么构造、LLM 怎样回答：看 `manager_agent.py`、`executor_agent.py` 和 `fast_agent.py`。
- 工具怎样执行：看 `ToolRegistry` 和 `actions.py`。
- 具体 Android/iOS driver 实现来自 `mobilerun-core-local` 依赖；本文件负责选择、连接和注入它。
- 内部 event 被转发和收录用于观测/轨迹的逻辑在 `handle_stream_event`。

总体可以记成：**Agent 决策由子 Agent 完成；MobileAgent 决定流程路由；ToolRegistry 负责工具分发；ActionContext 把工具连接到当前设备；LlamaIndex Workflow 提供步骤编排、事件路由和流式观察。**
