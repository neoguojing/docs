# RoPE（Rotary Position Embedding，旋转位置编码）全景解析

---

## 1. 核心设计动机与背景

### 1.1 传统绝对位置编码的局限
在经典的 Transformer 架构中，由于自注意力机制（Self-Attention）本身是置换不变的（无法分辨输入词序），传统方法通常使用**绝对位置编码做加法**：

$$x_m = e_m + p_m$$

其中 $e_m$ 为词嵌入向量， $p_m$ 为固定的正弦编码或可学习的位置向量。

这种直接相加的方式存在明显缺陷：
* **耦合严重**：计算注意力内积时，展开项（如 $e_m^\top p_n$、 $p_m^\top e_n$）将内容与位置信息强行交织在一起。
* **难以体现相对距离**：模型很难天然识别“第 1 与第 2 个 token”与“第 100 与第 101 个 token”之间同样相隔距离 1。

### 1.2 RoPE 的数学目标
RoPE 的出发点不是做加法，而是寻找一个变换函数 $f(x, \text{pos})$，使得 Query 向量 $q$（位置 $m$）与 Key 向量 $k$（位置 $n$）在注入位置编码后的内积，**严格只取决于各自的词内容以及两者的相对位置差 $(m - n)$**：

$$\langle f(q, m), f(k, n) \rangle = g(q, k, m - n)$$

---

## 2. 2D 几何直观与数学推导

### 2.1 钟表指针比喻
* **词义（语义）**：决定指针的长度与最初的初始朝向角度。
* **绝对位置（第几个词）**：决定把这根指针依步长顺时针/逆时针旋转多少度。
* **相对位置**：注意力内积主要衡量两根指针的夹角。当两根指针都在各自位置上转动后，求夹角时绝对转过的角度会相互抵消，**只剩下相对夹角**。

---

### 2.2 2D 几何推导
设二维向量 $q = [q_0, q_1]^\top$ 的模长为 $\Vert q \Vert$、初始方向角为 $\alpha$；向量 $k = [k_0, k_1]^\top$ 的模长为 $\Vert k \Vert$、初始方向角为 $\beta$。

向量的标准几何内积公式为：

$$q^\top k = \Vert q \Vert \Vert k \Vert \cos(\alpha - \beta)$$

对位于位置 $m$ 的 $q$ 旋转角度 $m\theta$，对位于位置 $n$ 的 $k$ 旋转角度 $n\theta$：
* 旋转后 $q$ 的方向角变为： $\alpha + m\theta$
* 旋转后 $k$ 的方向角变为： $\beta + n\theta$
* 正交旋转变换不改变向量的模长

此时计算变换后的内积：

$$\langle \tilde{q}_m, \tilde{k}_n \rangle = \Vert q \Vert \Vert k \Vert \cos\big((\alpha + m\theta) - (\beta + n\theta)\big) = \Vert q \Vert \Vert k \Vert \cos\big((\alpha - \beta) + (m - n)\theta\big)$$

**结论**：绝对位置标号 $m$ 与 $n$ 在做差时完全抵消，保留下来的位置变量只有相对差值 $(m - n)$。

---

### 2.3 2D 旋转矩阵表达
二维平面上逆时针旋转 $\phi$ 角的标准正交变换矩阵为：

$$\boldsymbol{R}(\phi) = \begin{pmatrix} \cos\phi & -\sin\phi \cr \sin\phi & \cos\phi \end{pmatrix}$$

将旋转角设为 $\phi = m\theta$，变换过程写为矩阵乘法：

$$\tilde{q}_m = \boldsymbol{R}_m q = \begin{pmatrix} \cos m\theta & -\sin m\theta \cr \sin m\theta & \cos m\theta \end{pmatrix} \begin{pmatrix} q_0 \cr q_1 \end{pmatrix} = \begin{pmatrix} q_0\cos m\theta - q_1\sin m\theta \cr q_0\sin m\theta + q_1\cos m\theta \end{pmatrix}$$

代入注意力内积：

