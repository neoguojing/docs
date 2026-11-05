# ARTEMIS 手机端操作架构与准确性说明

本文按代码实现说明 ARTEMIS 如何观察、理解并操作 Android 设备，怎样把操作送到设备，以及执行失败后的反馈和恢复方式。重点是主任务执行链路；`ask_explorer` 等视觉子代理作为辅助定位路径单独说明。

## 1. 总览：从任务到设备

```text
用户目标
  │
  ▼
Planner ── 任务计划 / 子目标 / 验收条件
  │
  ▼
Perception ── 截图 + Android UI 层级 +（可选）OCR
  │
  ▼
Operator LLM ── 结合目标、计划、历史和当前屏幕，选择工具及操作
  │ structured_decisions（动作 JSON）
  ▼
Execution Check ── 检查计划/完成状态，决定是否执行
  │
  ▼
Validator ── 单动作安全网 / 坐标转换 / 执行动作 / 记录结果
  │
  ├── 成功或失败结果、截图、trace ──► Summarizer / 下一轮
  └── 失败 ──► Execution Incident ──► 下一轮 Operator 重新观察并决策
```

主图由 LangGraph 编排，常见循环为 `perception → operator → execution_check → validator → summarizer → perception`。Planner 负责建立任务计划；Operator 是执行时的主要决策 LLM；Validator 负责执行和执行前检查；Checker 在子目标或任务结束时独立评估验收条件。Validator 不负责判断整个用户目标最终是否达成。

相关实现：[graph.py](../artemis/graph/graph.py)、[perception.py](../artemis/graph/perception.py)、[operator.py](../artemis/agents/operator/operator.py)、[validator.py](../artemis/agents/validator/validator.py)、[checker.py](../artemis/agents/checker/checker.py)。

## 2. 屏幕观察和提供给模型的信息

### 2.1 观察数据的产生

Perception 每轮获取设备屏幕数据，并按需等待界面稳定。观察数据包括截图、UI 层级和屏幕宽高；OCR 配置可用时还会对截图运行 OCR，再将 OCR 文本与 XML 元素融合。OCR 失败时继续使用 XML，不会因此丢弃整个观察结果。

核心 State 字段如下：

| 字段 | 主要内容 / 用途 |
|---|---|
| `initial_goal` | 用户原始目标；任务期间保留。 |
| `latest_screenshot` | 最新截图的持久化路径。 |
| `latest_ui_hierarchy` | 当前 XML/OCR 融合后的 UI 元素。 |
| `operator_raw_data` | 原始截图 base64、原始 XML、OCR 结果、宽高。 |
| `indexed_elements` / `indexed_points` | 当前观察建立的编号元素和点位索引，供动作目标引用。 |
| `structured_decisions` | Operator 产出的待执行动作列表 JSON。 |
| `last_execution_result` | Validator 最近一次执行报告。 |
| `open_incident` / `last_closed_incident` | 尚未解决的执行故障及最近关闭提示。 |
| `operator_feedback` | Checker、Planner 或 final check 提供的来源标记反馈。 |

State 使用 Pydantic 显式字段并禁止额外字段；LangGraph 的状态 reducer 决定每轮字段如何更新，避免未声明数据静默混入状态。定义见 [state.py](../artemis/graph/state.py)。

### 2.2 Operator LLM 实际看到什么

Operator 的提示主要由系统指令、工具声明和当前轮用户观察组成：

1. **系统指令**：Android 操作角色、精确操作原则、等待转场、计划和执行历史的约束、操作工具用法、验收/恢复规则。工具可用性会结合执行器能力动态过滤，避免提示模型调用后端不支持的动作。见 [operator.json](../artemis/agents/operator/operator.json) 和 [prompts.py](../artemis/agents/operator/prompts.py)。
2. **当前截图**：以图像输入提供，是判断屏幕真实状态的首要依据。
3. **Visible UI Elements 列表**：从 UI 层级/OCR 生成的辅助索引，包含编号、文本或提示文本、归一化边界；用于把目标指定为元素序号。底层索引还保存中心点、像素边界、类别、resource id、OCR 来源等字段。
4. **任务上下文**：用户目标、当前任务计划、摘要后的执行历史；启用 transcript 记忆时，还会按记忆策略保留近期消息与必要的计划回顾。
5. **动态反馈**：安全网拒绝原因、执行 incident、Checker 验收发现、用户注入指令，以及背景任务状态等（存在时）。
6. **工具 schema**：当前可用的动作、历史/笔记、Explorer、诊断等工具及其参数说明。

