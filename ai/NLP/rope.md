从最简单的**二维平面几何**与**高中三角函数**开始，一步步推导出 RoPE 的数学原理，并给出一套带具体数字的计算示例。

---

### 第一步：我们要解决的数学问题是什么？

在 Self-Attention 中，模型需要计算 Query 向量和 Key 向量的点积（内积）：


$$\text{Score} = q^\top k$$

若 $q$ 在位置 $m$，$k$ 在位置 $n$：

* 我们需要找到一个函数 $f(x, \text{pos})$，把位置信息融进去，得到带位置的向量 $\tilde{q}_m = f(q, m)$ 和 $\tilde{k}_n = f(k, n)$。
* **终极数学目标**：它们的内积必须只与**内容**和**相对位置差 $(m - n)$** 有关，不能残留绝对位置 $m$ 或 $n$：

$$\langle \tilde{q}_m, \tilde{k}_n \rangle = g(q, k, m - n)$$



---

### 第二步：2D 几何推导与旋转矩阵

先退回到最简单的二维向量：$x = [x_0, x_1]^\top$。

#### 1. 为什么“旋转”能天然实现内积只差 $(m - n)$？

在二维平面上，向量 $q$ 的长度为 $\Vert{}q\Vert{}$，与横轴夹角设为 $\alpha$；向量 $k$ 的长度为 $\Vert{}k\Vert{}$，与横轴夹角设为 $\beta$。

向量的内积公式为：


$$q^\top k = \Vert{}q\Vert{} \Vert{}k\Vert{} \cos(\alpha - \beta)$$

现在，给位置为 $m$ 的向量 $q$ 旋转一个角度 $m\theta$，给位置为 $n$ 的向量 $k$ 旋转角度 $n\theta$：

* 旋转后的 $q$ 角度变为：$\alpha + m\theta$
* 旋转后的 $k$ 角度变为：$\beta + n\theta$
* 旋转不改变向量的长度（模长不变）

此时计算它们旋转后的内积：


$$\langle \tilde{q}_m, \tilde{k}_n \rangle = \Vert{}q\Vert{} \Vert{}k\Vert{} \cos\big((\alpha + m\theta) - (\beta + n\theta)\big) = \Vert{}q\Vert{} \Vert{}k\Vert{} \cos\big((\alpha - \beta) + (m - n)\theta\big)$$

绝对位置 $m$ 和 $n$ 在相减时直接抵消，**只剩下了相对距离 $(m - n)$**。

#### 2. 二维旋转用矩阵如何表达？

线性代数中，平面向量逆时针旋转 $\phi$ 角的标准变换矩阵为：


$$\boldsymbol{R}(\phi) = \begin{pmatrix} \cos\phi & -\sin\phi \\ \sin\phi & \cos\phi \end{pmatrix}$$

将旋转角设为 $\phi = m\theta$，变换公式即为：


$$\tilde{q}_m = \boldsymbol{R}_m q = \begin{pmatrix} \cos m\theta & -\sin m\theta \\ \sin m\theta & \cos m\theta \end{pmatrix} \begin{pmatrix} q_0 \\ q_1 \end{pmatrix} = \begin{pmatrix} q_0\cos m\theta - q_1\sin m\theta \\ q_0\sin m\theta + q_1\cos m\theta \end{pmatrix}$$

---

### 第三步：具体数字计算示例（2D 验证）

我们设定具体的数值来走一遍完整算式：

1. **基本设定**：
* 旋转基频设为：$\theta = 30^\circ$
* 向量 $q = [1, 0]^\top$（模长为 1，初始角度 $\alpha = 0^\circ$）
* 向量 $k = [0, 1]^\top$（模长为 1，初始角度 $\beta = 90^\circ$）


2. **情况 A：位置差距为 1**（设 $m = 2, n = 1$，差值为 $m - n = 1$）
* $q$ 在位置 2，旋转角度 $2 \times 30^\circ = 60^\circ$：

$$\tilde{q}_2 = \begin{pmatrix} \cos 60^\circ & -\sin 60^\circ \\ \sin 60^\circ & \cos 60^\circ \end{pmatrix} \begin{pmatrix} 1 \\ 0 \end{pmatrix} = \begin{pmatrix} 0.5 \\ \frac{\sqrt{3}}{2} \end{pmatrix}$$


* $k$ 在位置 1，旋转角度 $1 \times 30^\circ = 30^\circ$：

$$\tilde{k}_1 = \begin{pmatrix} \cos 30^\circ & -\sin 30^\circ \\ \sin 30^\circ & \cos 30^\circ \end{pmatrix} \begin{pmatrix} 0 \\ 1 \end{pmatrix} = \begin{pmatrix} -0.5 \\ \frac{\sqrt{3}}{2} \end{pmatrix}$$


* 计算内积：

$$\tilde{q}_2^\top \tilde{k}_1 = (0.5) \times (-0.5) + \left(\frac{\sqrt{3}}{2}\right) \times \left(\frac{\sqrt{3}}{2}\right) = -0.25 + 0.75 = 0.5$$




3. **情况 B：位置差距同样为 1，但绝对位置变大**（设 $m = 5, n = 4$，差值依然为 $m - n = 1$）
* $q$ 在位置 5，旋转 $5 \times 30^\circ = 150^\circ$：