$$\tilde{q}_m^\top \tilde{k}_n = (\boldsymbol{R}_m q)^\top (\boldsymbol{R}_n k) = q^\top \boldsymbol{R}_m^\top \boldsymbol{R}_n k$$

利用旋转矩阵的正交性与角度可加性：

$$\boldsymbol{R}_m^\top = \boldsymbol{R}_{-m}, \quad \boldsymbol{R}_{-m}\boldsymbol{R}_n = \boldsymbol{R}_{n - m}$$

可得：

 $$\tilde{q}_m^\top \tilde{k}_n = q^\top \boldsymbol{R}_{n - m} k$$

---

### 2.4 2D 具体数值验算示例

设定：
* 旋转步长基频： $\theta = 30^\circ$
* 初始向量： $q = [1, 0]^\top$（模长 1，角度 $0^\circ$），$k = [0, 1]^\top$（模长 1，角度 $90^\circ$）

#### 情况 1：位置 $m = 2, n = 1$（相对距离 $m - n = 1$）
* $\tilde{q}_2$（旋转 $2 \times 30^\circ = 60^\circ$）：

$$\tilde{q}_2 = \begin{pmatrix} \cos 60^\circ & -\sin 60^\circ \cr \sin 60^\circ & \cos 60^\circ \end{pmatrix} \begin{pmatrix} 1 \cr 0 \end{pmatrix} = \begin{pmatrix} 0.5 \cr \frac{\sqrt{3}}{2} \end{pmatrix}$$

* $\tilde{k}_1$（旋转 $1 \times 30^\circ = 30^\circ$）：

$$\tilde{k}_1 = \begin{pmatrix} \cos 30^\circ & -\sin 30^\circ \cr \sin 30^\circ & \cos 30^\circ \end{pmatrix} \begin{pmatrix} 0 \cr 1 \end{pmatrix} = \begin{pmatrix} -0.5 \cr \frac{\sqrt{3}}{2} \end{pmatrix}$$

* 计算内积：

$$\tilde{q}_2^\top \tilde{k}_1 = (0.5) \times (-0.5) + \left(\frac{\sqrt{3}}{2}\right) \times \left(\frac{\sqrt{3}}{2}\right) = -0.25 + 0.75 = 0.5$$

#### 情况 2：位置 $m = 5, n = 4$（相对距离同样为 $m - n = 1$）
* $\tilde{q}_5$（旋转 $5 \times 30^\circ = 150^\circ$）：

$$\tilde{q}_5 = \begin{pmatrix} \cos 150^\circ & -\sin 150^\circ \cr \sin 150^\circ & \cos 150^\circ \end{pmatrix} \begin{pmatrix} 1 \cr 0 \end{pmatrix} = \begin{pmatrix} -\frac{\sqrt{3}}{2} \cr 0.5 \end{pmatrix}$$

* $\tilde{k}_4$（旋转 $4 \times 30^\circ = 120^\circ$）：

$$\tilde{k}_4 = \begin{pmatrix} \cos 120^\circ & -\sin 120^\circ \cr \sin 120^\circ & \cos 120^\circ \end{pmatrix} \begin{pmatrix} 0 \cr 1 \end{pmatrix} = \begin{pmatrix} -\frac{\sqrt{3}}{2} \cr -0.5 \end{pmatrix}$$

* 计算内积：

$$\tilde{q}_5^\top \tilde{k}_4 = \left(-\frac{\sqrt{3}}{2}\right) \times \left(-\frac{\sqrt{3}}{2}\right) + (0.5) \times (-0.5) = 0.75 - 0.25 = 0.5$$

**验算结论**：当相对距离保持为 1 时，无论 token 处于句首还是后文，内积结果严格等于 $0.5$。

---

## 3. 高维拓展与多频机制

### 3.1 拓展到 $d$ 维空间
实际单头注意力维度 $d$（偶数）通常为 64 或 128。RoPE 将 $d$ 维向量切分成 $d/2$ 个相互正交的二维子空间：

