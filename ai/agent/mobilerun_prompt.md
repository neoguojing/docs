---
title: Agent Prompt 设计与流程
---

# Agent Prompt 设计与流程

本文说明 Mobilerun Agent 流程中各个 LLM 调用的 prompt 属于哪个 Agent、何时构造、传入哪些字段、字段的用途，以及不同执行模式之间的差异。

## 先看流程：一次任务有多次、不同职责的 LLM 调用

主 Agent 有两种运行方式，由 `agent.reasoning` 配置决定：

```text
reasoning = false
  FastAgent（单 Agent、工具调用循环）
    prompt -> LLM 生成工具调用 XML -> ToolRegistry 执行
    -> 工具结果进入对话 -> 下一轮 prompt -> complete 结束

reasoning = true
  Manager（规划） -> Executor（单步执行） -> Manager（重新规划/判断完成）
    Manager prompt -> 输出 plan / current_subgoal / answer
    Executor prompt -> 输出单个 JSON action
    ToolRegistry 执行 -> 执行结果进入共享状态 -> 下一轮 Manager prompt
```

推理模式下，`manager.stateless=true` 会将普通 Manager 替换为 Stateless Manager。Stateless Manager 仍负责规划，但不向 LLM 发送累积对话历史，而是每次用当前共享状态重新构造一份 prompt。

此外，Android 的 `open_app` 按名称或描述启动时可能触发辅助的 AppStarter LLM 调用，用来把用户说的应用名称匹配到已安装应用包名。它不是每轮主 Agent prompt 的一部分。

## Prompt 从哪里来

默认模板是 Jinja2 文件，Agent 根据配置路径载入并渲染：

| Agent 调用 | 默认模板 |
|---|---|
| FastAgent system | [`fast_agent/system.jinja2`](../../mobilerun/config/prompts/fast_agent/system.jinja2) |
| FastAgent 初始 user | [`fast_agent/user.jinja2`](../../mobilerun/config/prompts/fast_agent/user.jinja2) |
| 有状态 Manager system | [`manager/system.jinja2`](../../mobilerun/config/prompts/manager/system.jinja2) |
| Stateless Manager 单轮 prompt | [`manager/stateless.jinja2`](../../mobilerun/config/prompts/manager/stateless.jinja2) |
| Executor 单动作 prompt | [`executor/system.jinja2`](../../mobilerun/config/prompts/executor/system.jinja2) |

模板路径及默认选择由 [`config_manager.py`](../../mobilerun/config_manager/config_manager.py) 中的 `FastAgentConfig`、`ManagerConfig`、`ExecutorConfig` 和 `AgentConfig.get_*_prompt_path()` 管理。Manager 未指定 `system_prompt` 时，会按 `manager.stateless` 自动选择 `manager/system.jinja2` 或 `manager/stateless.jinja2`；明确配置了路径则使用用户指定的模板。

`PromptResolver` 会先按 key 查找运行时自定义模板，例如 `fast_agent_system`、`fast_agent_user`、`manager_system`、`executor_system`；如果没有自定义模板，才加载 AgentConfig 指定的模板文件。两种来源都由 [`PromptLoader`](../../mobilerun/config_manager/prompt_loader.py) 用 Jinja2 渲染。缺失字段默认为空字符串；额外传入但模板没有引用的字段不会出现在最终文本里。

相关实现：

- [`prompt_resolver.py`](../../mobilerun/agent/utils/prompt_resolver.py)：自定义 prompt key 与覆盖规则。
- [`prompt_loader.py`](../../mobilerun/config_manager/prompt_loader.py)：文件读取和 Jinja2 渲染。
- [`droid_agent.py`](../../mobilerun/agent/droid/droid_agent.py)：创建 `PromptResolver` 并向子 Agent 传递。

`manager/rev1.jinja2`、`manager/trained.jinja2` 和 `executor/rev1.jinja2` 是可由配置路径显式选用的模板；它们不是当前默认自动选择的路径。

## FastAgent：系统规则 + 初始请求 + 累积工具对话

### 调用与消息结构

