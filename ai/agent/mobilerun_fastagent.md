## 总览

`FastAgent` 是直接执行模式下的 Agent Workflow。它采用类似 ReAct 的循环：**读取设备观察 → 请求 LLM → 解析 Tool Call → 执行工具 → 将结果反馈给 LLM → 再观察，直到完成或触发停止条件**。

它不是让模型直接操作手机：模型只生成工具调用文本，程序负责解析、分发和执行。这个设计把模型的决策与设备操作隔开，便于记录失败、拒绝无效调用并控制执行顺序。

## 1. 用到的主要技术

| 技术/组件 | FastAgent 中的用途 |
|---|---|
| LlamaIndex Workflow | `FastAgent(Workflow)`，用 `@step` 将对话、解析、工具执行拆成事件驱动的步骤。 |
| 异步 Python | `async`/`await` 处理 LLM 请求、截图、UI 状态读取和设备动作；工具调用按顺序执行。 |
| LlamaIndex `ChatMessage` | 管理 system/user/assistant 消息，并用 `TextBlock`、`ImageBlock` 保留文本、图像等多模态内容。 |
| XML 协议、`ElementTree`、正则 | 将模型输出中的 `<function_calls>` 解析为结构化工具调用；XML 不合法时不执行该调用。 |
| 共享状态与 Workflow Context | `MobileAgentState` 保存任务级状态和历史；`Context.store` 暂存本轮待执行工具调用等 Workflow 数据。 |
| Jinja2 prompt | 渲染工具说明、设备平台、可用凭据 ID、视觉模式、用户变量等提示内容。 |
| 重试、错误归一 | LLM 请求经 `acall_with_retries`；工具运行结果统一为 `ActionResult`。 |
| tracing 与用量统计 | 可记录截图 tracing、LLM token usage 和 Workflow events；它们用于观测，不负责判断任务是否语义正确。 |

## 2. 执行流程：从启动到下一轮

FastAgent 内部共有 6 个 `@step`。它们主要靠输入和返回的 Event 类型形成控制流。

```text
StartEvent
  → prepare_chat
  → FastAgentInputEvent
  → handle_llm_input
  → FastAgentResponseEvent
  → handle_llm_output
      ├─ FastAgentToolCallEvent → execute_code → FastAgentOutputEvent
      │                                       → handle_execution_result
      │                                       → FastAgentInputEvent（循环）
      ├─ FastAgentInputEvent（要求模型重试/补充）
      └─ FastAgentEndEvent → finalize → StopEvent
```

### A. 初始化会话：`prepare_chat`

`prepare_chat`：

- 加载可用凭据的 **ID**，但不是凭据值。
- 构造 system prompt 和初始 user prompt。
- 清空 `MobileAgentState.message_history`，加入本任务目标。
- 发出 `FastAgentInputEvent`，开始第一轮。

system prompt 来源于 `config/prompts/fast_agent/system.jinja2`，也可以被自定义 prompt 覆盖。它会向模型说明工具列表、XML 格式、设备上下文和完成规则。

### B. 观察设备并请求 LLM：`handle_llm_input`

`handle_llm_input` 每轮首先检查 `max_steps`，未达到时递增 `MobileAgentState.step_number`，再读取截图和设备 UI。

UI 由 `state_provider.get_state()` 生成 `UIState`，更新共享状态中的当前/上一轮状态、焦点文本、无障碍树和应用信息；同时将最新 UI 放到 `action_ctx.ui`，供后续动作函数定位元素。

发给 LLM 的消息大致由以下内容组成：

```text
system prompt（工具协议、工具签名和规则）
+ 有限对话历史
+ 当前 memory
+ 当前 device_state
+ 可选 screenshot
+ 前一轮 device_state（附加到上一条 user message）
```

历史通过 `limit_history` 控制为最多 `LLM_HISTORY_LIMIT * 2` 条消息，即默认 200 条；模型输入是历史的深拷贝，因此注入本轮设备状态和截图不会改写原始历史。截图会按模型能力和坐标约定进行缩放。

之后通过 `acall_with_retries` 请求 LLM，并调用 `parse_tool_calls_detailed` 解析回答。解析结果区分三种状态：合法工具调用、没有工具调用、工具调用格式错误。

