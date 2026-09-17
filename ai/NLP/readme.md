# 大语言模型核心知识体系：原理、训练、推理与多模态面试笔记

> **目标**：用“**是什么 → 为什么 → 怎么做 → 数学/数值例子 → 面试要点**”串起 LLM 核心知识。保持紧凑，重点覆盖架构、训练、推理、上下文、评测与多模态。

---

## I. 基础架构层

### 1.1 Tokenizer

**作用**：文本 $\rightarrow$ Token ID $\rightarrow$ Embedding。Tokenizer 决定文本如何离散化，直接影响上下文长度与训练/推理成本。

| 方法 | 单位 | 特点 |
| :--- | :--- | :--- |
| **Character** | 字符 | 词表小，但序列长 |
| **Word** | 单词 | 词表大，容易出现 OOV (Out of Vocabulary) 问题 |
| **Subword** | 子词 | 在词表大小与序列长度之间取得折中 |

*   **BPE (Byte Pair Encoding)**：从字符开始，统计相邻 Token 的频率，反复合并高频 pair，形成最终词表。
*   **数值例子**：`unhappiness` 可拆成 `un + happi + ness`；中文通常按字/子词切分。实际 Token 数取决于具体 Tokenizer，不能固定认为“1 字 = 1 Token”。
*   **面试要点**：Tokenizer 不只是预处理，它深刻影响上下文利用率、KV Cache 大小以及整体训练/推理成本。

### 1.2 Embedding

**作用**：Token ID $\rightarrow$ 连续向量。把离散符号映射到高维语义空间。

**余弦相似度**：
$$
sim(a,b)=\frac{a \cdot b}{||a|| ||b||}
$$
*例如*：两个向量 $a=(1,0), b=(0.8,0.6)$：
$$
sim=0.8
$$
表示两者方向较接近。

**关键点**：
*   Embedding 不是“天然具有语义”，而是在训练目标作用下形成了几何结构。
*   对比学习常通过“拉近正样本、推远负样本”来形成良好的语义空间。
*   Embedding 适合语义检索，但对精确事实、复杂逻辑并不天然可靠。

### 1.3 Transformer：QKV $\rightarrow$ Attention $\rightarrow$ FFN $\rightarrow$ Output

一个 Decoder Block 的经典数据流可抽象为：

```text
Token → Embedding / RoPE
          ↓
     Q/K/V Projection
          ↓
      Attention
          ↓
   Residual + Norm
          ↓
       MLP / FFN
          ↓
   Residual + Norm
          ↓
      Hidden State
          ↓
       LM Head
          ↓
        Logits
```

**Q / K / V 的直觉含义**：
*   **Q (Query)**：我现在想找什么？
*   **K (Key)**：每个历史 Token 提供什么索引/特征？
*   **V (Value)**：找到后真正取回什么内容？

$$
Q=XW_Q,\quad K=XW_K,\quad V=XW_V
$$

**标准 Attention 计算**：
$$
Attention(Q,K,V)=softmax\left(\frac{QK^T}{\sqrt{d_k}}\right)V
$$
*数值例子*：假设有 4 个 Token，每个 head 的维度（$d_k$）为 64，则 $QK^T$ 的形状是 $4 \times 4$；每个 Query 都要与 4 个 Key 计算相关性。

**复杂度**：
*   **计算复杂度**：$O(N^2d)$
*   **Attention Matrix 显存/内存**：$O(N^2)$

**LM Head**：
$$
logits=hW_{vocab}^T
$$
*数值例子*：若 hidden size=4096、词表大小=100K，则单个 Token 输出的 logits 维度为 100K。

### 1.4 RoPE：让 Attention 感知位置

RoPE (Rotary Position Embedding) 不直接给 Token 加上绝对位置向量，而是对 Q/K 施加与位置相关的旋转变换。

**二维旋转矩阵**：
$$
R_\theta=
\begin{bmatrix}
\cos\theta & -\sin\theta \\
\sin\theta & \cos\theta
\end{bmatrix}
$$
对于位置 $m$ 和 $n$：
$$
q_m=R_m q, \quad k_n=R_n k
$$
计算内积时：
$$
q_m^T k_n = q^T R_m^T R_n k = q^T R_{n-m} k
$$
由此，Attention 得分自然包含了相对位置差 $(n-m)$ 的信息。

