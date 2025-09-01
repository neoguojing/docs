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
## QWEN3

## Gemma3

## GPTOSS
