# 自然语言处理

## 常用模型架构

> 注：以下内容是常见的**模型/网络结构**，而不是框架（PyTorch、TensorFlow 等才是框架）。

### 循环神经网络
- RNN：最基础的循环神经网络，通过隐状态逐步读取序列，但存在梯度消失/爆炸问题，难以捕捉长距离依赖
- LSTM：在 RNN 基础上引入门控机制（输入门、遗忘门、输出门）和细胞状态，缓解长距离依赖问题
- - 转了90读的残差网络
- GRU：LSTM 的简化版，将门控合并为更新门和重置门，参数更少，效果与 LSTM 接近

### 基于 Transformer 的模型
- Transformer：基于自注意力机制（Self-Attention）的编码器-解码器架构，天然支持并行训练，是现代大语言模型的基石
- BERT：基于 Transformer **编码器**的双向预训练模型（Bidirectional Encoder Representations from Transformers），适合理解类任务（分类、NER、QA）
- GPT：基于 Transformer **解码器**的自回归（causal）模型，通过预测下一个 token 的方式生成文本，适合生成类任务

### 序列标注
- CRF（条件随机场）：一种概率图模型，常用于命名实体识别（NER）等序列标注任务，与 BiLSTM 等特征提取器组合使用（BiLSTM-CRF）

## Tokenizer
- **Tokenizer** 的作用是将文本按照一定规则分割成一个个独立的词语或标记（tokens）
- 它可以将连续的文本序列转化为离散的语言单位，如单词、子词或字符，以便计算机能够更好地理解和处理文本数据
- 常见的分词粒度：
  - 词级（Word）：以单词为单位，如早期的 word2vec；对未登录词（OOV）不友好
  - 字符级（Character）：以字符为单位，词表小但序列过长
  - 子词级（Subword）：兼顾二者，如 BPE（Byte Pair Encoding）、WordPiece、Unigram LM；主流大模型普遍采用子词分词
- 常见工具：HuggingFace `tokenizers` 库、SentencePiece 等
- [https://huggingface.co/docs/tokenizers](https://huggingface.co/docs/tokenizers/quicktour)

## 概念
- 
### RLHF (Reinforcement Learning from Human Feedback)：基于人类反馈的强化学习
- 包括：预训练、SFT，打分模型（Bradley-Terry），PPO奖励
- 策略模型，参考模型，奖励模型和价值模型，超参数难调整
- 强化学习的核心思想：**通过**学习最优的策略（Policy），使代理（Agent）可以在给定的环境下选择出能够最大化长期奖励的行动序列
- 代理通过不断尝试和学习，优化其策略，使其能够在不同的状态下做出最佳的决策
- 核心要素：
  - 环境（Environment）：代理与之交互的外部环境，可以是真实世界、模拟环境或虚拟环境。环境会根据代理的行为给出反馈，例如奖励或惩罚
  - 代理（Agent）：强化学习的学习主体，它通过与环境的交互来学习最佳行为策略。代理基于当前状态选择行动，并接收来自环境的奖励或惩罚信号
  - 状态（State）：描述环境的当前情况和特征的变量。代理根据观察到的状态选择行动，并根据环境的反馈转移到新的状态
  - 行动（Action）：代理在给定状态下采取的行为或决策。代理根据当前状态选择行动来与环境进行交互
  - 奖励（Reward）：环境根据代理的行动提供的反馈信号。奖励可以是正数（表示鼓励）或负数（表示惩罚），用于评估代理的行为。代理的目标是最大化长期累积的奖励
- 在 LLM 中的 RLHF 训练流程通常分为三个阶段：
  1. **SFT（监督微调）**：用人类高质量问答数据微调预训练模型，使其学会"按指令回答"
  2. **奖励建模（RM）**：收集人类对同一 prompt 多个回答的偏好排序，训练奖励模型为回答打分
  3. **PPO 强化学习**：以奖励模型的分数作为奖励信号，用 PPO 等算法优化语言模型策略，使其输出更符合人类偏好

### DPO 数学替换
- 不训练奖励模型
- 只需要策略模型和参考模型
- 缺点：不会探索，偏好数据和当前数据存在偏移

### GRPO 可验证任务
- REINFORCE的变体 + PPO
- baseline 省略算法
- 适用于：数学和代码

### SFT（Supervised Fine-Tuning，监督微调）
- 数据格式： 多伦对话
- SFT 是一种迁移学习的方法，即有监督的微调
- **优**势在于它能够利用预训练模型在大规模数据上学到的通用特征和知识，并将其迁移到特定任务上
- 通常使用 (prompt, completion) 形式的指令数据，在预训练模型的基础上继续训练，使模型具备遵循指令、对话式生成的能力
- 是 RLHF 流程中的第一步，也是目前很多应用（如 ChatGLM、LLaMA 的指令版本）落地的基础

### PPO（Proximal Policy Optimization，近端策略优化）
- PPO 是一种策略梯度类的强化学习算法，用于训练策略（Policy），适用于连续动作空间和离散动作空间（LLM 的 token 生成即离散动作空间，RLHF 中大量使用 PPO）
- 核心思想：通过**裁剪（clip）**新旧策略的概率比率，限制每次策略更新的幅度，避免更新过大导致训练不稳定
- 相比 TRPO 等之前的算法，PPO 实现更简单、计算开销更低，是目前 RLHF 中最常用的优化算法之一

### 奖励 / 偏好建模（Reward / Preference Modeling，RM）
- 基于人类反馈的强化学习（RLHF）的关键组件
- 目标：训练一个奖励模型，给定 (prompt, response) 对，预测人类对该回答的偏好程度（打分）
- 训练方式：
  - 收集偏好数据：同一个 prompt 对应的多个回答，由人工标注出"更好/更差"的排序（chosen / rejected）
  - 损失函数常用成对比较损失（pairwise ranking loss）：`-log σ(r(chosen) - r(rejected))`，其中 σ 为 sigmoid，r 为奖励模型输出
- 在 RLHF 流程中，奖励模型为 PPO 提供奖励信号；训练完成后通常可丢弃，仅保留优化后的语言模型
- 注意：RM 学到的是"偏好"而非绝对质量，因此偏好数据的质量和覆盖度直接决定 RM 的效果
