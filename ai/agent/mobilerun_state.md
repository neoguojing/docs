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


## `ActionContext` 由谁创建？

由顶层工作流 `MobileAgent` 在 `start_handler` 中创建，而不是由 Manager、Executor 或 FastAgent 各自创建。此时 driver、state provider 和 tool registry 等依赖已经准备好，创建代码把它们以及共享状态装入同一个 `ActionContext`。初始 `ui=None`，因为当前 UI 快照要等 agent 采集设备状态后才有。见 `start_handler 创建 ActionContext`。

Manager、Executor 在 `MobileAgent` 初始化时先以 `action_ctx=None` 构造；启动阶段再被接入这个实例。reasoning 模式中 Manager 和 Executor 持有的是**同一个 `ActionContext` 对象**，不是各自的一份副本。见 `Manager/Executor 初始化` 和 `start_handler 注入依赖`。

FastAgent 创建时也拿到同一个 `action_ctx`，见 `FastAgent 创建`。所以无论采用 reasoning 流程还是 FastAgent 流程，动作执行都沿用 MobileAgent 创建的上下文。

## 哪些地方引用和使用？

| 使用方 | 怎么使用 |
|---|---|
| **Manager** | 通过 `action_ctx.driver` 截取截图；每次采集到新 UI 后设置 `action_ctx.ui`；也会通过 `credential_manager` 获取可用凭据列表。见 `ManagerAgent`。 |
| **Stateless Manager** | 同样用上下文访问 driver，并在采集设备状态后刷新 `action_ctx.ui`。见 `StatelessManagerAgent`。 |
| **Executor** | 用 `state_provider` 处理截图坐标空间，并把 `action_ctx` 传给工具注册表执行动作。见 `ExecutorAgent` 和 `Executor 执行动作`。 |
| **FastAgent** | 用上下文截屏、刷新当前 UI，并将上下文传给工具注册表执行动作。见 `FastAgent` 和 `FastAgent 工具执行`。 |
| **动作函数** | 由 `ToolRegistry.execute()` 以关键字参数 `ctx` 传入；动作从 `ctx` 取得 driver、UI 快照、共享状态等依赖。见 `ToolRegistry.execute`。 |
| **MobileAgent 自身** | 也会通过 `action_ctx.driver` 做最终阶段的设备操作，例如截取最终截图。见 `droid_agent.py`。 |

## 它在架构上的作用

`ActionContext` 是一个**动作执行依赖的统一入口**，可以理解为 dependency bundle / dependency injection boundary：

```text
MobileAgent 创建并组装依赖
        │
        ├── Manager / Executor / FastAgent
        │
        └── ToolRegistry.execute(..., ctx=action_ctx)
                         │
                         └── action 函数使用 ctx.driver / ctx.ui / ...
```

动作函数不必依赖 `MobileAgent` 或某个具体 agent，也不需要把 driver、状态、凭据等依赖分别作为一长串参数传递。它只接收一个 `ctx`，从中取自己需要的内容。这样可以让动作实现与工作流编排相对解耦，并让多种 agent 共用相同的执行依赖。

注意，**`ActionContext` 不负责拥有或维护所有状态**：

- `shared_state` 是任务级跨 agent 状态的引用。
- `ui` 是当前轮 `UIState` 快照的引用；每轮采集后由 agent 更新它。
- `driver` 和 `state_provider` 是设备操作及观察能力的引用。
- 其他字段则提供凭据管理、流式配置、应用启动 LLM 和宏记录器等可选依赖。

因此它更像是一个“**动作执行时的依赖视图**”，而不是另一份独立的 agent 状态。字段定义见 `ActionContext`。
