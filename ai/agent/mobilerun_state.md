我会沿着这些类型的定义与调用链梳理状态来源、共享方式和各 agent 的使用方式，再给出对应源码位置。先定位相关符号与项目结构。

Search for `MobileAgentState|AndroidStateProvider|UIState|ActionContext`

Ran terminal command: git status --short && printf '\nTop-level:\n' && find . -maxdepth 2 -type f | sort | sed -n '1,100p'

Read 
Completed with input: {
  "path": "/workspaces/mobilerun/mobilerun/agent/droid/state.py"
}

Read 
Completed with input: {
  "path": "/workspaces/mobilerun/mobilerun/agent/action_context.py"
}

Read 
Completed with input: {
  "path": "/workspaces/mobilerun/mobilerun/tools/ui/state.py"
}

Read 
Completed with input: {
  "path": "/workspaces/mobilerun/mobilerun/tools/ui/provider.py"
}

Search for `shared_state\.(\w+)|action_ctx\.(\w+)|ctx\.(\w+)|state_provider\.get_state|An...`

初步定位显示这里有两层“状态”：设备采集形成的 `UIState`，以及由多个 agent 共享的 `MobileAgentState`；`ActionContext` 则把共享状态和设备依赖交给动作函数。接下来我会核对状态字段定义、初始化/刷新点及 Manager、Executor、FastAgent 的读写路径。

Read , lines 1 to 210
Completed with input: {
  "path": "/workspaces/mobilerun/mobilerun/agent/droid/state.py",
  "view_range": [
    1,
    210
  ]
}

Read , lines 1 to 230
Completed with input: {
  "path": "/workspaces/mobilerun/mobilerun/tools/ui/state.py",
  "view_range": [
    1,
    230
  ]
}

Read , lines 1 to 250
Completed with input: {
  "path": "/workspaces/mobilerun/mobilerun/tools/ui/provider.py",
  "view_range": [
    1,
    250
  ]
}

Read , lines 290 to 335
Completed with input: {
  "path": "/workspaces/mobilerun/mobilerun/agent/droid/droid_agent.py",
  "view_range": [
    290,
    335
  ]
}

Read , lines 720 to 870
Completed with input: {
  "path": "/workspaces/mobilerun/mobilerun/agent/droid/droid_agent.py",
  "view_range": [
    720,
    870
  ]
}

Search for `MobileAgentState\(|shared_state=|state_provider=|action_ctx\.ui|\.a11y_tree|p...`

Read , lines 199 to 410
Completed with input: {
  "path": "/workspaces/mobilerun/mobilerun/tools/ui/provider.py",
  "view_range": [
    199,
    410
  ]
}

Read , lines 180 to 240
Completed with input: {
  "path": "/workspaces/mobilerun/mobilerun/tools/ui/state.py",
  "view_range": [
    180,
    240
  ]
}

Read , lines 185 to 215
Completed with input: {
  "path": "/workspaces/mobilerun/mobilerun/agent/droid/state.py",
  "view_range": [
    185,
    215
  ]
}

Read , lines 390 to 565
Completed with input: {
  "path": "/workspaces/mobilerun/mobilerun/agent/manager/manager_agent.py",
  "view_range": [
    390,
    565
  ]
}

Read , lines 285 to 425
Completed with input: {
  "path": "/workspaces/mobilerun/mobilerun/agent/fast_agent/fast_agent.py",
  "view_range": [
    285,
    425
  ]
}

Read , lines 75 to 250
Completed with input: {
  "path": "/workspaces/mobilerun/mobilerun/agent/executor/executor_agent.py",
  "view_range": [
    75,
    250
  ]
}

Read , lines 365 to 410
Completed with input: {
  "path": "/workspaces/mobilerun/mobilerun/tools/ui/provider.py",
  "view_range": [
    365,
    410
  ]
}

Read , lines 425 to 485
Completed with input: {
  "path": "/workspaces/mobilerun/mobilerun/agent/droid/droid_agent.py",
  "view_range": [
    425,
    485
  ]
}

Read , lines 910 to 960
Completed with input: {
  "path": "/workspaces/mobilerun/mobilerun/agent/droid/droid_agent.py",
  "view_range": [
    910,
    960
  ]
}

Read , lines 235 to 350
Completed with input: {
  "path": "/workspaces/mobilerun/mobilerun/agent/executor/executor_agent.py",
  "view_range": [
    235,
    350
  ]
}

Search for `ctx\.(shared_state|ui|state_provider|driver|app_opener_llm|credential_manager...`

Read , lines 90 to 150
Completed with input: {
  "path": "/workspaces/mobilerun/mobilerun/agent/tool_registry.py",
  "view_range": [
    90,
    150
  ]
}

Read , lines 60 to 225
Completed with input: {
  "path": "/workspaces/mobilerun/mobilerun/agent/manager/stateless_manager_agent.py",
  "view_range": [
    60,
    225
  ]
}

