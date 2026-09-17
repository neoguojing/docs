可以。下面直接给你**精简、紧凑版正文**，保留前面文档的核心内容，并把之前遗漏的 Transformer Block、QKV、LM Head、KV Cache、FlashAttention、Continuous Batching、Chunked Prefill、ZeRO、Context Engineering、Lost in the Middle、Claim Verification 等补齐。

# 大语言模型核心知识体系：原理、训练、推理与多模态

> 核心主线：**Tokenizer → Transformer → Pretraining → SFT/RL → Inference → KV Cache → 推理优化 → Context → RAG/评测 → Multimodal**

---

# I. 基础架构层

## 1.1 Tokenizer

**作用**：将文本转换成模型可以处理的 Token ID。

```text
文本 → Tokenizer → Token IDs → Embedding → 向量
```

常见方式：

| 方法        | 单位 | 特点            |
| --------- | -- | ------------- |
| Character | 字符 | 词表小，但序列长      |
| Word      | 单词 | 词表大，存在 OOV    |
| Subword   | 子词 | 最常见，平衡词表与序列长度 |

### BPE

BPE 从字符开始，不断统计相邻 Token 的频率，将高频 pair 合并。

例如：

```text
unhappiness
↓
un + happi + ness
```

中文 Token 数取决于具体 Tokenizer，不能简单认为“1 个汉字 = 1 Token”。

**面试重点**：

Tokenizer 会直接影响：

* Context Length
* KV Cache
* 推理成本
* 训练成本
* 多语言效率

---

## 1.2 Embedding

**作用**：

```text
Token ID → Dense Vector
```

将离散 Token 映射到连续向量空间。

常用相似度：

$$
cos(a,b)=\frac{a\cdot b}{||a||||b||}
$$

例如：

```text
a = (1,0)
b = (0.8,0.6)
```

则：

$$
cos(a,b)=0.8
$$

说明两个向量方向较接近。

**核心理解**：

Embedding 的“语义”不是人为写进去的，而是在训练过程中逐渐形成的。

---

# II. Transformer 架构

## 2.1 Transformer Block

Decoder-only LLM 可以简化为：

```text
Token
 ↓
Embedding
 ↓
RoPE
 ↓
Q/K/V
 ↓
Attention
 ↓
Residual + Norm
 ↓
FFN / MLP
 ↓
Residual + Norm
 ↓
Hidden State
 ↓
LM Head
 ↓
Logits
 ↓
Softmax / Sampling
 ↓
Next Token
```

一个 Block 的核心：

```text
Attention：Token 与 Token 之间交互
FFN：对每个 Token 做非线性变换
Residual：帮助深层网络训练
Norm：稳定训练
RoPE：提供位置信息
```

---

## 2.2 Q / K / V

输入：

$$
X
$$

通过三个矩阵得到：

$$
Q=XW_Q
$$

$$
K=XW_K
$$

$$
V=XW_V
$$

可以简单理解：

* **Q**：我要找什么？
* **K**：我有什么可以被匹配？
* **V**：匹配成功后提供什么信息？

例如：

```text
Query：“谁是苹果公司的 CEO？”

Key：每个历史 Token 的语义特征
Value：对应 Token 携带的信息
```

---

## 2.3 Attention

核心公式：

$$
Attention(Q,K,V)
=
softmax(\frac{QK^T}{\sqrt{d_k}})V
$$

分成三步：

```text
QKᵀ
 ↓
相关性 Score
 ↓
Softmax
 ↓
Attention Weight
 ↓
× V
 ↓
加权信息
```

### 数值例子

假设有 4 个 Token：

```text
Q：4×64
K：4×64
V：4×64
```

那么：

$$
QK^T
$$

得到：

```text
4 × 4
```

即每个 Token 都需要与其他 Token 计算相关性。

因此标准 Attention：

$$
O(N^2d)
$$

其中：

* N：序列长度
* d：Head Dimension

Attention Matrix 本身需要：

$$
O(N^2)
$$

内存。

---

## 2.4 LM Head

Transformer 最终得到：

$$
h\in R^d
$$

通过 LM Head 投影到词表：

$$
logits=hW_{vocab}^T
$$

例如：

```text
hidden size = 4096
vocab size = 100K
```

