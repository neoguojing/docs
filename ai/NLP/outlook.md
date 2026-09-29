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
* **Attention 内部（并行与串行）：**
```text
Head 1 ─┐
Head 2 ─┤
Head 3 ─┤ 并行
...     ┤
Head H ─┘
        ↓
      Concat
        ↓
      o_proj  (串行)

```



**结论：** Block 之间串行；Attention 与 MLP 串行；Multi-Head 之间并行。

---

## 2. 核心超参数

**参数设定：**

* $L$ = sequence length (序列长度)
* $d$ = $d_{model}$ = hidden size (隐藏层维度)
* $H$ = num_heads (注意力头数)
* $d_h$ = $d_{head} = d / H$ (每个头的维度)
* $d_{ff}$ = MLP hidden size (前馈网络隐藏层维度)
* $V$ = vocab_size (词表大小)
* $N$ = num_layers (层数)

**具体数值示例：**

* $L = 3$
* $d = 4$
* $H = 2$
* $d_h = 2$
* $d_{ff} = 8$
* $V = 6$
* $N = 2$

**由此得出的张量维度：**

* $X$: `[3, 4]`
* $Q / K / V$ (每个Head): `[3, 2]`
* Attention矩阵: `[3, 3]`
* MLP变换: `[3, 4] → [3, 8] → [3, 4]`
* Logits: `[3, 6]`

---

## 3. Embedding

**Token 输入：**
$t = [2, 5, 1]$

**Embedding 参数：**


$$E \in R^{V \times d} = R^{6 \times 4}$$

**查表得到：**


$$X = \begin{bmatrix} 1 & 0 & 1 & 0 \\ 0 & 1 & 0 & 1 \\ 1 & 1 & 0 & 0 \end{bmatrix}$$

$$X \in R^{L \times d} = R^{3 \times 4}$$

**数学表达：**


$$X_i = E[t_i]$$

若使用可加式位置编码：


$$X_i = E[t_i] + P_i$$


*(注：现代 LLM 常使用 RoPE 等方式注入位置信息。)*

---

## 4. Attention：Q/K/V 投影

**输入：**


$$X \in R^{L \times d}$$

**参数：**


$$W_Q, W_K, W_V \in R^{d \times d_h}$$

**计算：**


$$Q = XW_Q \quad K = XW_K \quad V = XW_V$$

**维度变化：**

* $X$: `[3, 4]`
* $W_Q$: `[4, 2]`
* **$Q, K, V$**: 均得到 `[3, 2]`

**对应模型参数命名：**

* `q_proj` → $W_Q$
* `k_proj` → $W_K$
* `v_proj` → $W_V$

---

## 5. Attention 核心公式

$$S = \frac{QK^T}{\sqrt{d_h}}$$

**其中维度：**

* $Q$: `[L, d_h]`
* $K^T$: `[d_h, L]`
* **$S$**: `[L, L]`

**加入 Mask：**


$$S' = S + M$$


其中禁止访问的位置为：$M_{ij} = -\infty$

**然后计算 Attention 权重：**


