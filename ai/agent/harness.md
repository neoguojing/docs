
# 企业级 Agent Harness (智能体治理基座) 架构设计方案

## 核心定位

**Agent Harness** 并非重新发明底层的 LLM 路由算法，而是建立在 LangGraph 等状态机之上的“确定性防线与上下文工程引擎”。其使命是用工程架构的刚性边界，驯服大模型概率性输出带来的“幻觉、上下文腐烂与越权破坏”。

## 一、 Harness 全景架构拓扑

采用四层架构设计，逐层提纯上下文并收拢安全边界：

* **Layer 1: 规划与状态外化层 (Planning & State)**
* 目标解析 (Goal Parsing)
* 全局进度板 (Externalized TODO List)


* **Layer 2: 上下文工程引擎 (Context Engineering Engine)**
* 虚拟文件系统 (Virtual FS IO)
* 渐进式技能树 (JIT Skills)
* 子智能体隔离 (Sub-Agents)
* 滑动摘要 (Summary)


* **Layer 3: 声明式权限防线 (Security & ACL Firewall)**
* 路由绑定沙箱 (Backends)
* 基于路径的首匹配拦截 (FilesystemPermission)
* 人类在环中断挂起 (HITL Interrupt)


* **Layer 4: 确定性运行时与持久化 (Runtime & Memory)**
* LangGraph 状态机流转 (State Graph)
* 跨会话记忆库 (Memory Store)



## 二、 核心中枢设计细节

### 1. 状态与规划中枢 (State Externalization)

* **核心机制**：建立**外化进度板 (TODO List)**。Agent 不再依赖 Prompt 中的海量历史记录来推断下一步动作，而是直接读取 Planner 维护的 TODO 状态。
* **架构价值**：根除多轮对话中的“目标漂移 (Drift)”，将长任务分解为确定性的短路径。

### 2. 上下文工程中枢 (Context Engine)

* **虚拟文件系统 (Virtual FS)**：提供沙箱文件读写工具。长篇搜索结果或代码分析直接写入物理文件，主 Prompt 中仅保留“文件指针（路径）”和核心摘要，防止上下文腐烂。
* **渐进式装载 (Progressive Disclosure)**：执行特定任务时，动态注入对应领域的 SOP（如 `skills/database_SOP.md`），用完即卸载，维持 System Prompt 的高缓存命中率。
* **子节点隔离 (Subagent Isolation)**：子 Agent 产生的检索记录和报错堆栈随其生命周期结束而物理销毁，仅向主 Agent 返回提纯后的 `Summary`，实现上下文防污染。

### 3. 声明式权限防线 (Declarative Permissions)

* **基于路径的 Glob 管控**：底层注入 `FilesystemPermission`，采用**首匹配生效 (First-match-wins)**。高危特异性规则前置（如拒绝访问 `/secrets/**`），兜底泛化规则后置。
* **原生三元态阻断 (HITL)**：触发高危路径操作时，状态变更为 `mode="interrupt"`，DAG 图状态被持久化 Checkpoint 物理挂起，等待人类工程师 Review 审批。
* **子节点权限降级 (PoLP)**：派生审查 Agent 等子节点时，强制覆写为 Read-Only 模式，从根本上锁死漏洞爆炸半径。
* **复合后端路由绑定 (Composite Backend Routing)**：权限管控规则必须挂载在指定的后端路由命名空间下，防止 Agent 通过沙箱中的 Bash 命令绕过 Python 层的路径管控。

## 三、 架构设计核心哲学 (面试答辩精炼)

> 🛡️ **论工程边界：“用架构的上限，兜底 LLM 的下限”**
> “我绝不信任大模型在 Prompt 里做出的‘安全承诺’。系统的安全性必须建立在声明式的 `FilesystemPermission` 和路由级沙箱之上。工程的作用，就是用物理边界约束模型的概率性幻觉。”

> 🧩 **论多智能体协作：“双重隔离与最小权限”**
> “引入 Subagents 是为了实现上下文降噪与权限降级。脏数据被就地销毁只传 Summary；读写权限被强制降级，完美落地最小权限原则 (PoLP)。”

> 🧠 **论长文本悖论：“信噪比永远大于 Token 数量”**
> “Harness 的核心是上下文工程 (Context Engineering)——通过 TODO 状态外化、虚拟文件系统和能力按需加载 (JIT)，始终向模型投喂信噪比最高的极简上下文，彻底解决‘中间迷失’问题。”
