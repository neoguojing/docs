# Transformer 总纲 (GitHub 兼容优化版)

## 0. 全局参数与符号定义 (Global Definitions)

为了保证全局推演的一致性，本文所有公式与数值演算均基于以下同一套超参数：

**核心超参数：**

* $L=3$ (Sequence length，序列长度)
* $d=4$ ( $d_{model}$ ，隐藏层维度)
* $H=2$ (num_heads，注意力头数)
* $d_h=2$ ( $d_{head}=d/H$ ，每个头的维度)
* $d_{ff}=8$ (MLP 隐藏层维度，通常为 $d$ 的倍数)
* $V=6$ (Vocab size，词表大小)
* $N=2$ (层数)

**参数与网络位置严格对应表：**

| 网络模块 | 数学公式符号 | 代码层命名 | 张量维度 | 具体本例维度 |
| --- | --- | --- | --- | --- |
| **Embedding** | $E$ | `embed_tokens` | `[V, d]` | `[6, 4]` |
| **Attention** | $W_Q$ | `q_proj` | `[d, H·d_h]` | `[4, 4]` |
| **Attention** | $W_K$ | `k_proj` | `[d, H·d_h]` | `[4, 4]` |
| **Attention** | $W_V$ | `v_proj` | `[d, H·d_h]` | `[4, 4]` |
| **Attention** | $W_O$ | `o_proj` | `[H·d_h, d]` | `[4, 4]` |
| **MLP** | $W_{gate}$ | `gate_proj` | `[d, d_ff]` | `[4, 8]` |
| **MLP** | $W_{up}$ | `up_proj` | `[d, d_ff]` | `[4, 8]` |
| **MLP** | $W_{down}$ | `down_proj` | `[d_ff, d]` | `[8, 4]` |
| **Norm** | $\gamma, \beta$ | `input_layernorm` | `[d]` | `[4]` |
| **Output** | $W_{lm}$ | `lm_head` | `[d, V]` | `[4, 6]` |

*(注：标准 Multi-Head Attention 中 $H \cdot d_h = d$ ，因此投影矩阵通常为 $R^{d \times d}$ 。)*

---

## 1. 整体网络

以 Decoder-only LLM 为例：

```text
               Token IDs
                   │
                   ▼
               Embedding
                E[V,d]
                   │
                   ▼
                X[L,d]
                   │
                   ▼
┌─────────────────────────────────────────┐
│ Transformer Block × N                   │
│                                         │
│ X                                       │
│ │                                       │
│ ├─ Norm ─→ Attention ─→ + X             │
│ │             │                         │
│ │        ┌────┼────┐                    │
│ │        ▼    ▼    ▼                    │
│ │      Head1 Head2 ... HeadH            │
│ │        └────┼────┘                    │
│ │             ▼                         │
│ │          Concat                       │
│ │             ↓                         │
│ │          o_proj                       │
│ │             │                         │
│ └─────────────┼─────────────────────────┤
│               ▼                         │
│              Norm                       │
│               │                         │
│        ┌──────▼──────┐                  │
│        │     MLP     │                  │
│        │ gate/up     │                  │
│        │    ↓        │                  │
│        │   Act ×     │                  │
│        │    ↓        │                  │
│        │   down      │                  │
│        └──────┬──────┘                  │
│               │                         │
│              +X                         │
└───────────────┬─────────────────────────┘
                │
             Block 1
                ↓
             Block 2
                ↓
               ...
                ↓
             Block N
                │
                ▼
           Hidden State
                │
                ▼
             lm_head
                │
                ▼
              Logits
                │
                ▼
         Softmax / Decode

```

**串行与并行关系：**

* **Block 之间（串行）：** `Block 1 → Block 2 → ... → Block N`
* **Block 内部（串行）：** `Norm → Attention → Norm → MLP`
* **Attention 内部（并行与串行）：** `Head 1~H` (并行) `→ Concat → o_proj` (串行)

---

## 2. Embedding

**输入 Token IDs ( $L=3$ )：**

$$t = [2, 5, 1]$$

**Embedding 查表参数 ( $E \in R^{6 \times 4}$ )：**
根据词表索引查表后，得到输入张量 $X$ ：

$$X = \begin{bmatrix} 1 & 0 & 1 & 0 \\ 0 & 1 & 0 & 1 \\ 1 & 1 & 0 & 0 \end{bmatrix}$$

