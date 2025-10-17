# 多模态
### 输入序列编码

- 文本 token → 文本解码器

- 图像/视频 token → vision backbone → 特征 token

- 音频 token → audio backbone → 特征 token

### 序列融合

- 通过 特殊 token（audio_token_id / image_token_id / video_token_id）标记不同模态

- Transformer 接收统一序列，使用 RoPE/位置编码 保留时间与顺序信息

### 思考-生成循环

- codec_think_bos_id → 生成 latent 思考表示

- codec_think_eos_id → 结束思考

- codec_bos_id → 开始生成音频/语音

- codec_eos_id → 结束生成

- accept_hidden_layer → 可选隐藏层用于 refine 或决定生成步骤

### 输出 token

- 音频 → VQ 码本 token（num_code_groups 多码本）

- 文本 → 语义 token

- 视觉 → 一般作为 encoder embedding，与文本/音频融合

| 子模块      | 对应配置类                        | 功能                          |
| -------- | ---------------------------- | --------------------------- |
| Thinker  | `Qwen3OmniMoeThinkerConfig`  | 语义推理、上下文理解和跨模态规划            |
| Talker   | `Qwen3OmniMoeTalkerConfig`   | 文本/音频/图像 token 的自回归生成       |
| Code2Wav | `Qwen3OmniMoeCode2WavConfig` | 离散音频 token → waveform（音频生成） |

## AudioEncoder
| 概念                            | 含义                         | 数学/直观理解    |
| ----------------------------- | -------------------------- | ---------- |
| **频谱（Spectrum）**              | 音频信号在不同频率上的能量分布            | 由 FFT 得到   |
| **Mel 频率（Mel Scale）**         | 模拟人耳的非线性听觉感知的频率标尺          | 高频对数，低频线性  |
| **Mel 滤波器组（Mel Filter Bank）** | 一组三角形滤波器，用于在感知尺度上平滑功率谱     | 模拟听觉带通滤波   |
| **Mel 频谱图（Mel Spectrogram）**  | 滤波器组加权后的能量随时间变化的二维图        | “人耳视角”的时频图 |
| **MFCC（梅尔倒谱系数）**              | 对 Mel 频谱图取对数再做 DCT 压缩的紧凑特征 | 类似“音频指纹”   |

### 配置
| 参数名                       | 类型      | 默认值      | 说明                                                  |
| ------------------------- | ------- | -------- | --------------------------------------------------- |
| `num_mel_bins`            | `int`   | 128      | 每个输入特征的 Mel 滤波器数量，需与 `Qwen3OmniMoeProcessor` 保持一致。  |
| `encoder_layers`          | `int`   | 32       | Transformer 编码层的数量。                                 |
| `encoder_attention_heads` | `int`   | 20       | 每个注意力层的注意力头数量。                                      |
| `encoder_ffn_dim`         | `int`   | 5120     | 前馈网络的隐藏层维度。                                         |
| `d_model`                 | `int`   | 1280     | 模型主维度（embedding 维度）。                                |
| `dropout`                 | `float` | 0.0      | 编码层及嵌入层的 dropout 概率。                                |
| `attention_dropout`       | `float` | 0.0      | 注意力权重的 dropout 概率。                                  |
| `activation_function`     | `str`   | `"gelu"` | 激活函数，可选 `"gelu"`, `"relu"`, `"silu"`, `"gelu_new"`。 |
| `activation_dropout`      | `float` | 0.0      | 激活层的 dropout 概率。                                    |
| `scale_embedding`         | `bool`  | False    | 是否按 √d_model 缩放 embedding。                          |
| `initializer_range`       | `float` | 0.02     | 权重矩阵初始化的标准差。                                        |
| `max_source_positions`    | `int`   | 1500     | 最长输入序列长度（log-mel 序列长度上限）。                           |
| `n_window`                | `int`   | 100      | AudioEncoder 内部卷积和 FlashAttention 的块大小。             |
| `output_dim`              | `int`   | 3584     | 音频编码器输出特征维度。                                        |
| `n_window_infer`          | `int`   | 400      | 推理时使用的块大小。                                          |
| `conv_chunksize`          | `int`   | 500      | 卷积特征块的尺寸。                                           |
| `downsample_hidden_size`  | `int`   | 480      | 下采样隐藏层的维度。                                          |

## VisionEncoder
### 配置
| 参数                         | 默认值                 | 说明                          |
| -------------------------- | ------------------- | --------------------------- |
| `depth`                    | 27                  | 模型的层数（Transformer Block 层数） |
| `hidden_size`              | 1152                | 每层隐藏状态的维度                   |
| `hidden_act`               | "gelu_pytorch_tanh" | 激活函数类型                      |
| `intermediate_size`        | 4304                | FFN 中间层维度                   |
| `num_heads`                | 16                  | 多头注意力机制中的头数                 |
| `in_channels`              | 3                   | 输入图像通道数（RGB 图像为 3）          |
| `patch_size`               | 16                  | 图像切分成 patch 的大小（16x16）      |
| `spatial_merge_size`       | 2                   | 空间合并操作的尺度                   |
| `temporal_patch_size`      | 2                   | 时间维度 patch 大小（视频帧或序列处理）     |
| `out_hidden_size`          | 3584                | 输出隐藏特征维度                    |
| `num_position_embeddings`  | 2304                | 位置编码向量数量                    |
| `deepstack_visual_indexes` | [8, 16, 24]         | 用于 DeepStack 的特定层索引         |
| `initializer_range`        | 0.02                | 权重初始化范围                     |