**面试要点**：长上下文扩展（如位置插值、YaRN 等），本质是缓解训练长度与推理长度之间的**分布外推问题**，而不只是单纯地“增加位置编号”。

### 1.5 Attention 演进

*   **标准 Attention**：
    $$
    softmax(QK^T)V
    $$
    序列长度 $N$ 增长时存在 $N^2$ 级别的交互瓶颈。
*   **Linear Attention**：通过 Kernel Trick / 结合律改写计算顺序：
    $$
    \sum_j \phi(q_i)^T \phi(k_j)v_j
    $$
    可先对历史状态聚合：
    $$
    S=\sum_j \phi(k_j)v_j^T
    $$
    再与 Query 计算，从而把序列维度上的二次项降维。固定 head 维度时，复杂度约为 $O(Nd^2)$。**注意**：并非任何情况下都比高度优化的标准 Attention（如 FlashAttention）快。
*   **GLA / DeltaNet / Hybrid**：
    *   **GLA**：引入门控，历史状态按维度衰减或保留。
    *   **DeltaNet**：增量更新状态，强调“修正已有记忆”。
    *   **Hybrid 架构**：结合两者优势，局部精确检索用标准 Attention，长程记忆用线性结构。

### 1.6 经典架构对比：BERT / T5 / GPT

| 模型系列 | 核心架构 | 典型训练目标 / 核心用途 |
| :--- | :--- | :--- |
| **BERT** | Encoder-only | MLM（掩码语言模型），用于文本理解、分类、表示 |
| **T5** | Encoder-Decoder | Seq2Seq，适用于文本转换、翻译、摘要 |
| **GPT** | Decoder-only | Causal LM（因果语言模型），主导当前的文本生成 |

GPT 类模型通过自回归（Next Token Prediction）进行训练：
$$
P(x)=\prod_t P(x_t|x_{<t})
$$
Decoder 中的因果 Mask 严格保证当前位置无法“看到”未来的 Token。

---

## II. 训练与对齐

### 2.1 Pretraining 与 Scaling Law

**核心目标**：最小化负对数似然：
$$
L=-\sum_t \log P(x_t|x_{<t})
$$

**Chinchilla 粗略计算公式**：
$$
C \approx 6ND
$$
其中 $N$ 为参数量，$D$ 为训练 Token 数。
*数值例子*：70B 参数模型在 1.4T Token 上训练所需的 FLOPs：
$$
C \approx 6 \times 70\times 10^9 \times 1.4\times 10^{12} = 5.88\times 10^{23} \text{ FLOPs}
$$
*注意*：“约 20 Token/参数”是 Chinchilla 时代的经验，现代模型（如 Llama 3 训练了 15T Token）往往远超这个比例。

**Scaling Law**：扩大模型参数、数据量、计算量，通常会带来可预测的 Loss 下降；但现实中也受数据质量、网络架构、推理成本等强约束。

**ICL (In-Context Learning)**：推理时不更新权重参数，而是通过 Prompt 中的示例/上下文隐式改变模型的输出分布。

### 2.2 SFT (Supervised Fine-Tuning)

使用高质量的 `(instruction, response)` 对齐数据继续训练。
目标函数形式上不变，但通常**只对 Assistant 的输出部分计算 Loss**：
$$
L=-\sum_t \log P(y_t|x, y_{<t})
$$
*例如*：`用户: 解释 Raft 协议 \n 助手: Raft 是...` (仅对 "Raft 是..." 计算 Loss)。

### 2.3 RLHF / GRPO

**RLHF 常见演进路线**：
`Pretrain → SFT → Reward Model / Preference Data → RL → Aligned Model`

**GRPO (Group Relative Policy Optimization)**：对同一问题采样一组回答，利用组内 Reward 做相对优势计算。
$$
A_i=\frac{r_i-\mu_r}{\sigma_r+\epsilon}
$$
*数值例子*：一组回答的 rewards 为 $[8,6,4,2]$，则：
$$
\mu=5, \quad \sigma \approx 2.24
$$
第一条回答的优势函数：
$$
A_1 \approx (8-5)/2.24 = 1.34
$$
（最后一条约为 -1.34）。
*直觉*：不一定需要单独训练笨重的 Value Model，利用同题多样本的相对质量即可进行稳定的策略更新。

### 2.4 DAPO (Direct Alignment for Policy Optimization 范畴)