元素清单是模型的辅助索引，不是屏幕事实的唯一来源。Operator 提示要求先看截图，再用元素清单定位；未列出的可视目标可由 Explorer 辅助定位。

格式化列表的实现见 [visualization.py](../artemis/utils/visualization.py)；Perception 对模型可用状态字段的写入见 [perception.py](../artemis/graph/perception.py)；截图和列表加入提示见 [prompts.py](../artemis/agents/operator/prompts.py)。

### 2.3 不同 LLM 的输入和职责

- **Planner**：围绕用户目标构造任务计划；不会直接触屏。
- **Operator**：读取上述任务上下文与当前截图，发出工具调用。动作工具调用会先被转换和校验，再保存为 `structured_decisions`；模型调用的工具不等于立即执行的设备命令。
- **Explorer**：`ask_explorer` 的屏幕元素定位代理。输入包含定位问题、当前截图、Operator 上下文反馈和初始编号 UI 元素列表；可用文本搜索、坐标审计、OCR、视觉检测、局部裁剪等工具逐步定位，最后返回候选项，不直接触屏。
- **Validator 的像素安全网**：必要时由视觉模型比较决策时和当前屏幕的目标局部图，并提供目标描述等上下文，判断动作目标是否仍在。
- **Checker**：读取计划验收项及其证据，产出逐项结构化 verdict；它与 Operator 分离，且检查工具设计为只读。

Explorer 的最终返回形状为 `{"candidates":[{"label":"...","description":"...","coords":[nx,ny]}],"fallback_message":"..."}`，坐标为 0–1000 归一化值。Flash tier 可采用一次性视觉检测；Pro/Ultra 路径可以进行多轮感知工具调用。见 [explorer.py](../artemis/agents/explorer/explorer.py)、[tool_declarations.py](../artemis/agents/explorer/tool_declarations.py)、[screen_index.py](../artemis/agents/explorer/screen_index.py)。

## 3. 动作数据结构与坐标流转

### 3.1 Operator 的动作参数

动作 schema 的单一来源是 [action_specs.py](../artemis/mcp/action_specs.py)。典型动作工具有：

- `click(target, target_description?, times?, delay_ms?)`
- `long_press(target, target_description?, duration?)`
- `input_text(text, target?, target_description?, clear_exist?)`
- `swipe(...)`、`press_key(key)`、`manage_app(action, app_name)`、`wait_for_delay(time_in_ms)`

点击/长按/输入的 `target` 可以是：

- **元素序号**：例如 `3`，按当前可见元素列表取得已观察到的元素中心、文本、边界、resource id 和 class。
- **归一化坐标**：例如 `[520, 440]`，两个分量范围为 0–1000；必须同时提供 `target_description`，说明坐标处的目标是什么。

二者的来源有意分开：元素序号附带观察到的元素属性；坐标目标只记录模型自己的文字描述，不伪装成 UI 观察值。见 [operator.py](../artemis/agents/operator/operator.py) 和 [action_executor.py](../artemis/mcp/action_executor.py)。

### 3.2 Validator 动作记录示例

Operator 动作会转换为待 Validator 执行的动作列表。下例展示了索引目标经转换后的概念性记录（字段会随动作类型不同而变化）：

```json
[
  {
    "action": "tap",
    "coordinates": [561, 1056],
    "coordinate_space": "pixel",
    "normalized_coordinates": [520, 440],
    "target_text": "Search",
    "target_bounds": [420, 1000, 700, 1112],
    "target_resource_id": "com.example:id/search",
    "target_class": "android.widget.EditText"
  }
]
```

字段意义：

| 字段 | 说明 |
|---|---|
| `action` | Validator 使用的规范化动作名，例如 `tap`、`long_press_on`、`focus_and_input_text`。 |
| `coordinates` | 驱动器执行用的屏幕像素点（或滑动端点）。 |
| `coordinate_space` | 记录坐标单位，避免对同一坐标重复转换。 |
| `normalized_coordinates` | 0–1000 坐标，用于模型可读记录及跨分辨率表达。 |
| `target_text` / `target_bounds` / `target_resource_id` / `target_class` | 从已观察 UI 元素提取的目标身份信息；坐标目标一般使用 `target_description`。 |
| 动作参数 | 例如 `text`、`duration`、`times`、方向或等待时长。 |
| 执行后字段 | 失败时可能包含 `attempts`；报告还包含状态、错误、burst 标记和 incident。 |