### C. 判断响应怎么处理：`handle_llm_output`

`handle_llm_output` 依据解析状态分支：

- **格式错误**：追加修正提示，让模型重新输出；连续错误达到 3 次后失败结束，避免无限重试。
- **没有工具调用**：要求模型补充工具调用，或使用 `complete` 明确结束任务。
- **有工具调用**：继续到 `execute_code`。
- 如果回答没有 thought，代码会向对话追加提示；这属于 prompting 约束，不是数学或语义上的正确性证明。

只有第一个 `<function_calls>` 块会被解析执行；之后的工具块会被忽略。保存 assistant 消息时，会截断第一个调用块之后的文本，避免下一轮把被忽略的内容当成执行事实重新送回模型。同时保留 `ChatMessage` 中的 thinking/non-text blocks 和附加元数据。

如果 thought 中包含 `<add_memory>`，则提取内容并追加到共享 `agent_memory`。待执行的解析结果放进 Workflow 的 `ctx.store`，通过 `FastAgentResponseEvent` 继续传递。

### D. 执行 Tool Call：`execute_code`

`execute_code` 从 `ctx.store` 取出已解析的调用，再按顺序逐一执行：

1. 如果该调用参数解析失败，不调用工具，构造失败结果。
2. 否则交给 `ToolRegistry.execute`。
3. Registry 按名字找到函数，并将 `ActionContext` 作为 `ctx` 参数传入。
4. 工具返回 `ActionResult`，例如 `success` 和 `summary`。
5. 结果格式化成 `<function_results>`，供下一轮模型参考。

例如 `click(index)` 会通过 `ctx.ui` 查找元素坐标，再调用 `ctx.driver.tap(...)`；具体动作实现见 `actions.py`。

Prompt 配置中的 `parallel_tools` 控制模型是否被鼓励在同一个 XML 块里提出多个调用，但当前代码是**按顺序执行这些调用，不是并行执行**。如果某个动作失败，后续调用会跳过，因为它们可能依赖前面动作成功。

如果执行的是 `complete`，它会通过 `MobileAgentState.complete` 设置完成状态。若此时有等待处理的外部用户消息，Agent 会清除完成标记、把消息和结果交给下一轮继续处理；否则发出 `FastAgentEndEvent`。

### E. 回填结果并循环：`handle_execution_result`

`handle_execution_result` 将工具结果作为新的 user message 追加到对话历史。等待中的用户消息也会在此时附加并排空队列，然后发出新的 `FastAgentInputEvent`。

因此，普通工具执行后的动作不是立刻结束，而是重新观察设备并让模型根据**新 UI + 工具结果**做下一步决定。

### F. 结束：`finalize`

`finalize` 清除共享状态中的 `finished` 标记、发出结束事件，并返回 `success`、`reason` 和 `tool_call_count`。

## 3. 关键业务状态

状态定义在 `MobileAgentState`。这里几个计数和字段用途不同：

| 字段 | 含义 |
|---|---|
| `step_number` | Agent 的观察/推理轮数，受 `max_steps` 限制；不是工具调用数。 |
| `tool_call_counter` | FastAgent 解析并尝试处理的调用数量；调用前递增，所以参数解析失败的调用也会计入，不代表成功调用数。 |
| `message_history` | 当前任务的聊天记录，包括模型响应和工具结果；FastAgent 开始时会清空。 |
| `agent_memory` | 从 `<add_memory>` 得到的追加式工作记忆，供后续 prompt 使用。当前没有长度上限或跨任务持久化。 |
| `formatted_device_state` / `previous_formatted_device_state` | 当前与上一轮界面文本，帮助模型比较变化。 |
| `finished` / `success` / `answer` | 由 `complete` 工具设置，供 FastAgent 判断是否结束。 |
| `pending_user_messages` | 运行中到达的新指令队列，在安全的轮次边界注入，并在任务结束或达到上限时处理/丢弃。 |

`Context.store` 用于 Workflow 内部传递本轮待执行工具调用；它不是任务记忆。FastAgent 的内部事件定义在 `fast_agent/events.py`。

## 4. Tool 协议和校验机制