则一个 Token 最终产生：

```text
100K 个 logits
```

然后：

```text
Logits
 ↓
Temperature
 ↓
Top-K / Top-P
 ↓
Sampling
 ↓
Next Token
```

---

# III. RoPE 与 Attention 演进

## 3.1 RoPE

RoPE 不直接把位置 embedding 加到 Token 上，而是根据位置旋转 Q/K。

二维旋转：

$$
R_\theta=
\begin{bmatrix}
cos\theta&-sin\theta\\
sin\theta&cos\theta
\end{bmatrix}
$$

位置 m：

$$
q_m=R_mq
$$

位置 n：

$$
k_n=R_nk
$$

则：

$$
q_m^Tk_n
=
q^TR_m^TR_nk
=
q^TR_{n-m}k
$$

因此 Attention 自然获得了**相对位置关系**。

### 长上下文

常见方法：

* Position Interpolation
* YaRN
* RoPE Scaling

核心问题是：

> 训练长度和推理长度存在分布差异。

不是简单地“把位置编号增加到 128K”即可。

---

## 3.2 Linear Attention

标准 Attention：

$$
softmax(QK^T)V
$$

核心问题是：

$$
O(N^2)
$$

Linear Attention 尝试通过 Kernel/Associativity 改变计算顺序。

例如：

$$
\sum_j\phi(q_i)^T\phi(k_j)v_j
$$

先计算：

$$
S=\sum_j\phi(k_j)v_j^T
$$

再计算 Query。

固定 d 时可以达到类似：

$$
O(Nd^2)
$$

因此序列维度从二次变成线性。

但：

> Linear Attention 不意味着任何情况下都比 FlashAttention 更快。

---

## 3.3 GLA / DeltaNet / Hybrid

### GLA

通过 Gate 控制历史状态：

```text
旧状态
 ↓
Gate
 ↓
保留 / 衰减
 ↓
新状态
```

### DeltaNet

通过增量更新状态，对已有记忆进行修正。

### Hybrid

结合：

```text
Attention
+
Linear / State Space
```

目标：

> 用 Attention 解决精确检索，用线性状态结构承担长程记忆。

---

# IV. BERT / T5 / GPT

| 模型   | 架构              | 特点      |
| ---- | --------------- | ------- |
| BERT | Encoder         | 双向理解    |
| T5   | Encoder-Decoder | Seq2Seq |
| GPT  | Decoder         | 自回归生成   |

GPT 的目标：

$$
P(x)=\prod_tP(x_t|x_{<t})
$$

即：

> 根据前面的 Token，预测下一个 Token。

Decoder 使用 Causal Mask：

```text
Token1 → 看不到未来
Token2 → 只能看 Token1
Token3 → 可以看 Token1、2
```

---

# V. Training

## 5.1 Pretraining

核心目标：

$$
L=-\sum_t logP(x_t|x_{<t})
$$

即 Next Token Prediction。

训练数据：

```text
Raw Data
 ↓
Cleaning
 ↓
Quality Filter
 ↓
Dedup
 ↓
Data Mix
 ↓
Tokenizer
 ↓
Training
```

---

## 5.2 Chinchilla

经典粗略计算：

$$
C\approx6ND
$$

其中：

* N：参数量
* D：训练 Token 数

例如：

```text
N = 70B
D = 1.4T
```

则：

$$
C
≈6×70×10^9×1.4×10^{12}
$$

$$
≈5.88×10^{23}FLOPs
$$

“约 20 Token/参数”是 Chinchilla 时代的重要经验关系，但不是现代所有模型的固定最优值。

---

## 5.3 Scaling Law

核心思想：

> 增大模型、数据和计算规模，Loss 通常呈现较稳定的规模化变化。

但实际效果还受到：

* 数据质量
* 数据分布
* 模型架构
* 训练目标
* 优化器
* 推理成本

影响。

---

## 5.4 ICL

In-Context Learning：

```text
Prompt
 + Example
 + Example
 ↓
模型临时适应任务
```

不修改模型参数。

例如：

```text
A → 1
B → 2
C → ?
```

模型根据上下文推断 `C` 的模式。

---

# VI. SFT / RL

## 6.1 SFT

Supervised Fine-Tuning：