实际动作记录在转换时会包含必要信息，但不保证每个字段总是出现；例如坐标目标不会有虚构的 `target_text`，滑动也可能记录开始/结束点而没有单一目标元素。

坐标通常在 LLM 接口使用 0–1000 空间，在后端用设备当前宽高换算到像素，并限制在屏幕范围内；设备屏幕大小来自当前观察/设备上下文，而不是假定所有手机分辨率一致。Operator 生成的索引坐标先解析为当前索引元素中心，再交给执行侧。

### 3.3 结果结构

设备动作通过结构化 `ActionResult` 返回，而不是靠分析错误字符串判断成功：

```json
{
  "ok": false,
  "code": "DEVICE_ERROR",
  "action": "click",
  "message": "Error executing click: ...",
  "detail": "...",
  "normalized_coordinates": [520, 440],
  "duration_ms": null
}
```

`code` 可区分 `INVALID_ARGS`、`TARGET_NOT_FOUND`、`DEVICE_ERROR`、`PACKAGE_NOT_FOUND`、`TIMEOUT`、`UNSUPPORTED` 等；`detail` 用于诊断底层原因。观察结果 `ObserveResult` 则包含 `ok`、截图路径/名称、分辨率、格式化元素清单、结构化元素，以及 `hierarchy_ok`。动作结果成功仅表示命令被执行器/设备接口接受，并不等于目标 UI 已达到预期状态。

定义见 [action_types.py](../artemis/mcp/action_types.py)。

## 4. 准确性保障及其边界

### 4.1 执行前的目标核对

对一个 turn 中的**单个动作**，Validator 先运行 Safety Net：

1. 有 UI 元素元数据时优先拉取实时层级，并将当前元素与决策时目标按 resource id、文本相似度、边界 IoU 和坐标包含关系综合匹配。
2. 针对小幅位移，在目标身份匹配足够可信的情况下可自我修正动作中心点；不确定目标是否为同一元素时则保守拒绝。
3. 若 UI XML 不可用/超时或没有有效候选，会退回像素级安全网，比较原始截图目标区域和当前截图目标区域。
4. 若坐标目标没有元素元数据，则主要依靠像素安全网。
5. 发现目标已消失、偏移过大、或目标位置被其它控件遮挡时，不发送该动作，而是返回分类错误和证据。

XML 检查最多尝试 3 次，间隔 0.4 秒；像素检查有最多 3 次尝试，主要针对基础设施/模型调用异常。如果像素模型以较高置信度判断目标不在，会拒绝动作；若置信度低，或无法取得截图/模型不可用，则检查可能以 `PIXEL_BYPASSED` 放行，以免安全网阻塞整个任务。此处是明确的可用性与安全性权衡，不是绝对保证。

细节见 [precondition_xml.py](../artemis/agents/validator/precondition_xml.py)、[precondition_pixel.py](../artemis/agents/validator/precondition_pixel.py)。

### 4.2 Schema、状态与执行一致性

- **动作定义共用**：动作名称、参数、工具提示和传输端参数由 manifest/schema 生成，减少 Operator、Flash 和 MCP wire 之间的 schema 漂移。
- **索引有范围检查**：元素序号必须在本轮可见列表内；越界、缺少坐标目标描述、无效参数等会在执行前返回错误，不盲目操作。
- **清晰坐标单位**：归一化坐标和像素坐标带有单位/尺寸上下文，避免把 0–1000 点误当像素。
- **动作结果结构化**：`ActionResult.ok/code` 是机器判定依据；设备拒绝和 transport/protocol 错误分开报告。
- **单设备动作串行化**：`ActionSession` 的 owner task 和请求队列串行处理动作与观察，避免子代理和 Validator 并发交错操作同一设备。
- **观察后再决策**：Perception 在下一轮重新读取屏幕；Explorer 与 Flash 路径也可在动作后重新观察。执行报告的 `dispatched` 不被当作 UI 效果的证明。
- **最终验收独立**：Checker 会根据检查项和只读设备/历史证据验证子目标或最终目标。执行成功不能替代验收成功。

