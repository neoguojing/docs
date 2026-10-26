
# ManagerAgent：关键流程与业务数据

`ManagerAgent` 是推理模式中的**规划 Agent**。它每轮读取设备状态和任务进展，请求 LLM 输出下一步计划或最终答案；它**不直接执行手机动作**。父级 `MobileAgent` 根据 Manager 的结果决定交给 Executor，还是结束任务。

可以先记住这条边界：

```text
ManagerAgent：根据当前状态决定“下一步做什么”
ExecutorAgent：把子目标变成具体动作并执行
MobileAgent：编排 Manager 与 Executor，决定流程走向
```

## 一轮 Manager 流程

Manager 是 LlamaIndex `Workflow`，由 4 个 `@step` 组成，步骤间通过 Event 传递结果：

```text
StartEvent
  → prepare_context
  → ManagerContextEvent
  → get_response
  → ManagerResponseEvent
  → process_response
  → ManagerPlanDetailsEvent
  → finalize
  → StopEvent（返回计划/答案给父流程）
```

这几个 step 的实现位于 `manager_agent.py`，Manager 内部事件定义在 `manager/events.py`。

### 1. `prepare_context`：收集观察并准备本轮上下文

`prepare_context` 是每轮推理的输入准备阶段：

1. **按需截取截图**：视觉模式、截图流或轨迹记录开启时，向 driver 获取截图。截图获取失败时会记录 warning；设备断连则向上抛出。
2. **读取当前 UI**：调用 `state_provider.get_state()` 得到 `UIState`，并将它放入 `action_ctx.ui`，供后续动作读取。
3. **更新共享状态**：保存当前/上一轮的格式化设备状态，以及焦点文本、无障碍树、手机状态和当前应用信息。
4. **加载 App Card**：根据当前 app、任务目标和平台加载应用专属提示，结果保存到 `shared_state.app_card`。
5. **接入用户中途消息**：排空 `pending_user_messages` 队列，将消息附加到本轮 user message，并发布已应用事件。
6. **写入对话历史**：把本轮上下文消息追加到 `shared_state.message_history`，然后发出 `ManagerContextEvent`。

这里的截图有两个用途：可以发布给事件流供观察或记录；只有 Manager 配置启用了视觉，且存在截图时，截图才会作为图像块放进模型请求。

### 2. `get_response`：构造 prompt、调用并校验 LLM

`get_response` 调用 LLM 前会组装两部分内容：

- `_build_system_prompt`：填入任务指令、日期、App Card、平台、错误历史、自定义工具说明、可用凭据 ID、用户变量和可选输出 schema。
- `_build_messages_with_context`：建立 system message，复制已有对话历史，然后把记忆、当前设备状态和可选截图追加到最新 user message；上一轮设备状态附加到前一条 user message。

随后调用 `acall_with_retries` 请求 LLM。Manager 的响应需要符合 `parse_manager_response` / `validate_manager_response` 定义的结构，校验逻辑在 `manager/prompts.py`：

- 必须恰好有一个**非空 `<plan>`**，或恰好有一个**非空最终答案标签**。
- 最终答案标签可以是 `<request_accomplished>` 或 `<answer>`，并且必须有 `success="true"` 或 `success="false"`。
- 不允许同时给计划和最终答案、重复结果标签或错误嵌套的控制标签。
- 初始响应无效时，最多再请求 LLM 3 次。因此一次 Manager 响应校验流程最多尝试 **4 次**；仍无效就抛出 `ManagerResponseValidationError`，再由父级处理。

LLM 调用失败会被包装为 Manager 调用错误；用量提取失败则记 warning，不改变规划结果。

### 3. `process_response`：把模型文本转为业务状态

`process_response` 将 LLM 响应解析成结构化数据，然后更新共享状态：

- `<add_memory>` 内容追加到 `agent_memory`。
- 原始 assistant 响应追加到 `message_history`。
- 当前 `plan`、`current_subgoal`、`last_thought` 和 `answer` 写入 `MobileAgentState`。
- 若响应提供 `<progress_summary>`，则更新进度摘要。
- 生成 `ManagerPlanDetailsEvent`，其中包含计划、子目标、最终答案、成功状态和 memory update。

解析器会从 `<plan>` 第一行提取当前 subgoal，并去掉常见列表编号或 bullet 标记。若第一项是 `<script>`，则提取为带 `<script>` 标签的 subgoal。

### 4. `finalize`：返回本轮决策给父 Workflow

`finalize`（路径中的实际文件为 `manager_agent.py`）将 `ManagerPlanDetailsEvent` 包装为 `StopEvent`，返回 plan、subgoal、thought、answer 和 success 等数据。父级 `MobileAgent.run_manager` 等待这个子 Workflow 完成后，再包装为 `ManagerPlanEvent`，交给 `handle_manager_plan` 路由。

## 关键业务数据：是什么、做什么、为什么