工具说明来自 `ToolRegistry.get_tool_descriptions_xml`，参数类型来自 `get_param_types`。FastAgent 在构造时取得这两份数据，用于生成 prompt 和解析参数。

XML parser `xml_parser.py` 会：

- 检查工具调用块是否存在、是否完整，区分 `NO_MARKUP` 和 `MALFORMED`。
- 解析 XML 结构，只接受 `<invoke>` 和 `<parameter>` 的预期结构；校验工具名、参数标签和参数名是否存在。
- 把数字、列表等参数转换为对应类型；不能转换时将该 Tool Call 标记为参数错误，不执行它。
- 只执行第一个工具块，忽略后续块；结果中会提示模型被忽略了多少块。
- 保留参数中的普通文本内容，对 XML 特殊字符进行转义，以免参数值里的 `<`、`>`、`&` 破坏 XML 解析。

`ToolRegistry.execute` 进一步处理未知工具、同步/异步函数和运行异常，将结果统一转为 `ActionResult`，并可发布 `ToolExecutionEvent`。工具能力也会在运行时按 driver/provider 支持情况过滤。

## 5. 哪些机制提升准确性？它们的边界是什么？

可以把准确性拆成**调用格式正确、工具实际执行、任务语义正确**三层：

### 调用与执行层面的保护

- **明确 prompt 合同**：告诉模型工具名、参数、XML 结构、何时观察、何时结束；可用工具由 registry 动态提供。
- **结构化解析**：格式非法时不执行；数据类型不符合要求时生成参数错误结果。
- **防止尾随内容伪装执行结果**：只执行首个调用块；截断之后的文本和调用块，避免未执行的内容进入下一轮。
- **失败短路**：某次动作失败后跳过同一批之后的调用，防止继续执行基于失败前提的动作。
- **设备/UI 防护**：基于当前 `UIState` 的元素索引定位；坐标操作通过 UI 的坐标转换和契约检查；具体动作失败作为错误结果返回，而不是假设成功。
- **有限重试和上限**：LLM 请求重试由 inference helper 处理；格式错误最多连续纠正 3 次；总观察轮数由 `max_steps` 限制。
- **结果反馈闭环**：动作结果回到对话，下一轮重新观察实际 UI。它有机会发现前一步效果与预期不同。

相关实现可看 `xml_parser.py`、`actions.py` 和 `ToolRegistry`。

### 不能保证的部分

这些保护**不等于任务结果绝对正确**：

- XML 格式正确，只能说明调用结构能解析，不证明模型选择的目标、参数或业务判断正确。
- 工具返回 `success=True`，通常表示动作函数认为调用成功，不一定证明用户目标最终达成。比如点击成功不代表目标页面一定正确。
- Prompt 会要求完成后调用 `complete`，但 `complete` 的 `success` 和 `message` 仍由模型提供；代码没有对所有任务做独立的语义验证。
- 代码把“最多调用数”和“最大轮数”分开管理；超出轮数会失败结束，不会自动推断任务已成功。
- 读取设备状态发生一般异常时，当前代码记录警告后继续；在一些情况下模型可能使用上一次成功保存的状态。因此这不是“每轮观察必然成功”的保证。
- `get_param_types` 使用按参数名建立的类型映射，并非完整的、按工具区分的 JSON Schema 验证。实际函数仍负责接受参数并返回失败结果。

所以这套设计更准确地说是：**在执行边界上做协议检查、参数转换、失败隔离和反馈闭环，降低错误调用的风险；任务语义是否正确仍依赖模型判断、设备实际状态和具体动作的结果反馈。**

## 6. 建议结合测试阅读

这些测试直接覆盖 FastAgent 的关键保护行为：

- `test_fast_agent_xml_parser.py`：XML 格式分类、参数解析、工具块处理和 memory 标签。
- `test_fast_agent_malformed_tool_guard.py`：畸形调用重试、连续错误退出和恢复行为。
- `test_fast_agent_multi_block.py`：只执行首个工具块、截断后续内容和 pending user message 处理。
- `test_fast_agent_block_failure.py`：多调用顺序执行及失败后短路行为。

建议顺着“状态和 prompt → parser → registry → action → 执行结果回填”阅读，而不是先从某个 UI 动作函数单独看起。