```text
Instruction
+
Answer
↓
继续训练
```

例如：

```text
用户：解释 Raft
助手：Raft 是一种……
```

目标：

$$
L=-\sum_t logP(y_t|x,y_{<t})
$$

通常只对 Assistant 输出部分计算 loss。

---

## 6.2 RLHF

典型流程：

```text
Pretraining
 ↓
SFT
 ↓
Preference / Reward
 ↓
RL
 ↓
Aligned Model
```

目标：

> 让模型不仅“会回答”，还更符合人类偏好/任务要求。

---

## 6.3 GRPO

GRPO 可以针对同一个问题采样多个回答：

```text
Question
 ├─ Answer A → reward 8
 ├─ Answer B → reward 6
 ├─ Answer C → reward 4
 └─ Answer D → reward 2
```

常见优势形式：

$$
A_i=
\frac{r_i-\mu}{\sigma+\epsilon}
$$

例如：

```text
reward = [8,6,4,2]
mean = 5
std ≈ 2.24
```

第一条：

$$
A_1≈1.34
$$

最后：

$$
A_4≈-1.34
$$

核心：

> 通过同题多个回答之间的相对质量产生训练信号。

---

## 6.4 DAPO

核心思想是改善大规模 RL 训练稳定性与效率。

主要概念：

* Clip-Higher
* Dynamic Sampling
* Token-Level Policy Gradient
* Overlong Reward Shaping

面试时重点理解：

> DAPO 不是一个简单的“新 Reward Model”，而是一组针对 RL 训练效率、样本利用和稳定性的改进。

---

## 6.5 Process Reward / Outcome Reward

### Outcome Reward

只判断最终结果：

```text
最终答案正确 → +1
最终答案错误 → 0
```

### Process Reward

评价中间步骤：

```text
Step1 ✓
Step2 ✓
Step3 ✗
```

优点：

> 能解决更细粒度的 Credit Assignment。

缺点：

> 标注和验证成本更高。

---

# VII. LoRA

## 7.1 原理

冻结：

$$
W
$$

只训练：

$$
\Delta W=BA
$$

因此：

$$
W'=W+BA
$$

例如：

```text
W = 4096 × 4096
rank = 8
```

原参数：

$$
4096^2=16,777,216
$$

LoRA：

$$
4096×8+8×4096=65,536
$$

占比：

$$
65536/16777216≈0.39\%
$$

所以训练成本大幅下降。

### 为什么低秩有效？

经验上很多下游任务需要的参数更新存在低维结构。

但：

> “低秩有效”是经验性结构假设，不是数学定理。

---

# VIII. 推理

## 8.1 Logits → Sampling

模型输出：

```text
z = [2,1]
```

Softmax：

$$
p_i=
\frac{e^{z_i/T}}
{\sum_j e^{z_j/T}}
$$

结果：

| Temperature | 概率               |
| ----------- | ---------------- |
| 0.5         | `[0.881, 0.119]` |
| 1           | `[0.731, 0.269]` |
| 2           | `[0.622, 0.378]` |

结论：

```text
T ↓ → 更确定
T ↑ → 更随机
```

`T=0` 通常由实现特殊处理为 Greedy。

### Top-K

只保留概率最高的 K 个 Token。

### Top-P

按照概率从高到低累加，保留累计概率达到 P 的最小集合。

---

# IX. KV Cache

## 9.1 为什么需要 KV Cache？

Decode：

```text
生成 Token1
 ↓
生成 Token2
 ↓
生成 Token3
```

如果每次重新计算整个历史：

```text
Token1 + Token2 + Token3 ...
```

会大量重复计算。

因此缓存历史：

```text
K1 V1
K2 V2
K3 V3
...
```

下一步只计算新 Token 的 Q/K/V，然后读取历史 KV。

---

## 9.2 KV Cache 大小

常见公式：

$$
M_{KV}
=
2×L×H_{kv}×D×B×S×bytes
$$

其中：

* 2：K + V
* L：Layer
* Hkv：KV Head
* D：Head Dimension
* B：Batch
* S：Sequence Length

例如：

```text
L = 32
Hkv = 32
D = 128
B = 1
S = 8192
FP16 = 2 bytes
```

则：

$$
M≈4GiB
$$

