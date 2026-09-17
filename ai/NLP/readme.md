# 大语言模型核心知识体系：原理、训练、推理与多模态面试笔记

> 目标：用“**是什么 → 为什么 → 怎么做 → 数学/数值例子 → 面试要点**”串起 LLM 核心知识。保持紧凑，重点覆盖架构、训练、推理、上下文、评测与多模态。

---

## I. 基础架构层

### 1.1 Tokenizer

**作用**：文本 → Token ID → Embedding。Tokenizer 决定文本如何离散化，直接影响上下文长度与训练/推理成本。

| 方法 | 单位 | 特点 |
|---|---|---|
| Character | 字符 | 词表小，但序列长 |
| Word | 单词 | 词表大，OOV 问题 |
| Subword | 子词 | 在词表与序列长度间折中 |

**BPE**：从字符开始，统计相邻 Token 的频率，反复合并高频 pair，形成词表。

**数值例子**：`unhappiness` 可拆成 `un + happi + ness`；中文通常按字/子词切分，实际 Token 数取决于具体 Tokenizer，不能固定认为“1 字 = 1 Token”。

**面试**：Tokenizer 不只是预处理，它影响上下文利用率、KV Cache 大小、训练/推理成本。

### 1.2 Embedding

**作用**：Token ID → 连续向量；把离散符号映射到高维语义空间。

余弦相似度：
\[
sim(a,b)=\frac{a\cdot b}{||a||||b||}
\]

例如两个向量 `a=(1,0), b=(0.8,0.6)`：
\[
sim=0.8
\]
表示方向较接近。

**关键点**：
- Embedding 不是“天然具有语义”，而是在训练目标作用下形成几何结构。
- 对比学习常通过拉近正样本、推远负样本形成语义空间。
- Embedding 适合语义检索，但对精确事实、复杂逻辑并不天然可靠。

### 1.3 Transformer：QKV → Attention → FFN → Output

一个 Decoder Block 可抽象为：

