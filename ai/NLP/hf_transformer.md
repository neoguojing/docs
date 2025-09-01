# transfomer 库详解
> python utils/modular_model_converter.py :基于mudular文件生成model文件，避免重复代码
## 通用库
### 通用参数名称对照表：
- scaling： attention的缩放参数，对应公式除的根号d
- q_proj： XWq的函数，投影矩阵
- k_proj： XWk的函数
- v_proj： XWv的函数
- q_norm：
- k_norm：
- sliding_window
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
### PreTrainedModel:
### GradientCheckpointingLayer
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
- initializer_range: 权重矩阵初始标准差
- rms_norm_eps：归一化层的参数0.02，防止除0；RMSNorm
- use_cache：true，使用kvcache，加速推理
- tie_word_embeddings：共享输入输出层的Embding；false
- rope_theta：RoPE的旋转周期；10000
- attention_bias：是否attention层偏置；false
- use_sliding_window (bool, 默认值=False)是否使用滑动窗口注意力（SWA）。
- sliding_window (int, 默认值=4096)滑动窗口注意力的窗口大小。
- max_window_layers (int, 默认值=28)前 max_window_layers 层使用全局注意力，之后的层使用滑动窗口注意力。
- layer_types (list, 可选)每一层的注意力模式定义（比如混合使用全局注意力和 SWA）。
```
self.layer_types = [
                "sliding_attention"
                if self.sliding_window is not None and i >= self.max_window_layers
                else "full_attention"
                for i in range(self.num_hidden_layers)
            ]
```
- attention_dropout (float, 默认值=0.0)注意力概率的 dropout 比例。
#### 并行策略
##### 张量并行
> colwise 列并行
> rowwise 行并行
```
base_model_tp_plan = {
    "layers.*.self_attn.q_proj": "colwise",
    "layers.*.self_attn.k_proj": "colwise",
    "layers.*.self_attn.v_proj": "colwise",
    "layers.*.self_attn.o_proj": "rowwise",
    "layers.*.mlp.gate_proj": "colwise",
    "layers.*.mlp.up_proj": "colwise",
    "layers.*.mlp.down_proj": "rowwise",
}
```
#####流水线并行
```
base_model_pp_plan = {
    "embed_tokens": (["input_ids"], ["inputs_embeds"]),
    "layers": (["hidden_states", "attention_mask"], ["hidden_states"]),
    "norm": (["hidden_states"], ["hidden_states"]),
}
```
## Gemma3

## GPTOSS
