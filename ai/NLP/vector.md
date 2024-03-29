# Embedding 嵌入技术
- 词向量是自然语言处理中表示词的数值化表示法,主要的计算方法包括:
- 词向量*权重矩阵 = Embedding矩阵，是一个概率矩阵，反映词之间的关系
- 也说Embedding是从原始数据中提取的特征
- 对One-hot向量进行降维
```
embedding的每维特征都可以看出词的一个特征，比如人可以通过身高，体重，地址，年龄等多个特征表示，对于每个词embedding的每个维度的具体含义，不用人为定义，模型自己去学习。
这样，在d维空间上，语义相近的词的向量就比较相似了，同时embedding还能起到降维的作用，将one-hot的[s,vocab_size]大小变成了[s,d]。
```
## One-hot向量
One-hot向量使用高维稀疏向量来表示词,将词映射到一个大的向量空间中,每个词对应一个索引位置,将该位置设为1,其他位置为0。这种方法可以区分不同词,但没有表征词的语义信息。

## 词频-反文档频率(TF-IDF)
TF-IDF利用词频和反文档频率来表示词的重要程度。词频衡量词在文档中出现的频率,反文档频率衡量词的稀有程度。TF-IDF能反映出词的重要性,但也没有词的语义信息。

## Word2Vec
Word2Vec通过神经网络学习得到词的分布式向量表示,可以表示词的语义信息。主要有CBOW和Skip-Gram两种模型,能学到词的语义类比关系。
- Skip-Gram：用输入单词作为中心单词去预测周边单词的方式
- CBOW：用输入单词作为周边单词去预测中心单词的方式叫
```
# 一般embedding在语言模型的最开始，也就是词token操作之后
# vocab_size 词汇表大小，hidden_size 隐藏层维度大小
word_embeddings= nn.Embedding(config.vocab_size, config.hidden_size, self.padding_idx)
# input_ids是句子分词后词id，比如上述的“我喜欢”可转换成为[0,1,2],数字是在词汇表【我，喜，欢，吃,面】中的索引，即token id
embeddings = word_embeddings(input_ids)  # embeddings的shape为[b,s,d],b:batch,s:seq_len,d:embedding size
```

## GloVe
GloVe通过统计词与词之间的共现关系生成词向量,也能表示词的语义信息。它利用统计方法和词的全局共现矩阵进行矩阵分解学习词向量。

## BERT
BERT使用大规模无监督语料 pretrain 深层双向语言模型,得到词、句子的语义表示。可 fine-tune 应用到下游NLP任务中。

# 分词
## 分词（Word Segmentation）：
在自然语言处理中，分词是将连续的文本序列切分成一系列有意义的词或标记的过程。分词是基本的文本预处理任务，将连续的字符序列切分为有意义的单词有助于后续的语言理解和处理。
## 子词化（Subword Segmentation）：
子词化是将词或短语进一步切分为更小的单元（子词）的过程。子词化旨在处理词汇中的复杂性，如未登录词、复合词和低频词。通过将单词切分为子词，可以更好地捕捉语言中的内部结构和语义信息。
## 标记：（Token）
是指将文本切分成较小的单位或符号的过程
## Tokenizer（分词器）
是自然语言处理（NLP）中的一个重要概念，指的是将文本切分成标记（tokens）的工具或算法。Tokenizer 在文本处理中起到了关键作用，它将连续的文本序列划分为离散的标记，例如单词、子词、字符或其他语言片段。
### 分词标记
- EOS（End-of-Sentence）：EOS 是表示句子结束的特殊标记。在标记化过程中，文本通常被分割成单词或子词的序列。当到达句子的结尾时，可以使用 EOS 标记来指示句子的结束。EOS 标记对于语言模型、机器翻译和生成任务等任务中非常重要。
- PAD（Padding）：PAD 是用于填充序列长度的特殊标记。在序列数据中，不同的样本可能具有不同的长度。在进行批处理时，为了使每个批次的样本长度一致，可以使用 PAD 标记填充较短的样本，使其与最长样本的长度相匹配。填充通常在序列的末尾添加 PAD 标记。
- EOD（End-of-Document）：EOD 是表示文档结束的特殊标记。与 EOS 类似，EOD 标记用于指示文档的结束。在某些情况下，处理整个文档而不仅仅是单个句子时，可以使用 EOD 标记来标记文档的结束。
- UNK（Unknown）：UNK 是表示未知词或未登录词的特殊标记。当遇到模型词汇表中没有包含的词或标记时，可以使用 UNK 标记来表示。
- BOS（Beginning-of-Sentence）：BOS 是表示句子开头的特殊标记。类似于 EOS 标记，BOS 标记用于指示句子的开始。
- SEP（Separator）：SEP 是表示分隔符的特殊标记。在某些情况下，可以使用 SEP 标记来分隔句子、段落或其他文本单元。
- MASK（Mask）：MASK 是表示掩码操作的特殊标记。在一些任务中，可以使用 MASK 标记来屏蔽或隐藏文本的一部分，以进行模型的训练或推理。
# 区别和联系
- 标记化是将文本转换为计算机可以处理的形式，以便进行后续的自然语言处理任务
- 词向量它是将单词转换为连续的实数向量，以便进行机器学习和深度学习等任务
- 词向量可以看作是对标记的表示形式之一
- 标记化是在自然语言处理任务的预处理阶段完成的
- 词向量的生成可以是预训练模型或其他词嵌入技术进行的
- 词向量可以用于模型的输入表示，以提供更丰富的语义信息和上下文理解能力

## SentencePiece ：开源分词工具
- 用于进行分词和子词化（subword segmentation）任务
- 支持监督学习和未监督学习
- 支持训练和推理
- 还提供了其他功能，如模型保存和加载、词汇表的导出和导入、词频统计等
- Unigram Language Model（一元语言模型）：Unigram Language Model 是 SentencePiece 中的一种语言模型，用于学习文本中的标记（tokens）的概率分布。它基于标记的出现频率来计算其概率，用于生成词汇表和进行编码。
- Vocabulary Size（词汇表大小）：Vocabulary Size 是指在 SentencePiece 中生成的词汇表的大小，即词汇表中的唯一标记数量。较大的词汇表能够更好地覆盖不常见的标记，但会增加模型的复杂性和计算成本。
- Model Type（模型类型）：Model Type 是 SentencePiece 中的一个参数，用于指定模型的类型。常见的模型类型有 "unigram"、"bpe"（Byte Pair Encoding）和 "word"。不同的模型类型具有不同的切分策略和性能特点。
- Input Sentence Size（输入句子大小）：Input Sentence Size 是指用于训练 SentencePiece 模型的输入句子的大小。它可以是句子的字符数或标记数，用于控制训练数据的规模和模型的训练效果。
- Character Coverage（字符覆盖率）：Character Coverage 是用于控制 SentencePiece 模型训练的一个参数，用于指定训练数据中要覆盖的字符比例。较高的字符覆盖率可以保留更多的字符信息，但可能导致词汇表增加和训练时间增加。
- Normalization Rule（规范化规则）：Normalization Rule 是用于指定 SentencePiece 模型中文本的规范化规则。它可以包括对字符的大小写转换、标点符号的处理、Unicode 规范化等操作，以便在训练和切分过程中统一文本的表示形式。
- https://github.com/google/sentencepiece