Read , lines 995 to 1150
Completed with input: {
  "path": "/workspaces/mobilerun/mobilerun/agent/droid/droid_agent.py",
  "view_range": [
    995,
    1150
  ]
}

Read , lines 930 to 1015
Completed with input: {
  "path": "/workspaces/mobilerun/mobilerun/agent/droid/droid_agent.py",
  "view_range": [
    930,
    1015
  ]
}

Read , lines 1 to 60
Completed with input: {
  "path": "/workspaces/mobilerun/mobilerun/agent/action_context.py",
  "view_range": [
    1,
    60
  ]
}

Search for `class MobileAgentState|instruction: str|step_number: int|formatted_device_sta...`

Search for `class ActionContext|def __init__|self\.(driver|ui|shared_state|state_provider...`

Search for `class AndroidStateProvider|async def get_state|def _state_from_data|return se...`

Search for `self\.shared_state = shared_state|step_number \+=|action_history\.append|summ...`

Search for `shared_state\.(width|height|current_package_name|current_activity_name|visite...`

Read , lines 21 to 45
Completed with input: {
  "path": "/workspaces/mobilerun/mobilerun/agent/action_context.py",
  "view_range": [
    21,
    45
  ]
}

Read , lines 130 to 205
Completed with input: {
  "path": "/workspaces/mobilerun/mobilerun/agent/tool_registry.py",
  "view_range": [
    130,
    205
  ]
}

调用路径已确认：reasoning 模式中 Manager 与 Executor 收到同一个共享状态对象和同一个 ActionContext；FastAgent 也接收同一对象，但它直接执行工具而非经由 Executor。最后我会核对一些边界点（例如快照与共享状态哪些字段并非一一同步），避免把 `UIState` 和工作流状态混为一谈。

Search for `MobileAgentState\(|self\.shared_state = shared_state|ActionContext\(|self\.ma...`

Search for `^class UIState|^    def __init__|^        self\.(elements|formatted_text|focu...`

Search for `^class AndroidStateProvider|^    supported =|^    async def get_state|^    de...`

Search for `^class MobileAgentState|^    (instruction|step_number|platform|device_date|fo...`

Search for `^class ActionContext|^    def __init__|^        self\.(driver|ui|shared_state...`

## 一句话区分

- **`MobileAgentState`**：整个任务生命周期内由多个 agent 共同读写的协调状态。
- **`AndroidStateProvider`**：从设备驱动获取当前设备/UI 数据，并把原始数据处理成快照。
- **`UIState`**：一次设备观察的快照，提供元素查找与坐标转换能力。
- **`ActionContext`**：把动作执行所需的驱动、当前 UI 快照、共享状态等依赖打包，传给每个 action；它本身不是状态存储。

## 各层分别是什么

### `MobileAgentState`：跨 agent 的共享工作状态

它是 Pydantic `BaseModel`，在 `MobileAgent` 初始化时创建一次，保存任务指令、设备观察、计划、动作历史、记忆、完成标记和外部消息队列等字段。定义见 `MobileAgentState`。

按用途可分为：

| 状态类别 | 关键字段 | 主要用途 |
|---|---|---|
| 任务与运行 | `instruction`、`step_number`、`platform`、`device_date` | 任务上下文、步数限制及平台信息 |
| 当前/上一轮设备观察 | `formatted_device_state`、`previous_formatted_device_state`、`focused_text`、`a11y_tree`、`phone_state`、`screenshot` | 给 Manager、Executor、FastAgent 构造提示和工具执行上下文 |
| 应用跟踪 | `current_package_name`、`current_activity_name`、`visited_packages`、`visited_activities`、`app_card` | 追踪当前/已访问应用，向规划提示提供应用背景 |
| 计划与进展 | `plan`、`current_subgoal`、`previous_plan`、`progress_summary`、`last_thought` | Manager 规划并向 Executor 传递子目标；不同 agent 共用进度 |
| 动作结果 | `action_history`、`summary_history`、`action_outcomes`、`error_descriptions`、`last_action`、`last_summary` | Executor 与 Manager 回顾动作、判断错误并调整计划 |
| 记忆与对话 | `agent_memory`、`message_history` | 保存跨轮记忆；有状态对话 agent 还会维护 LLM 消息历史 |
| 完成与协作 | `finished`、`success`、`answer`、`pending_user_messages`、`workflow_completed` | 完成任务、处理 `complete()`、在运行中接收外部消息 |