约：

$$
0.5MiB/token
$$

---

# X. MHA / GQA / MQA

假设：

```text
Q Head = 32
```

### MHA

```text
Q = 32
K = 32
V = 32
```

### GQA

```text
Q = 32
K = 8
V = 8
```

KV Cache 约为 MHA：

$$
8/32=1/4
$$

### MQA

```text
Q = 32
K = 1
V = 1
```

KV Cache 约为：

$$
1/32
$$

核心：

> GQA/MQA 通过减少 KV Head 降低 KV Cache 和 Decode 数据读取量。

---

# XI. Prefill / Decode

## 11.1 Prefill

```text
Prompt
 ↓
一次处理大量 Token
 ↓
KV Cache
```

特点：

> 并行度高，通常偏 Compute Bound。

---

## 11.2 Decode

```text
KV Cache
 +
上一个 Token
 ↓
Next Token
```

特点：

> 每次计算量小，但需要读取大量 KV，因此更容易 Memory/Bandwidth Bound。

所以：

```text
Prefill → Compute Bound
Decode  → Memory Bound
```

---

# XII. 推理优化

## 12.1 Memory Wall

GPU：

```text
Compute
↑↑↑ 很快
```

但：

```text
Memory Bandwidth
↑
```

增长相对慢。

Decode 中：

> 计算一个 Token 的 FLOPs 可能不大，但需要搬运大量模型权重和 KV Cache。

所以现代 LLM 推理优化的核心之一：

> **减少数据搬运，而不仅仅是减少 FLOPs。**

---

## 12.2 FlashAttention

普通 Attention：

$$
S=QK^T
$$

$$
P=softmax(S)
$$

$$
O=PV
$$

问题：

```text
QKᵀ
 ↓
N×N Matrix
 ↓
写 HBM
 ↓
读取
 ↓
Softmax
 ↓
再次读写
```

FlashAttention 使用：

* Tiling
* Online Softmax
* SRAM / Shared Memory

减少 HBM IO。

**关键结论**：

$$
O(N^2)
$$

没有变。

变化的是：

> **Memory IO 大幅下降。**

---

## 12.3 PagedAttention

KV Cache 不再要求连续大块显存，而是：

```text
Logical Sequence
 ↓
Logical Blocks
 ↓
Block Table
 ↓
Physical GPU Blocks
```

类似虚拟内存。

解决：

* 显存碎片
* 动态分配
* 不同长度请求
* Prefix Sharing

注意：

> PagedAttention 不会减少 KV Cache 的理论数据量。

---

## 12.4 Continuous Batching

传统 Batch：

```text
A ────────────────
B ────────────────
C ────────────────
```

A 很短时，A 完成后 Batch 可能存在空洞。

Continuous Batching：

```text
Step1：A B C
Step2：A B C
Step3：A C D
Step4：A D E
```

请求可以动态：

```text
加入
退出
```

因此：

> 提高 GPU 利用率和吞吐。

---

## 12.5 Chunked Prefill

长 Prompt：

```text
10K Tokens
```

如果一次 Prefill：

```text
Prefill 10K
──────────────→
Decode 请求被阻塞
```

可以拆：

```text
Chunk1
 ↓
Decode
 ↓
Chunk2
 ↓
Decode
 ↓
Chunk3
```

解决：

> 长 Prefill 阻塞在线 Decode 的问题。

---

## 12.6 Speculative Decoding

```text
Small Model
 ↓
Draft 4~8 Tokens
 ↓
Large Model
 ↓
一次验证
 ↓
接受 / 修正
```

例如：

```text
Small：A B C D
Large：A B C X
```

则：

```text
A B C 接受
D 被拒绝
X 修正
```

优势：

> 减少大模型逐 Token Decode 次数。

正确实现需要保持目标模型的采样分布。

---

# XIII. Quantization

## 13.1 基本原理

降低：

```text
FP16
 ↓
INT8
 ↓
INT4
```

减少：

* Weight Memory
* KV/Activation（视方案而定）
* Memory Bandwidth

简单对称量化：

$$
q=round(x/s)
$$

反量化：

$$
\hat{x}=sq
$$

例如：

```text
x = 0.73
s = 0.1
```

则：

$$
q=round(7.3)=7
$$

