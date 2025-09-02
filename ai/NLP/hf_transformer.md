# transfomer 库详解
> python utils/modular_model_converter.py :基于mudular文件生成model文件，避免重复代码
## 通用库
### attention计算优化
- flash_attention_2/3	GPU kernel 优化，高速	超长序列训练	高
- flex_attention	灵活块大小，自适应	可变序列长度	高
- paged_attention	分页计算，减少 O(seq²)	超长序列	高
- sdpa	标准公式，直观	普通序列	中
- sdpa_paged	分页优化	长序列	高
- eager_paged	即时分页计算	推理/训练长序列	高
### 通用参数名称对照表：
- scaling： attention的缩放参数，对应公式除的根号d
- q_proj = XWq   输入：隐藏状态 ；输出：查询向量 Q
- k_proj = XWk
- v_proj = XWv
- o_proj = [head1​,head2​,...,headh​]Wo； 输入：拼接后的多头注意力结果；输出：回到模型隐藏维度hidden_size，以便传给后续的 FFN 或残差连接
- q_norm：对 Q向量做 RMSNorm（归一化），保证数值稳定，提升训练和推理效果
- k_norm：对 Q 向量做 RMSNorm（归一化），保证数值稳定，提升训练和推理效果
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
##### 流水线并行
```
base_model_pp_plan = {
    "embed_tokens": (["input_ids"], ["inputs_embeds"]),
    "layers": (["hidden_states", "attention_mask"], ["hidden_states"]),
    "norm": (["hidden_states"], ["hidden_states"]),
}
```
### Qwen3Attention继承自LlamaAttention
```
#(B,L,d) -> (B,L)
input_shape = hidden_states.shape[:-1]
#(B,L,-1,head_dim) -1表示自动推断，在后面计算过程中自动推断
hidden_shape = (*input_shape, -1, self.head_dim)
#view 会自动推断-1的纬度 ,transpose交互1和2维，得到(B,head_num,L,head_dim)
query_states = self.q_norm(self.q_proj(hidden_states).view(hidden_shape)).transpose(1, 2)
key_states = self.k_norm(self.k_proj(hidden_states).view(hidden_shape)).transpose(1, 2)
value_states = self.v_proj(hidden_states).view(hidden_shape).transpose(1, 2)
#输入的位置编码矩阵
cos, sin = position_embeddings
query_states, key_states = apply_rotary_pos_emb(query_states, key_states, cos, sin)
# k,v缓存
if past_key_values is not None:
    # sin and cos are specific to RoPE models; cache_position needed for the static cache
    cache_kwargs = {"sin": sin, "cos": cos, "cache_position": cache_position}
    key_states, value_states = past_key_values.update(key_states, value_states, self.layer_idx, cache_kwargs)
# 选择attention的计算策略
attention_interface: Callable = eager_attention_forward
if self.config._attn_implementation != "eager":
    attention_interface = ALL_ATTENTION_FUNCTIONS[self.config._attn_implementation]

attn_output, attn_weights = attention_interface(
    self,
    query_states,
    key_states,
    value_states,
    attention_mask,
    dropout=0.0 if not self.training else self.attention_dropout,
    scaling=self.scaling,
    sliding_window=self.sliding_window,  # diff with Llama
    **kwargs,
)
# (B, L, hidden_dim)
attn_output = attn_output.reshape(*input_shape, -1).contiguous()
# 多头拼接
attn_output = self.o_proj(attn_output)
return attn_output, attn_weights
```
#### 位置编码 
> 原理1：i⋅(x+iy)=ix+i^2y=−y+ix   等价与旋转90度
> 原理2：旋转任意角度:(x+iy)⋅eiθ=(xcosθ−ysinθ)+i(xsinθ+ycosθ)
> 原理3： i(xsinθ+ycosθ) 就等价于复数乘法，对x,y旋转90度
```
cos = cos.unsqueeze(unsqueeze_dim)
sin = sin.unsqueeze(unsqueeze_dim)
q_embed = (q * cos) + (rotate_half(q) * sin)
k_embed = (k * cos) + (rotate_half(k) * sin)
```
## Gemma3

## GPTOSS