面向大规模 RL 训练稳定性与效率的一组工程/算法改进（常包含于现代 PPO 变体中）：
1.  **Clip-Higher**：调整 PPO clipping，避免过度惩罚高概率且正确（高 Reward）的提升。
2.  **Dynamic Sampling**：动态过滤或补充高质量样本。
3.  **Token-Level Policy Gradient**：在 Token 粒度上分配奖励（Credit Assignment）。
4.  **Overlong Reward Shaping**：针对过长回答进行惩罚/整形，避免“越长得分越高”的作弊行为。

### 2.5 Process Reward vs Outcome Reward

*   **Outcome Reward**：只看最终结果对不对（如：最终答案 ✗）。
*   **Process Reward**：对中间推理步骤（如 CoT）逐一评价。
    *   步骤1：列方程 ✓
    *   步骤2：移项 ✓
    *   步骤3：计算错误 ✗
*   **对比**：Outcome 只能提供稀疏的失败信号；Process Reward 能精准定位步骤 3 的错误。工程上 Process Reward 粒度更细但标注极贵；Outcome Reward 廉价但存在“信用分配 (Credit Assignment)”难题。

### 2.6 LoRA (Low-Rank Adaptation)

冻结 Base Model，仅训练低秩增量矩阵：
$$
W' = W + \Delta W, \quad \Delta W = BA
$$
*数值例子*：假设原矩阵 $W$ 为 $4096 \times 4096$，设置 rank $r=8$。
*   LoRA 需训练参数：$4096 \times 8 + 8 \times 4096 = 65,536$
*   原矩阵参数：$4096^2 = 16,777,216$
*   仅占约 $0.39\%$。

**为什么有效？** 经验性假设认为，大量下游任务的参数更新存在低维（低秩）固有结构。
**关键超参**：`rank (r)`, `target modules`, `learning rate`, `alpha`, `dropout`。

---

## III. 推理与部署

### 3.1 Logits $\rightarrow$ Softmax $\rightarrow$ Sampling

模型最后一层输出 Logits：
$$
z = [2, 1]
$$
加入 Temperature ($T$) 的 Softmax 转换概率：
$$
p_i = \frac{e^{z_i/T}}{\sum_j e^{z_j/T}}
$$
*数值分布变化*：
*   $T=1$：$[0.731, 0.269]$
*   $T=0.5$：$[0.881, 0.119]$ (更尖锐，趋向 Greedy)
*   $T=2$：$[0.622, 0.378]$ (更平缓，增加随机性)

*   **Top-K**：仅在概率最高的 K 个 Token 中采样。
*   **Top-P (Nucleus)**：按概率降序累加，保留累计概率刚达到 P 的候选集合。

### 3.2 VRAM：训练 vs 推理

**训练期显存大户**：`Weights + Gradients + Optimizer States + Activations`
使用混合精度 AdamW 的粗略估算：
*   FP16/BF16 weights：2 bytes
*   FP16/BF16 gradients：2 bytes
*   FP32 Adam states：8 bytes (momentum + variance)
*   合计基础占用约：$12\text{ bytes/param}$

*数值例子*：12B 模型基础训练显存：
$$
12B \times 12 \approx 144\text{ GB}
$$
（注：实际还受 Activation、并行策略影响）。

### 3.3 KV Cache

Decoder 推理时，避免重复计算历史 Token 的 Q/K/V，将 K/V 缓存。
**显存估算公式**：
$$
M_{KV} = 2 \times L \times H_{kv} \times D \times B \times S \times \text{bytes}
$$
*数值例子*：层数 $L=32$，$H_{kv}=32$，头维 $D=128$，Batch $B=1$，序列长 $S=8192$，精度 FP16 (2 bytes)：
$$
M = 2 \times 32 \times 32 \times 128 \times 8192 \times 1 \times 2 \approx 4\text{ GiB}
$$
(约 $0.5\text{ MiB/token}$)。

**GQA / MQA 优化**：直接减少 $H_{kv}$。
假设 Query Heads = 32：
*   **MHA**：32 KV heads
*   **GQA**：例如 8 KV heads $\rightarrow$ Cache 直接降为 MHA 的 1/4。
*   **MQA**：1 KV head $\rightarrow$ Cache 降为 1/32。

### 3.4 Prefill vs Decode

