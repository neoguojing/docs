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
| 参数                        | 默认值    | 说明                 |
| ------------------------- | ------ | ------------------ |
| `audio_token_id`          | 151646 | 音频输入 token ID      |
| `image_token_id`          | 151655 | 图像输入 token ID      |
| `video_token_id`          | 151656 | 视频输入 token ID      |
| `audio_start_token_id`    | 151647 | 音频序列起始 token ID    |
| `user_token_id`           | 872    | 用户 token ID        |
| `position_id_per_seconds` | 25     | 每秒位置 ID 增量，用于时间步编码，例如 25 → 每秒增加 25 个位置 ID |
| `initializer_range`       | 0.02   | 权重初始化标准差           |

> 标识输入序列的特殊标记（special tokens）或时间步/位置的起点
> 每种 token 都像一个“模态标签”，告诉模型接下来应该使用哪个编码器和处理方式。
> 对文本 token，通常使用标准的位置编码
> 对视觉/音频 token，位置编码可以结合 patch 或帧索引 + RoPE 方式
- audio_config → Qwen3OmniMoeAudioEncoderConfig

- vision_config → Qwen3OmniMoeVisionEncoderConfig

- text_config → Qwen3OmniMoeTextConfig

## Talker 
### 配置
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
| `upsample_rates`    | `(8,5,4,3)` | 每层特征上采样倍数       |
| `upsampling_ratios` | `(2,2)`     | 反卷积上采样比例        |
| `decoder_dim`       | 1536        | 输出特征维度，映射到波形生成前 |

## Processor
#### from_pretrained
- AutoProcessor.from_pretrained() 读取配置；
- 发现 "image_processor_type" 字段；
- 调用 get_possibly_dynamic_module("Qwen2VLImageProcessor")；
- 检查是否在 transformers 模块中定义 → 没找到；
- 进入 IMAGE_PROCESSOR_MAPPING._extra_content；
- 找到注册过的 Qwen2VLImageProcessor 类；
- 返回类对象；
#### 注册
```
from transformers import AutoImageProcessor

AutoImageProcessor.register(
    "qwen2-vl",  # key
    Qwen2VLImageProcessor  # class
)

```
### ProcessorMixin
> 汇总各种处理器，处理统一输入

| 属性名                 | 输入数据     | 子处理器类型 |
| ------------------- | -------- | ------ |
| `tokenizer`         | `text`   | 文本处理   |
| `image_processor`   | `images` | 图像处理   |
| `video_processor`   | `videos` | 视频帧处理  |
| `feature_extractor` | `audio`  | 音频特征提取 |

### Qwen3OmniMoeProcessor
```
Qwen3OmniMoeProcessor
│
├── image_processor → Qwen2VLImageProcessor
├── video_processor → Qwen2VLVideoProcessor
├── feature_extractor → WhisperFeatureExtractor
├── tokenizer → Qwen2TokenizerFast
│
├── __call__() → 主入口
│    ├── 音频提取
│    ├── 图像处理
│    ├── 视频处理
│    ├── 文本多模态占位符替换
│    ├── 文本分词
│    └── 合并为 BatchFeature
│
├── replace_multimodal_special_tokens() → 多模态 token 展开，将占位符替换为实际token
├── get_chunked_index() → 时间片索引划分
├── apply_chat_template() → 应用对话模板
└── model_input_names → 输入字段总览

```
#### 配置
| 字段                                                    | 含义                                   |
| ----------------------------------------------------- | ------------------------------------ |
| `"processor_class": "Qwen3OmniMoeProcessor"`          | 顶层处理器类名，负责协调图像和音频两个子模块；是主入口。         |
| `"image_processor_type": "Qwen2VLImageProcessor"`     | 图像处理器类型，用于 resize、normalize、打 patch。 |
| `"feature_extractor_type": "WhisperFeatureExtractor"` | 音频特征提取类，用于将波形转换为频谱特征。                |
| 字段                              | 含义                       |
| ------------------------------- | ------------------------ |
| `"feature_size": 128`           | 每帧音频特征的维度，通常是 Mel 滤波器数量  |
| `"n_fft": 400`                  | STFT 的窗口长度               |
| `"hop_length": 160`             | 帧移，即连续帧之间的步长             |
| `"n_samples": 4800000`          | 最大采样点数，通常用于截断长音频         |
| `"sampling_rate": 16000`        | 音频采样率（16 kHz）            |
| `"dither": 0.0`                 | 加性噪声参数（用于防止量化伪影）         |
| `"padding_value": 0.0`          | 填充时的默认值                  |
| `"return_attention_mask": true` | 是否返回掩码，用于模型区分 padding 区域 |
| 字段                              | 含义                               |
| ------------------------------- | -------------------------------- |
| `"image_mean": [0.5, 0.5, 0.5]` | 图像归一化均值（通道维度）                    |
| `"image_std": [0.5, 0.5, 0.5]`  | 图像归一化方差（通道维度）                    |
| `"patch_size": 16`              | ViT patch 大小（每 16×16 像素为一 patch） |
| `"temporal_patch_size": 2`      | 时间维 patch（多帧视频或时间维切块）            |
| `"max_pixels": 12845056`        | 输入图像允许的最大像素数，用于限制内存占用            |
| `"min_pixels": 3136`            | 最小分辨率限制                          |
| `"merge_size": 2`               | 图像 patch 合并尺寸，用于降采样或多尺度融合        |
| `"nb_max_frames": 30000`        | 视频或序列最大帧数                        |
#### WhisperFeatureExtractor
> class WhisperFeatureExtractor(SequenceFeatureExtractor):