FastAgent 在 `reasoning=false` 路径使用。入口是 [`FastAgent.prepare_chat()`](../../mobilerun/agent/fast_agent/fast_agent.py)，每个执行步骤在 `handle_llm_input()` 中获取当前设备状态并调用 LLM。

第一次请求的消息大致如下：

```text
SYSTEM: fast_agent/system.jinja2 渲染结果
USER:   fast_agent/user.jinja2 渲染结果（原始 goal）
```

后续请求会将 LLM 回复、工具结果、必要的纠正消息累加到共享 `message_history`。调用模型前构造一份临时消息副本，并把最新状态信息和截图附到最新的 user 消息中；原始历史本身不会被这一步的状态注入修改。

### 字段和用途

#### system prompt 渲染字段

[`FastAgent._build_system_prompt()`](../../mobilerun/agent/fast_agent/fast_agent.py) 提供以下上下文：

| 字段 | 用途 |
|---|---|
| `tool_descriptions` | Registry 生成的工具 XML 签名与说明；告诉模型可调用哪些工具及参数。 |
| `available_tools` | 当前过滤后 Registry 中的工具名集合；模板用它判断示例是否适用。 |
| `available_secrets` | 可用凭据 ID，仅在 `type_secret` 已注册且凭据管理器可用时读取；不包含秘密值。 |
| `parallel_tools` | 是否允许/建议一个 XML block 中并行输出多个工具调用；模板据此调整规则与示例。 |
| `vision` | 是否向模型附上截图；模板说明截图信息是否可用。 |
| `screenshot_only` | 当前是否没有索引化无障碍树、需要只根据截图使用坐标工具。 |
| `platform` | 当前设备平台，填入角色说明。 |
| `output_schema` | 可选的结构化输出约束；模板要求 `complete` 的最终消息包含指定字段。 |
| `variables` | 用户自定义变量；默认 FastAgent system 模板当前不引用它，定制模板可选择使用。 |

#### 初始 user prompt 字段

[`FastAgent._build_user_prompt()`](../../mobilerun/agent/fast_agent/fast_agent.py) 传入：

| 字段 | 用途 |
|---|---|
| `goal` | 当前用户任务，默认模板以 “Current Request” 呈现。 |
| `variables` | 自定义变量；默认 user 模板只显示 `goal`，自定义模板可以引用变量。 |

#### 每轮追加的动态上下文

[`FastAgent.handle_llm_input()`](../../mobilerun/agent/fast_agent/fast_agent.py) 每轮读取状态，并向发送给模型的副本追加：

| 内容 | 注入位置 / 用途 |
|---|---|
| `<memory>` | 最新 user 消息；累积 Agent 记忆，提示模板要求离开含重要信息的页面前保存。 |
| `<device_state>` | 最新 user 消息；当前格式化 UI 状态，通常含可交互元素索引、文本和 phone state。 |
| `<previous_device_state>` | 第二新的 user 消息（如存在）；帮助对比页面/状态变化。 |
| Screenshot image | 最新 user 消息；仅 `vision=true` 且本轮取得截图时添加。 |
| Tool results | 以 user 消息追加到对话历史；让模型观察操作成功与否后再决定下一步。 |
| `<external_user_message>` | 新的用户纠正或追加指令；按最新指令更新动作。 |

FastAgent 的系统模板还直接定义 XML 工具调用协议、记忆协议 `<add_memory>`、完成协议 `complete`，以及批量工具调用规则。模型回复由 `parse_tool_calls_detailed()` 解析；工具调用 XML 与执行结果形成一个来回循环。

## 有状态 Manager：长期对话中的高层规划器

### 调用与消息结构

Manager 用于 `reasoning=true` 且 `manager.stateless=false` 的情况。`prepare_context()` 每轮读取设备状态、可选截图，并把用户任务、上轮动作摘要、外部用户消息写入共享 `message_history`。随后 `get_response()` 生成 system prompt 并组装消息：

```text
SYSTEM: manager/system.jinja2 渲染结果
...   : 共享的累积 message_history（先前的 user / assistant 消息）
USER:   本轮上层上下文（上一轮 thought、action、description 等）
        + memory
        + current device_state
        + 可选 screenshot
```

