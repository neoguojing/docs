# 语言模型整理
- 大语言模型无监督训练方法是 "从互联网上的大量语料库中学习根据上文来预测下一个词"，它做的是个生成任务
- 已有的大语言模型似乎并不能遵循用户的意图，而且还可能会输出不真实、有毒的内容或者是废话。换句话说，这些模型与他们的用户没有对齐 (Align)。
## 数据集
- https://huggingface.co/datasets/HuggingFaceH4/stack-exchange-preferences
## 伯克利 vicuna 
## meta LLaMA-2
- LLaMA 2 其实是两种模型：LLaMA 2 和 LLaMA 2-CHAT，分别是仅仅预训练过的模型，和预训练过之后再经过人类指令微调的模型
- LLaMA 2 的输入度由 LLaMA 的 2k 增加到 4k，训练数据由 1.4T tokens 增加到 2.0T tokens
- Tokenizer 的做法和 LLaMA 一样，是基于 SentencePieceProcessor[4]，使用 bytepair encoding (BPE) 算法。
- RMSNorm[5] 归一化函数对每个 Transformer 的子层的输入进行归一化
- SwiGLU 激活函数[6]替换 ReLU 非线性以提高性能
- LLaMA 去掉了绝对位置编码，使用旋转位置编码 (Rotary Positional Embeddings, RoPE)
- 分组查询的注意力机制 (Grouped-Query Attention, GQA)
- 优化器使用 AdamW，，使用 cosine 学习率衰减策略，2000 步的 warm-up，最终学习率等于最大学习率的 10%，使用 0.1 的权重衰减和 1.0 的梯度裁剪
- 有监督微调：Supervised Fine-Tuning (SFT)
- 人类反馈强化学习：Reinforcement Learning with Human Feedback (RLHF)
## open ai Dall-E 2
## stable diffusion
## google T5 Flan T5

## Pile 大型公共数据集