$$\boldsymbol{R}_{\Theta, m}^d = \begin{pmatrix} \boldsymbol{R}_{m\theta_0} & 0 & \dots & 0 \cr 0 & \boldsymbol{R}_{m\theta_1} & \dots & 0 \cr \vdots & \vdots & \ddots & \vdots \cr 0 & 0 & \dots & \boldsymbol{R}_{m\theta_{d/2 - 1}} \end{pmatrix}$$

展开为完整的对角块矩阵：

$$\boldsymbol{R}_{\Theta, m}^d = \begin{pmatrix} \cos m\theta_0 & -\sin m\theta_0 & 0 & 0 & \dots & 0 & 0 \cr \sin m\theta_0 & \cos m\theta_0 & 0 & 0 & \dots & 0 & 0 \cr 0 & 0 & \cos m\theta_1 & -\sin m\theta_1 & \dots & 0 & 0 \cr 0 & 0 & \sin m\theta_1 & \cos m\theta_1 & \dots & 0 & 0 \cr \vdots & \vdots & \vdots & \vdots & \ddots & \vdots & \vdots \cr 0 & 0 & 0 & 0 & \dots & \cos m\theta_{\frac{d}{2}-1} & -\sin m\theta_{\frac{d}{2}-1} \cr 0 & 0 & 0 & 0 & \dots & \sin m\theta_{\frac{d}{2}-1} & \cos m\theta_{\frac{d}{2}-1} \end{pmatrix}$$

---

### 3.2 频率分配公式与多频机制解析

角频率定义沿用了 Transformer 正弦位置编码的底数规则：

$$
\theta_i = 10000^{-2i/d}, \quad i \in \{0, 1, \dots, \frac{d}{2} - 1\}
$$
#### 频率跨度数值演示（以 $d = 6$ 为例，划分为 3 个平面）
* **平面 0（$i = 0$，高频/秒针）**：

$$\theta_0 = 10000^{-0} = 1.0 \text{ rad} \approx 57.3^\circ$$

* **平面 1（ $i = 1$ ，中频/分针）**：

$$\theta_1 = 10000^{-2/6} = 10000^{-1/3} \approx 0.0464 \text{ rad} \approx 2.66^\circ$$

* **平面 2（ $i = 2$ ，低频/时针）**：

$$\theta_2 = 10000^{-4/6} = 10000^{-2/3} \approx 0.00215 \text{ rad} \approx 0.123^\circ$$

#### 位置演进对照表

| 位置 $m$ | 平面 0 转动角度 ($m\theta_0$) | 平面 1 转动角度 ($m\theta_1$) | 平面 2 转动角度 ($m\theta_2$) | 作用与物理意义 |
| :--- | :--- | :--- | :--- | :--- |
| **m = 0** | 0° | 0° | 0° | 起始零点 |
| **m = 1** | **57.3°** | 2.66° | 0.123° | 平面 0 变化显著，敏锐捕捉**微观相邻语法依赖** |
| **m = 2** | **114.6°** | 5.32° | 0.246° | 高频子空间迅速变动，准确分辨词序先后 |
| **m = 6** | **343.8° ≈ 360°** | 15.96° | 0.74° | 平面 0 转满一圈，进入周期性重叠 |
| **m = 100** | 旋转约 16 圈（高频模糊） | **266°** | **12.3°** | 平面 1 和 2 稳定转动，接力提供辨识度 |
| **m = 1000** | 高度混叠 | 旋转约 7 圈 | **123°** | **仅平面 2 未满一圈**，单调递增区分长程上下文 |

---

### 3.3 高维旋转计算数值演示（$d=4$）

设 $d = 4$（拆为两个 2D 平面：平面 0 为前两维，平面 1 为后两维）。
* 平面 0 设 $\theta_0 = 90^\circ$
* 平面 1 设 $\theta_1 = 30^\circ$
* 输入向量：$q = [1, 0, 0, 2]^\top$
* 目标位置：$m = 1$

#### 构造分块旋转矩阵
在 $m = 1$ 时：
* 平面 0 旋转角 $90^\circ$：$\cos 90^\circ = 0, \sin 90^\circ = 1$
* 平面 1 旋转角 $30^\circ$：$\cos 30^\circ = \frac{\sqrt{3}}{2} \approx 0.866, \sin 30^\circ = 0.5$

