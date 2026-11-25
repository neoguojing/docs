基于你提供的 Gemma4 31B IT 配置，可以把模型分为两条主要路径：

1. **文本路径**：Embedding → 60 层 Transformer → LM Head。
2. **视觉路径**：Vision Encoder → 多模态特征处理 → 融合到文本模型。

模型的核心特点是：**60 层 Transformer 混合使用 Sliding Attention 和 Full Attention，每 6 层中有 5 层 Sliding Attention、1 层 Full Attention。**

# **1. Gemma4 整体模型结构**

```
                       Gemma4ForConditionalGeneration
┌──────────────────────────────────────────────────────────────┐
│                                                              │
│                      多模态输入                               │
│                                                              │
│       ┌────────────────────┐    ┌──────────────────────┐     │
│       │    Text Input      │    │     Image Input      │     │
│       │    Token IDs       │    │      Image           │     │
│       └─────────┬──────────┘    └──────────┬───────────┘     │
│                 │                          │                 │
│                 ▼                          ▼                 │
│       ┌──────────────────┐       ┌────────────────────┐      │
│       │  Token Embedding │       │   Vision Encoder   │      │
│       │                  │       │                    │      │
│       │ Vocab: 262144    │       │ Hidden: 1152       │      │
│       │ Hidden: 5376     │       │ Layers: 27         │      │
│       └────────┬─────────┘       │ Heads: 16          │      │
│                │                 │ Head Dim: 72       │      │
│                │                 │ Output: 280 tokens │      │
│                │                 └──────────┬─────────┘      │
│                │                            │                │
│                │                            ▼                │
│                │                 ┌────────────────────┐      │
│                │                 │ Multimodal Feature │      │
│                │                 │ Processing / Fusion│      │
│                │                 └──────────┬─────────┘      │
│                │                            │                │
│                └────────────┬───────────────┘                │
│                             ▼                                │
│                  ┌────────────────────┐                      │
│                  │  Transformer 60层  │                      │
│                  │                    │                      │
│                  │  Hidden: 5376      │                      │
│                  │  Intermediate:21504│                      │
│                  │                    │                      │
│                  │  Sliding Attention │                      │
│                  │  + Full Attention  │                      │
│                  └──────────┬─────────┘                      │
│                             │                                │
│                             ▼                                │
│                  ┌────────────────────┐                      │
│                  │      LM Head       │                      │
│                  │                    │                      │
│                  │  5376 → 262144     │                      │
│                  └──────────┬─────────┘                      │
│                             │                                │
│                             ▼                                │
│                       Output Tokens                          │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

说明：视觉特征处理和融合模块的内部投影维度没有在当前配置清单中完整列出，因此不在图中虚构具体维度。

# **2. Transformer 60 层结构**

根据 `layer_types` 配置，60 层可以分为 10 个重复组，每组 6 层。

```
Input
  │
  ▼
┌──────────────────────────────────────────────┐
│              Transformer Layers              │
│                                              │
│  Group 0                                     │
│  ┌──────────────┐                            │
│  │ Layer 0      │ Sliding Attention          │
│  │ Layer 1      │ Sliding Attention          │
│  │ Layer 2      │ Sliding Attention          │
│  │ Layer 3      │ Sliding Attention          │
│  │ Layer 4      │ Sliding Attention          │
│  │ Layer 5      │ Full Attention             │
│  └──────────────┘                            │
│                                              │
│  Group 1                                     │
│  ┌──────────────┐                            │
│  │ Layer 6      │ Sliding Attention          │
│  │ Layer 7      │ Sliding Attention          │
│  │ Layer 8      │ Sliding Attention          │
│  │ Layer 9      │ Sliding Attention          │
│  │ Layer 10     │ Sliding Attention          │
│  │ Layer 11     │ Full Attention             │
│  └──────────────┘                            │
│                                              │
│                  ...                         │
│                                              │
│  Group 9                                     │
│  ┌──────────────┐                            │
│  │ Layer 54     │ Sliding Attention          │
│  │ Layer 55     │ Sliding Attention          │
│  │ Layer 56     │ Sliding Attention          │
│  │ Layer 57     │ Sliding Attention          │
│  │ Layer 58     │ Sliding Attention          │
│  │ Layer 59     │ Full Attention             │
│  └──────────────┘                            │
│                                              │
└──────────────────────────────────────────────┘
  │
  ▼
