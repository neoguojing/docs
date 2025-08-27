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
## 参数分布：
| 层 | 参数 | 维度 |
|-----|------|------|
| **Embedding** | Token Embedding `E` | `(V, d_model)` |
|  | Position Embedding `P` | `(L, d_model)` |
| **Multi-Head Attention** | Query `W_Q` | `(d_model, h*d_k)` |
|  | Key `W_K` | `(d_model, h*d_k)` |
|  | Value `W_V` | `(d_model, h*d_v)` |
|  | 输出线性 `W_O` | `(h*d_v, d_model)` |
|  | 偏置 `b_Q, b_K, b_V, b_O` | `(h*d_k / d_v / d_model)` |
| **Feed Forward Network** | 第一层权重 `W_1` | `(d_model, d_ff)` |
|  | 第一层偏置 `b_1` | `(d_ff,)` |
|  | 第二层权重 `W_2` | `(d_ff, d_model)` |
|  | 第二层偏置 `b_2` | `(d_model,)` |
| **LayerNorm** | scale `γ` | `(d_model,)` |
|  | shift `β` | `(d_model,)` |
| **输出层** | 词表投影 `W_vocab` | `(d_model, V)` |
|  | 偏置 `b_vocab` | `(V,)` |

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
## Embding （Embding矩阵：（V，d））
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
- sin(pos+k) = sin(pos)cos(k) + cos(pos)sin(k)
- 基于此公式，我们可以推导位置间的关系，即位置关系可以线性推导
- 位置与相对位置k有关系
- 频率由2i/d 确定，分母越大频率越低
- 唯一性：sin 和 cos组合的唯一性
- 可扩展到任意长度
- 相对位置：通过相对位置理解距离
- PE(pos,k): pos是token的顺序位置，0<=k<d/2−1,计算出一个d维的向量，标识位置pos的token，在d维空间的位置
#### RoPE 旋转位置编码:旋转矩阵纬度（d,d）
- 复数：利用了复数的概念
- (x,y) 旋转一个角度的公式如下
-- x'=x⋅cos(θ)−y⋅sin(θ)
-- y′=x⋅sin(θ)+y⋅cos(θ)
- RoPE 的设计灵感来自复数。它将词向量的每一对维度（比如第 0 和 1 维，第 2 和 3 维等）看作一个复平面上的点。然后，通过一个与词语位置相关的角度，旋转这个复数向量。
- 两个向量的点积，在旋转后保持不变，但它们的相对位置信息会被编码到结果中
- 1.RoPE 的核心在于其点积的线性特性。数学上可以证明，两个旋转向量的点积，其结果只与它们的相对位置 m−n 有关。A⋅B=∣A∣⋅∣B∣⋅cos(α) q 
> qm′⋅kn′=∣qm∣⋅∣kn∣⋅cos((m−n)θ)
- 2.由于 RoPE 能够直接编码相对位置，它在处理比训练时更长的序列时，表现更加稳健
- 3.RoPE 将位置信息直接集成到了注意力机制的计算过程中
##### 计算过程
- m：词语在序列中的位置（从 0 开始）
- k：向量的维度索引，从 0 到 d/2−1
- d：向量的总维度
- 1.计算旋转角度：计算旋转角度 θk=10000^2k/d
- 2.构建旋转矩阵R,d维的对角方阵：将向量每两个作为一组，
- 3.应用旋转矩阵：x' = Rx  ,将向量X进行旋转
## Encoder：把输入序列编码成上下文表示
> Input -> Multi-Head Self-Attention -> Add & Norm -> Feed Forward -> Add & Norm -> Output
- Multi-Head Self-Attention输入序列中的每个 token 与其他 token 交互（没有 mask）。
- Add & Norm（残差 + LayerNorm）
- Feed Forward Network (FFN)
## Decoder：根据上下文生成输出序列
> Input (目标序列嵌入 + 位置编码) → Masked Multi-Head Self-Attention → Add & Norm → Multi-Head Attention over Encoder Output → Add & Norm → Feed Forward → Add & Norm → Output
> decode only：Input (嵌入 + 位置编码) → Masked Multi-Head Self-Attention → Add & Norm → Feed Forward → Add & Norm → Output
> 在 Decoder 里生成任务时，不能让模型看到未来的 token（即右侧的 token），否则会导致信息泄露
> Masked Self-Attention 的核心就是 在注意力矩阵上应用掩码（mask），禁止模型关注未来信息。
### 每一层 Decoder Layer 包含：
- Masked Multi-Head Self-Attention（防止看未来）
- Add & Norm
- Encoder-Decoder Attention（Query 来自 Decoder，Key/Value 来自 Encoder）
- Add & Norm
- Feed Forward Network
- Add & Norm
### 掩码注意力机制
- 掩码矩阵纬度：L * L
-- j <= i 的位置：值为0，保留原始值
-- j > i的位置，值为无穷大，Softmax 后对应权重为 0
- attention在softmax之前，加上掩码矩阵
- softmax之后，演变为一个下三角矩阵
## Self-Attention 机制 
> Q.K 是点积计算
> XW_Q、XW_K、XW_V 就是这样的矩阵乘法
> 
- 输入x纬度：序列长度L*d
- 输出纬度：L*d
- 单头注意力下：Q，K，V的投影参数矩阵w的纬度是：d * d
- 1.Q=XWQ,K=XWK,V=XWV： 输入和投影矩阵相乘得到Q K V
- - Q:要查询的信息；K:可以提供的信息，可检索标签；QK的点积：标识在K中寻找最相关的Q的信息，结果越大相关性越高
  - V表示真正返回给用户的信息，