此时维度：

$$X \in R^{L \times d} = R^{3 \times 4}$$

**数学表达：**

$$X_i = E[t_i]$$

若注入位置信息（如可加式）：

$$X_i = E[t_i] + P_i$$

*(现代 LLM 常使用 RoPE 进行旋转位置编码)*

---

## 3. Attention：Q/K/V 投影

**输入：**

$$X \in R^{3 \times 4}$$

**参数 (对应 `q_proj`, `k_proj`, `v_proj`)：**
对于第 1 个 Head，权重矩阵为 $W_Q, W_K, W_V \in R^{d \times d_h} = R^{4 \times 2}$ 。

**计算：**

$$Q = XW_Q \quad K = XW_K \quad V = XW_V$$

**维度变化：**

* $X$ : `[3, 4]`
* $W_Q$ : `[4, 2]`
* 得到 **$Q, K, V$** : 均被投影为 `[3, 2]` ( $L \times d_h$ )。

---

## 4. Attention 核心公式

 $$ \boxed{Z = \operatorname{softmax}\left(\frac{QK^T}{\sqrt{d_h}} + M\right)V} $$ 

**公式分解与维度：**

1. 相似度计算： $S = \frac{QK^T}{\sqrt{d_h}}$ （维度： `[L, d_h] × [d_h, L] = [L, L]` ）
2. 掩码操作： $S' = S + M$ （加入 Causal Mask，禁止访问未来信息）
3. 概率分布： $A = \operatorname{softmax}(S')$ （维度： `[L, L]` ）
4. 信息聚合： $Z = AV$ （维度： `[L, L] × [L, d_h] = [L, d_h]` ）

---

## 5. Attention 数值计算 (含 Causal Mask)

我们以第 1 个 Head 的实际矩阵进行 $3 \times 3$ 严谨推演。
假设投影后的 $Q, K, V \in R^{3 \times 2}$ 为：

$$Q = \begin{bmatrix} 1.414 & 0 \\ 0 & 1.414 \\ 1.414 & 1.414 \end{bmatrix}, \quad K = \begin{bmatrix} 2 & 0 \\ 0 & 2 \\ 0 & 0 \end{bmatrix}, \quad V = \begin{bmatrix} 1 & 0 \\ 0 & 1 \\ 1 & 1 \end{bmatrix}$$

已知缩放因子 $\sqrt{d_h} = \sqrt{2} \approx 1.414$ 。

**① 计算 $QK^T$ (维度 `[3, 3]`)**

$$QK^T = \begin{bmatrix} 1.414 & 0 \\ 0 & 1.414 \\ 1.414 & 1.414 \end{bmatrix} \begin{bmatrix} 2 & 0 & 0 \\ 0 & 2 & 0 \end{bmatrix} = \begin{bmatrix} 2.828 & 0 & 0 \\ 0 & 2.828 & 0 \\ 2.828 & 2.828 & 0 \end{bmatrix}$$

**② Scale 缩放计算 $S$**

$$S = \frac{QK^T}{1.414} = \begin{bmatrix} 2 & 0 & 0 \\ 0 & 2 & 0 \\ 2 & 2 & 0 \end{bmatrix}$$

**③ 加入 Causal Mask 矩阵 $M$**

$\boxed{\text{注意：Mask 必须加在 Softmax 之前}}$

$$M = \begin{bmatrix} 0 & -\infty & -\infty \\ 0 & 0 & -\infty \\ 0 & 0 & 0 \end{bmatrix}$$

$$S' = S + M = \begin{bmatrix} 2 & -\infty & -\infty \\ 0 & 2 & -\infty \\ 2 & 2 & 0 \end{bmatrix}$$

**④ Softmax 归一化得到 $A$**

* **Row 1** : $e^{-\infty}=0$ ，仅第一列有效 $\rightarrow [1, 0, 0]$
* **Row 2** : $\frac{e^0}{e^0+e^2} \approx 0.12, \frac{e^2}{e^0+e^2} \approx 0.88 \rightarrow [0.12, 0.88, 0]$
* **Row 3** : $\frac{e^2}{\sum}, \frac{e^2}{\sum}, \frac{e^0}{\sum} \rightarrow [0.468, 0.468, 0.063]$

$$A \approx \begin{bmatrix} 1 & 0 & 0 \\ 0.12 & 0.88 & 0 \\ 0.468 & 0.468 & 0.063 \end{bmatrix}$$

**⑤ 计算 $A \times V$ 得到最终输出 $Z_1$**

$$Z_1 = AV = \begin{bmatrix} 1 & 0 & 0 \\ 0.12 & 0.88 & 0 \\ 0.468 & 0.468 & 0.063 \end{bmatrix} \begin{bmatrix} 1 & 0 \\ 0 & 1 \\ 1 & 1 \end{bmatrix} = \begin{bmatrix} 1 & 0 \\ 0.12 & 0.88 \\ 0.531 & 0.531 \end{bmatrix}$$

输出张量维度完美契合：

$$Z_1 \in R^{L \times d_h} = R^{3 \times 2}$$

---

## 6. Multi-Head Attention

**多头并行处理与拼接：**
我们设定 $H=2$ 。刚才已算出 Head 1 的输出 $Z_1$ 。
假设 Head 2 也完成独立计算得到 $Z_2$ ：

$$Z_1 = \begin{bmatrix} 1 & 0 \\ 0.12 & 0.88 \\ 0.531 & 0.531 \end{bmatrix}_{3 \times 2}, \quad Z_2 = \begin{bmatrix} 0 & 1 \\ 0.5 & 0.5 \\ 0.2 & 0.8 \end{bmatrix}_{3 \times 2}$$

**Concat 拼接：**

$$Z = \operatorname{Concat}(Z_1, Z_2) = \begin{bmatrix} 1 & 0 & 0 & 1 \\ 0.12 & 0.88 & 0.5 & 0.5 \\ 0.531 & 0.531 & 0.2 & 0.8 \end{bmatrix}$$

拼接后维度恢复为： `[L, d] = [3, 4]` 。

**输出投影 (`o_proj`)：**

$$O = ZW_O$$

其中 $W_O \in R^{4 \times 4}$ 。最终注意力模块输出：

$$O \in R^{3 \times 4}$$

---

## 7. Residual + Norm

Attention 模块输出：

$$O \in R^{3 \times 4}$$

**残差连接：**

$$Y = X + O$$

（两者均为 `[3, 4]` ，直接相加。作用：保留原始信息，同时让网络更容易进行深层梯度回传。）

**Norm (如 LayerNorm / RMSNorm)：**

$$\mu = \frac{1}{d}\sum_{j=1}^{d}x_j, \quad \sigma^2 = \frac{1}{d}\sum_{j=1}^{d}(x_j-\mu)^2$$

$$LN(x)_j = \gamma_j\frac{x_j-\mu}{\sqrt{\sigma^2+\epsilon}} + \beta_j$$

---

## 8. MLP / FFN

输入经过 Norm 后维度依然为：

$$X_{norm} \in R^{3 \times 4}$$

**经典 FFN 结构：**

$$FFN(x) = W_2\operatorname{Activation}(W_1x+b_1) + b_2$$

**现代 LLM 的 Gated MLP 结构：**

$$\boxed{MLP(x) = W_{down}\left(\operatorname{Act}(W_{gate}x)\odot W_{up}x\right)}$$

**严格对应的数值演示：**
基于全篇设定的 $d=4, d_{ff}=8$ 。我们抽取第一个 Token 的特征向量 $x = [1, 2, -1, 0]$ (维度 `1x4`)：

* **gate_proj**: $W_{gate}x \in R^8$ 。假设 $= [1, 2, 0, -1, 0, 0, 0, 0]$
* **up_proj**: $W_{up}x \in R^8$ 。假设 $= [3, 4, 1, 1, 1, 1, 1, 1]$
* **SiLU 激活**: $\operatorname{SiLU}(x) = x \cdot \sigma(x)$
近似计算： $\operatorname{SiLU}([1, 2...]) \approx [0.731, 1.762, 0, -0.269, 0, 0, 0, 0]$
* **逐元素相乘 ( $\odot$ )**:
$[0.731, 1.762, 0, -0.269, 0, 0, 0, 0] \odot [3, 4, 1, 1, 1, 1, 1, 1]$
得到中间态维度 `[8]` 的张量： $M_{inner} = [2.193, 7.048, 0, -0.269, 0, 0, 0, 0]$
* **down_proj**:
$y = W_{down}M_{inner}$ (其中 $W_{down} \in R^{8 \times 4}$ )。
最终将向量从 $R^8$ 降维回 $R^4$ 。完美闭环。

---

## 9. 一个完整 Transformer Block

一个独立 Block 的数据流转历程：

$$X \rightarrow \operatorname{Norm} \rightarrow \operatorname{Attention} \rightarrow \operatorname{Residual} \rightarrow \operatorname{Norm} \rightarrow \operatorname{MLP} \rightarrow \operatorname{Residual}$$

**公式表达：**

$$H_1 = X + \operatorname{Attention}(\operatorname{Norm}(X))$$

$$H_2 = H_1 + \operatorname{MLP}(\operatorname{Norm}(H_1))$$

$$\boxed{X_{out} = H_2}$$

**层间堆叠：**

$$X_0 \rightarrow \operatorname{Block}_1 \rightarrow X_1 \rightarrow \operatorname{Block}_2 \rightarrow X_2 \rightarrow \cdots \rightarrow \operatorname{Block}_N$$

$$\boxed{\text{Block} \times N = \text{串行}}$$

---

## 10. 输出与 Token 预测

经过 $N$ 层后，最终隐藏状态：

$$H \in R^{L \times d} = R^{3 \times 4}$$

**Logits 计算 (`lm_head`)：**

$$Logits = HW_{lm}$$

其中：

$$W_{lm} \in R^{d \times V} = R^{4 \times 6}$$

维度相乘： `[3, 4] × [4, 6] = [3, 6]` (包含了 3 个 Token 在词表上的概率投影)

**数值预测示例（针对最后一个 Token，触发下一次生成）：**
假设第三个 Token 的最终 Hidden State 为 $H_3 = [1.0, -1.0, 2.0, 0.5]$ (维度 4)

* 与 `lm_head` `[4, 6]` 矩阵相乘，得到 6 个词元的分数：
Logits = `[1.2, -0.3, 4.1, 0.8, -0.5, 1.0]`
* **概率转化 (Softmax)：**

$$P_i = \frac{e^{z_i}}{\sum_je^{z_j}}$$

* **选择 / 采样策略：**

$$Token_{next} \sim P(Token \mid Context)$$

拿到新的 Token IDs，将其拼接入 Context，开启下一轮自回归（Auto-Regressive）生成。

---

## 11. Transformer 最核心的公式

**Attention**

$$\boxed{Q = XW_Q, \quad K = XW_K, \quad V = XW_V}$$

$$\boxed{\operatorname{Attention}(Q, K, V) = \operatorname{softmax}\left(\frac{QK^T}{\sqrt{d_h}} + M\right)V}$$

**Multi-Head**

$$\boxed{\operatorname{MHA}(X) = \operatorname{Concat}(\text{Head}_1, \dots, \text{Head}_H)W_O}$$

**Residual**

$$\boxed{Y = X + F(X)}$$

**MLP (Gated)**

$$\boxed{\operatorname{MLP}(x) = W_{down}\left(\operatorname{Act}(W_{gate}x)\odot W_{up}x\right)}$$

**输出概率**

$$\boxed{Logits = HW_{lm}}$$

$$\boxed{P(token_i) = \frac{e^{z_i}}{\sum_je^{z_j}}}$$

---

## 12. 一张图记住 Transformer

```text
                 Token IDs
                     │
                     ▼
               Embedding [L,d]
                     │
                     ▼
        ┌──────────────────────────┐
        │      Transformer Block   │
        │                          │
        │  Norm                    │
        │   │                      │
        │   ▼                      │
        │  ┌────────────────────┐  │
        │  │ Attention          │  │
        │  │                    │  │
        │  │ Head1 ─┐           │  │
        │  │ Head2 ─┤ 并行      │  │
        │  │ ...    ┤           │  │
        │  │ HeadH ─┘           │  │
        │  │    ↓               │  │
        │  │ Concat → o_proj    │  │
        │  └────────┬───────────┘  │
        │           ↓              │
        │       Residual           │
        │           ↓              │
        │          Norm            │
        │           ↓              │
        │  gate_proj ─┐            │
        │             × → down_proj│
        │  up_proj ───┘            │
        │           ↓              │
        │       Residual           │
        └───────────┬──────────────┘
                    │
              Block × N
                 串行
                    │
                    ▼
                 lm_head
                    │
                    ▼
                 Logits
                    │
                    ▼
              Next Token

```
