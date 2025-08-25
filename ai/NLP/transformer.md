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
- 词表问题：解决词表过大（英文单词太多），未登录词（不在词表中的词）组合问题（期望通过组合得到任意词）
### tokennizer方法
- 单词级别：词表过大
- 字符级别： 句子的token标识过长
- 子词级别：权衡上面两者：BPE和WordPiece
### BPE
- 主要依据 出现频率 进行合并：每次选择训练语料中出现最频繁的符号对（字符或子词），将其合并成一个新单元
#### 训练
- 把词拆成字符 + </w>；
- 反复：数所有相邻对的频次 → 合并出现最多的那个对 → 重写全语料；
- 直到达到预定词表大小或再也没有高频可合并对；
- 产出“合并规则表（有序）”，按照合并顺序建立的表，通常模型文件下的merge.txt
#### 推理
- 新文本同样先拆成字符 + </w>；
- 按合并规则表的顺序，能合并就合并，直到不能再合并。
### Byte-level BPE
> 从 UTF-8 字节开始，合并高频字节对构建子词
- 先把所有文本分解为字节
- 统计相邻对
- 按频率合并字节码，而不是字符（中文用3个字节编码）
- 得到最终词表
## Embding
- llm的Embding通过训练得到
- 纬度[词表大小，d]，d是模型的超参数，决定模型的表达力,有v行，d列
- 所有词投影到一个 d 维语义空间，相似意思的词聚在一起，意思差异大的词相隔开
### token id 到Embding
- 查询Embding矩阵，矩阵的每一行表示该token的向量
### embding的训练过程
- 初始化	随机或预训练向量，矩阵 E ∈ R^(V×d)
- 前向	token → embedding → 加位置编码 → Transformer → 输出概率
- 损失	语言建模或对比学习等目标
- 反向	梯度传回 embedding，只更新输入 token 对应行
- 优化	Adam/AdamW 等优化器更新 embedding
- 收敛后	embedding 空间形成语义结构，向量相似度表示语义相似性
### W_vocab输出词表向量矩阵
- 与embding共享权值
- 隐藏向量映射到词表概率空间的矩阵
### Hidden State
- 输入的中间层表示
- 每个 token 在每一层都有自己的 hidden state，随着层数增加，它的语义表达越来越丰富。
- Embedding	seq_len × d	输入 token → 初始向量
- Hidden State	seq_len × d	Transformer 内部中间表示，捕捉上下文语义
- Logits	seq_len × V	Hidden state × W_vocab^T → token 概率
### 位置编码
- 位置向量让 Transformer 知道“这个 token 在第几位
- 与 token embedding 相加 → Transformer 隐状态 hidden state 同时包含 语义 + 顺序
#### 正余弦位置编码
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