> class SequenceFeatureExtractor(FeatureExtractionMixin):

> 把 raw waveform → log-mel fbank，供 Whisper 模型使用

> 输入：(num_samples,)  输出：（num_mel_filters, num_frames）

| 函数                    | 作用             | 输出              | 备注                         |
| --------------------- | -------------- | --------------- | -------------------------- |
| `torch.hann_window`   | 生成 Hann 窗      | Tensor(n_fft)   | 用于加窗                       |
| `torch.randn`         | 生成正态随机噪声       | Tensor          | 用于抖动 (dither)              |
| `torch.stft`          | 短时傅里叶变换        | Tensor(complex) | `return_complex=True` 返回复数 |
| `.abs()`              | 取复数幅值          | Tensor          | 用于功率谱计算                    |
| `**2`                 | 幅值平方 → 功率谱     | Tensor          |                            |
| `@`                   | 矩阵乘法           | Tensor          | 频谱 → Mel 频率                |
| `torch.clamp(min)`    | 限制最小值          | Tensor          | 避免 log(0)                  |


##### waveform
- 它表示的是声音随时间变化的振幅（Amplitude）
- 表示在时间上连续的离散采样
- (num_samples,)： 单声道是一维数组
- 每个值标识1/sample秒的声音振幅
##### 分帧
- 在极短的时间片段中，语音的频谱特征基本不变
- 对每一帧计算频谱
- n_fft - hop_length 是帧直接的重叠长度
- num_frames = int(1 + np.floor((waveform.size - frame_length) / hop_length)) ： 总帧数计算
##### 加窗
- buffer[:frame_length] *= window：点积
- window是一个frame_length长度的数组，值为浮点数
- 目的在于帧数据*一个系数，是的边界数据比较平滑（边缘系数很小，边缘的值会变小）
- 帧边界平滑过渡 → 频谱干净、特征更稳定。
#### 傅里叶变化
- DFT（离散傅里叶变换): 把信号投影到一系列“正弦 + 余弦”基上，得到每个频率分量的强度和相位
- FFT（快速傅里叶变换）: 通过分治算法，把复杂度降低
- rFFT（实值快速傅里叶变换）: 负频率部分是正频率的镜像,只需要存储一半数据
- 结果是复数数组，因为频率既有大小又有相位
- np.fft.fft(x)输出结果示例：[ 0.+0.j  0.+0.j  0.-4.j  0.+0.j  0.+0.j  0.+0.j  0.+4.j  0.+0.j]
- 计算结果形状：(num_frames, num_frequency_bins)
- 线性频率： 每个频率 bin 是等间隔的 Hz 值
##### 频谱处理
- np.abs(spectrogram)：就是取 每个频率 bin 的幅度
- 幅度谱：每个频率分量的振幅大小，也就是波形在某一频率的“强度”
- 功率谱：每个频率分量的能量，单位是振幅平方
##### Mel谱转换
- 低频：人耳敏感 → 需要更多滤波器
- 人耳不敏感 → 滤波器间隔可大
- 每个滤波器是一个列向量，长度 = num_freq_bins
- 滤波器矩阵 shape = (num_freq_bins, num_mel_filters)
- 对 FFT 幅度谱做矩阵乘法：mel_spectrogram = mel_filters.T @ spectrogram
- 输出结果：（num_mel_filters, num_frames）
##### log mel转换
- 人耳对声强的感知是 对数关系（例如音量增加 10 倍，感觉只大约增加 2 倍）
- log_mel 选项就是把线性谱 → 对数谱，符合人耳感知，方便机器学习建模
- log / log10 / dB 的区别主要在 对数底数 与 是否换算成分贝单位