*   **Prefill (预填充)**：接收长 Prompt，一次性并行计算大量 Token 并填充 KV Cache。**计算密集型 (Compute Bound)**。
*   **Decode (解码)**：利用 KV Cache 逐个生成新 Token。每次只需计算一个 Token，但需从显存搬运整个历史 KV Cache。**显存带宽受限 (Memory / Bandwidth Bound)**。

### 3.5 Memory Wall (内存墙)

GPU 的算力 (FLOPs) 飞速增长，但显存带宽 (HBM Bandwidth) 增长缓慢。
**核心结论**：在 LLM 推理（特别是 Decode 阶段）中，优化的终极目标往往不是“减少计算量”，而是“减少 HBM 数据搬运”。

### 3.6 FlashAttention

普通 Attention：
$$
S = QK^T, \quad P = softmax(S), \quad O = PV
$$
中间会生成巨大的 $N \times N$ 矩阵，频繁读写 HBM 导致极慢。
**FlashAttention 解决方案**：
利用 **Tiling**（分块计算） + **Online Softmax**，将计算锁定在高速 SRAM 中完成，大幅减少 HBM I/O。
*面试注意*：FlashAttention 的计算复杂度依然是 $O(N^2d)$，它优化的是 **I/O 和内存访问**。

### 3.7 PagedAttention

KV Cache 管理的痛点：动态变长、容易产生显存碎片。
**思想借鉴 OS 虚拟内存**：
`Sequence → Logical Blocks → Block Table → Physical GPU KV Blocks`
*   允许 KV Cache 不连续存放。
*   极大减少显存碎片，提高并发 Batch Size。
*   原生支持 Prefix Sharing（如共享 System Prompt 的 KV Cache）。

### 3.8 Continuous Batching (In-flight Batching)

*   **传统 Static Batching**：以最长的请求为准，短请求完成后 GPU 会空转（出现气泡）。
*   **Continuous Batching**：以 Token / Iteration 为调度粒度。一个请求完成，立即踢出并动态插入新请求，时刻保持 GPU 的 Batch 被塞满，极大提升吞吐率 (Throughput)。

### 3.9 Chunked Prefill

超长 Prompt 的 Prefill 计算时间极长，如果一次性计算，会导致同 Batch 内正在 Decode 的其他请求被长时间阻塞（卡顿）。
**方案**：将 Long Prompt 截断成多个 Chunk，在各个 Chunk 计算的间隙穿插 Decode 计算。
**收益**：兼顾在线服务的 TTFT (首字延迟) 和 TPOT (单字延迟) 的公平性。

### 3.10 Speculative Decoding (投机解码)

利用小模型 (Draft) 和大模型 (Target) 协同：
`Small Model 生成 k 个候选 Token → Large Model 仅做一次 Forward 并行验证 → 接受正确前缀 + 修正第一个错误 Token`
*   **核心收益**：大模型的单次 Forward (Memory Bound) 可以替代多次自回归生成的耗时。
*   **关键要求**：算法需保证采样分布与目标模型严格一致（如基于 Rejection Sampling），而非简单地“只要不错就接受”。

### 3.11 Quantization (量化)

用低精度比特 (INT8, INT4) 表达权重或激活值，缓解 Memory Wall。
**对称量化直觉**：
$$
q = round(x/s), \quad x \approx s \cdot q
$$
*数值例子*：$x=0.73, s=0.1$
$$
q = round(7.3) = 7, \quad \hat{x} = 0.7
$$
量化误差：$|0.73 - 0.7| = 0.03$

*   **PTQ**：训练后量化（GPTQ, AWQ, SmoothQuant）。
*   **QAT**：量化感知训练（在训练时模拟量化截断）。

---

## IV. 训练工程

### 4.1 数据 Pipeline

`Raw Data → Cleaning → Quality Filter → Dedup (去重) → Data Mix → Tokenization → Training`
**Dedup (去重) 极其重要**：重复数据不仅浪费算力，还会导致模型产生“记忆性过拟合”，并且污染评测集。

### 4.2 Learning Rate 调度

经典曲线：`Warmup → Peak LR → Cosine Decay`
**Cosine Decay 公式**：
$$
lr(t) = lr_{min} + \frac{1}{2}(lr_{max}-lr_{min})(1+\cos(\pi t/T))
$$
Warmup 防止早期训练崩溃，Decay 帮助模型收敛到更平缓的极小值。

### 4.3 损失函数 Loss 监控