| 数据 | 来源与用途 | 为什么需要 |
|---|---|---|
| `instruction` | 任务目标；进入 Manager 的 system prompt。 | 让模型每轮围绕原始目标判断进度和下一步。 |
| 当前/上一轮 UI 状态 | `StateProvider` 返回的 `UIState` 被写入 `formatted_device_state`、`previous_formatted_device_state` 等字段。 | 让模型基于当前屏幕规划，并能比较前后状态。 |
| `screenshot` | 从 `action_ctx.driver.screenshot()` 获取；视觉开启时作为 `ImageBlock` 输入 LLM。 | 辅助理解无障碍树不易表达的视觉界面。 |
| `plan` / `current_subgoal` | 从 `<plan>` 解析；`current_subgoal` 通常是计划的第一项。 | 把较大的用户目标拆成下一步可执行的任务，交给 Executor。 |
| `answer` / `success` | 从最终答案标签的正文和属性解析。 | 表示任务完成结果；父级据此结束或报告失败。 |
| `action_history` 等 | 父级在 Executor 执行后写入动作、摘要、结果与错误；Manager 可将近期错误放进 prompt。 | 让规划根据实际执行结果调整，避免忽略连续失败。 |
| `message_history` | 每轮追加 user 和 assistant `ChatMessage`；构建 prompt 时复制后注入当前上下文。 | 保持普通 Manager 的多轮对话连续性，同时避免 prompt 注入修改原始历史。 |
| `agent_memory` | 解析响应中的 `<add_memory>` 并追加；下一轮注入 prompt。 | 保存跨步骤仍有价值的信息，如从界面获得的姓名、日期或任务事实。它是当前共享状态中的工作记忆，不是持久化数据库。 |
| `app_card` | App Card Provider 根据当前应用、任务和平台加载。 | 为特定应用提供额外使用提示。 |
| `external_user_message` | `prepare_context` 从待处理队列取出并放入本轮上下文。 | 允许任务运行中接收新的说明或纠正。 |

共享业务状态定义在 `MobileAgentState`。Manager 通过同一个对象读写状态，因此 Planner 的输出和 Executor 的执行结果能够在父子 Workflow 之间衔接。

## App Card 的作用

Manager 根据配置创建 App Card Provider：禁用时不加载；启用时可选本地、服务端或组合模式。Provider 的选择和失败回退在 `_initialize_app_card_provider`，实际加载发生在 `_load_app_card`。

它提供的是**辅助提示上下文**，不会替代 UI 观察，也不负责工具执行。加载异常会被记录，当前实现会将 `app_card` 设为空字符串继续。

## Manager 如何决定下一步？

Manager 本身只产生“计划或最终答案”，不决定如何执行手机动作。父流程在 `MobileAgent.handle_manager_plan` 中分支：

```text
Manager 返回计划
  → 取 current_subgoal
  → Executor 执行一个动作
  → MobileAgent 更新动作历史
  → 下一轮 Manager 读取新 UI 和执行结果

Manager 返回最终答案
  → MobileAgent 生成 FinalizeEvent
  → 结束主任务
```

所以 Manager 的“决策”是对**下一子目标或完成状态**的决策；具体要调用 `click`、`type` 还是其他工具，由 Executor 根据 subgoal 选择。

## 使用的 LlamaIndex 功能

- `Workflow` 和 `@step`：把准备上下文、LLM 推理、响应处理和返回结果拆成事件驱动步骤。
- `StartEvent` / 自定义 `Event` / `StopEvent`：表达工作流输入、步骤间数据和本轮子 Workflow 完成。
- `Context` 与 `ctx.write_event_to_stream`：发布截图、UI、响应等过程事件，供父流程或观察者消费。
- `ChatMessage`、`TextBlock`、`ImageBlock`：构造文本和多模态 LLM 输入。
- `LLM` 接口：以统一方式调用不同 provider 的模型。

其中事件类定义在 `manager/events.py`，与跨 Agent 协调的事件不同；父级协调事件位于 `droid/events.py`。

## 设计边界与准确性

结构化响应解析和重试能降低“模型没有给出可路由计划/答案”的概率，但**不能证明计划本身正确或任务确实完成**。Manager 校验主要验证标签结构、数量和必需字段；它不会独立检查模型声称的事实是否真实，也不会操作设备确认任务成功。执行后的实际验证要依赖下一轮重新读取 UI，并让 Manager 根据新状态与 Executor 的动作结果继续判断。

此外，普通 `ManagerAgent` 会累积 `message_history`，而这段实现没有像 FastAgent 那样对其做固定消息数裁剪；记忆和设备状态也会进入 prompt。因此长任务仍可能遇到模型上下文窗口限制。stateless 变体的 prompt 构造策略不同，可对照 `stateless_manager_agent.py`。

**建议阅读顺序：** `prepare_context` → `_build_system_prompt` / `_build_messages_with_context` → `get_response` → `parse_manager_response` → `process_response` → 父级 `handle_manager_plan`。
