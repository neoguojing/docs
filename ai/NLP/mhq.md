# GQA / MQA / MHA

## 1. 核心区别

设：

$$
d_{model}=4096,\quad n_q=32,\quad d_{head}=128
$$

则：

$$
n_qd_{head}=32\times128=4096=d_{model}
$$

三者的核心区别是 **K/V Head 数量**：

| Attention | Q Heads | KV Heads | K/V 共享     | KV Cache |
| --------- | ------: | -------: | ---------- | -------: |
| MHA       |      32 |       32 | 不共享        |     100% |
| GQA       |      32 |        8 | 每 4 个 Q 共享 |      25% |
| MQA       |      32 |        1 | 所有 Q 共享    |   3.125% |

统一表示：

$$
Q\in R^{B\times n_q\times L\times d_{head}}
$$

$$
K,V\in R^{B\times n_{kv}\times L\times d_{head}}
$$

其中：

$$
n_{kv}=n_q\Rightarrow MHA
$$

$$
1<n_{kv}<n_q\Rightarrow GQA
$$

$$
n_{kv}=1\Rightarrow MQA
$$

---

## 2. MHA

输入：

$$
X\in R^{B\times L\times d_{model}}
$$

投影：

$$
Q=XW_Q,\quad K=XW_K,\quad V=XW_V
$$

其中：

$$
W_Q,W_K,W_V\in R^{d_{model}\times d_{model}}
$$

得到：

```text
Q/K/V: [B, L, 4096]
       ↓ reshape + transpose
[B, 32, L, 128]
```

每个 Query Head 都有独立的 K/V：

```text
Q1 → K1,V1
Q2 → K2,V2
...
Q32 → K32,V32
```

Attention：

$$
S=\frac{QK^T}{\sqrt{d_{head}}}
$$

$$
A=Softmax(S)
$$

$$
O=AV
$$

---

## 3. MQA

MQA 保留多个 Q Head，但所有 Q Head 共享一组 K/V。

```text
Q1 ─┐
Q2 ─┤
Q3 ─┼── K1,V1
Q4 ─┘
```

例如：

$$
n_q=32,\quad n_{kv}=1
$$

Q：

```text
[B, 32, L, 128]
```

K/V：

```text
[B, 1, L, 128]
```

因此：

$$
W_Q\in R^{4096\times4096}
$$

$$
W_K,W_V\in R^{4096\times128}
$$

---

## 4. GQA

GQA 是 MHA 与 MQA 的折中。

例如：

$$
n_q=32,\quad n_{kv}=8
$$

每个 KV Head 服务：

$$
g=\frac{n_q}{n_{kv}}
=\frac{32}{8}=4
$$

个 Query Head。

```text
Q1  ─┐
Q2  ─┤
Q3  ─┤── K1,V1
Q4  ─┘

Q5  ─┐
Q6  ─┤
Q7  ─┤── K2,V2
Q8  ─┘

...
```

统一映射：

$$
KVHead(q)=
\left\lfloor
\frac{q}{n_q/n_{kv}}
\right\rfloor
$$

例如 `32 Q / 8 KV`：

```text
Q0~Q3     → KV0
Q4~Q7     → KV1
Q8~Q11    → KV2
...
Q28~Q31   → KV7
```

Tensor Shape：

```text
Q: [B, 32, L, 128]
K: [B,  8, L, 128]
V: [B,  8, L, 128]
```

---

## 5. GQA 数值实现

假设：

```text
B = 1
L = 3
nq = 4
nkv = 2
d_head = 2
```

则：

```text
Q: [1, 4, 3, 2]
K: [1, 2, 3, 2]
V: [1, 2, 3, 2]
```

分组：

```text
Q0 Q1 → KV0
Q2 Q3 → KV1
```

对于 Q0：

$$
O_0=
Softmax
\left(
\frac{Q_0K_0^T}{\sqrt2}
\right)V_0
$$

对于 Q1：

$$
O_1=
Softmax
\left(
\frac{Q_1K_0^T}{\sqrt2}
\right)V_0
$$

对于 Q2/Q3：

$$
O_2=
Softmax
\left(
\frac{Q_2K_1^T}{\sqrt2}
\right)V_1
$$

$$
O_3=
Softmax
\left(
\frac{Q_3K_1^T}{\sqrt2}
\right)V_1
$$

**Q 不共享，K/V 在 Group 内共享。**

---

## 6. 工程实现

最直观的实现是复制 K/V：

```python
group_size = n_q // n_kv

K = repeat_kv(K, group_size)
V = repeat_kv(V, group_size)

scores = Q @ K.transpose(-1, -2)
scores = scores / sqrt(d_head)
A = softmax(scores)
O = A @ V
```

但是生产环境通常**不会真的复制 K/V**。

Kernel 直接根据：

$$
kv\_head=
\left\lfloor
\frac{q\_head}{group\_size}
\right\rfloor
$$

读取对应的 K/V。

因此：

```text
逻辑：
Q0 Q1 Q2 Q3 → KV0

物理：
KV0 只保存一份
```

避免额外显存和 Memory Bandwidth。

---

## 7. GQA 为什么影响模型性能？

GQA 减少 KV Head，会产生两个方向的影响。

### 优势

$$
n_{kv}\downarrow
$$

导致：

* K/V Projection 参数减少
* K/V 计算量减少
* KV Cache 减少
* GPU Memory Bandwidth 降低
* Decode 吞吐提高
* 同一 GPU 可支持更大的 Batch / Context

