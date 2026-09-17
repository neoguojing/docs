# 大语言模型核心知识体系：原理、训练、推理与多模态面试笔记

> 目标：以“是什么 → 为什么 → 怎么做 → 数学/数值例子 → 面试要点”为主线，覆盖 LLM 基础架构、训练、对齐、推理部署、上下文、幻觉评测与多模态。内容保持紧凑，适合面试复习。

---

## I. 基础架构层

### 1.1 Tokenizer

**作用**：文本 → Token ID → Embedding。Tokenizer 决定文本如何离散化，并直接影响上下文长度与训练/推理成本。

| 方法 | 单位 | 特点 |
| --- | --- | --- |
| **Character** | 字符 | 词表小，但序列长 |
| **Word** | 单词 | 词表大，存在 OOV |
| **Subword** | 子词 | 在词表规模与序列长度之间折中 |

* **BPE**：从字符开始，统计相邻 Token 的频率，反复合并高频 pair，最终形成词表。
* **例子**：`unhappiness` 可以拆成 `un + happi + ness`。中文 Token 数取决于具体 Tokenizer，不能简单固定为“1 字 = 1 Token”。
* **面试**：Tokenizer 不只是预处理，它会影响上下文利用率、KV Cache 大小以及训练/推理成本。

### 1.2 Embedding

**作用**：Token ID → 连续向量，把离散符号映射到高维语义空间。

$$sim(a,b)=\frac{a\cdot b}{\vert{}\vert{}a\vert{}\vert{}\vert{}\vert{}b\vert{}\vert{}}$$

例如 $a=(1,0)$，$b=(0.8,0.6)$：

$$sim(a,b)=0.8$$

**关键点**：

* Embedding 的语义结构来自训练目标，而不是天然存在。
* 对比学习通过拉近正样本、推远负样本形成语义空间。
* Embedding 适合语义检索，但不天然保证事实准确或复杂推理能力。

### 1.3 Transformer：QKV → Attention → FFN → LM Head

Decoder Block 可抽象为：

```text
Token → Embedding / RoPE
             ↓
       Q / K / V Projection
             ↓
          Attention
             ↓
       Residual + Norm
             ↓
           MLP/FFN
             ↓
       Residual + Norm
             ↓
        Hidden State
             ↓
          LM Head
             ↓
           Logits

```

**Q/K/V**：

* **Q**：当前 Token 想查询什么。
* **K**：历史 Token 提供什么匹配信息。
* **V**：匹配后真正取回的信息。

$$Q=XW_Q,\quad K=XW_K,\quad V=XW_V$$

**标准 Attention**：


$$Attention(Q,K,V)=softmax\left(\frac{QK^T}{\sqrt{d_k}}\right)V$$

数值例子：4 个 Token、head dimension=64，则 $QK^T$ 是 4×4；每个 Query 都与 4 个 Key 计算相关性。

**复杂度**：

* 计算：常写 $O(N^2d)$
* Attention Matrix 内存：$O(N^2)$

**LM Head**：


$$logits=hW_{vocab}^{T}$$


例如 hidden size=4096、词表=100K，则单个 Token 最终得到约 100K 维 logits。

### 1.4 RoPE：让 Attention 感知位置

二维旋转矩阵：


$$R_\theta=\begin{bmatrix}\cos\theta&-\sin\theta\\\sin\theta&\cos\theta\end{bmatrix}$$

位置 $m,n$：


$$q_m=R_mq,\quad k_n=R_nk$$

于是：


$$q_m^Tk_n=q^TR_m^TR_nk=q^TR_{n-m}k$$

因此 Attention 自然包含相对位置差。
**长上下文**：位置插值、YaRN 等方法用于缓解训练长度与推理长度之间的分布/外推问题，并不只是“增加位置编号”。

### 1.5 Attention 演进

**标准 Attention**


$$softmax(QK^T)V$$


序列长度 $N$ 增长时存在 $N^2$ 交互。

**Linear Attention**
利用 Kernel/Associativity 改变计算顺序，例如：


$$\sum_j\phi(q_i)^T\phi(k_j)v_j$$


先聚合：


$$S=\sum_j\phi(k_j)v_j^T$$


再计算 Query，从而降低序列维度上的二次项。固定 head dimension 时常见复杂度可写为 $O(Nd^2)$，并非所有场景都比高度优化的标准 Attention 快。

**GLA / DeltaNet / Hybrid**