##### 输出
- 如果原始音频长度不同，需要 padding 对齐 batch
- mask 告诉模型 哪些帧是真实音频，哪些是 padding，防止模型对 padding 做无意义计算
#### Qwen2VLImageProcessor
##### 配置
| 参数                    | 默认值                                                    | 作用                      |
| --------------------- | ------------------------------------------------------ | ----------------------- |
| `do_resize`           | True                                                   | 是否调整图片尺寸                |
| `size`                | `{"shortest_edge": 56*56, "longest_edge": 28*28*1280}` | resize 的目标尺寸范围          |
| `resample`            | BICUBIC                                                | resize 时使用的插值方法         |
| `do_rescale`          | True                                                   | 是否缩放像素值                 |
| `rescale_factor`      | 1/255                                                  | 缩放比例（通常把 0-255 映射到 0-1） |
| `do_normalize`        | True                                                   | 是否做均值/方差归一化             |
| `image_mean`          | `[0.48145466, 0.4578275, 0.40821073]`                  | 归一化均值                   |
| `image_std`           | `[0.26862954, 0.26130258, 0.27577711]`                 | 归一化标准差                  |
| `do_convert_rgb`      | True                                                   | 是否将图片转换为 RGB            |
| `min_pixels`          | 56*56                                                  | resize 的最小像素限制          |
| `max_pixels`          | 28*28*1280                                             | resize 的最大像素限制          |
| `patch_size`          | 14                                                     | 空间 patch 尺寸             |
| `temporal_patch_size` | 2                                                      | 时间 patch 尺寸（视频时） 每个时间 patch 由 2 帧组成        |
| `merge_size`          | 2                                                      | 局部 patch 合并大小           |
#### 转换
- 用于将输入的图片或视频帧处理成模型可以直接使用的视觉特征输入
- 把图片/视频切成很多小方块（patch），每个方块变成一个小向量，然后排成一排，送进模型去理解
- merge_size 不是固定的 → 可以动态调整模型序列长度
- merge_size 可控 → 可以选择在保留局部特征的同时减少 token
- 小 patch → merge = 先切小方块，再拼成大方块，输出维度更容易匹配 Transformer 隐藏层需求
- grid_h=height/patch_size
- grid_w=W/patch_size
- merge_size = 2 → 每 2×2 个 patch 合并
- grid_h=grid_h/merge_size
- grid_w=grid_w/merge_size
- grid_t=num_frames/temporal_patch_size
- patch_dim=channel∗temporal_patch_size∗patch_size∗patch_size∗merge_size2：merge参数使得 输出维度更容易匹配 Transformer 隐藏层需求
- num_patches=grid_t​∗grid_h​∗grid_w
- flatten_patches.shape = (num_patches, patch_dim)
#### Qwen2VLVideoProcessor
> 帧抽样 → 图像裁剪/缩放 → 标准化/归一化 → 切 patch 并合并成 tokens最终生成模型可直接输入的 视频向量化表示。
##### 配置 大部分参数等同于图像处理
- do_sample_frames bool 是否进行帧采样
##### 类
- class Qwen2VLVideoProcessor(BaseVideoProcessor)
- class BaseVideoProcessor(BaseImageProcessorFast):
- class BaseImageProcessorFast(BaseImageProcessor):
- class BaseImageProcessor(ImageProcessingMixin):
##### 运行
- _decode_and_sample_videos： 解码或者采样视频帧，返回列表：[torch.Tensor(num_frames, 3, H, W)]
- sample_frames： 安装输入的帧数或者fps，进行均匀采样，返回的是帧的帧数索引数组：indices = [0, k, 2k, 3k, ...]
- - torch.arange(start, end, step)：在 [start, end) 区间内，按固定间隔 step 生成一个一维张量
- fetch_videos： 负责下载视频，组装图片，以及调用视频解码器，解码帧，返回[num_frames, height, width, 3]
- _prepare_input_videos：格式化输入为[(num_frames, C, H, W)]
- _preprocess:
- - group_videos_by_shape：按尺寸分组视频，方便对齐
  - - grouped_videos：dict[(T,H,W), Tensor]每组视频堆叠后的批次数据
  - - grouped_videos_index：dict[int, ((T,H,W), idx)]原始索引 → (分组键, 分组内位置)
- smart_resize: 高、宽都能被 factor 整除（一般是模型 patch size 的倍数，比如 28）;总像素数在 [min_pixels, max_pixels] 范围内;尽可能保持原图宽高比（不拉伸变形
- 输出：
- - pixel_values_videos.shape： (N, P, D) ;
  - N：批内视频数量（总视频数）
  - P：每个视频的 patch 数量，等于 grid_t * grid_h * grid_w
  - D：每个 patch 的向量长度（维度），等于 C * temporal_patch_size * patch_size * patch_size
  - video_grid_thw:(N, 3)
  - 每行表示 [grid_t, grid_h, grid_w]（时、空高、空宽）
## OmniMoe
### 配置
| Token 名称                                                     | 作用                   |
| ------------------------------------------------------------ | -------------------- |
| `im_start_token_id` / `im_end_token_id`                      | 图像输入序列的起始/结束标识       |
| `tts_pad_token_id` / `tts_bos_token_id` / `tts_eos_token_id` | 语音生成（TTS）的填充、开始、结束标记 |
| `system_token_id` / `user_token_id` / `assistant_token_id`   | 多轮对话中标识系统、用户和助手发言    |

