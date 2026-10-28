## AppCard 是什么？

**AppCard 是针对某个移动 App 的简明操作指南**，一般用 Markdown 编写，提供该应用特有的界面位置、操作习惯、搜索语法和常见流程。例如，Gmail 卡片说明怎样打开文件夹、使用搜索过滤器、撰写邮件。项目内置示例见 `gmail.md`。

它不是设备当前 UI 状态，也不是自动化脚本，而是补充给 Agent 的**应用领域知识**。当前设备 UI 仍由 StateProvider 提供；AppCard 告诉 Agent 如何解释和操作特定应用。

## 为什么需要？

没有 AppCard 时，Agent 主要依赖通用知识和当前屏幕信息，可能需要反复探索应用特有的导航、手势或功能。AppCard 把这些可复用知识提前给 Agent，目标是减少试错和多余步骤，帮助完成应用内的复杂任务。

它也不是必需条件：没找到对应卡片时，Agent 仍会基于常规提示和设备状态工作，只是少了针对该应用的补充说明。

## 架构与加载流程

```text
设备状态中的当前应用
  → Manager 读取 package/activity 与 platform
  → AppCardProvider 按 app ID 和平台查找卡片
  → 内容写入 MobileAgentState.app_card
  → 注入 Manager / Executor prompt
  → LLM 将卡片作为应用操作背景
```

架构支持三种 provider：

- **Local**：从本地文件加载，适合开发和自定义内容。
- **Server**：通过 HTTP 服务获取卡片。
- **Composite**：优先请求服务端，服务端没有返回内容时再查本地。

通用接口是 `AppCardProvider.load_app_card()`，具体实现分别在 `LocalAppCardProvider`、`ServerAppCardProvider` 和 `CompositeAppCardProvider`。

## 在代码里哪里被使用？

1. **选择 provider**：Manager 根据 `app_cards.enabled` 和 `mode` 创建对应 provider；配置定义在 `AppCardConfig`，工厂逻辑在 `ManagerAgent._initialize_app_card_provider()`。
2. **按当前应用加载**：Manager 根据当前 package name、任务指令和平台调用 provider，再把结果存入共享状态 `MobileAgentState.app_card`。入口见 `ManagerAgent._load_app_card()`；加载发生在 `prepare_context`。
3. **注入提示词**：Manager 把 `app_card` 作为 prompt 变量；Executor 也从共享状态取同一内容。见 `Manager prompt 变量` 和 `Executor prompt 变量`。对应 prompt 模板中，只有变量非空时才插入卡片： `Manager 模板`、`Executor 模板`。

当前文档说明 AppCard 供 **reasoning 模式的 Manager 和 Executor** 使用；Stateless Manager 不加载 AppCard。Direct/FastAgent 模式也不经过这个 Manager 加载流程。

## 如何使用？

默认配置启用本地卡片，可在 `config_example.yaml` 中看到 `enabled: true`、`mode: local` 和卡片目录设置。文件按平台组织，用 JSON 将 App ID 映射到 Markdown 卡片：

```text
config/app_cards/
├── android/
│   ├── app_cards.json
│   └── gmail.md
└── ios/
    ├── app_cards.json
    └── gmail.md
```

Android 的 `app_cards.json` 示例：

```json
{
  "com.google.android.gm": "gmail.md"
}
```

创建后用 reasoning 模式运行，并开启 debug 日志确认加载信息：

```sh
mobilerun run "搜索 Gmail 中来自某人的邮件" --reasoning --debug
```

还可以通过配置切换到 server 或 composite 模式。具体目录规则、HTTP 参数、缓存及配置说明见 `App Instruction Cards 文档`。

## 使用时需要注意

- **按平台区分**：Android 使用 package name，iOS 使用 bundle ID；本地目录中 `android/`、`ios/` 各自维护 `app_cards.json`。
- **工作目录卡片会替代 bundled cards**：自定义目录不是与内置卡片自动合并；需要保留的卡片也应加入自定义映射。
- **Server 模式会发送任务指令**：请求 payload 包含 `package_name` 和 `instruction`，可选地包含 `platform`。配置远程服务时要考虑任务文本的数据边界，见 `ServerAppCardProvider`。
- **Stateless Manager 不加载**：如果配置了 `manager.stateless: true`，不要预期它会自动取得 AppCard。
- **已有测试覆盖**：平台映射、缓存/路径解析、Server 请求平台参数、Composite 本地回退，以及 Executor prompt 注入均有测试，见 `test_app_card_platforms.py` 和 `test_app_cards.py`。
