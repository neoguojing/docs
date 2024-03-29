# 自然语言处理

## 常用框架
- RNN
- LSTM
- GRU
- Transformer
- BERT： 基于Transformer
- GPT： 基于Transformer
- CRF

## Tokenizer
- 的作用是将文本按照一定规则分割成一个个独立的词语或标记（tokens）
- 它可以将连续的文本序列转化为离散的语言单位，如单词、子词或字符，以便计算机能够更好地理解和处理文本数据
- [https://huggingface.co/docs/tokenizers](https://huggingface.co/docs/tokenizers/quicktour)

## 概念
### RLHF (Reinforcement Learning)：基于人类反馈的强化学习
- 过学习最优的策略（Policy），使代理可以在给定的环境下选择出能够最大化长期奖励的行动序列。
- 代理通过不断尝试和学习，优化其策略，使其能够在不同的状态下做出最佳的决策
- 环境（Environment）：代理与之交互的外部环境，可以是真实世界、模拟环境或虚拟环境。环境会根据代理的行为给出反馈，例如奖励或惩罚。
- 代理（Agent）：强化学习的学习主体，它通过与环境的交互来学习最佳行为策略。代理基于当前状态选择行动，并接收来自环境的奖励或惩罚信号。
- 状态（State）：描述环境的当前情况和特征的变量。代理根据观察到的状态选择行动，并根据环境的反馈转移到新的状态。
- 行动（Action）：代理在给定状态下采取的行为或决策。代理根据当前状态选择行动来与环境进行交互。
- 奖励（Reward）：环境根据代理的行动提供的反馈信号。奖励可以是正数（表示鼓励）或负数（表示惩罚），用于评估代理的行为。代理的目标是最大化长期累积的奖励。
### SFT(Supervised fine-tuning)是一种迁移学习的方法，有监督的微调
- 势在于它能够利用预训练模型在大规模数据上学到的通用特征和知识，并将其迁移到特定任务上
### Proximal Policy Optimization（PPO）是一种强化学习算法，用于训练策略（Policy）来解决连续动作空间的强化学习问题

### 奖励 / 偏好建模 (Reward / preference modeling，RM)。
基于人类反馈的强化学习 (RLHF)。