Output
```

| 层类型               | Layer Index（从 0 开始）          | 数量 |
| ----------------- | ---------------------------- | -- |
| Sliding Attention | 0–4、6–10、...、54–58           | 50 |
| Full Attention    | 5、11、17、23、29、35、41、47、53、59 | 10 |
| 合计                | 0–59                         | 60 |

# **3. Transformer 单层结构及维度**

每一层 Transformer 都可以抽象成 Attention 和 FFN 两个主要模块。

```
                Input
                  │
                  │ [B, S, 5376]
                  ▼
             ┌──────────┐
             │ RMSNorm  │
             └────┬─────┘
                  │
                  ▼
          ┌───────────────┐
          │   Attention   │
          │               │
          │ Q Projection  │
          │ K Projection  │
          │ V Projection  │
          │ RoPE          │
          │ Attention     │
          │ O Projection  │
          └───────┬───────┘
                  │
                  ▼
             ┌──────────┐
             │ Residual │◄──────── Input
             └────┬─────┘
                  │
                  ▼
             ┌──────────┐
             │ RMSNorm  │
             └────┬─────┘
                  │
                  ▼
          ┌───────────────┐
          │      FFN      │
          │               │
          │ 5376 → 21504  │
          │ Activation   │
          │ 21504 → 5376  │
          └───────┬───────┘
                  │
                  ▼
             ┌──────────┐
             │ Residual │◄──────── Attention Output
             └────┬─────┘
                  │
                  ▼
                Output
                  │
                  │ [B, S, 5376]
                  ▼
```

这里的 `B` 是 Batch Size，`S` 是当前输入序列长度。

## **3.1 Sliding Attention 层维度**

普通 Sliding Attention 层：

| 操作               | 维度             | 说明              |
| ---------------- | -------------- | --------------- |
| Input            | `[B,S,5376]`   | 输入隐藏状态          |
| Q Projection     | `[B,S,8192]`   | 32 × 256        |
| K Projection     | `[B,S,4096]`   | 16 × 256        |
| V Projection     | `[B,S,4096]`   | 16 × 256        |
| Q Heads          | `[B,32,S,256]` | 32 个 Query Head |
| K Heads          | `[B,16,S,256]` | 16 个 KV Head    |
| V Heads          | `[B,16,S,256]` | 16 个 KV Head    |
| Attention Output | `[B,S,8192]`   | 32 × 256        |
| O Projection     | `[B,S,5376]`   | 投影回隐藏维度         |
| FFN Intermediate | `[B,S,21504]`  | FFN 中间维度        |
| FFN Output       | `[B,S,5376]`   | 恢复隐藏维度          |

```
Input: [B, S, 5376]
       │
       ├── Q: [B, 32, S, 256]
       │
       ├── K: [B, 16, S, 256]
       │
       └── V: [B, 16, S, 256]
                    │
                    ▼
             GQA Attention
             Window = 1024
                    │
                    ▼
           [B, S, 8192]
                    │
                    ▼
           O Projection
           8192 → 5376
                    │
                    ▼
           [B, S, 5376]
                    │
                    ▼
                  FFN
        5376 → 21504 → 5376
                    │
                    ▼
           [B, S, 5376]
```

## **3.2 Full Attention 层维度**

Full Attention 层通过 `per_layer_config` 覆盖了部分 Attention 参数：

| 配置项                 | 普通层  | Full Attention 层 |
| ------------------- | ---- | ---------------- |
| num_attention_heads | 32   | 32\*             |
| num_key_value_heads | 16   | 4                |
| head_dim            | 256  | 512              |
| Q 总维度               | 8192 | 16384\*          |
| K 总维度               | 4096 | 2048             |
| V 总维度               | 4096 | 2048             |
| Attention 输出维度      | 8192 | 16384\*          |
| Sliding Window      | 1024 | 不使用局部窗口          |

\* Full Attention 的 num_attention_heads 没有在前述 per_layer_config 中单独覆盖。这里按基础配置中的 32 进行维度推算，最终应以模型实现中的实际投影参数为准。

```
Input: [B, S, 5376]
       │
       ├── Q: [B, 32, S, 512]
       │
       ├── K: [B,  4, S, 512]
       │
       └── V: [B,  4, S, 512]
                    │
                    ▼
             Full Attention
                    │
                    ▼
           [B, S, 16384]
                    │
                    ▼
           O Projection
           16384 → 5376
                    │
                    ▼
           [B, S, 5376]
                    │
                    ▼
                  FFN
        5376 → 21504 → 5376
                    │
                    ▼
           [B, S, 5376]
```

# **4. Vision Encoder 结构**

根据 Vision Config，视觉编码器的主要参数如下：

| 配置项                          | 实际值    | 说明                      |
| ---------------------------- | ------ | ----------------------- |
| hidden_size                  | 1152   | Vision 隐藏层维度            |
| num_hidden_layers            | 27     | Vision Transformer 层数   |
| num_attention_heads          | 16     | Attention Head 数量       |
| num_key_value_heads          | 16     | KV Head 数量              |
| head_dim                     | 72     | 每个 Head 的维度             |
| intermediate_size            | 4304   | FFN 中间层维度               |
| patch_size                   | 16     | 图像 Patch 大小             |
| default_output_length        | 280    | 默认输出长度                  |
| vision_soft_tokens_per_image | 280    | 每张图片对应的视觉 Soft Token 数量 |
| max_position_embeddings      | 131072 | 视觉位置编码长度上限              |

```
Image
  │
  ▼