**交叉熵损失 (Cross Entropy)**：
$$
L=-\frac{1}{N}\sum_{i=1}^{N}\log p(y_i|x, y_{<i})
$$
*数值例子*：若预测正确 Token 的概率为 0.8，则：
$$
L=-\log(0.8) \approx 0.223
$$
如果 Loss 突然飞升 (Spike)，通常意味着梯度爆炸、脏数据或硬件故障。

### 4.4 分布式训练与并行化

*   **DP (Data Parallel)**：多卡各存一份完整模型，切分数据 Batch。
*   **TP (Tensor Parallel)**：将一层网络里的矩阵运算（如 FFN 的巨大权重）沿维度切开，分配给多卡计算（通信极密集，通常限节点内 NVLink）。
*   **PP (Pipeline Parallel)**：按层切分（如 1-8层给 GPU0，9-16层给 GPU1），产生微批次 Pipeline 以减少气泡。
*   **ZeRO 系列 (Zero Redundancy Optimizer)**：将模型状态打散（ZeRO-1: Optimizer States；ZeRO-2: + Gradients；ZeRO-3: + Parameters），本质是用通信带宽换取单卡显存。

### 4.5 Activation Checkpointing (Gradient Checkpointing)

反向传播需要前向传播的中间激活值 (Activations)。
如果全存：显存爆炸。
**策略**：前向时只保存部分关键节点的 Checkpoint，反向时从 Checkpoint 重新计算丢弃的激活值。
*   本质：**Compute ↔ Memory Trade-off (用额外计算量换取显存)**。

---

## V. 上下文与推理能力

### 5.1 Context Engineering

目标不是“把所有资料无脑塞进 Prompt”，而是：
**找到最小的高信噪比 Token 集合，最大化期望结果的生成概率。**
$$
Context\ Quality \gg Context\ Length
$$
常见手段：外部检索 (RAG)、总结压缩、结构化排版 (Markdown/XML)。

### 5.2 Lost in the Middle

长文本现象：模型对 Prompt 开头和结尾的指令/内容吸收最好，对**中间部分**的信息利用率大幅下降。
*   工程应对：关键指令置首或置尾；长文档分块 RAG 检索并重排序 (Re-rank)；强迫模型输出前先进行摘要回顾。

### 5.3 CoT (Chain-of-Thought)

原理：通过强迫模型输出中间推理步骤，赋予其更多的“外部化计算空间”和“自回归推演 Token”。
`直接回答：Q → A`
`CoT回答：Q → Step 1 → Step 2 → Step 3 → A`
*注意*：生成更多 Token 代表消耗了更多算力，但简单任务或存在严重误导上下文时，强行 CoT 反而容易“带偏”模型。

---

## VI. 幻觉与评测

### 6.1 幻觉 (Hallucination) 为什么发生？

LLM 的第一性原理是优化下一个 Token 的条件概率 $P(x_t|x_{<t})$，它**只是一个基于统计的语言生成器，而不是事实验证机**。
核心诱因：
*   参数化知识过期或不足。
*   自回归生成的 Exposure Bias（一旦说错一个字，容易顺着错下去）。
*   Attention 失焦或 Lost in the Middle。
*   对齐税（模型为了“讨好”人类表现得自信，即使它不知道答案）。

### 6.2 RAG 与 Claim Verification

防幻觉的高阶模式：
`生成回答 → 拆分为原子声明 (Atomic Claims) → RAG 检索可信证据 → Claim 与 Evidence 交叉验证 (NLI: Entailment/Contradiction) → 修正输出`

### 6.3 LLM-as-a-Judge

利用强模型（如 GPT-4）对候选模型打分。
*   **优势**：可扩展、低成本、统一评分维度。
*   **偏差陷阱 (Biases)**：
    *   **Position Bias**：倾向于给放在前面（或后面）的答案打高分。
    *   **Length Bias**：倾向于给“长篇大论”打高分（即使废话多）。
    *   **Self-Preference**：模型更喜欢自己生成的句子风格。

---

## VII. 多模态与图像生成

### 7.1 Visual Token 化

将图像切分成 Patch，再展平送入 Transformer。
*数值例子*：图片分辨率 $1024 \times 1024$，Patch 大小 $16 \times 16$。
Patch 数量：
$$
(1024/16)^2 = 64^2 = 4096
$$
如果不经下采样，相当于直接塞入了 4096 个“视觉 Token”。

### 7.2 生成范式：Autoregressive vs Diffusion