当前状态和记忆附加到最新 user 消息；上一设备状态会附加到前一条 user 消息（如有）。Manager 不直接选择/执行某个工具，而是返回高层计划、下一子目标或最终答案。`parse_manager_response()` 和响应验证逻辑负责解析/校验输出；格式无效时会将错误反馈给模型并重试。

### 字段和用途

[`ManagerAgent._build_system_prompt()`](../../mobilerun/agent/manager/manager_agent.py) 传入：

| 字段 | 用途 |
|---|---|
| `instruction` | 原始用户任务，作为 `<user_request>`。 |
| `platform` | 设备平台，说明规划目标环境。 |
| `device_date` | 设备日期/时间；帮助理解相对日期和日历任务。 |
| `app_card` | 当前应用的操作说明；提供应用专属帮助。 |
| `important_notes` | 预留字段；目前构造为 `""`，默认模板条件块不会呈现。 |
| `error_history` | 仅进入错误规划状态时提供最近失败动作摘要，提示模型重新评估是否卡住。 |
| `custom_tools_descriptions` | 已注册的非标准扩展工具描述；Manager 可在计划中要求 Executor 使用。 |
| `available_secrets` | 凭据 ID 列表，只有 `type_secret` 可用时填充；模型只获得 ID，不获得值。 |
| `output_schema` | 可选输出模型的 JSON Schema；默认模板将字段约束转成自然语言要求 Manager 最终答复包含的数据。 |
| `variables` | 自定义变量；默认 Manager 模板当前没有引用该字段，定制模板可使用。 |

当前轮的 user 消息由 `_build_user_message_content()` 和 `prepare_context()` 构造，主要含：

| 内容 | 用途 |
|---|---|
| `<thought>` | 上一轮规划的思考摘要，供下一轮接续。 |
| `<last_action>` | 上一轮 Executor 执行的动作对象。 |
| `<last_action_description>` | 上一轮动作摘要或错误文本。 |
| `<external_user_message>` | 执行期间到达的新用户指令，明确优先于旧假设。 |
| `<memory>` | 跨步骤累积的任务事实。 |
| `<device_state>` | 最新 UI 文本/设备状态。 |
| `<previous_device_state>` | 较前一轮状态，用于比较变化。 |
| Screenshot image | 仅启用 Manager 视觉且成功取图时加入最新 user 消息。 |

默认 Manager 模板也直接包含安全/任务边界类的任务规则、记忆规范、规划输出格式和终止格式。结构输出中，`<plan>` 表示继续执行并提供子目标；`<request_accomplished success="...">` 表示任务终止。

## Stateless Manager：每轮重建的一次性规划上下文

Stateless Manager 仍然是规划器，不是另一个动作执行器。它使用 `manager/stateless.jinja2`，但每轮不复用 Manager 的对话 message history。

### 调用与消息结构

[`StatelessManagerAgent._build_prompt()`](../../mobilerun/agent/manager/stateless_manager_agent.py) 渲染出一条完整 prompt；`get_response()` 将它作为单条 user 消息发送：

```text
USER: manager/stateless.jinja2 渲染结果
      + 可选 screenshot image
```

它以共享状态字段模拟“跨轮记忆”：之前计划、之前设备状态、累积 memory、上轮 thought、progress summary、最近动作历史、当前设备状态都会按需重新塞入本轮 prompt。动作历史最多取最近 5 条。

### 字段和用途

| 字段 | 用途 |
|---|---|
| `instruction` | 原始任务目标。 |
| `platform` | 设备平台；当前 stateless 默认模板正文未引用。 |
| `device_date` | 设备日期/时间。 |
| `previous_plan` | 上一轮计划，避免无历史对话时丢失规划方向。 |
| `previous_state` | 上轮设备状态，供当前状态比较。 |
| `memory` | 累积事实记忆。 |
| `last_thought` | 上轮规划的简短思考。 |
| `progress_summary` | 累积已完成内容和当前状态。 |
| `action_history` | 最近最多 5 条 Executor 动作、摘要、结果和错误。 |
| `current_state` | 当前设备状态，是本轮决策的主要 UI 依据。 |