* **GLA**：通过门控控制历史状态的保留与衰减。
* **DeltaNet**：通过增量更新状态修正已有记忆。
* **Hybrid**：在需要精确检索的位置保留 Attention，同时利用线性结构处理长程状态。

### 1.6 BERT / T5 / GPT

| 模型架构 | 典型用途 |
| --- | --- |
| **BERT** (Encoder-only) | MLM、理解、分类、表示 |
| **T5** (Encoder-Decoder) | Seq2Seq、文本转换 |
| **GPT** (Decoder-only) | Causal LM、生成 |

GPT 类模型通过：


$$P(x)=\prod_tP(x_t\vert{}x_{<t})$$


进行 Next Token Prediction。
Decoder 使用 Causal Mask，保证当前位置不能看到未来 Token。

---

## II. 训练与对齐

### 2.1 Pretraining 与 Scaling Law

核心目标：


$$L=-\sum_t\log P(x_t\vert{}x_{<t})$$

Chinchilla 常用粗略计算：


$$C\approx6ND$$


其中 $N$ 为参数量，$D$ 为训练 Token 数。

例如 70B × 1.4T：


$$C\approx6\times70\times10^9\times1.4\times10^{12}=5.88\times10^{23}\text{ FLOPs}$$

“约 20 Token/参数”是 Chinchilla 时代的经典经验关系，不是所有现代模型的统一最优值。
**Scaling Law**：模型、数据、计算规模增加通常带来可预测的损失变化，但还受数据质量、架构、训练目标和推理成本约束。
**ICL**：In-Context Learning 不更新模型参数，而是利用 Prompt 中的示例/上下文改变当前行为。

### 2.2 SFT (Supervised Fine-Tuning)

使用高质量 (instruction, response) 数据继续训练。


$$L=-\sum_t\log P(y_t\vert{}x,y_{<t})$$

例如：

> 用户：解释 Raft
> 助手：Raft 是……

实际训练中通常只对 Assistant 输出部分计算 loss。

### 2.3 RLHF / GRPO

典型 RLHF：
`Pretrain → SFT → Preference / Reward → RL → Aligned Model`

GRPO 的常见形式：对同一问题采样一组回答，利用组内 reward 计算相对优势：


$$A_i=\frac{r_i-\mu_r}{\sigma_r+\epsilon}$$

例如 rewards 为 $[8,6,4,2]$，$\mu=5,\quad\sigma\approx2.24$。
第一条：$A_1\approx(8-5)/2.24=1.34$
最后一条约为 -1.34。
直觉：利用同题多样本的相对质量产生训练信号，不一定需要传统 PPO 中独立的 Value Model。

### 2.4 DAPO

可理解为面向大规模 RL 训练稳定性与效率的一组改进，核心概念包括：

* **Clip-Higher**：调整 clipping 行为，减少过度限制。
* **Dynamic Sampling**：动态筛选/补充有效样本。
* **Token-Level Policy Gradient**：更细粒度地利用 Token 级训练信号。
* **Overlong Reward Shaping**：处理过长回答带来的 reward 问题。

### 2.5 Process Reward vs Outcome Reward

* **Outcome Reward**：只看最终结果。
* **Process Reward**：评价中间推理步骤。

例如：

> 步骤1：列方程 ✓
> 步骤2：移项 ✓
> 步骤3：计算错误 ✗
> 最终答案 ✗

Outcome 只能知道最终错误；Process Reward 可以定位步骤 3。
**取舍**：Process Reward 更细，但标注/验证成本更高；Outcome Reward 简单，但信用分配更困难。

### 2.6 LoRA

冻结 Base Model，仅训练低秩增量：


$$W'=W+\Delta W,\quad\Delta W=BA$$

若：

* $W=4096\times4096$
* rank $r=8$

LoRA 参数：$4096\times8+8\times4096=65,536$
原矩阵：$4096^2=16,777,216$
仅约 0.39%。

**为什么有效**：下游任务的参数更新经常存在低维/低秩结构，这是经验性假设，并非普适定理。
**关键超参**：rank、target modules、learning rate、alpha、dropout。

---

## III. 推理与部署

### 3.1 Logits → Softmax → Sampling

模型输出：$z=[2,1]$

Softmax：


$$p_i=\frac{e^{z_i/T}}{\sum_j e^{z_j/T}}$$

近似结果：

| T | 概率 |
| --- | --- |
| **0.5** | [0.881, 0.119] |
| **1** | [0.731, 0.269] |
| **2** | [0.622, 0.378] |