$$
\hat{x}=0.7
$$

误差：

$$
0.03
$$

---

## 13.2 PTQ / QAT

**PTQ**

```text
训练完成
 ↓
Quantization
```

**QAT**

```text
训练过程中模拟 Quantization
 ↓
模型学习适应量化误差
```

---

## 13.3 GPTQ / AWQ

### GPTQ

利用近似二阶信息，在后训练阶段优化权重量化误差。

### AWQ

Activation-aware Weight Quantization。

核心：

> 根据激活统计识别对量化更敏感的权重结构。

---

# XIV. Training Engineering

## 14.1 Data Cleaning

```text
Raw
 ↓
Cleaning
 ↓
Quality Filter
 ↓
Dedup
 ↓
Data Mix
```

Dedup 很重要：

* 避免训练资源浪费
* 降低重复数据过拟合
* 减少评测污染

---

## 14.2 Learning Rate

典型：

```text
Warmup
 ↓
Peak LR
 ↓
Decay
```

Cosine：

$$
lr(t)
=
lr_{min}
+
\frac12(lr_{max}-lr_{min})
(1+\cos(\pi t/T))
$$

Warmup：

> 避免训练初期参数更新过大。

---

## 14.3 Loss

Causal LM：

$$
L=-\frac1N\sum_i logP(y_i|x,y_{<i})
$$

如果真实 Token 概率：

```text
P = 0.8
```

则：

$$
L=-log(0.8)\approx0.223
$$

概率越低，Loss 越高。

---

# XV. 分布式训练

## 15.1 DP

Data Parallel：

```text
GPU0 → Model + Data A
GPU1 → Model + Data B
GPU2 → Model + Data C
```

每张 GPU 有模型副本。

---

## 15.2 TP

Tensor Parallel：

```text
一个矩阵
 ↓
GPU0：一部分
GPU1：一部分
GPU2：一部分
```

解决：

> 单卡放不下/算不动大型矩阵。

---

## 15.3 PP

Pipeline Parallel：

```text
GPU0：Layer 1~8
GPU1：Layer 9~16
GPU2：Layer 17~24
```

模型按 Layer 切分。

---

## 15.4 ZeRO

ZeRO 对训练状态做切分：

```text
ZeRO-1 → Optimizer States
ZeRO-2 → + Gradients
ZeRO-3 → + Parameters
```

目标：

> 降低单 GPU 显存。

---

## 15.5 Activation Checkpointing

普通训练：

```text
保存大量 Activation
 ↓
显存高
```

Checkpoint：

```text
只保存部分 Activation
 ↓
反向时重新计算
 ↓
显存 ↓
计算 ↑
```

本质：

> Compute ↔ Memory Trade-off。

---

# XVI. Context Engineering

## 16.1 核心目标

不是：

> 给 Agent 更多 Context。

而是：

> **找到最小的高信噪比 Token 集合，最大化任务成功概率。**

---

## 16.2 主要方法

### 压缩

```text
历史 Context
 ↓
Summary
 ↓
重新初始化
```

### 外化记忆

```text
Context
 ↓
重要信息
 ↓
File / DB
```

### Retrieval

```text
全部知识
 ↓
Retriever
 ↓
相关知识
 ↓
Prompt
```

### 结构化 Context

例如：

```text
Task
Constraints
Memory
Tool Results
Current State
```

分区组织，而不是全部拼接。

---

# XVII. Lost in the Middle

长 Context 中：

```text
开始       中间       结尾
 ↑          ↓          ↑
较容易     可能下降    较容易
```

因此：

> Context Window 很大，不代表模型能同等有效地利用所有位置的信息。

工程方法：

* 关键内容放在显眼位置
* 长文档切块
* Retrieval
* Ranking
* Summary
* 删除无关信息

---

# XVIII. CoT

Chain-of-Thought：

```text
问题
 ↓
Step1
 ↓
Step2
 ↓
Step3
 ↓
Answer
```

相比：

```text
问题 → Answer
```

CoT 增加了中间计算步骤，相当于提供更多“外部化工作空间”。

但：

> 更多 Token ≠ 一定更准确。

简单问题、错误推理或过长推理可能增加错误。

---

# XIX. Hallucination

