# LLama2
- llama只用了tranformer的decoder部分，是decoder-only结构
- bert采用Encoder-only，google t5采用encoder-decoder结构
- 32个Decoder层
- LayerNorm换成了RMSNorm
- Multi-Head Attention换成了GQA
- postionnal换成了RotatyEmbedding（RoPE相对位置编码）

## QKV
- MHA： 多头注意力，kv和q有相同的头数；
- MQA：kv只有一头，所有q共享：吞吐量提高30-40，即推理加速
- GQA：折中方案

## RoPE 相对位置编码
- RoPE是利用绝对位置编码表示相对位置的一种方式，不仅保持位置编码，还能保持相对位置的关系。
- 绝对位置编码表示相对位置的一种方式，不仅保持位置编码，还能保持相对位置的关系。
- 将位置m转成β进制，构成一个d//2维的向量，用这个向量就能计算出位置。

## RMSNorm，简化了LayerNorm，提升计算速度
- 
## beam search集束搜索
-num_beams： 当搜索策略为束搜索（beam search）时，该参数为在束搜索（beam search）中所使用的束个数，当num_beams=1时，实际上就是贪心搜索（greedy decoding）
- greedy search：每步取概率最大的输出，然后将从开始到当前步的输出作为输入，取预测下一步，直到句子结束

## 扩展token
- RoPE外推：结合RoPE的特性，可以通过位置插值，扩展token的长度。最简单的方法就是线性插值
- NTK
- 动态插值算法
