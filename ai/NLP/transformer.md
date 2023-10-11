# transformer
- 由 Encoder 和 Decoder 两个部分组成,分别由多个block组成
- 输入矩阵 = 单词Embedding+单词位置Embedding，纬度：单词个数*向量纬度，每行标识一个单词
- 输入矩阵经过Encoder，得到编码矩阵
- 解码过程中，需要掩码掩盖i+1的信息，不让获取
- Transformer 与 RNN 不同，可以比较好地并行训练。
- Transformer 本身是不能利用单词的顺序信息的，因此需要在输入中添加位置 Embedding，否则 Transformer 就是一个词袋模型了。
- Transformer 的重点是 Self-Attention 结构，其中用到的 Q, K, V矩阵通过输出进行线性变换得到。
- Transformer 中 Multi-Head Attention 中有多个 Self-Attention，可以捕获单词之间多种维度上的相关系数 attention score。
## 输入
- 单词的 Embedding 有很多种方式可以获取，例如可以采用 Word2Vec、Glove 等算法预训练得到，也可以在 Transformer 中训练得到
- 位置 Embedding：保存单词在序列中的绝对或者相对位置
## 自注意力 Self-Attention
- 输入：Q(查询),K(键值),V(值)，每一行都表示一个单词
- Q乘以K的转置后，得到的矩阵行列数都为 n，n 为句子单词数，这个矩阵可以表示单词之间的 attention 强度
- 除以向量纬度，防止内积过大
- Softmax 计算每一个单词对于其他单词的 attention 系数
- Softmax 矩阵之后可以和V相乘，得到最终的输出Z
## Multi-Head Attention
- 含多个 Self-Attention 层
- 对多个Self-Attention 层的结果拼接，然后做线性变换
## 编码
- 由 Multi-Head Attention, Add & Norm, Feed Forward, Add & Norm 组成的
- Add是残差连接，和resnet一样，让网络关注差异
- Norm：将输入转换为均值方差，加快收敛
- Feed Forward：两层全连接
## 解码
- 包含两个 Multi-Head Attention 层。
- 第一个 Multi-Head Attention 层采用了 Masked 操作。Mask 操作是在 Self-Attention 的 Softmax 之前使用的
- 第二个 Multi-Head Attention 层的K, V矩阵使用 Encoder 的编码信息矩阵C进行计算，而Q使用上一个 Decoder block 的输出计算。
- 最后有一个 Softmax 层计算下一个翻译单词的概率。
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