如 Manager 开启视觉且截图获取成功，图片附在同一 user 消息中。stateless 默认模板**没有**有状态 Manager system 模板中的 `app_card`、`error_history`、`custom_tools_descriptions`、`available_secrets` 和 `output_schema` 区块；其运行时 prompt 字段集合也没有传入这些变量。输出则使用 `<think>`、`<add_memory>`、`<progress_summary>`、`<plan>` 或 `<answer success="...">`。Stateless 响应同样通过 Manager 的解析/验证代码校验。

## Executor：单个子目标对应一个原子动作

### 调用与消息结构

Manager 产生 `current_subgoal` 后，DroidAgent 调用 `ExecutorAgent.run(subgoal=...)`。Executor 每次构造一条新的 prompt 并请求模型，不携带 FastAgent/Manager 的累积 chat history：

```text
USER: executor/system.jinja2 渲染结果
      + vision 开启且有截图时附 screenshot image
```

注意默认模板文件名虽然是 `system.jinja2`，Executor 当前仍把渲染后的全文放在 `ChatMessage(role="user")` 中，而不是 `system` role。

### 字段和用途

[`ExecutorAgent.prepare_context()`](../../mobilerun/agent/executor/executor_agent.py) 传入：

| 字段 | 用途 |
|---|---|
| `instruction` | 原始用户目标，为子目标提供总任务背景。 |
| `platform` | 当前设备平台。 |
| `app_card` | 当前应用的操作提示。 |
| `device_state` | 当前可见 UI 状态/元素索引。 |
| `plan` | Manager 生成的整体计划，让 Executor 的单步操作与总目标一致。 |
| `subgoal` | 本次必须完成的当前子目标。 |
| `progress_status` | 累积进度摘要。 |
| `atomic_actions` | 当前 Registry 可用工具签名、参数和描述；`complete` 被排除。 |
| `action_history` | 最近最多 5 条已执行动作、结果及错误，避免重复失败动作。 |
| `available_secrets` | `type_secret` 可用时提供凭据 ID；不提供实际值。 |
| `variables` | 用户自定义变量；默认 Executor 模板当前不引用它，定制模板可使用。 |

Executor 模板的核心要求是按子目标执行**一个**原子动作，输出以下协议：

```text
### Thought
...
### Action
{"action": "click", "index": 3}
### Description
...
```

`parse_executor_response()` 从三个段落中抽取 thought、动作 JSON 和描述；`ExecutorAgent.execute()` 解析动作名称/参数后调用 `ToolRegistry.execute()`。因此 Executor 的 prompt 是离动作执行最近的 LLM prompt。

## 辅助调用：AppStarter 的应用匹配 prompt

Android 默认的 `open_app(text=...)` 工具可能调用 [`AppStarter`](../../mobilerun/agent/oneflows/app_starter_workflow.py)。它先调用 `driver.get_apps(include_system=True)`，再将已安装应用列表和用户描述拼成一段普通文本 prompt，通过 `acomplete_with_retries()` 请求 LLM：

| 输入 | 用途 |
|---|---|
| `apps_list` | 已安装应用 label 和 package name 列表，限定可选结果范围。 |
| `app_description` | 工具 `open_app` 收到的用户目标应用名称/描述。 |
| 输出格式要求 | 只返回 `{"package": "..."}` 或 `{"package": null}`。 |

解析包名后，AppStarter 直接调用 `driver.start_app(package_name)`。这是**专用应用匹配调用**，没有使用 FastAgent/Manager/Executor 的模板，也没有以 ToolRegistry 再分派一次。

如果运行配置走 `open_bundle_id` 精确应用 ID 分支，则不需要这个匹配 LLM 调用。

## 流程对比