## 19.1 为什么会幻觉？

LLM 优化：

$$
P(x_t|x_{<t})
$$

目标是：

> 预测下一个 Token。

而不是：

> 验证这个事实是否真实。

主要原因：

1. 参数知识是压缩后的统计知识。
2. 知识存在边界和时效性。
3. RAG 可能检索错误。
4. Context 可能缺失。
5. Attention 可能没有正确利用信息。
6. Long Context 可能出现 Lost in the Middle。
7. 自回归生成存在 Exposure Bias。
8. Alignment 不等于事实验证。

---

# XX. RAG + Claim Verification

可靠系统可以：

```text
Answer
 ↓
Atomic Claims
 ↓
Retrieve Evidence
 ↓
Claim ↔ Evidence
 ↓
Entail / Contradict / Unknown
 ↓
Final Answer
```

例如：

> “A 公司 2025 年营收 100 亿。”

拆成：

```text
主体：A 公司
时间：2025
指标：营收
数值：100 亿
```

分别寻找证据。

核心思想：

> 不要只验证“整段回答”，而要验证“原子事实”。

---

# XXI. 多次采样一致性

同一个问题生成多次：

```text
A A A A
```

说明模型比较稳定。

而：

```text
A B C D
```

说明不确定性更高。

但：

> **一致 ≠ 正确。**

因此应结合：

* Search
* RAG
* Database
* Rules
* Cross Model Verification

---

# XXII. LLM-as-a-Judge

让一个 LLM 评价另一个 LLM：

```text
Answer
 ↓
Judge
 ↓
Rubric
 ↓
Score / Comparison
```

优势：

* 成本低
* 可扩展
* 自动化程度高

问题：

* Position Bias
* Length Bias
* Style Bias
* Self Preference
* Knowledge Boundary

改进：

```text
明确 Rubric
+
A/B 顺序交换
+
Multi-Judge
+
客观指标
+
人工抽检
```

---

# XXIII. Multimodal

## 23.1 Visual Token

例如：

```text
Image = 1024×1024
Patch = 16×16
```

Patch 数：

$$
(1024/16)^2=4096
$$

因此理论上约：

```text
4096 Visual Tokens
```

实际模型通常还会经过：

```text
Vision Encoder
 ↓
Downsampling / Compression
 ↓
Visual Tokens
 ↓
LLM
```

所以真实 Token 数取决于架构。

---

# XXIV. Autoregressive vs Diffusion

## Autoregressive

```text
Token1
 ↓
Token2
 ↓
Token3
 ↓
...
```

优点：

> 统一序列建模。

缺点：

> 生成具有串行依赖。

---

## Diffusion

```text
Noise
 ↓
Denoise
 ↓
Denoise
 ↓
...
 ↓
Image
```

通过多步去噪逐渐生成图像。

---

# XXV. Diffusion 数学

Forward：

$$
x_t=
\sqrt{\bar{\alpha}_t}x_0+
\sqrt{1-\bar{\alpha}_t}\epsilon
$$

例如：

```text
x0 = 0.8
alpha_bar = 0.5
epsilon = 0.1
```

则：

$$
x_t
=
\sqrt{0.5}×0.8+
\sqrt{0.5}×0.1
$$

$$
≈0.636
$$

训练模型学习从噪声状态恢复相关信息，再进行 Reverse Denoising。

---

# XXVI. 图像生成难点

核心：

### 1. Global Consistency

```text
人物
身体
手
物体
空间关系
```

需要整体一致。

### 2. Local Detail

```text
纹理
边缘
文字
细节
```

### 3. Text-Visual Alignment

Prompt：

```text
一个人
+
一辆车
+
车在人的左边
```

模型需要同时满足多个约束。

### 4. 计算成本

分辨率越高：

```text
Visual Tokens / Latent
↑
计算量 ↑
显存 ↑
```

---

# XXVII. LLM 系统整体知识地图

