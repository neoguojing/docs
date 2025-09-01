# transfomer 库详解

## 通用库
### PretrainedConfig 模型配置：
- 类属性： 如model_type 等
- 公共属性（所有子类都有）
-- vocab_size (int)：词表大小（非文本模型可能没有）。
-- hidden_size (int)：隐藏层维度。
-- num_attention_heads (int)：注意力头数。
-- num_hidden_layers (int)：模型层数。
- 初始化参数：is_decoder 等
- 微调相关参数
- tokenizer
-- tokenizer_class (str)：分词器类名。
-- prefix (str)：输入文本前缀。
-- bos_token_id：序列开始 token id。
-- pad_token_id：填充 token id。
-- eos_token_id：序列结束 token id。
-- decoder_start_token_id：解码起始 token id。
-- sep_token_id：分隔 token id。
## Qwen3
### Qwen3Config
- vocab_size： 151936
- hidden_size： 4096
- intermediate_size： 前馈网络的升维纬度，22016
- - 在「接近 4~6 倍」的区间内，挑一个能被硬件友好分割的数
  - GPU 上矩阵乘法（GEMM）通常要求维度是 64 或 128 的倍数，才能利用 Tensor Core 高效运算。22016 ÷ 128 = 172 → 恰好是整数
- num_hidden_layers：32；block的层数
  - Block = Attention + FFN + Norm + Residual
- num_attention_heads: 32;Attention的头数
- num_key_value_heads：kv头的数量，适用于MQA GQA
- head_dim： 128；单个头的纬度；
- hidden_act：激活函数；silu：SiLU(x)=x⋅σ(x)
- max_position_embeddings：32768；模型的最大支持的序列长度
## Gemma3

## GPTOSS