| 对比项 | FastAgent | 有状态 Manager | Stateless Manager | Executor |
|---|---|---|---|---|
| 启用条件 / 位置 | `reasoning=false` 主流程 | `reasoning=true` 的规划阶段 | `reasoning=true` 且 stateless | Manager 给出子目标后 |
| 职责 | 自主循环：看状态、直接选工具、判断完成 | 高层规划、更新计划、给出下一子目标或最终答复 | 与 Manager 相同，但每轮重建上下文 | 将一个子目标落成一个原子动作 |
| 默认 prompt 模板 | system + user 两份 | manager system | manager stateless 单份 | executor system 单份 |
| 消息角色 | system + user + 历史往返 | system + 累积历史 | 单条 user 消息 | 单条 user 消息 |
| 主要状态历史 | 对话历史、memory、当前/上一 UI state、tool result | 对话历史、memory、当前/上一 UI state、last action | previous plan/state、memory、最多 5 条 action history、current state | plan/subgoal、当前状态、最多 5 条 action history |
| 工具信息 | 所有 Registry 工具的 XML 描述 | 标准动作由规划规则描述；自定义/MCP 扩展工具单独放入 `custom_actions` | 默认模板没有传工具签名或自定义工具说明 | Registry 实际可用动作签名，排除 `complete` |
| 输出协议 | XML `<function_calls>`；`complete` 终止 | `<plan>` 或 `<request_accomplished>` | `<plan>` 或 `<answer>` | `### Thought / Action / Description`，Action 是 JSON |
| 截图 | vision 开启时附到当前 user 消息 | vision 开启时附到当前 user 消息 | vision 开启时附到单条 user 消息 | vision 开启时附到本轮单条 user 消息 |
| 对话 token 形态 | 随轮次累积；有 `limit_history()` 限制历史消息 | 累积 `message_history`，无此处的裁剪步骤 | 不传累积对话，每轮重建 | 不传累积对话，每个子目标一次新 prompt |

## 关键设计差异与理解建议

1. **Prompt 的职责边界映射 Agent 职责**：FastAgent 获得完整工具说明并直接调用工具；Manager 提供任务级上下文并输出子目标；Executor 获取当前工具签名并执行单步。不要把 Manager 的 `<plan>` 当成工具调用，也不要把 Executor 当成任务最终回答者。
2. **字段有静态与动态两部分**：模板变量形成系统规则/说明；每轮动态注入的状态、记忆、动作结果和截图构成“当前任务现场”。同一个 Agent 的完整 prompt 不只等于 `.jinja2` 文件内容。
3. **Manager 两种模式的上下文策略不同**：有状态模式靠完整对话历史承接上下文；Stateless 模式把必要上下文显式复制到每轮 prompt。两者不应只按模板文字差异理解。
4. **默认 prompt 里的可选字段依条件出现**：例如无可用凭据时没有秘密 ID 区块；无截图时没有图片；未配置 output model 时没有输出字段要求。
5. **传入字段不代表最终 prompt 一定显示**：Jinja 模板必须引用变量才会渲染它。当前有些默认模板不使用 `variables` 或 `platform`，但运行时仍传了这些值；自定义模板可以引用它们。
6. **响应协议是解析器契约**：FastAgent 期待 XML 工具调用；Manager 期待规定的 XML 风格标签；Executor 期待 Markdown 标题段落和 JSON action。改模板时应同步考虑对应解析和验证代码。

## 关键代码入口

| 组件 | 关键代码 |
|---|---|
| 主流程选择、有状态/无状态 Manager 装配 | [`droid_agent.py`](../../mobilerun/agent/droid/droid_agent.py) |
| FastAgent prompt 构造、消息历史与状态注入 | [`fast_agent.py`](../../mobilerun/agent/fast_agent/fast_agent.py) |
| Manager prompt 构造、历史消息与 UI 注入 | [`manager_agent.py`](../../mobilerun/agent/manager/manager_agent.py) |
| Stateless Manager 字段和单消息构造 | [`stateless_manager_agent.py`](../../mobilerun/agent/manager/stateless_manager_agent.py) |
| Manager 输出标签解析与校验 | [`prompts.py`](../../mobilerun/agent/manager/prompts.py) |
| Executor 字段、截图和动作解析调用 | [`executor_agent.py`](../../mobilerun/agent/executor/executor_agent.py)、[`prompts.py`](../../mobilerun/agent/executor/prompts.py) |
| AppStarter 应用匹配 prompt | [`app_starter_workflow.py`](../../mobilerun/agent/oneflows/app_starter_workflow.py) |
| 默认模板目录 | [`config/prompts/`](../../mobilerun/config/prompts/) |