* $T$ 越低分布越尖锐；$T$ 越高越随机。
* $T=0$ 通常由实现特殊处理为 Greedy/近似 Greedy，而不是执行除零。
* **Top-K**：只保留概率最高 K 个 Token。
* **Top-P**：保留累计概率达到 P 的最小候选集合。

### 3.2 VRAM：训练 vs 推理

训练显存主要包括：Weights + Gradients + Optimizer States + Activations。

混合精度 AdamW 的常见粗略估算：

* Weights：2 bytes/param
* Gradients：2 bytes/param
* Adam States：8 bytes/param
约：$12\text{ bytes/param}$

例如 12B：$12B\times12\approx144\text{GB}$
这是静态粗估；实际还会受到 Master Weight、Activation、临时 Buffer、并行策略等影响。

### 3.3 KV Cache

历史 Token 的 K/V 在 Decode 阶段可以缓存，避免重复计算。


$$M_{KV}=2\times L\times H_{kv}\times D\times B\times S\times \text{bytes}$$

例如：$L=32$，$H_{kv}=32$，$D=128$，$B=1$，$S=8192$，FP16=2 bytes。


$$M=2\times32\times32\times128\times8192\times2\approx4\text{GiB}$$


约 0.5 MiB/token。

GQA/MQA 通过减少 KV Head 降低 Cache（当 Q Head=32 时）：

* **MHA**：32 KV heads
* **GQA**：8 KV heads → Cache 约为 MHA 的 1/4
* **MQA**：1 KV head → 约为 MHA 的 1/32

### 3.4 Prefill vs Decode

* **Prefill**：`Prompt → 一次性处理大量 Token → KV Cache`。GPU 并行度高，通常偏 Compute Bound。
* **Decode**：`KV Cache + 上一个 Token → 下一个 Token`。每步通常只生成一个 Token，但需要读取大量历史 KV，因此更容易受到显存带宽影响。

常见判断：

* Prefill → Compute Bound
* Decode → Memory / Bandwidth Bound

### 3.5 Memory Wall

GPU FLOPs 增长很快，但数据搬运速度相对成为瓶颈。Decode 中，即使每次只计算一个新 Token，也可能需要读取大量 KV Cache。LLM 推理优化不仅是减少 FLOPs，更重要的是减少数据搬运、提高显存带宽利用率。

### 3.6 FlashAttention

普通 Attention：


$$S=QK^T,\quad P=softmax(S),\quad O=PV$$


中间 $N\times N$ 矩阵频繁读写 HBM，造成大量 IO。

FlashAttention 使用：

* Tiling
* Online Softmax
* GPU 片上高速存储

**重点**：

* 数学复杂度仍是 $O(N^2d)$。
* 核心优化是 IO / Memory Access。
* 不是把 Attention 变成 $O(N)$。

### 3.7 PagedAttention

把 KV Cache 按固定 Block 管理：
`Sequence → Logical Blocks → Block Table → Physical KV Blocks`

类似虚拟内存：

* 减少显存碎片
* 降低连续大块分配要求
* 支持 Prefix/Block Sharing
* 便于动态管理不同长度请求

它不改变 KV Cache 的理论大小，主要优化显存管理与利用率。

### 3.8 Continuous Batching

Static Batching 容易出现短请求结束后的 GPU 空洞：

> Step 1: A B C
> Step 2: A B C
> Step 3: A C D   ← B 完成，D 加入
> Step 4: A D E

Continuous Batching 以迭代/Token 为粒度动态加入、移除请求，提高 GPU 利用率和吞吐。

### 3.9 Chunked Prefill

长 Prompt 一次性 Prefill 可能长时间阻塞 Decode。因此拆成：
`Long Prompt → Chunk1 → Chunk2 → Chunk3 → ...`
在 Chunk 之间穿插 Decode，改善在线服务的公平性与延迟表现。

### 3.10 Speculative Decoding

`Small Model → Draft k Tokens → Large Model 一次验证 → 接受部分 + 修正`

小模型负责便宜地产生候选，大模型负责验证，从而减少大模型逐 Token Decode 次数。正确实现需要保持目标模型的采样分布，而不是简单“全部接受 Draft”。

### 3.11 Quantization

目标：低比特表示权重/激活，降低显存和带宽需求。
一种常见对称量化：


$$q=\text{round}(x/s),\quad\hat{x}=sq$$

例如 $x=0.73$，$s=0.1$：


$$q=\text{round}(7.3)=7,\quad\hat{x}=0.7$$


误差：$\vert{}0.73-0.7\vert{}=0.03$

常见：