*   **Autoregressive (自回归)**：像语言一样，将视觉 Token 展平为 1D 序列，串行逐个预测（如早期 DALL-E，部分原生多模态）。
*   **Diffusion (扩散模型)**：通过向纯噪声不断执行多步去噪，最终还原出高维图像空间的数据分布。

### 7.3 DDPM 扩散模型数学直觉

**Forward (加噪过程)**：
$$
x_t = \sqrt{\bar{\alpha}_t}x_0 + \sqrt{1-\bar{\alpha}_t}\epsilon
$$
*   训练时：模型学习如何从带有时间步 $t$ 的噪声图像 $x_t$ 中预测出加入的噪声 $\epsilon$（或原图 $x_0$）。
*   推理时 (Reverse)：从纯高斯噪声开始，用模型一步步减去预测出的噪声，最后得到清晰图片。

---

## 八、核心知识地图

```text
                    LLM
                     │
       ┌─────────────┼─────────────┐
       ↓             ↓             ↓
   基础架构         Training      Inference
       │             │             │
 Tokenizer        Pretrain        Logits
 Embedding        SFT             Sampling
 Transformer      RLHF/GRPO       KV Cache
 Q/K/V            LoRA            Prefill/Decode
 Attention        DP/TP/PP        FlashAttention
 RoPE             ZeRO            PagedAttention
 FFN              Checkpoint      Continuous Batching
 LM Head                          Speculative Decoding
                                   Quantization
                     │
                     ↓
             Context / Reasoning
                     │
       Context Engineering / CoT
       Lost in the Middle / Memory
                     │
                     ↓
               Hallucination
                     │
       RAG / Verification / Judge
                     │
                     ↓
               Multimodal
           Vision Token / Diffusion
```

---

## 九、面试高频问答速记

1.  **为什么 Decode 比 Prefill 更容易受显存带宽限制 (Memory Bound)？**
    Decode 每步只生成 1 个新 Token（计算量极小），但必须从显存中完整读取整个历史的 KV Cache（访存极大）。数据搬运的时间远超 GPU 计算时间。
2.  **FlashAttention 为什么快？**
    它并未把 Attention 复杂度从 $O(N^2)$ 降到 $O(N)$，而是利用 Tiling 和 Online Softmax 将中间庞大的 $N \times N$ 矩阵计算封锁在 GPU 高速 SRAM 中，消除了昂贵的 HBM 读写开销。
3.  **PagedAttention 解决了什么？**
    解决了传统 KV Cache 预分配导致的显存碎片问题。类似 OS 虚拟内存，通过非连续的块管理，大幅提升了显存利用率和线上并发吞吐量。
4.  **GQA 为什么能省显存？**
    KV Cache 的大小与 K/V 的 Head 数量线性相关。GQA 保持多 Query Heads 的同时大幅缩减 KV Heads 数量，从而直接砍掉大部分 KV Cache 显存占用，且几乎不损耗模型能力。
5.  **Continuous Batching 为什么能提高吞吐？**
    打破静态 Batch 必须等待最长请求结束的限制，实现在 Token 生成中动态踢出完成的请求、加入新请求，消灭 GPU 气泡，让算力始终打满。
6.  **LoRA 为什么能显著降低训练成本？**
    通过旁路低秩矩阵 $B \times A$ 模拟参数更新，冻结原庞大模型权重。参数更新量骤降至 $< 1\%$，使得 Optimizer States 和 Gradients 显存占用极具下降，单卡即可微调大模型。
7.  **为什么模型有长上下文，但不一定能“记住”所有内容？**
    Context Window 只是理论“容量”，实际受限于 Attention 的失焦、RoPE 的外推能力、训练数据分布以及 Lost in the Middle 现象，有效信息提取能力随长度衰减。
8.  **LLM 系统优化应该从哪里排查入手？**
    *   **质量差** $\rightarrow$ 数据清洗 / Prompt 优化 / RAG 构建
    *   **首字慢 (TTFT)** $\rightarrow$ 优化 Prefill 耗时 / Chunked Prefill
    *   **每字慢 (TPOT)** $\rightarrow$ 优化 Decode (KV 命中率 / 带宽受限 / 算子优化)
    *   **OOM (显存炸)** $\rightarrow$ Quantization / PagedAttention / 调小 Batch
    *   **吞吐低** $\rightarrow$ Continuous Batching / vLLM / Tensor Parallel