```text
                         LLM
                          │
        ┌─────────────────┼─────────────────┐
        ↓                 ↓                 ↓
    基础架构            Training          Inference
        │                 │                 │
 Tokenizer            Pretrain            Logits
 Embedding             SFT                Sampling
 Transformer           RLHF               KV Cache
 Q/K/V                 GRPO               Prefill
 Attention             DAPO               Decode
 RoPE                  LoRA               GQA/MQA
 FFN                   DP/TP/PP           FlashAttention
 LM Head               ZeRO               PagedAttention
                                            Continuous Batch
                                            Chunked Prefill
                                            Speculative
                                            Quantization
                          │
                          ↓
                 Context Engineering
                          │
            Memory / Retrieval / CoT
            Lost in the Middle
                          │
                          ↓
                    Reliability
                          │
            RAG / Verification / Judge
                          │
                          ↓
                    Multimodal
               Vision Token / Diffusion
```

---

# XXVIII. 面试最重要的 15 个问题

### 1. Transformer 为什么需要 Attention？

让不同 Token 之间进行动态信息交互。

### 2. Q/K/V 分别是什么？

```text
Q：查询
K：匹配
V：信息
```

### 3. Attention 为什么是 O(N²)？

因为每个 Query 都需要与 N 个 Key 计算关系，共 N×N。

### 4. RoPE 解决什么问题？

给 Attention 注入位置信息，并自然表达相对位置关系。

### 5. 为什么 LLM 使用 Decoder-only？

Causal LM 与自回归生成天然匹配，架构简单、扩展性强。

### 6. 为什么 KV Cache 能加速 Decode？

缓存历史 K/V，避免每生成一个 Token 都重新计算历史 Token。

### 7. 为什么 Decode 容易 Memory Bound？

每步计算量小，但需要读取大量模型权重和 KV Cache。

### 8. FlashAttention 优化了什么？

主要优化 GPU Memory IO，而不是改变 Attention 的 O(N²) 数学复杂度。

### 9. PagedAttention 解决什么？

KV Cache 的动态显存管理、碎片和共享。

### 10. GQA 为什么减少显存？

减少 KV Head：

$$
KV\ Memory\propto H_{kv}
$$

### 11. Continuous Batching 为什么提高吞吐？

动态加入/退出请求，减少 Batch 空洞，提高 GPU 利用率。

### 12. LoRA 为什么便宜？

只训练低秩矩阵：

$$
\Delta W=BA
$$

### 13. ZeRO 解决什么？

切分：

```text
Optimizer
Gradient
Parameter
```

降低单卡训练显存。

### 14. Context Engineering 是什么？

不是无限增加 Context，而是：

> **选择最小的高质量信息集合。**

### 15. 为什么 LLM 会幻觉？

因为：

$$
Next\ Token\ Prediction
\neq
Fact\ Verification
$$

所以需要：

```text
RAG
+
Tools
+
Verification
+
Structured Context
```

---

# XXIX. 最终面试主线

如果面试官让你“整体讲一下 LLM”，建议按这个顺序：

```text
1. Tokenizer
   ↓
2. Embedding
   ↓
3. Transformer
   ├─ Q/K/V
   ├─ Attention
   ├─ RoPE
   ├─ FFN
   └─ LM Head
   ↓
4. Pretraining
   └─ Next Token Prediction
   ↓
5. SFT
   ↓
6. RLHF / GRPO / DAPO
   ↓
7. LoRA
   ↓
8. Inference
   ├─ Logits
   ├─ Sampling
   ├─ KV Cache
   ├─ Prefill / Decode
   └─ GQA
   ↓
9. Inference Optimization
   ├─ FlashAttention
   ├─ PagedAttention
   ├─ Continuous Batching
   ├─ Chunked Prefill
   ├─ Speculative Decoding
   └─ Quantization
   ↓
10. Context Engineering
    ├─ Memory
    ├─ Retrieval
    ├─ Compression
    └─ CoT
   ↓
11. Reliability
    ├─ Hallucination
    ├─ RAG
    ├─ Claim Verification
    └─ LLM Judge
   ↓
12. Multimodal
    ├─ Visual Token
    └─ Diffusion
```

## 一句话总结

> **LLM 本质上是基于 Transformer 的概率序列模型；训练阶段通过 Next Token Prediction、SFT 和 RL 获得能力，推理阶段通过 KV Cache、Attention 优化、Batching、量化等技术解决计算、显存和带宽问题，而真正构建可靠 Agent/LLM 系统，还需要 Context Engineering、Memory、RAG、Tools 和 Verification。**
