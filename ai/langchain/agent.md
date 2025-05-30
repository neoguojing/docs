# agent

## ReAct resoning and action
- ReAct: Synergizing Reasoning and Acting in Language Models
> 人类在解决复杂任务时，会将内在的语言推理和外部动作紧密结合，通过不断“思考—行动—观察”的循环完成任务。论文提出希望借助大型语言模型实现这种推理与执行的协同，从而提升模型在知识密集型问答、事实核查以及交互式决策任务中的表现，同时改善模型的可解释性和鲁棒性。
- 推理（Reasoning）： 通过生成语言思考，模型能够拆解任务、追踪进展、调整计划并判断何时需要外部信息。
- 行动（Acting）： 模型通过具体动作与外部环境交互，例如搜索或查找信息，获取观察结果，再将结果反馈回推理过程。

## Multi-agent network
> 论文提出利用多智能体对话的方式，将任务求解过程抽象为智能体之间的交流与协作。多智能体不仅可以通过各自擅长的能力互补，还能通过对话形成动态、灵活的控制流，进而应对复杂且多样化的应用场景。这一方法能够激发多样性思维、提高推理准确性，并为开发新一代LLM应用提供统一且灵活的基础设施。
- AutoGen框架通过可对话智能体和对话式编程两大核心概念，实现了多智能体协同工作，极大简化了复杂LLM应用的开发流程。同时，它在模块化、可扩展性和人机混合协作方面表现出色，为不同领域的应用提供了灵活且高效的解决方案。
- Conversable Agents（可对话智能体）：
每个智能体都被设计为具备发送和接收信息的能力，能够维护自己的内部状态，并根据收到的消息做出响应。

智能体可以由LLM、工具、甚至人类输入来驱动，其能力可以通过配置进行定制，从而满足特定任务的需求。

- Conversation Programming（对话式编程）：
将复杂的任务求解流程视为一系列智能体之间的对话，重点在于对话中的“计算”（即各智能体执行相应操作生成回复）和“控制流”（即消息传递的顺序和条件）。

开发者可以利用自然语言和编程语言的融合方式，自由定义智能体之间的交互模式，无论是固定的静态对话还是动态、开放式的多智能体协作。

- 多智能体协同：
框架支持多种对话模式，例如一对一对话、群聊以及层级对话等。

通过定义明确的角色（如助手、用户代理、执行者、监管者等），各个智能体可以分工协作，互相补充，从而实现任务的高效求解。

## Plan and resolve
>  Plan-and-Solve (PS) Prompting 方法，旨在改进 零样本（Zero-shot）链式思维（Chain-of-Thought, CoT）推理，使得大型语言模型（LLMs）能够更有效地解决多步推理问题。核心思想包括：
- 先规划 (Plan)：在解题前，先让模型生成一个解决方案的计划，将任务拆解为多个子任务。
- 再求解 (Solve)：按照计划逐步执行推理过程，生成最终答案。
- Explicit long term planning (which even really strong LLMs can struggle with)
- Ability to use smaller/weaker models for the execution step, only using larger/better models for the planning step

## Reflection
> 在 LLM 智能体（LLM Agent）构建的背景下，反思（Reflection） 指的是通过提示（Prompting）引导 LLM 回顾其过去的执行步骤（包括来自工具或环境的观察结果），以评估所选行动的质量。这一过程随后可用于 重新规划（Re-planning）、搜索（Search）或评估（Evaluation） 等下游任务。
### ToT
> 《Tree of Thoughts: Deliberate Problem Solving with Large Language Models》提出了一种新的语言模型推理框架——“思想树”（Tree of Thoughts, ToT），旨在克服传统自回归语言模型（LM）在复杂问题解决中的局限性。传统方法如“思维链”（Chain of Thought, CoT）依赖逐 token、从左到右的生成，缺乏探索性和前瞻性规划能力，而 ToT 通过构建一个思想树，允许模型进行多路径推理、自我评估和全局决策，从而显著提升问题解决能力。ToT 借鉴了人类认知中的“双过程理论”（System 1 和 System 2）以及经典人工智能的搜索方法，将语言模型的生成能力与系统性搜索算法结合，适用于需要规划和探索的任务。
ToT 将问题解决过程建模为在思想树上的搜索，每个节点代表一个部分解（状态），每个分支代表一个推理步骤（思想）。其核心方法包括以下四个关键步骤：
- 思想分解：根据问题特性，将解决过程分解为若干中间思想步骤。思想是一个连贯的语言序列，规模适中，既便于生成多样性样本，又足以评估其进展。
- 思想生成：从当前状态生成多个候选思想，可通过独立采样（适用于开放性任务，如创意写作）或顺序提议（适用于约束性任务，如24点游戏）。
- 状态评估：使用语言模型自我评估不同状态的进展，作为搜索启发式。可独立评估每个状态（生成数值或分类），或通过比较投票选出最优状态。
- 搜索算法：结合广度优先搜索（BFS）或深度优先搜索（DFS）等算法，系统性地探索思想树，支持前瞻和回溯。
## SELF-DISCOVER
> 《SELF-DISCOVER: Large Language Models Self-Compose Reasoning Structures》提出了一种创新框架——SELF-DISCOVER，旨在提升大型语言模型（LLM）在复杂推理任务中的能力。传统提示方法（如思维链 CoT）通常依赖单一、预设的推理模式，而 SELF-DISCOVER 允许模型从一组原子推理模块（如“批判性思维”、“逐步思考”）中自我发现并组合出任务特定的推理结构。这种方法受人类如何通过已有知识和技能设计问题解决策略的启发，强调任务固有的推理结构的重要性。SELF-DISCOVER 在性能上显著优于 CoT 等方法，同时计算效率高，且推理结构具有跨模型通用性和可解释性，与人类推理模式存在共通之处。
- 阶段 1：任务级推理结构发现
SELECT（选择）：从一组通用的原子推理模块描述（如表 2 列出的 39 个模块）中，基于任务示例（无需标签）选择相关模块。
ADAPT（调整）：将选中的模块描述重新表述，使其更贴合具体任务，例如将“分解问题”调整为“按顺序计算每个算术操作”。
IMPLEMENT（实现）：将调整后的模块转化为可操作的推理结构（JSON 格式），提供逐步指导。整个过程通过三个元提示（meta-prompts）引导 LLM，无需训练或标签。
- 阶段 2：实例级问题解决
使用阶段 1 生成的推理结构，提示 LLM 按结构逐步填充具体值，解决任务的每个实例。