* FP16/BF16
* INT8
* INT4
* **PTQ**：训练后量化。
* **QAT**：训练时模拟量化。
* **GPTQ**：利用近似二阶信息进行后训练权重量化。
* **AWQ**：根据激活统计识别对量化敏感的权重结构。
**难点**：Outlier、层间敏感度、激活分布、任务精度损失。

---

## IV. 训练工程

### 4.1 数据

`Raw Data → Cleaning → Quality Filter → Dedup → Data Mix → Tokenization → Training`

* **Dedup**：减少重复数据、训练浪费以及潜在评测污染。
* **Data Mix**：代码、网页、书籍、数学、对话等比例会影响最终能力。

### 4.2 Learning Rate

典型：`Warmup → Peak LR → Decay`

Cosine Decay：


$$lr(t)=lr_{min}+\frac{1}{2}(lr_{max}-lr_{min})(1+\cos(\pi t/T))$$


Warmup 避免训练初期更新过大；后期降低 LR 有利于稳定收敛。

### 4.3 Loss

Causal LM Cross Entropy：


$$L=-\frac{1}{N}\sum_i\log p(y_i\vert{}x,y_{<i})$$

若真实 Token 概率为 0.8：


$$L=-\log(0.8)\approx0.223$$


真实概率越低，loss 越大。

### 4.4 Precision / Throughput

常见：

* **FP32**：精度高、显存大
* **FP16/BF16**：训练常用
* **FP8**：部分现代训练/推理场景

关注：Tokens/s、Samples/s、MFU/HFU、GPU 利用率、Communication Overhead。

### 4.5 DP / TP / PP / ZeRO

* **DP（Data Parallel）**：每张 GPU 一份模型，切分数据。
* **TP（Tensor Parallel）**：把矩阵/Layer 内部计算切到多 GPU。
* **PP（Pipeline Parallel）**：按 Layer 切分（GPU0: Layer 1-8, GPU1: 9-16, GPU2: 17-24）。
* **ZeRO**：
* ZeRO-1：切 Optimizer States
* ZeRO-2：+ Gradients
* ZeRO-3：+ Parameters



核心：降低单卡显存。

### 4.6 Activation Checkpointing

不保存所有中间 Activation，只保存部分 Checkpoint，反向传播时重新计算。显存 ↓，计算量 ↑。本质是 Compute ↔ Memory Trade-off。

### 4.7 Fine-Tuning

可选路径：Full Fine-Tuning、LoRA / Adapter、QLoRA、Prompt Tuning。选择主要取决于模型规模、GPU 预算、任务规模以及是否需要大范围修改参数。

---

## V. 上下文与推理能力

### 5.1 Context Engineering

目标：找到最小的高信噪比 Token 集合，最大化期望结果概率。
主要手段：

* **压缩/重启**：摘要历史上下文。
* **外化记忆**：把关键信息写入文件/数据库。
* **检索**：只取当前任务相关内容。
* **结构化上下文**：任务状态、约束、工具结果分区。
* **工具输出控制**：避免 Tool Context Explosion。

核心：$Context\ Quality\gg Context\ Length$

### 5.2 Lost in the Middle

长上下文中，中间位置的信息利用率可能下降。工程方法：

* 关键信息放在显眼位置
* 长文档分块检索
* 对检索结果排序
* 摘要压缩
* 避免无关上下文进入 Prompt

### 5.3 CoT (Chain-of-Thought)

通过增加中间推理 Token，提供更多外部化工作空间：

* 直接回答：`A → Answer`
* CoT：`A → Step1 → Step2 → Step3 → Answer`

更多 Token 通常意味着更多推理计算，但不保证一定更准确；简单任务或过长推理可能产生额外错误。

---

## VI. 幻觉与评测

### 6.1 为什么会幻觉？

LLM 基本目标优化的是 Next Token 概率 $P(x_t\vert{}x_{<t})$，而不是事实真实性。
主要来源：

* 参数知识是压缩后的统计表示。
* 知识存在边界与时效性。
* RAG/检索可能失败。
* Attention 可能未正确提取相关信息。
* 长上下文存在 Lost in the Middle。
* 自回归生成存在 Exposure Bias。
* 对齐训练不等于事实验证。

### 6.2 RAG / Claim Verification

`Answer → Atomic Claims → Retrieve Evidence → Claim ↔ Evidence → Entail / Contradict / Unknown → Final Verification`

例如：“公司 A 在 2025 年营收 100 亿。”
拆成：主体（公司 A）、时间（2025）、指标（营收）、数值（100 亿）。再分别寻找证据。