### 代价

K/V 共享增加：

$$
n_{kv}\downarrow
\Rightarrow
K/V\ 表达自由度\downarrow
$$

因此理论上存在一定模型能力损失。

但现代模型通常从训练阶段就采用 GQA，使这种损失得到控制。

**不能简单认为 MHA 一定比 GQA 更“聪明”。**最终效果还取决于模型规模、数据、训练方法、FFN、层数等整体架构。

---

## 8. KV Cache：GQA 最大工程价值

Decode 阶段，每生成一个 Token：

```text
Q_new
   ↓
读取历史 K/V Cache
   ↓
Attention
   ↓
生成 Token
```

KV Cache：

$$
Memory=
2\times
B\times L\times n_{kv}\times d_{head}\times bytes
$$

其中：

* `2`：K + V
* `B`：Batch
* `L`：上下文长度
* `n_kv`：KV Head 数
* `d_head`：Head 维度
* `bytes`：数据类型字节数

因此：

$$
KV\ Cache\propto n_{kv}
$$

---

## 9. 数值示例

假设：

```text
Layers = 32
Batch = 1
Context = 8192
d_head = 128
FP16 = 2 Bytes
n_q = 32
```

### MHA

$$
n_{kv}=32
$$

总 KV Cache：

$$
2\times32\times8192\times32\times128\times2
\approx4GB
$$

### GQA

$$
n_{kv}=8
$$

因为：

$$
\frac{8}{32}=25\%
$$

所以：

$$
KV\ Cache\approx1GB
$$

### MQA

$$
n_{kv}=1
$$

所以：

$$
KV\ Cache\approx128MB
$$

即：

```text
MHA    4 GB
GQA    1 GB
MQA    128 MB
```

相同条件下，GQA 将 KV Cache 降低到 MHA 的 **1/4**。

注意：

> KV Cache 降低 4 倍 ≠ 整体推理速度提升 4 倍，因为实际性能还受 GPU 算力、显存带宽、Attention Kernel、Batch、PagedAttention 等因素影响。

---

## 10. 参数量

MHA：

$$
W_K,W_V\in
R^{d_{model}\times d_{model}}
$$

GQA：

$$
W_K,W_V\in
R^{d_{model}\times(n_{kv}d_{head})}
$$

对于：

$$
d_{model}=4096,\quad d_{head}=128
$$

### MHA

```text
WK = 4096 × 4096
WV = 4096 × 4096
```

### GQA-8

$$
8\times128=1024
$$

```text
WK = 4096 × 1024
WV = 4096 × 1024
```

### MQA

```text
WK = 4096 × 128
WV = 4096 × 128
```

因此 GQA/MQA 不仅减少 KV Cache，也减少 K/V Projection 参数和计算。

---

## 11. 为什么 GQA 对 Decode 更重要？

### Prefill

一次处理大量 Prompt：

```text
Prompt
 ↓
Q/K/V
 ↓
Attention
```

通常计算量较大，更偏：

$$
Compute\ Bound
$$

### Decode

每次只生成一个 Token：

```text
Token
 ↓
Q_new
 ↓
读取大量历史 KV Cache
 ↓
Attention
```

更容易受到：

$$
Memory\ Bandwidth
$$

限制。

所以：

$$
n_{kv}\downarrow
\Rightarrow
KV\ Cache\downarrow
\Rightarrow
Memory\ Read\downarrow
\Rightarrow
Decode\ Efficiency\uparrow
$$

这也是 GQA 在 LLM 推理系统中的核心价值。

---

## 12. 三者统一理解

```text
                 Attention
                     │
          ┌──────────┴──────────┐
          │                     │
         Q                     K/V
          │                     │
      n_q heads              n_kv heads
          │                     │
          │             ┌───────┼───────┐
          │             │       │       │
          │            MHA     GQA     MQA
          │             │       │       │
          │           n_kv=nq  < nq      1
          │
          └───────────────┐
                          ↓
                    Attention
                          ↓
                     Output
```

核心只有一个参数：

$$
\boxed{n_{kv}}
$$

它决定了：

$$
n_{kv}\uparrow
\Rightarrow
K/V表达能力\uparrow,\quad
KV Cache\uparrow
$$

$$
n_{kv}\downarrow
\Rightarrow
K/V共享\uparrow,\quad
KV Cache\downarrow
$$

---

## 13. 与现代 LLM 的关系

当前模型架构可以大致理解为：

```text
MHA
 │
 ├── MQA
 │
 └── GQA ← 主流折中方案
       │
       ↓
   KV Cache 优化
       │
       ├── PagedAttention
       ├── FlashAttention
       └── KV Cache Quantization

另一条路线：

MHA/GQA
   ↓
MLA
   ↓
Latent KV Compression
```

其中 MLA（Multi-head Latent Attention）不是简单减少 KV Head，而是进一步压缩 KV 的表示方式。

---

## 14. 面试总结

> **GQA（Grouped-Query Attention）保持多个 Query Head，同时让多个 Query Head 共享一个 K/V Head。它本质上是在 Attention 表达能力与推理成本之间做 Trade-off。相比 MHA，GQA 可以显著降低 KV Cache、K/V Projection 参数和 Decode 阶段的内存带宽压力；相比 MQA，又保留了更多 K/V 表达能力，因此成为现代 LLM 中常见的 Attention 设计。**
