# transformer
- 由 Encoder 和 Decoder 两个部分组成,分别由多个block组成
- 输入矩阵 = 单词Embedding+单词位置Embedding，纬度：单词个数*向量纬度，每行标识一个单词
- 输入矩阵经过Encoder，得到编码矩阵
- 解码过程中，需要掩码掩盖i+1的信息，不让获取
- Transformer 与 RNN 不同，可以比较好地并行训练。
- Transformer 本身是不能利用单词的顺序信息的，因此需要在输入中添加位置 Embedding，否则 Transformer 就是一个词袋模型了。
- Transformer 的重点是 Self-Attention 结构，其中用到的 Q, K, V矩阵通过输出进行线性变换得到。
- Transformer 中 Multi-Head Attention 中有多个 Self-Attention，可以捕获单词之间多种维度上的相关系数 attention score。
> Input sequence -> Embedding + Positional Encoding -> N x Encoder Layer -> Context
> Context + Decoder Input -> N x Decoder Layer -> Output probabilities
## Tokenizer
### 为什么tokenize
- 计算机只能识别数字
- 词表问题：解决词表过大（英文单词太多），
## Embding

## Encoder-Decoder 架构（原始 Transformer）

## Encoder：把输入序列编码成上下文表示

## Decoder：根据上下文生成输出序列

## Self-Attention 机制

## Position Encoding

## Feed-Forward Network (FFN)

## Layer Normalization

## Residual Connection

## Masked Attention（在 Decoder 中）
## 投影
- q_proj：q_proj 是指查询投影（Query Projection）。在自注意力机制中，输入被分为查询（query）、键（key）和值（value）三部分。q_proj 负责将查询部分进行线性变换投影，以便与键和值进行匹配。

- v_proj：v_proj 是指值投影（Value Projection）。在自注意力机制中，v_proj 负责将值部分进行线性变换投影，以便与查询进行匹配。

- k_proj：k_proj 是指键投影（Key Projection）。在自注意力机制中，k_proj 负责将键部分进行线性变换投影，以便与查询进行匹配。

- o_proj：o_proj 是指输出投影（Output Projection）。在自注意力机制中，o_proj 负责将经过自注意力计算后的结果进行线性变换投影，以产生最终的输出。

- gate_proj：gate_proj 是指门投影（Gate Projection）。在某些模型中，门机制用于控制信息的流动。gate_proj 负责对门的输入进行线性变换投影，以计算门的开启程度。

- down_proj：down_proj 是指下投影（Down Projection）。在某些结构中，down_proj 负责将输入向下投影到较低维度的特征空间，常用于降低计算复杂度或提取更抽象的特征。

- up_proj：up_proj 是指上投影（Up Projection）。在某些结构中，up_proj 负责将输入向上投影到较高维度的特征空间，常用于恢复特征的维度或扩展表征能力。
- c_attn：c_attn 是指上下文注意力（Context Attention）。在一些模型中，c_attn 用于计算上下文信息的注意力权重，以便在生成输出时对输入的相关部分进行加权考虑。

- c_proj：c_proj 是指上下文投影（Context Projection）。在一些模型中，c_proj 负责对上下文信息进行线性变换投影，以产生最终的上下文表示。

- w1：w1 是一个参数或权重矩阵，常用于模型中的计算过程。具体的含义可能因上下文而异，需要根据具体的模型和上下文来确定 w1 的作用和功能。

- w2：w2 也是一个参数或权重矩阵，常用于模型中的计算过程。具体的含义可能因上下文而异，需要根据具体的模型和上下文来确定 w2 的作用和功能。
- wte：wte 是一个缩写，通常表示词汇表嵌入（Word Token Embedding）。在自然语言处理中，词汇表嵌入是将文本中的单词或标记映射到低维度向量表示的过程。wte 可能是特定模型或框架中用于表示词汇表嵌入的术语。
- lm_head：lm_head 是一个缩写，通常表示语言模型头（Language Model Head）。在语言模型中，lm_head 是模型的最后一层或输出层，负责将模型的隐藏表示转化为对应词汇表的概率分布。lm_head 用于生成下一个词或预测文本的下一个标记。

## 概念
- sequence length：指输入给模型的文本序列的长度；长的序列长度通常意味着更多的上下文信息。这可以帮助模型更好地理解和生成文本，但也会增加计算和内存的需求
- n_layers:指的是编码器（Encoder）或解码器（Decoder）中的自注意力层（self-attention layer）的数量；主要关注模型的深度，即自注意力层的数量
- n_heads:指的是多头自注意力机制中的注意力头的数量。每个注意力头都有自己的权重矩阵，通过计算输入序列的查询、键和值来产生注意力权重
- d_model 是 Transformer 模型中的一个超参数，指定了隐藏层的维度或特征维度
- "vocab size"（词汇表大小）是指在自然语言处理任务中使用的词汇表中不同单词的数量