$$A = \operatorname{softmax}(S')$$

**最后乘以 Value：**


$$Z = AV$$

**完整公式：**


$$\boxed{Z = \operatorname{softmax} \left( \frac{QK^T}{\sqrt{d_h}} + M \right) V}$$

**最终维度推导：**

* $QK^T$: `[3, 3]`
* Softmax: `[3, 3]`
* $V$: `[3, 2]`
* **$Z$**: `[3, 2]`

---

## 6. Attention 数值计算

为了看清楚计算过程，使用一个 Head 演示：


$$Q = K = \begin{bmatrix} 2 & 0 \\ 0 & 2 \end{bmatrix}, \quad V = \begin{bmatrix} 2 & 0 \\ 0 & 2 \end{bmatrix}$$


这里设定 $d_h = 2$。

**① 计算 $QK^T$**


$$QK^T = \begin{bmatrix} 2 & 0 \\ 0 & 2 \end{bmatrix} \begin{bmatrix} 2 & 0 \\ 0 & 2 \end{bmatrix} = \begin{bmatrix} 4 & 0 \\ 0 & 4 \end{bmatrix}$$

**② Scale 缩放**


$$\sqrt{d_h} = \sqrt{2} \approx 1.414$$

$$S = \begin{bmatrix} 4/1.414 & 0 \\ 0 & 4/1.414 \end{bmatrix} = \begin{bmatrix} 2.828 & 0 \\ 0 & 2.828 \end{bmatrix}$$

**③ Softmax 计算**
第一行概率计算：


$$\frac{e^{2.828}}{e^{2.828} + e^0} \approx 0.944, \quad \frac{e^0}{e^{2.828} + e^0} \approx 0.056$$


得出注意力分布矩阵：


$$A \approx \begin{bmatrix} 0.944 & 0.056 \\ 0.056 & 0.944 \end{bmatrix}$$

**④ A × V**


$$Z = AV = \begin{bmatrix} 0.944 & 0.056 \\ 0.056 & 0.944 \end{bmatrix} \begin{bmatrix} 2 & 0 \\ 0 & 2 \end{bmatrix} = \begin{bmatrix} 1.888 & 0.112 \\ 0.112 & 1.888 \end{bmatrix}$$


输出张量：

$$Z \in R^{L \times d_h}$$

---

## 7. Causal Mask

对于 Decoder-only LLM 的可见性限制：

| Query \ Key | 1 | 2 | 3 |
| --- | --- | --- | --- |
| **1** | ✓ | × | × |
| **2** | ✓ | ✓ | × |
| **3** | ✓ | ✓ | ✓ |

**Mask 矩阵 $M$：**


$$M = \begin{bmatrix} 0 & -\infty & -\infty \\ 0 & 0 & -\infty \\ 0 & 0 & 0 \end{bmatrix}$$

$$\boxed{\text{注意：Mask 必须加在 Softmax 之前}}$$

**计算示例：**


$$S = \begin{bmatrix} 2.8 & 1.2 & 0.5 \\ 0.1 & 2.5 & 1.0 \\ 0.3 & 0.7 & 2.2 \end{bmatrix}$$


加 Mask 后：


$$S + M = \begin{bmatrix} 2.8 & -\infty & -\infty \\ 0.1 & 2.5 & -\infty \\ 0.3 & 0.7 & 2.2 \end{bmatrix}$$

Softmax 后，未来 Token 的对应概率为 $e^{-\infty} = 0$，因此模型严格无法看到未来信息。

---

## 8. Multi-Head Attention

每个 Head 独立进行参数投影与 Attention 计算：


$$Q_i = XW_{Q_i}, \quad K_i = XW_{K_i}, \quad V_i = XW_{V_i}$$

$$Z_i = \operatorname{Attention}(Q_i, K_i, V_i)$$

**多头并行处理与拼接：**

```text
             ┌─ Head 1 → Z₁
             │
X ───────────┼─ Head 2 → Z₂
             │
             ├─ ...
             │
             └─ Head H → Z_H
                    │
                 并行
                    ↓
                 Concat

```

$$Z = \operatorname{Concat}(Z_1, \dots, Z_H)$$

**维度变化 (假设 $H = 2, d_h = 2$)：**

* $Z_1$: `[L, 2]`
* $Z_2$: `[L, 2]`
* Concat 拼接后: `[L, 4]`

**输出投影：**


$$O = ZW_O$$


其中 

$$W_O \in R^{(H \cdot d_h) \times d}$$


本例中：

$$W_O \in R^{4 \times 4}$$


最终输出：

$$O \in R^{L \times d}$$

**对应模型参数命名：**

* `o_proj` → $W_O$

---

## 9. Residual + Norm

Attention 模块输出：

$$O \in R^{L \times d}$$


**残差连接：** 

$$Y = X + O$$


*作用：保留原始信息，同时让网络更容易进行深层训练。*

**LayerNorm 形式 (依模型而定)：**


$$\mu = \frac{1}{d} \sum_{j=1}^{d}x_j$$

$$\sigma^2 = \frac{1}{d} \sum_{j=1}^{d}(x_j - \mu)^2$$

$$LN(x)_j = \gamma_j \frac{x_j - \mu}{\sqrt{\sigma^2 + \epsilon}} + \beta_j$$


其中可学习参数：

$$\gamma, \beta \in R^d$$


*(注：现代 LLM 常使用 RMSNorm 代替标准 LayerNorm。)*

---

## 10. MLP / FFN

输入：

$$X \in R^{L \times d}$$

**经典 FFN 结构：**


$$FFN(x) = W_2 \operatorname{Activation}(W_1x + b_1) + b_2$$


维度转换：$W_1 \in R^{d \times d_{ff}}$, $W_2 \in R^{d_{ff} \times d}$
流程：`[L, 4] → (W1) → [L, 8] → (Act) → [L, 8] → (W2) → [L, 4]`

**现代 LLM 的 Gated MLP 结构：**


$$\boxed{MLP(x) = W_{down} \left( \operatorname{Act}(W_{gate}x) \odot W_{up}x \right)}$$

**对应模型参数命名与维度：**

* `gate_proj`: `[d, d_ff]`
* `up_proj`: `[d, d_ff]`
* `down_proj`: `[d_ff, d]`

**数值计算示例：**
设 $x = [1, 2, 3, 4]$
假设：$W_{gate}x = [1, 2]$, $W_{up}x = [3, 4]$
使用 SiLU 激活函数：$\operatorname{SiLU}(x) = x \cdot \sigma(x)$
近似计算：$\operatorname{SiLU}([1, 2]) \approx [0.731, 1.762]$
逐元素乘法 ($\odot$)：


$$[0.731, 1.762] \odot [3, 4] = [2.193, 7.048]$$


最后经过下投影：


$$y = W_{down}[2.193, 7.048]$$


最终输出：

$$y \in R^d$$

---

## 11. 一个完整 Transformer Block

一个独立 Block 的计算流：


$$X \rightarrow \operatorname{Norm} \rightarrow \operatorname{Attention} \rightarrow \operatorname{Residual} \rightarrow \operatorname{Norm} \rightarrow \operatorname{MLP} \rightarrow \operatorname{Residual}$$

**公式表达：**


$$H_1 = X + \operatorname{Attention}(\operatorname{Norm}(X))$$

$$H_2 = H_1 + \operatorname{MLP}(\operatorname{Norm}(H_1))$$

$$\boxed{X_{out} = H_2}$$

**层间堆叠：**


$$X_0 \rightarrow \operatorname{Block}_1 \rightarrow X_1 \rightarrow \operatorname{Block}_2 \rightarrow X_2 \rightarrow \cdots \rightarrow \operatorname{Block}_N$$

$$\boxed{\text{Block} \times N = \text{串行}}$$

---

## 12. 输出与 Token 预测

最终隐藏状态：

$$H \in R^{L \times d}$$

**Logits 计算：**


$$Logits = H W_{lm}$$


其中：

$$W_{lm} \in R^{d \times V}$$


维度相乘：`[L, d] × [d, V] = [L, V]`

**例如：**

* Hidden = `[1, 2, 3, 4]`
* $W_{lm}$ = `[4, 6]`
* Logits = `[1.2, 0.3, 4.1, 0.8, 0.5, 1.0]`

**概率转化为：**


$$P_i = \frac{e^{z_i}}{\sum_j e^{z_j}}$$

**选择 / 采样策略：**


$$Token_{next} \sim P(Token \mid Context)$$


然后把新 Token 放回模型，继续下一轮生成迭代。

---

## 13. 参数与网络位置对应

| 网络位置 | 参数名 | 对应代码命名 | 维度 |
| --- | --- | --- | --- |
| **Embedding** | $E$ | - | `[V, d]` |
| **Attention** | $W_Q$ | `q_proj` | `[d, H·d_h]` |
| **Attention** | $W_K$ | `k_proj` | `[d, H·d_h]` |
| **Attention** | $W_V$ | `v_proj` | `[d, H·d_h]` |
| **Attention** | $W_O$ | `o_proj` | `[H·d_h, d]` |
| **MLP** | $W_{gate}$ | `gate_proj` | `[d, d_ff]` |
| **MLP** | $W_{up}$ | `up_proj` | `[d, d_ff]` |
| **MLP** | $W_{down}$ | `down_proj` | `[d_ff, d]` |
| **Norm** | $\gamma, \beta$ | - | `[d]` |
| **Output** | $W_{lm}$ | `lm_head` | `[d, V]` |

*注：在标准 MHA 中 $H \cdot d_h = d$，因此注意力层的权重矩阵通常为 $W_Q, W_K, W_V, W_O \in R^{d \times d}$。*

---

## 14. Transformer 最核心的公式

**Attention**


$$\boxed{Q = XW_Q, \quad K = XW_K, \quad V = XW_V}$$

$$\boxed{\operatorname{Attention}(Q, K, V) = \operatorname{softmax} \left( \frac{QK^T}{\sqrt{d_h}} + M \right) V}$$

**Multi-Head**


$$\boxed{\operatorname{MHA}(X) = \operatorname{Concat}(\text{Head}_1, \dots, \text{Head}_H) W_O}$$

**Residual**


$$\boxed{Y = X + F(X)}$$

**MLP (经典 vs Gated)**


$$\boxed{\operatorname{FFN}(x) = W_2 \operatorname{Act}(W_1x)}$$

$$\boxed{\operatorname{MLP}(x) = W_{down} \left( \operatorname{Act}(W_{gate}x) \odot W_{up}x \right)}$$

**输出概率**


$$\boxed{Logits = H W_{lm}}$$

$$\boxed{P(token_i) = \frac{e^{z_i}}{\sum_j e^{z_j}}}$$

---

## 15. 一张图记住 Transformer

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

> **核心理解：**
> Attention 负责 Token 与 Token 之间的信息交互；MLP 负责单个 Token 内部的特征变换；Multi-Head 并行学习不同关系；Transformer Block 逐层串行堆叠。