需要注意：Operator 的多动作快速 burst 是明确的延迟优化例外。一个 turn 里有两个或更多结束动作时，它们背靠背执行，不在动作间拍屏、不运行 Safety Net、也不重试；首个失败会终止 burst，后续动作标为未执行。此路径比单动作更依赖 Operator 的目标选择和排序。

## 5. 出错、执行失败与恢复

### 5.1 动作级失败

- Operator 工具参数校验失败时，错误 `ToolMessage` 返回给同一轮 LLM，模型可修正工具参数。
- 单动作通过 Safety Net 后，普通动作最多执行 2 次（间隔 0.5 秒）；`launch_app` 为 1 次。burst 每个动作最多尝试 1 次。
- 动作失败会尝试获取失败截图，动作执行和结果写入 DataEngine step/trace。
- Validator 生成结构化 Execution Incident。常见类别包括 `target_disappeared`、`target_shifted`、`target_occupied`、`pixel_target_disappeared`、`xml_bypassed`、`pixel_bypassed`、`general`。
- incident 通过图状态和下一轮提示交给 Operator，包含失败动作、原因、失败类型、动作序号/burst 大小、连续失败次数，以及可用的目标位移/遮挡证据。
- 没有独立自动修复 agent。下一轮 Operator 必须先重新读 incident 并观察当前界面，再决定重试、改路径、等待或跳过已不需要的意图。后续动作成功只会关闭执行通道的 incident；Operator 仍需确认原目标是否已实现。

执行报告大致形状：

```json
{
  "execution": [{"action": "tap", "attempts": ["..."]}],
  "status": "failed",
  "burst": false,
  "incident": {
    "kind": "safety_net",
    "category": "target_shifted",
    "reason": "...",
    "action": {},
    "action_description": "...",
    "action_index": 0,
    "burst_size": 1,
    "step_number": 4,
    "consecutive_failures": 1,
    "evidence": {}
  }
}
```

报告和状态结构见 [execution_loop.py](../artemis/agents/validator/execution_loop.py)、[incidents.py](../artemis/agents/validator/incidents.py)；面向 Operator 的 incident 文本见 [prompts.py](../artemis/agents/operator/prompts.py)。

### 5.2 设备观察后端错误

截图成功但层级解析失败时，观察仍可携带截图并将 `hierarchy_ok=false` 标明；调用方可保留旧索引，而不把空列表冒充为可靠的新 UI。截图自身失败则 `ObserveResult.ok=false`。

默认的 `ARTEMIS_HIERARCHY_BACKEND=auto` 先用 Accessibility Helper，失败时按策略回退 UIAutomator2；`helper` 模式只用 helper，便于让错误显式暴露；`uiautomator` 模式使用 UIAutomator2。Helper 客户端对只读请求可重建 ADB tunnel 并重试一次；带 payload 的动作请求不会自动重放，因为动作可能已经执行。见 [screen_client_factory.py](../artemis/clients/screen_client_factory.py)、[accessibility_client.py](../artemis/clients/accessibility_client.py)。

### 5.3 MCP / LLM 错误

设备动作的 MCP 调用异常会作为执行错误返回；不支持的工具或 schema 错误则属于协议层错误。`ActionSession` 将所有 transport 请求放在其 owner task 串行执行；调用者超时只会放弃等待，不会取消正在执行的底层 MCP 请求（取消可能破坏共享内存 transport）。若 transport 本身退出，下一次获取 session 会创建新会话。

Explorer 的原生 Gemini 调用使用共享可靠性层对可重试错误退避重试，并将永久失败或重试耗尽显式记录；其他模型/代理还受各自 timeout 和 structured-output 处理约束。这类重试是模型/transport 可恢复性措施，不代表设备动作可以安全重放。

## 6. Python 控制层到 Android 设备的通信架构

### 6.1 默认本地 Android 动作通道

```text
Operator 产生的动作
  → Operator 转换为 structured_decisions
  → Validator.action_execution
  → ActionSession（in-process MCP client/server，串行队列）
  → build_action_server（只注册 actuator 支持的工具）
  → AdbActuator
  → UnifiedMobileController
  → AndroidAdbDriver
  → adbutils / ADB server
  → adb shell input tap/swipe/keyevent、应用管理等
  → Android 设备
```