例如，`append_memory()` 追加共享记忆，`complete()` 设置完成/成功/答案，`queue_user_message()` 与 `drain_user_messages()` 管理运行中注入的消息；`update_current_app()` 更新当前应用及访问记录。见 [`state.py` 中的方法`](/workspaces/mobilerun/mobilerun/agent/droid/state.py:122)。

### `AndroidStateProvider`：采集并整理设备状态

`AndroidStateProvider` 持有 `driver`，调用 `driver.get_ui_tree()` 获取设备状态，经过 tree filter 和 formatter 后返回一个 `UIState`。实现入口在 `AndroidStateProvider`：`get_state()` 负责读取，`_state_from_data()` 负责过滤、格式化和坐标配置，最终创建快照。它还处理读取重试、Android 无障碍服务恢复，以及屏幕截图与坐标空间对齐等逻辑。

有个容易误解的细节：共享字段名叫 `a11y_tree`，但 agent 刷新它时赋入的是 `ui_state.elements`——即 provider 处理后的元素树，不是直接把 driver 返回的原始树原封不动放进去。

### `UIState`：一次观察的 UI 快照

`UIState` 保存一轮观察的元素、格式化文本、焦点文本、手机状态、屏幕尺寸和坐标契约信息，定义见 `UIState`。

它的主要职责是让动作针对“当前这份快照”定位元素和处理坐标：

- `get_element()`、`get_element_info()`：按元素索引查找元素。
- `get_element_coords()`：从元素 bounds 求点击位置，并考虑已知遮挡。
- `convert_point()`：将模型提供的点转换成设备坐标；根据配置处理归一化坐标或缩放。

因此，`UIState` 不是跨轮历史：下一次 `get_state()` 会得到新的快照；动作执行时用的是 `ActionContext.ui` 当前指向的那份快照。

## 不同 agent 如何共享和使用状态

共享主要靠**对象引用**，不是通过每个 agent 单独复制或做数据库同步：

1. `MobileAgent` 初始化时创建一个 `MobileAgentState`，见 `droid_agent.py`。
2. reasoning 模式创建 Manager 和 Executor 时，把同一个 `self.shared_state` 传给它们，见 `Manager/Executor 初始化`。
3. 初始化工具执行上下文时，`ActionContext.shared_state` 也指向这个对象；Manager 和 Executor 还会被接入同一个 ActionContext，见 `ActionContext 装配`。
4. FastAgent 创建时同样接收 `self.shared_state`，见 `FastAgent 创建`。

所以，一方修改共享字段，后续运行的其他 agent 会从同一对象看到更新。

**Reasoning 模式：**

- Manager 每轮通过 `state_provider.get_state()` 获取快照，并把它设为 `action_ctx.ui`；同时更新共享的当前/上一轮文本状态、焦点文本、元素树、手机状态和截图。见 `Manager.prepare_context`。
- Manager 更新 `plan`、`current_subgoal`、`last_thought` 等共享计划字段。
- Executor 从共享状态读取指令、计划、子目标、进度、动作历史和设备状态，再调用动作。见 `Executor.prepare_context`。
- Executor 返回后，由外层工作流把动作、结果、错误和摘要写回 `MobileAgentState`，见 `run_executor`；Manager 后续可据此重新规划。
- `StatelessManagerAgent` 也更新这批共享 UI 字段；“stateless”不代表没有共享状态，而是它仍依赖同一共享状态构造每轮提示。见 `StatelessManagerAgent.prepare_context`。

**FastAgent 模式：**

FastAgent 自己递增 `step_number`、读取新 UI 快照并更新共享 UI 字段，再把设备状态、记忆和必要的截图放进 LLM 输入；它直接通过工具注册表执行动作，不经过 Executor。见 `FastAgent 单步流程`。该模式也使用同一个 `MobileAgentState`，例如工具调用的完成结果会被 FastAgent 读取。

## `ActionContext` 是什么？有什么关键要素？

`ActionContext` 是普通依赖容器类，不是 Pydantic 状态模型，也没有复杂的状态管理逻辑。定义在 `action_context.py`，构造时保存：

| 属性 | 用途 |
|---|---|
| `driver` | 实际执行设备操作，如点击、输入、滑动 |
| `ui` | 当前 `UIState` 快照；初始化时可为 `None`，每轮采集状态后更新 |
| `shared_state` | 让 action 读取或修改任务级共享状态 |
| `state_provider` | 动作需要时重新获取状态，或读取坐标能力/契约信息 |
| `app_opener_llm` | 为打开应用相关动作提供 LLM |
| `credential_manager` | 获取或解析凭据，供安全输入动作使用 |
| `streaming` | 向需要流式反馈的动作传递配置 |
| `macro_recorder` | 供宏记录相关动作使用 |

工具注册表执行 action 时，会把这个对象作为 `ctx` 传入；见 `ToolRegistry.execute`。动作实现随后通过 `ctx` 使用依赖：例如元素点击通过 `ctx.ui` 解析坐标、再由 `ctx.driver` 点击；`complete` action 则更新 `ctx.shared_state`。相应实现可见 `actions.py` 和 `actions.py 的完成动作`。

**概括来说**：`MobileAgentState` 解决“各 agent 如何协调并记住任务进展”；`StateProvider` 与 `UIState` 解决“当前设备看起来怎样、如何定位 UI”；`ActionContext` 解决“一个 action 如何拿到执行所需的对象”。这些职责相连，但不应当把三者视为同一种状态。