$$\tilde{q}_5 = \begin{pmatrix} \cos 150^\circ & -\sin 150^\circ \\ \sin 150^\circ & \cos 150^\circ \end{pmatrix} \begin{pmatrix} 1 \\ 0 \end{pmatrix} = \begin{pmatrix} -\frac{\sqrt{3}}{2} \\ 0.5 \end{pmatrix}$$


* $k$ 在位置 4，旋转 $4 \times 30^\circ = 120^\circ$：

$$\tilde{k}_4 = \begin{pmatrix} \cos 120^\circ & -\sin 120^\circ \\ \sin 120^\circ & \cos 120^\circ \end{pmatrix} \begin{pmatrix} 0 \\ 1 \end{pmatrix} = \begin{pmatrix} -\frac{\sqrt{3}}{2} \\ -0.5 \end{pmatrix}$$


* 计算内积：

$$\tilde{q}_5^\top \tilde{k}_4 = \left(-\frac{\sqrt{3}}{2}\right) \times \left(-\frac{\sqrt{3}}{2}\right) + (0.5) \times (-0.5) = 0.75 - 0.25 = 0.5$$





两次计算结果完全一致（都为 $0.5$），证明了**内积结果严格只取决于相对距离 $(m - n)$**。

---

### 第四步：从 2 维拓展到 $d$ 维空间

实际 Transformer 中的单头隐藏层维度通常是 $d = 64$ 或 $128$（$d$ 必为偶数）。

RoPE 的策略是：**将高维向量两两分组，拆成 $d/2$ 个相互独立的二维平面**。

对于向量 $x = [x_0, x_1, x_2, x_3, \dots, x_{d-2}, x_{d-1}]^\top$：

* 平面 0 由 $(x_0, x_1)$ 构成，旋转角度步长为 $\theta_0$
* 平面 1 由 $(x_2, x_3)$ 构成，旋转角度步长为 $\theta_1$
* 平面 $i$ 由 $(x_{2i}, x_{2i+1})$ 构成，旋转角度步长为 $\theta_i$

#### 1. 整体旋转矩阵形式

整体矩阵是一个对角线上全是 2D 旋转矩阵的**分块对角矩阵**：

$$\boldsymbol{R}_{\Theta, m}^d = \begin{pmatrix} \cos m\theta_0 & -\sin m\theta_0 & 0 & 0 & \dots & 0 & 0 \\ \sin m\theta_0 & \cos m\theta_0 & 0 & 0 & \dots & 0 & 0 \\ 0 & 0 & \cos m\theta_1 & -\sin m\theta_1 & \dots & 0 & 0 \\ 0 & 0 & \sin m\theta_1 & \cos m\theta_1 & \dots & 0 & 0 \\ \vdots & \vdots & \vdots & \vdots & \ddots & \vdots & \vdots \\ 0 & 0 & 0 & 0 & \dots & \cos m\theta_{\frac{d}{2}-1} & -\sin m\theta_{\frac{d}{2}-1} \\ 0 & 0 & 0 & 0 & \dots & \sin m\theta_{\frac{d}{2}-1} & \cos m\theta_{\frac{d}{2}-1} \end{pmatrix}$$

#### 2. 角频率 $\theta_i$ 的设计

每一个平面的旋转频率取法直接继承了经典 Transformer 的正弦波长设定：


$$\theta_i = 10000^{-2i / d}, \quad i \in \{0, 1, \dots, \frac{d}{2} - 1\}$$

* 当 $i = 0$ 时，$\theta_0 = 10000^0 = 1$（旋转频率极高，位置每变 1，转动弧度很大，负责刻画邻近 token 的精确相对位置）。
* 当 $i$ 变大时，$\theta_i$ 迅速趋近于 0（旋转频率极低，位置走很远才转一点点，负责捕捉远距离上下文的粗粒度位置）。

---

### 第五步：工程代码层面的向量化计算

在实际代码（PyTorch 等）中，不会真去构建一个稀疏的 $d \times d$ 矩阵来做矩阵乘法。

观察每个二维对的计算：


$$\begin{pmatrix} \tilde{x}_{2i} \\ \tilde{x}_{2i+1} \end{pmatrix} = \begin{pmatrix} x_{2i}\cos(m\theta_i) - x_{2i+1}\sin(m\theta_i) \\ x_{2i}\sin(m\theta_i) + x_{2i+1}\cos(m\theta_i) \end{pmatrix}$$

可以改写为两个向量的逐元素相乘（Hadamard 积 $\odot$）：

$$\tilde{x} = x \odot \begin{pmatrix} \cos m\theta_0 \\ \cos m\theta_0 \\ \cos m\theta_1 \\ \cos m\theta_1 \\ \vdots \end{pmatrix} + \begin{pmatrix} -x_1 \\ x_0 \\ -x_3 \\ x_2 \\ \vdots \end{pmatrix} \odot \begin{pmatrix} \sin m\theta_0 \\ \sin m\theta_0 \\ \sin m\theta_1 \\ \sin m\theta_1 \\ \vdots \end{pmatrix}$$

这种表示只需：

1. 翻转相邻维度并给奇数位添负号：`rotate_half(x)`；
2. 算一次向量逐元素乘法和加法：`x * cos + rotate_half(x) * sin`。

显存占用和算力开销均为最低的 $O(d)$。