```text
Token → Embedding/RoPE
          ↓
     Q/K/V Projection
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
- Q：我现在想找什么？
- K：每个历史 Token 提供什么索引/特征？
- V：找到后真正取回什么内容。
\[ Q=XW_Q,\quad K=XW_K,\quad V=XW_V \]

**标准 Attention**：
\[ Attention(Q,K,V)=softmax(\frac{QK^T}{\sqrt{d_k}})V \]
例如 4 个 Token，每个 head 的维度为 64，则 QKᵀ 是 4×4；每个 Query 都要与 4 个 Key 计算相关性。

**复杂度**：
- 计算：常写为 \(O(N^2d)\)
- Attention Matrix 内存：\(O(N^2)\)

**LM Head**：
\[ logits=hW_{vocab}^T \]
例如 hidden size=4096、词表=100K，则一个 Token 的 logits 维度为 100K。

### 1.4 RoPE：让 Attention 感知位置
RoPE 不直接给 Token 加一个位置向量，而是对 Q/K 做位置相关旋转。

二维旋转：
\[
R_\theta=
\begin{bmatrix}
\cos\theta&-\sin\theta\\
\sin\theta&\cos\theta
\end{bmatrix}
\]
位置 m, n：
\[ q_m=R_mq,\quad k_n=R_nk \]
于是：
\[ q_m^Tk_n=q^TR_m^TR_nk=q^TR_{n-m}k \]
因此 Attention 中自然包含相对位置差。

长上下文：常见方向包括位置插值、YaRN 等，本质是缓解训练长度与推理长度之间的分布/外推问题，而不只是“增加位置编号”。

### 1.5 Attention 演进
**标准 Attention**：
\[ softmax(QK^T)V \]
序列长度 N 增长时存在 \(N^2\) 交互。

**Linear Attention**：通过 Kernel/Associativity 改写计算顺序，例如：
\[ \sum_j \phi(q_i)^T\phi(k_j)v_j \]
可先聚合：
\[ S=\sum_j\phi(k_j)v_j^T \]
再计算 Query，从而把序列维度上的二次项降下来。固定 head dimension 时常见复杂度可写为 \(O(Nd^2)\)，并非任何情况下都比优化后的标准 Attention 快。

**GLA / DeltaNet / Hybrid**：
- GLA：引入门控，让历史状态按维度衰减/保留。
- DeltaNet：通过增量更新状态，更强调“修正已有记忆”。
- Hybrid：在局部/精确检索需求上保留 Attention，同时用线性结构承担长程记忆。

### 1.6 BERT / T5 / GPT
| 模型 | 架构 | 典型训练/用途 |
|---|---|---|
| BERT | Encoder-only | MLM、理解/分类/表示 |
| T5 | Encoder-Decoder | Seq2Seq、文本转换 |
| GPT | Decoder-only | Causal LM、生成 |

GPT 类模型通过：
\[ P(x)=\prod_tP(x_t|x_{<t}) \]
进行 Next Token Prediction。
Decoder 的因果 Mask 保证当前位置不能看到未来 Token。

## II. 训练与对齐

### 2.1 Pretraining 与 Scaling Law
核心目标：
\[ L=-\sum_t\log P(x_t|x_{<t}) \]

Chinchilla 常用粗略计算：
\[ C\approx6ND \]
其中 N=参数量，D=训练 Token 数。
例如 70B × 1.4T：
\[ C\approx6\times70\times10^9\times1.4\times10^{12}=5.88\times10^{23}FLOPs \]
“约 20 Token/参数”是 Chinchilla 时代的经典经验关系，并非所有现代模型的统一最优点。

Scaling Law：扩大模型、数据、计算通常带来可预测的损失下降；同时存在数据质量、架构、训练目标、推理成本等约束。

ICL：In-Context Learning 不更新参数，而是通过 Prompt 中的示例/上下文改变当前推理行为。

### 2.2 SFT
Supervised Fine-Tuning：使用高质量 (instruction, response) 数据继续训练。
目标仍可理解为：
\[ L=-\sum_t\log P(y_t|x,y_{<t}) \]
例如：
- 用户：解释 Raft
- 助手：Raft 是……
通常只对 Assistant 输出部分计算 loss。

### 2.3 RLHF / GRPO
RLHF 常见流程：
Pretrain → SFT → Reward Model / Preference → RL → Aligned Model

GRPO（Group Relative Policy Optimization）的常见形式：对同一问题采样一组回答，利用组内 reward 做相对优势。
\[ A_i=\frac{r_i-\mu_r}{\sigma_r+\epsilon} \]
例如 rewards：[8,6,4,2]
\[ \mu=5,\quad\sigma\approx2.24 \]
于是第一条回答：
\[ A_1\approx(8-5)/2.24=1.34 \]
最后一条约为 -1.34。
直觉：不一定需要单独训练一个 Value Model，而是利用同题多样本的相对质量进行更新。

### 2.4 DAPO
可把 DAPO 理解为面向大规模 RL 训练稳定性与效率的一组工程/算法改进，常见要点：
- Clip-Higher：调整 PPO 类 clipping 行为，避免过度限制高概率提升。
- Dynamic Sampling：动态过滤/补充有效样本。
- Token-Level Policy Gradient：更细粒度地进行 Token 级优化。
- Overlong Reward Shaping：处理过长回答，避免长度直接破坏训练信号。

### 2.5 Process Reward vs Outcome Reward
- Outcome Reward：只看最终答案对不对。
- Process Reward：评价中间推理步骤。
例如数学题：
- 步骤1：列方程 ✓
- 步骤2：移项 ✓
- 步骤3：计算错误 ✗
- 最终答案 ✗
Outcome 只能看到最终错误；Process Reward 可以定位步骤 3。
工程上：Process Reward 更细，但标注/验证成本更高；Outcome Reward 简单，但信用分配更困难。

### 2.6 LoRA
冻结 Base Model，仅训练低秩增量：
\[ W'=W+\Delta W,\quad\Delta W=BA \]
若：
- W: 4096×4096
- rank r=8
LoRA 参数：
\[ 4096\times8+8\times4096=65,536 \]
原矩阵：
\[ 4096^2=16,777,216 \]
仅约 0.39%。

为什么有效：大量下游任务的参数更新存在低维/低秩结构，这是经验性假设，不是普适定理。
关键超参：rank、target modules、learning rate、alpha、dropout。

## III. 推理与部署

### 3.1 Logits → Softmax → Sampling
模型输出 logits：
\[ z=[2,1] \]
Softmax：
\[ p_i=\frac{e^{z_i/T}}{\sum_j e^{z_j/T}} \]
大致：
- T=1：[0.731, 0.269]
- T=0.5：[0.881, 0.119]
- T=2：[0.622, 0.378]
T 越低分布越尖锐；T 越高越随机。
T=0 实际通常由实现特殊处理为 Greedy/近似 Greedy，而不是执行除零。

Top-K：只保留概率最高 K 个 Token。
Top-P：保留累计概率达到 P 的最小候选集合。

### 3.2 VRAM：训练 vs 推理
训练显存主要包括：Weights + Gradients + Optimizer States + Activations
混合精度 AdamW 的常见粗略估算：
- FP16/BF16 weights：2 bytes
- FP16/BF16 gradients：2 bytes
- FP32 Adam states：8 bytes
约：\(12\ bytes/param\)
例如 12B：
\[ 12B\times12\approx144GB \]
这只是静态粗估；实际还受 Master Weight、Activation、临时 Buffer、并行策略等影响。

### 3.3 KV Cache
Decoder 推理时，历史 Token 的 K/V 不再重复计算，只缓存下来。
常见估算：
\[ M_{KV}=2\times L\times H_{kv}\times D\times B\times S\times bytes \]
例如：
- L=32, Hkv=32, D=128, B=1, S=8192, FP16=2 bytes
\[ M=2\times32\times32\times128\times8192\times1\times2\approx4GiB \]
约 0.5 MiB/token。

GQA/MQA：减少 KV Head，直接降低 KV Cache。
例如 Q Head=32：
- MHA：32 KV heads
- GQA：8 KV heads → Cache 约为 MHA 的 1/4
- MQA：1 KV head → 约 1/32

### 3.4 Prefill vs Decode
- Prefill：Prompt → 一次性处理大量 Token → KV Cache (特点：计算密集，适合 GPU 并行)
- Decode：KV Cache + 上一个 Token → 下一个 Token (每步通常只计算一个新 Token，受显存带宽/数据搬运影响明显)
因此 LLM 推理常存在：
- Prefill：Compute Bound
- Decode：Memory/Bandwidth Bound

### 3.5 Memory Wall
GPU 计算能力增长很快，但 HBM/显存带宽与数据搬运成为瓶颈。
例如模型 Decode 时，即使一次只计算少量新 Token，也需要读取大量历史 KV；因此：
推理优化的核心之一不是“让 FLOPs 更少”，而是“让数据搬得更少、更快”。

### 3.6 FlashAttention
普通 Attention 可能形成：
\[ S=QK^T,\quad P=softmax(S),\quad O=PV \]
问题不是数学复杂度，而是中间 N×N 矩阵需要频繁读写 HBM。
FlashAttention 使用 Tiling + Online Softmax，把计算分块放入 GPU 片上高速存储，减少 HBM IO。
关键点：
- 数学复杂度仍是 \(O(N^2d)\)
- 主要优化 IO / Memory Access
- 不等于把 Attention 从 O(N²) 变成 O(N)

### 3.7 PagedAttention
KV Cache 按固定大小 Block/Page 管理：
Sequence → Logical Blocks → Block Table → Physical GPU KV Blocks
类似虚拟内存思想：
- 降低连续大块显存分配要求
- 减少碎片
- 支持 Prefix Sharing / Block Sharing
- 更容易动态管理不同长度请求
它不会改变 KV Cache 的数学大小，主要优化显存管理与利用率。

### 3.8 Continuous Batching
传统 Static Batching：短请求结束后 GPU 可能出现空洞。
Continuous Batching：以 Token/Iteration 为粒度动态加入/移除请求，提高 GPU 利用率和吞吐。

### 3.9 Chunked Prefill
长 Prompt 的 Prefill 如果一次执行太久，会阻塞 Decode 请求。
因此可以把 Long Prompt 切分为 Chunk1, Chunk2...，在 Chunk 之间穿插 Decode，提高在线服务的公平性与 TTFT/TPOT 平衡。

### 3.10 Speculative Decoding
使用小模型 Draft，大模型 Verify：
Small Model Draft k Tokens → Large Model 一次验证 → 接受部分 + 修正第一个不接受 Token。
核心收益：大模型一次 Forward 可以验证多个候选 Token。
重要：正确设计的 Speculative Decoding 应保持目标模型的采样分布，而不是简单“永远接受 Draft”。

### 3.11 Quantization
目标：用低比特表示权重/激活，降低显存与带宽。
一种常见对称量化：
\[ q=round(x/s),\quad x\approx s q \]
例如 x=0.73，s=0.1：
\[ q=round(7.3)=7,\quad\hat x=0.7 \]
误差：\(|0.73-0.7|=0.03\)
常见：FP16/BF16, INT8, INT4
PTQ：训练后量化。
QAT：训练时模拟量化。
GPTQ / AWQ 等进一步提升量化性能。
量化难点：Outlier、层间敏感度、激活分布、任务精度损失。

## IV. 训练工程

### 4.1 数据
核心流程：Raw Data → Cleaning → Quality Filter → Dedup → Data Mix → Tokenization → Training
Dedup 很重要：重复数据会浪费训练预算，并可能造成过拟合/评测污染。

### 4.2 Learning Rate
典型：Warmup → Peak LR → Decay
常见 Cosine Decay：
\[ lr(t)=lr_{min}+\frac12(lr_{max}-lr_{min})(1+\cos(\pi t/T)) \]

### 4.3 Loss
Causal LM Cross Entropy：
\[ L=-\frac1N\sum_{i=1}^{N}\log p(y_i|x,y_{<i}) \]
若真实 Token 概率为 0.8：
\[ L=-\log(0.8)\approx0.223 \]
概率越低，loss 越大。

### 4.4 Precision / Throughput
常见精度：FP32, FP16/BF16, FP8
训练吞吐常关注：Tokens/s, Samples/s, MFU/HFU, GPU 利用率, Communication Overhead

### 4.5 分布式训练：DP / TP / PP / ZeRO
- DP（Data Parallel）：每张 GPU 一份模型，输入数据切分。
- TP（Tensor Parallel）：把矩阵/Layer 切到多 GPU。
- PP（Pipeline Parallel）：按 Layer 切分到不同 GPU。
- ZeRO：切分训练状态 (ZeRO-1: Optimizer States; ZeRO-2: + Gradients; ZeRO-3: + Parameters)
核心目标：降低单卡显存。

### 4.6 Activation Checkpointing
训练时不保存所有中间 Activation，而是只保存部分 Checkpoint，反向传播时重新计算。
本质是 Compute ↔ Memory Trade-off。

### 4.7 Fine-Tuning
常见：Full Fine-Tuning, LoRA / Adapter, QLoRA, Prompt Tuning。

## V. 上下文与推理能力

### 5.1 Context Engineering
目标不是“塞更多信息”，而是：找到最小的高信噪比 Token 集合，最大化期望结果概率。
核心是：\(Context Quality \gg Context Length\)

### 5.2 Lost in the Middle
长上下文中，模型对位于开头和结尾的信息往往更容易利用，中间信息可能利用率下降。
工程应对：关键信息放在显眼位置，长文档分块检索，排序，摘要压缩等。

### 5.3 CoT
Chain-of-Thought 通过增加中间推理 Token，让模型拥有更多“外部化工作空间”。
更多 Token意味着更多计算步骤，但并不保证一定更准确；任务简单、上下文不足或推理过长时可能反而产生错误。

## VI. 幻觉与评测

### 6.1 幻觉为什么发生？
LLM 的基本目标是优化 \(P(x_t|x_{<t})\)（下一个 Token 的概率），而不是“事实真实性”。
主要来源：知识过期、RAG检索失败、Attention失焦、长上下文Lost in the middle、自回归Exposure Bias。

### 6.2 RAG / Claim Verification
更可靠的验证方式：Answer → Atomic Claims → Retrieve Evidence → Claim ↔ Evidence → Verify
拆成原子 Claim 分别寻找证据验证。

### 6.3 多次采样一致性
一致 ≠ 正确。需要结合外部证据验证。

### 6.4 LLM-as-a-Judge
让一个 LLM 对另一个 LLM 的答案评分/比较。
优势：扩展性高、成本低、可统一 Rubric。
风险：Position Bias, Length Bias, Style Bias, Self-preference。

## VII. 多模态与图像生成

### 7.1 Visual Token
图像可以转换成视觉 Token。
例如图片 1024×1024，Patch 16×16，Patch 数：
\[ (1024/16)^2=64^2=4096 \]
即约 4096 个视觉 Token。

### 7.2 Autoregressive vs Diffusion
- Autoregressive：串行 Token 生成。
- Diffusion：通过多步去噪生成连续图像。

### 7.3 DDPM 数学直觉
Forward 加噪：
\[ x_t=\sqrt{\bar\alpha_t}x_0+\sqrt{1-\bar\alpha_t}\epsilon \]
训练模型学习如何从噪声状态预测参数，再执行反向去噪。

### 7.4 为什么图像生成难？
全局一致性、局部细节、文本-视觉对齐、计算量。

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
 LM Head                          Speculative
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

## 九、面试高频问法

1. **为什么 Decode 比 Prefill 更容易受显存带宽影响？**
Decode 每步只生成少量新 Token，却需要读取历史 KV Cache；计算量相对小，数据搬运占比高，因此更容易 Memory Bound。

2. **FlashAttention 为什么快？**
不是把 Attention 从 \(O(N^2)\) 变成 \(O(N)\)，而是通过 Tiling + Online Softmax 减少中间矩阵的 HBM 读写。

3. **PagedAttention 解决什么？**
解决 KV Cache 动态分配、碎片和共享问题，提高显存利用率；不改变 KV Cache 的理论大小。

4. **GQA 为什么省显存？**
Q Head 可以很多，但 K/V Head 更少；KV Cache 按 Hkv 线性增长，因此直接减少 Cache。

5. **Continuous Batching 为什么提高吞吐？**
请求可以在 Token 生成过程中动态加入/退出，减少静态 Batch 中的空洞，提高 GPU 利用率。

6. **LoRA 为什么能减少训练成本？**
只训练低秩增量矩阵 BA，冻结大部分 Base 参数，因此参数、梯度和优化器状态都显著减少。

7. **Speculative Decoding 为什么有效？**
小模型负责便宜地提出多个候选 Token，大模型批量验证；若多数候选被接受，就减少大模型逐 Token Decode 次数。

8. **为什么长上下文不等于模型真的“记住”了所有内容？**
Context Window 是容量，不代表有效利用率；长上下文还受训练分布、Attention、位置编码和 Lost in the Middle 等因素影响。

9. **为什么 LLM 会幻觉？**
因为 Next Token Prediction 优化的是概率/语言流畅性，不是事实验证；需要 RAG、工具、检索与 Claim Verification 等外部机制增强可靠性。

10. **LLM 系统优化应该从哪里入手？**
- 质量问题 → 数据 / Prompt / RAG / 模型
- 延迟问题 → Prefill / Decode / KV / Batch
- 显存问题 → Quantization / GQA / PagedAttention
- 吞吐问题 → Continuous Batching / Parallelism
- 稳定性问题 → Scheduling / Timeout / Retry / Isolation