- 2.attn = softmax(​​QKT/​d^(1/2))V
- - 取d的平方根：对点积的结果进行缩放，防止梯度爆炸
  - softmax取点积的概率
- 3：概率与V的加权：让 Query 从上下文所有 Value 中，按相关性做加权平均，得到一个带上下文信息的新表示
### 多头注意力（MHA），单个w（投影矩阵）的纬度（d，d/h）
- 将参数纬度d拆分为h个
- h个self attention的组合：通常8，16
- 每个head的输出纬度：L*d/h
- 将输出做拼接：让最后一个纬度变为d：L*d
- W0线性变换矩阵：将输出混淆：纬度d*d
- 关注不同的关系
- 规避模型盲点
- 并行性更好，效率更高
- 参数可控

## Feed-Forward Network (FFN)
> 参数量大于MHA
- 输入：L*d
- 1.升维：xW1; W1：升维矩阵，纬度：df*d，df一般为d的2-4倍；结果纬度L*df
- 2.偏置：加上参数b1，做平移；纬度为：df；偏置是逐维加的
- 3.激活函数：非线性激活：ReLU，GELU，SwiGLU；用于引入通用近似能力
- 4.降维操作：W2*h；W2：降维矩阵，纬度：df*d；
- 5.偏置：b2，纬度:d
### 特性：
- 逐位置（Position-wise）：每个 token 的向量独立通过 FFN，不考虑序列上下文关系
- 升维-降维结构：像瓶颈层（bottleneck），增加模型容量又能保持输出维度不变。
- 并行计算：不同 token 的 FFN 可以同时计算，非常适合 GPU/TPU
## Layer Normalization
- 出现在多头注意力和FFN后面，和残差一起使用
- 1.把输入减去均值，保证特征的平均值为 0
- 2.把每个特征除以标准差，保证特征分布的方差为 1
- 3.仿射变换：引入可训练的缩放 γ 和平移 β
## Residual Connection
- 残差连接：y = x + F(x)
- 出现在多头注意力和FFN后面
- 缓解梯度消失/爆炸
- 信息保留
- 加速收敛

## 输出层
> 隐藏状态 h_t -> 线性映射 (投影到词表维度) -> logits 向量 -> softmax 归一化 -> 概率分布 -> 解码策略采样/选择 -> token id -> tokenizer 解码 -> 文本
> 每次预测一个token，加入输入序列，重新生成，直到遇到eos token或达到最大长度，停止
> 实际实现中不会每次都重新跑整个序列，而是利用 KV Cache（缓存注意力计算），只计算新 token 的 hidden state
### 线性投影 参数与Embding共享：纬度（V，d）
- 每个 token 的隐藏向量 与 词表矩阵 W 做内积，得到每个 token 属于词表中每个词的 logits
- logits:未归一化分数；纬度：L*v
- 为什么要映射到v的纬度？隐藏状态是抽象的，只有词表能表示真实含义；便于交叉熵做对比
### Softmax
- 对logits做softmax归一化，得到概率分布
### token选择策略
> 归一化采样 = 先筛掉一部分候选，再重新调整概率和为 1，然后再采样
- 贪心策略：选择最高的概率
- 随机选择
- TopK采样：只保留前 k 个最高概率 toke
- Top-p：取累计概率 ≥ p 的最小候选集合
- Beam Search：同时保留多个候选序列，选择整体概率最大的序列
### 交叉熵损失函数（训练）
- 训练时用于评估，做反向传播，更新参数
## kv cache
> 这样每次只算 1 个 token 的 Q/K/V，而历史的 KV 来自 cache，不用重复计算。
> HuggingFace Transformers：在 generate() 里自动维护 past_key_values（就是 KV cache）
> vLLM：用了更高级的 PagedAttention（把 KV 存在一个“大内存池”里，减少内存碎片，提高显存利用率）
> TensorRT / FasterTransformer：KV cache 会存到 GPU 的高效 buffer 里，避免频繁拷贝
- 存储MHA每层的 Key/Value 张量（维度 = [batch, heads, seq_len, head_dim]）
- 每次生成新 token 时只算它的 K/V
- 把新 K/V append 到缓存
- Attention 查询时用历史缓存 + 新 Q 计算
- 框架里一般通过 past_key_values 参数传入和更新

## 长上下文扩展
> 序列越长，计算量和显存消耗呈平方增长
降低复杂度：
- 稀疏注意力（Sparse Attention）:每个 token 只和局部窗口或少量全局 token 做注意力
- 滑动窗口 / 分块 Attention将序列切块，只在块内做 attention，再跨块传递信息
- 分段训练 / Gradient Checkpoint大序列分段训练，显存累积梯度
- KV Cache（主要用于推理）

不重复计算历史 token 的 K/V
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
- d_model 代表了模型中所有向量的维度，包括词向量（Word Embeddings）、位置编码向量（Positional Encodings）、以及注意力机制中的 Q、K、V 向量的维度。
- "vocab size"（词汇表大小）是指在自然语言处理任务中使用的词汇表中不同单词的数量