驱动工厂默认创建 Android ADB 驱动；mock 和云设备会走各自的驱动/客户端配置。关键实现：[action_session.py](../artemis/mcp/action_session.py)、[action_server.py](../artemis/mcp/action_server.py)、[actuators/adb.py](../artemis/mcp/actuators/adb.py)、[unified_controller.py](../artemis/controllers/unified_controller.py)、[adb_driver.py](../artemis/drivers/android/adb_driver.py)、[factory.py](../artemis/drivers/factory.py)。

以点击为例，actuator 接收 0–1000 的归一化点，根据设备宽高换算成像素，再由 Android ADB driver 通过 `adb shell input tap x y` 发送。滑动通过 `input swipe`，按键通过 Android keyevent。文本输入优先尝试剪贴板写入后粘贴；也可使用 ADBKeyboard，最后回退到 ADB `input text`。这描述的是默认驱动的主路径。

### 6.2 UI 层级 / 截图观察通道

屏幕观察同样经统一 Controller 和 device screen client 获取。默认 `auto` 后端会用 Artemis Accessibility Helper 提供层级；Helper 在设备本机 `127.0.0.1:18888` 监听，host 通过 ADB `forward` 暴露本地端口，并用随机 session token 认证请求。Android 11+ Helper 可在一个 snapshot 请求中同时取得 hierarchy 和截图；较老系统则可用 ADB screencap 补截图。Helper 不占用 Android 的 UiAutomation 单例连接，因而能与 Appium/Mobly/Espresso 并存。

若 Helper 不可用，auto backend 可回退 UIAutomator2；单次屏幕数据也会统一解析成下游可用的 UIAutomator 元素字典形状。注意：Helper APK 自身还实现 AccessibilityService `dispatchGesture` 手势接口，但常规 `AdbActuator → AndroidAdbDriver` 点击/滑动走的是上节所述 ADB shell 输入路径；不要把“Helper 能执行手势”误认为默认主动作都经 Helper 的 `/action` 发送。

实现见 [helper_manager.py](../artemis/runtime/helper_manager.py)、[accessibility_client.py](../artemis/clients/accessibility_client.py)、[screen_client_factory.py](../artemis/clients/screen_client_factory.py)、[CommandServer.java](../packages/artemis-accessibility-helper/app/src/main/java/com/artemis/helper/CommandServer.java)、[GestureController.java](../packages/artemis-accessibility-helper/app/src/main/java/com/artemis/helper/GestureController.java)。

### 6.3 MCP 服务边界

内部默认是进程内 MCP，而不是每次动作启动一个子进程。MCP server 根据 actuator manifest 注册所支持的工具；必需动作缺失会在 session 建立时被发现，未实现的可选动作则不会暴露给模型。wire 层只接收规范参数（如归一化坐标、时长），模型可用的索引目标/自然语言描述在客户端 adapter 中先转换，模型 schema 与设备传输 schema 因此分层。

仓库也保留可独立运行的 stdio ADB MCP server，适用于外部 MCP 客户端；它不是 Validator 默认执行所使用的 in-process session 路径。见 [adb_server.py](../artemis/mcp/adb_server.py)。

## 7. 持久化、追踪与验证

DataEngine 会保存 step、动作 trace、截图及对应 UI/OCR 数据，供执行历史摘要、回放、故障分析与后续验收使用。每个动作可记录开始/结束、成功标记、错误、执行耗时、归一化目标、burst 属性和后截图标识。Checker 的 verdict 包含验收项文本、类型、`passed/failed/inconclusive` 状态、具体证据和建议；机器侧据此决定是否继续、重试或以 blocked outcome 结束。

因此，项目的“准确性”是多层防线而不是单次模型调用的保证：多模态观察与结构化元素索引、schema/参数检查、单动作执行前核验、归一化坐标映射、串行动作通道、逐轮重新观察、incident 反馈、最终独立验收共同降低误操作和误判。快速 burst、无法取得可靠视觉判断、以及“命令已派发但界面未生效”等情形仍需后续观察与验收处理。

可进一步参考 [ARTEMIS_TASK_LIFECYCLE_GUIDE_CN.md](./ARTEMIS_TASK_LIFECYCLE_GUIDE_CN.md)、[ARTEMIS_AGENTS_GUIDE_CN.md](./ARTEMIS_AGENTS_GUIDE_CN.md)、[ARTEMIS_TOOLS_MCP_GUIDE_CN.md](./ARTEMIS_TOOLS_MCP_GUIDE_CN.md)。