$$\boldsymbol{R}_1 = \begin{pmatrix} 0 & -1 & 0 & 0 \cr 1 & 0 & 0 & 0 \cr 0 & 0 & 0.866 & -0.5 \cr 0 & 0 & 0.5 & 0.866 \end{pmatrix}$$

#### 矩阵乘法求解 $\tilde{q}_1 = \boldsymbol{R}_1 q$

$$\tilde{q}_1 = \begin{pmatrix} 0 & -1 & 0 & 0 \cr 1 & 0 & 0 & 0 \cr 0 & 0 & 0.866 & -0.5 \cr 0 & 0 & 0.5 & 0.866 \end{pmatrix} \begin{pmatrix} 1 \cr 0 \cr 0 \cr 2 \end{pmatrix}$$

分块并行运算：
* **前两维（平面 0）**：

$$\begin{pmatrix} 0 \times 1 + (-1) \times 0 \cr 1 \times 1 + 0 \times 0 \end{pmatrix} = \begin{pmatrix} 0 \cr 1 \end{pmatrix}$$

* **后两维（平面 1）**：

$$\begin{pmatrix} 0.866 \times 0 + (-0.5) \times 2 \cr 0.5 \times 0 + 0.866 \times 2 \end{pmatrix} = \begin{pmatrix} -1 \cr 1.732 \end{pmatrix}$$

**输出结果**：

$$\tilde{q}_1 = \begin{pmatrix} 0 \cr 1 \cr -1 \cr 1.732 \end{pmatrix}$$

---

## 4. 工程落地与计算加速

为避免显存占用和 $O(d^2)$ 的稀疏矩阵乘法开销，在 PyTorch 与 CUDA 算子实现中，统一改用**向量逐元素相乘（Hadamard 积 $\odot$）**。

将每个二维对的计算：

$$\begin{pmatrix} \tilde{x}_{2i} \cr \tilde{x}_{2i+1} \end{pmatrix} = \begin{pmatrix} x_{2i}\cos(m\theta_i) - x_{2i+1}\sin(m\theta_i) \cr x_{2i}\sin(m\theta_i) + x_{2i+1}\cos(m\theta_i) \end{pmatrix}$$

向量化重写为：

$$\tilde{x}_m = x \odot \cos\_vec + x_{rot} \odot \sin\_vec$$

其中：
* `x_rot` = $[-x_1, x_0, -x_3, x_2, \dots, -x_{d-1}, x_{d-2}]$
* `cos_vec` = $[\cos m\theta_0, \cos m\theta_0, \dots, \cos m\theta_{\frac{d}{2}-1}, \cos m\theta_{\frac{d}{2}-1}]$
* `sin_vec` = $[\sin m\theta_0, \sin m\theta_0, \dots, \sin m\theta_{\frac{d}{2}-1}, \sin m\theta_{\frac{d}{2}-1}]$

**计算复杂度**：显存占用和时间复杂度均为 $O(d)$。

---

## 5. 关键特性总结

| 核心特性 | 机制说明 | 带来的实际收益 |
| :--- | :--- | :--- |
| **范数保持（Norm Preserving）** | 旋转矩阵属于正交矩阵（$\Vert \boldsymbol{R}_m x \Vert = \Vert x \Vert$） | 不会放大或缩小词向量的模长，保障注意力分布计算数值稳定 |
| **远程衰减（Long-term Decay）** | 多频震荡波叠加干涉 | 相对距离越远，内积期望值趋于衰减，契合语言距离越远关联越弱的先验 |
| **相对位置内积不变性** | 矩阵乘法角度抵消：$\boldsymbol{R}_m^\top \boldsymbol{R}_n = \boldsymbol{R}_{n - m}$ | 无论在句首还是句尾，相同句式的相对注意力分数严格一致 |
| **长度外推与插值潜力** | 基于角频率 $\theta_i$ 参数化控制 | 可方便引入 NTK-Aware、Linear Scaling、YaRN 等位置外推算法扩展模型上下文 |
