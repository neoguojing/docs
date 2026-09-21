# Deepseek && 训练方法
## 监督学习

## 预训练
- 无监督学习
- 自监督学习:通过从数据本身构造监督信号（如遮蔽词预测、下一句预测等）来进行训练
- - GPT 采用因果语言建模（Causal Language Modeling, CLM），数据格式如下：
```
输入 X: "机器学习是一种"
目标 Y: "机器学习是一种 人工智能方法"
这里的目标 Y 由输入 X 右移一位得到，形成自回归学习目标。
```
- - BERT 采用掩码语言建模（Masked Language Modeling, MLM）：
```
输入 X: "机器学习是一种 [MASK] 方法"
目标 Y: "[MASK] = 人工智能"
这里 MASK 的内容是训练时从原始句子中随机遮蔽的词，模型需要预测被遮蔽的部分。
```
- 数据存储方式通常是纯文本（txt/jsonl）或 tokenized 格式，以适配 LLM 训练流程
## SFT 监督微调 Supervised Fine-Tuning
- 全参数微调
- 指令微调
```
{
    "prompt": "用户: 如何进行 SFT 训练？\n助手:",
    "completion": "SFT 训练需要有监督数据，使用交叉熵损失进行优化。"
}

```
- 链式推理（CoT）数据的长文本
- 模型蒸馏微调
### SFT 主要用于让 LLM 对齐人类期望，提升模型在特定领域的表现，如：
- 任务指令遵循（Instruction Following）
- 对话生成优化（Chatbot Training）
- 代码补全优化（如 Code Llama）
- 医学、法律、金融等领域的专用知识微调
### 参数高效微调（PEFT）

## Reinforcement Learning，RL 强化学习
- 强化学习是一种机器学习范式，它通过“试错”和“奖励反馈”让智能体（Agent）学习如何在环境（Environment）中做出最佳决策，以最大化长期累积奖励（Cumulative Reward）
### 强化学习广泛应用于多个领域，如：
游戏 AI（如 AlphaGo、Dota2 AI）
机器人控制（自动驾驶、机械臂控制）
金融交易（智能投资策略）
推荐系统（个性化推荐）
自然语言处理（对话系统优化，如 ChatGPT 的 RLHF）

### RLHF（人类反馈强化学习）
- PPO 近端策略优化 Proximal Policy Optimization
- - 强化学习优化（PPO）：使用强化学习优化 AI 生成的文本，使其更符合人类偏好
- GRPO 群体相对优化 Group Relative Policy Optimization
- 利用基于规则的奖励系统，主要包括准确性奖励（如数学问题的正确答案）和格式奖励（如思维过程的正确格式）