Patch Embedding
  │
  │ [B, N, 1152]
  ▼
┌─────────────────────────────┐
│       Vision Encoder        │
│                             │
│       27 Transformer Layers │
│                             │
│       Q: 16 × 72 = 1152     │
│       K: 16 × 72 = 1152     │
│       V: 16 × 72 = 1152     │
│                             │
│       FFN: 1152 → 4304      │
│            4304 → 1152      │
│                             │
└──────────────┬──────────────┘
               │
               ▼
       Vision Features
               │
               ▼
      Multimodal Processing
               │
               ▼
      280 Vision Soft Tokens
               │
               ▼
       Text Model Fusion
```

# **5. 模型关键维度汇总**

| 模块                      | 关键维度                 |
| ----------------------- | -------------------- |
| Text Embedding          | 262144 → 5376        |
| Transformer Layers      | 60                   |
| Text Hidden Size        | 5376                 |
| FFN Intermediate        | 21504                |
| Sliding Attention Q     | 32 × 256 = 8192      |
| Sliding Attention KV    | 16 × 256 = 4096      |
| Full Attention Q        | 按 32 × 512 推算为 16384 |
| Full Attention KV       | 4 × 512 = 2048       |
| Sliding Window          | 1024                 |
| Vision Encoder Layers   | 27                   |
| Vision Hidden Size      | 1152                 |
| Vision Attention        | 16 × 72 = 1152       |
| Vision FFN Intermediate | 4304                 |
| Vision Tokens Per Image | 280                  |
| LM Head                 | 5376 → 262144        |

**特别注意三个容易混淆的维度：**

1. `hidden_size=5376` 是 Transformer 层输入、输出的隐藏维度。
2. `num_attention_heads × head_dim` 是 Attention 的 Q 投影维度，不一定等于 hidden_size。
3. `intermediate_size=21504` 是 FFN 中间层维度，不是 Transformer 隐藏维度。

对于后续 vLLM 源码阅读，我建议重点跟踪 `Gemma4ForConditionalGeneration → Gemma4Model → Attention → MLP`，并对比普通层与 Full Attention 层的 QKV 投影权重 Shape。这能直接把模型配置和实际加载的权重、KV Cache Shape 对应起来。

```
                       用户输入
                          │
                          ▼
                       Tokenizer
                          │
                          ▼
                    Token IDs
                          │
                          ▼
                 Token Embedding
                          │
                          ▼
                 [B, S, 5376]
                          │
                          ▼
        ┌───────────────────────────────────┐
        │       Transformer Layer 0        │
        │                                   │
        │       Hidden: 5376                │
        │               │                   │
        │          RMSNorm                  │
        │               │                   │
        │       ┌───────┼───────┐           │
        │       ▼       ▼       ▼           │
        │       Q       K       V           │
        │       │       │       │           │
        │       ▼       ▼       ▼           │
        │      32H     16H     16H           │
        │       │       │       │           │
        │       │       └──┬────┘           │
        │       │          │                │
        │       └──────────┘                │
        │               │                   │
        │          GQA Attention             │
        │          Window = 1024             │
        │               │                   │
        │          Output Projection         │
        │          8192 → 5376               │
        │               │                   │
        │           Residual                 │
        │               │                   │
        │             RMSNorm                │
        │               │                   │
        │              FFN                  │
        │      5376 → 21504 → 5376           │
        │               │                   │
        │           Residual                 │
        └───────────────┬───────────────────┘
                        │
                        ▼
                 Transformer Layer 1
                 Sliding Attention
                        │
                       ...
                        │
                        ▼
                 Transformer Layer 5
                  Full Attention
                        │
                       ...
                        │
                        ▼
                 Transformer Layer 59
                  Full Attention
                        │
                        ▼
                   Final RMSNorm
                        │
                        ▼
                    LM Head
                 5376 → 262144
                        │
                        ▼
                   Logits
                        │
                        ▼
                 Sampling / Decode
                        │
                        ▼
                   Next Token
```
```
概念	Gemma4 配置	直观理解
Hidden Size	5376	一个 Token 的特征向量宽度
Sequence Length	最大运行长度 16000	一次输入有多少个 Token，即矩阵有多少行
Head	32 个 Q Head	把 Attention 分成 32 个并行计算组
Head Dim	256	每个 Q/K/V Head 内部的特征维度
KV Head	16	存储和提供 K、V 的 Head 数量
GQA	32 Q / 16 KV	每 2 个 Q Head 共享一组 K、V
Sliding Attention	Window 1024	每个 Token 只关注局部历史窗口
Full Attention	全局因果关注	可以关注全部允许访问的历史 Token
Intermediate Size	21504	FFN 中间层的宽度
LM Head	5376 → 262144	把隐藏特征映射为词表分数
```