### 6.3 多次采样一致性

* A A A A → 高一致性
* A B C D → 高不确定性

但：一致 ≠ 正确。因此最好结合外部证据、检索和规则验证。

### 6.4 LLM-as-a-Judge

让 LLM 按 Rubric 对答案评分/比较。

* **优势**：可扩展、成本低于全人工、可统一评价标准。
* **风险**：Position Bias、Length/Verbosity Bias、Style Bias、Self-preference、Knowledge Boundary。
* **改进**：明确 Rubric、A/B 顺序交换、Multi-Judge、客观指标、人工抽检。

---

## VII. 多模态与图像生成

### 7.1 Visual Token

图像可以先通过视觉编码器转换成视觉 Token。
例如图片 1024×1024，Patch 16×16，Patch 数：


$$(1024/16)^2=4096$$


约 4096 个视觉 Token。实际模型可能进行下采样/压缩，因此真实 Token 数取决于具体架构。

### 7.2 Autoregressive vs Diffusion

* **Autoregressive**：`Token1 → Token2 → Token3 → ...`。统一序列建模，但生成通常串行。
* **Diffusion**：`Noise → Denoise → Denoise → ... → Image`。通过多步去噪生成图像。

### 7.3 DDPM 数学直觉

Forward 加噪：


$$x_t=\sqrt{\bar\alpha_t}x_0+\sqrt{1-\bar\alpha_t}\epsilon$$

例如 $x_0=0.8$，$\bar\alpha_t=0.5$，$\epsilon=0.1$。
得到：


$$x_t\approx0.636$$


模型学习从噪声状态恢复原始数据相关信息，再执行反向去噪。

### 7.4 图像生成难点

* **全局一致性**：结构、人物、物体关系。
* **局部细节**：纹理、边缘、文字。
* **文本-视觉对齐**：多个对象与关系同时满足。
* **计算量**：高分辨率意味着更大的视觉表示。

---

## VIII. 核心知识地图

```text
                         LLM
                          │
        ┌─────────────────┼─────────────────┐
        ↓                 ↓                 ↓
    基础架构            Training          Inference
        │                 │                 │
 Tokenizer            Pretrain            Logits
 Embedding             SFT                Sampling
 Transformer           RLHF/GRPO          KV Cache
 Q/K/V                  LoRA               Prefill/Decode
 Attention              DP/TP/PP           FlashAttention
 RoPE                   ZeRO               PagedAttention
 FFN                    Checkpoint         Continuous Batching
 LM Head                                   Speculative
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

## IX. 高频面试题

1. **为什么 Decode 更容易受显存带宽影响？**
Decode 每步只生成少量新 Token，但要读取历史 KV Cache；计算量相对小，数据搬运占比高，因此更容易 Memory Bound。
2. **FlashAttention 为什么快？**
不是把 Attention 从 $O(N^2)$ 变成 $O(N)$，而是通过 Tiling + Online Softmax 减少 HBM IO。
3. **PagedAttention 解决什么？**
解决 KV Cache 动态分配、碎片和共享问题，提高显存利用率；不改变 KV Cache 理论大小。
4. **GQA 为什么省显存？**
KV Cache 与 $H_{kv}$ 成正比；减少 KV Head 就直接减少 Cache。
5. **Continuous Batching 为什么提高吞吐？**
请求可以在 Token 生成过程中动态加入/退出，减少静态 Batch 空洞，提高 GPU 利用率。
6. **LoRA 为什么减少训练成本？**
只训练低秩增量矩阵 BA，冻结大部分 Base 参数，因此参数、梯度、优化器状态都显著减少。
7. **Speculative Decoding 为什么有效？**
小模型生成候选，大模型批量验证；如果多数候选被接受，就减少大模型逐 Token Decode 次数。
8. **长上下文为什么不等于模型能有效利用全部内容？**
Context Window 是容量，不代表有效利用率；还受训练分布、位置编码、Attention 和 Lost in the Middle 影响。
9. **为什么 LLM 会幻觉？**
Next Token Prediction 优化语言概率而非事实验证；需要 RAG、工具、检索与 Claim Verification 增强可靠性。
10. **LLM 系统优化从哪里入手？**
* 质量问题 → 数据 / Prompt / RAG / 模型
* 延迟问题 → Prefill / Decode / KV / Batch
* 显存问题 → Quantization / GQA / PagedAttention
* 吞吐问题 → Continuous Batching / Parallelism
* 稳定性问题 → Scheduling / Timeout / Retry / Isolation