## Text
配置等同于语言模型
## Thinker
### 配置
> 标识输入序列的特殊标记（special tokens）或时间步/位置的起点
> 每种 token 都像一个“模态标签”，告诉模型接下来应该使用哪个编码器和处理方式。
> 对文本 token，通常使用标准的位置编码
> 对视觉/音频 token，位置编码可以结合 patch 或帧索引 + RoPE 方式
| 参数                        | 默认值    | 说明                 |
| ------------------------- | ------ | ------------------ |
| `audio_token_id`          | 151646 | 音频输入 token ID      |
| `image_token_id`          | 151655 | 图像输入 token ID      |
| `video_token_id`          | 151656 | 视频输入 token ID      |
| `audio_start_token_id`    | 151647 | 音频序列起始 token ID    |
| `user_token_id`           | 872    | 用户 token ID        |
| `position_id_per_seconds` | 25     | 每秒位置 ID 增量，用于时间步编码，例如 25 → 每秒增加 25 个位置 ID |
| `initializer_range`       | 0.02   | 权重初始化标准差           |

- audio_config → Qwen3OmniMoeAudioEncoderConfig

- vision_config → Qwen3OmniMoeVisionEncoderConfig

- text_config → Qwen3OmniMoeTextConfig
## Talker 
###配置
| Token 名称                | 默认值    | 功能说明                        |
| ----------------------- | ------ | --------------------------- |
| `audio_token_id`        | 151646 | 表示音频 token 的位置，用于编码音频片段     |
| `audio_start_token_id`  | 151669 | 标记音频生成的起始 token             |
| `image_token_id`        | 151655 | 表示图像输入在序列中的位置               |
| `video_token_id`        | 151656 | 表示视频输入在序列中的位置               |
| `vision_start_token_id` | 151652 | 标记视觉输入（图像/视频）开始             |
| `codec_bos_id`          | 4197   | 语音或音频生成序列的开始 token          |
| `codec_eos_token_id`    | 4198   | 语音生成序列的结束 token             |
| `codec_pad_id`          | 4196   | 语音序列 padding token          |
| `codec_nothink_id`      | 4203   | 表示无需思考步骤（think step）的 token |
| `codec_think_bos_id`    | 4204   | 思考阶段的开始 token               |
| `codec_think_eos_id`    | 4205   | 思考阶段的结束 token               |

## Code-to-Waveform
> 将离散音频码本（acoustic token）转换为高保真波形。它主要负责 音频生成阶段，通常作为 Talker 模型输出音频 token 的最终解码器
| 概念                         | 作用                | 在生成流程的位置                        |
| -------------------------- | ----------------- | ------------------------------- |
| 码本 (Codebook)              | 将连续音频向量离散化成 token | Talker 输出音频 token → Code2Wav 输入 |
| 多码本 (Multi-Codebook)       | 使用多层残差量化提升表示能力    | 每个 token 由多个码本索引组成              |
| 残差量化器 (Residual Quantizer) | 编码残差，逐步逼近原向量      | 多码本内部逐步量化                       |
| 上采样 (Upsampling)           | 扩展时间分辨率以生成波形      | Transformer 隐状态 → 连续波形          |

### 配置
| 参数               | 默认   | 功能                  |
| ---------------- | ---- | ------------------- |
| `codebook_size`  | 2048 | 每个残差码本的条目数量         |
| `num_quantizers` | 16   | 残差量化器数量，多码本表示音频更细粒度 |
| 参数                  | 默认          | 功能              |
| ------------------- | ----------- | --------------- |
| `upsample_rates`    | `(8,5,4,3)` | 每层特征上采样倍数       |
| `upsampling_ratios` | `(2,2)`     | 反卷积上采样比例        |
| `decoder_dim`       | 1536        | 输出特征维度，映射到波形生成前 |
## OmniMoe
### 配置
| Token 名称                                                     | 作用                   |
| ------------------------------------------------------------ | -------------------- |
| `im_start_token_id` / `im_end_token_id`                      | 图像输入序列的起始/结束标识       |
| `tts_pad_token_id` / `tts_bos_token_id` / `tts_eos_token_id` | 语音生成（TTS）的填充、开始、结束标记 |
| `system_token_id` / `user_token_id` / `assistant_token_id`   | 多轮对话中标识系统、用户和助手发言    |

