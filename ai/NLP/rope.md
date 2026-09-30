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
* 初始向量： $q = [1, 0]^\top$（模长 1，角度 $0^\circ$）， $k = [0, 1]^\top$ （模长 1，角度 $90^\circ$ ）

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
* **平面 0（ $i = 0$ ，高频/秒针）**：

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

### 3.3 高维旋转计算数值演示（ $d=4$ ）

设 $d = 4$ （拆为两个 2D 平面：平面 0 为前两维，平面 1 为后两维）。
* 平面 0 设 $\theta_0 = 90^\circ$
* 平面 1 设 $\theta_1 = 30^\circ$
* 输入向量： $q = [1, 0, 0, 2]^\top$
* 目标位置： $m = 1$

#### 构造分块旋转矩阵
在 $m = 1$ 时：
* 平面 0 旋转角 $90^\circ$ ： $\cos 90^\circ = 0, \sin 90^\circ = 1$
* 平面 1 旋转角 $30^\circ$ ： $\cos 30^\circ = \frac{\sqrt{3}}{2} \approx 0.866, \sin 30^\circ = 0.5$

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

为了**彻底解决**某些 Markdown 渲染器对下划线 `_` 极其严格的报错，最一劳永逸的方法是：**在数学公式中放弃使用代码风格的下划线命名法（如 `cos_vec`），而是改用标准的数学下标表示法（如 $v_{\cos}$ 和 $v_{\sin}$）。**

这样不仅绝对不会报错，而且公式看起来更加专业美观。下面是为您格式化并彻底消除下划线报错的纯净版模拟过程：

---

### 1. $m$ 和角度 $\theta$ 到底是如何取值的？

在真实的 Transformer 中，特征向量的维度 $d$ 通常很大（例如 $d=4096$），我们会把这 4096 个数字**两两分组**，分成 $4096 / 2 = 2048$ 个二维平面。

* **$m$（绝对位置）：** 代表当前 Token 在句子中的位置索引（从 0 开始）。
* 第一词：“我”，$m = 0$
* 第二词：“爱”，$m = 1$
* 第三词：“你”，$m = 2$


* **$\theta_i$（基础角度）：** 代表第 $i$ 组二维平面固定的旋转频率。通常通过一个以 10000 为底的指数公式自动计算：

$$\theta_i = 10000^{\frac{-2i}{d}}$$



*其中 $i$ 是分组的序号（$0, 1, 2 \dots \frac{d}{2}-1$）。*

**举个真实的例子（假设维度 $d=4$，有两组二维平面）：**

* **第 0 组** ($i=0$)：$\theta_0 = 10000^0 = 1$ （即 1 弧度，约 $57.3^\circ$）
* **第 1 组** ($i=1$)：$\theta_1 = 10000^{-2/4} = 10000^{-0.5} = 0.01$ （即 0.01 弧度，约 $0.57^\circ$）

**最终的实际旋转角度 $= m \times \theta_i$**
这就是为什么叫做“位置编码”：同一个二维平面，Token 所在的位置 $m$ 越靠后，转过的总角度 $m\theta_i$ 就越大，模型就能借此感知到词与词的距离！

---

### 2. 设定一个“方便手算”的数值场景

真实的 1 弧度和 0.01 弧度计算正余弦会得到无限不循环小数，无法直观演示。为了让您看清矩阵乘法向 Hadamard 积（$\odot$）转化的巧妙过程，我们人为设定两个整角来进行模拟演算：

* **向量维度：** $d=4$
* **输入向量：** $x = [1, 2, 3, 4]$
* **当前 Token 所在位置：** $m = 1$ （代表这句话的第 2 个词）
* **假设的基准角度：**
* 第一组 ($i=0$) 的总角度设定为 $m\theta_0 = 90^\circ$（即 $\pi/2$）
* 第二组 ($i=1$) 的总角度设定为 $m\theta_1 = 180^\circ$（即 $\pi$）



---

### 3. 构建 O(d) 复杂度的 1 维向量

我们不需要构建庞大且稀疏的二维旋转矩阵，而是直接在 PyTorch 中生成 4 个一维数组（向量）：

**① 原始向量 $x$**：


$$x = [1, 2, 3, 4]$$

**② 错位取反向量 $x_{rot}$**：
规则是：相邻两个元素互换位置，并将前一个取负号，即 $[-x_1, x_0, -x_3, x_2]$。


$$x_{rot} = [-2, 1, -4, 3]$$

**③ 余弦向量 $v_{\cos}$**：
将每个平面的 $\cos(m\theta_i)$ 复制两遍凑成维度 $d$。


$$v_{\cos} = [\cos(90^\circ), \cos(90^\circ), \cos(180^\circ), \cos(180^\circ)] = [0, 0, -1, -1]$$

**④ 正弦向量 $v_{\sin}$**：
将每个平面的 $\sin(m\theta_i)$ 复制两遍凑成维度 $d$。


$$v_{\sin} = [\sin(90^\circ), \sin(90^\circ), \sin(180^\circ), \sin(180^\circ)] = [1, 1, 0, 0]$$

---

### 4. 执行向量化计算 (Hadamard 积 $\odot$)

在底层 CUDA 算子中，RoPE 被统一重写为极简公式：


$$\tilde{x}_m = x \odot v_{\cos} + x_{rot} \odot v_{\sin}$$

**步骤 A：逐元素计算 $x \odot v_{\cos}$**


$$\begin{bmatrix} 1 \\ 2 \\ 3 \\ 4 \end{bmatrix} \odot \begin{bmatrix} 0 \\ 0 \\ -1 \\ -1 \end{bmatrix} = \begin{bmatrix} 0 \\ 0 \\ -3 \\ -4 \end{bmatrix}$$

**步骤 B：逐元素计算 $x_{rot} \odot v_{\sin}$**


$$\begin{bmatrix} -2 \\ 1 \\ -4 \\ 3 \end{bmatrix} \odot \begin{bmatrix} 1 \\ 1 \\ 0 \\ 0 \end{bmatrix} = \begin{bmatrix} -2 \\ 1 \\ 0 \\ 0 \end{bmatrix}$$

**步骤 C：两者相加，得到加入位置信息后的输出 $\tilde{x}_m$**


$$\tilde{x}_m = \begin{bmatrix} 0 \\ 0 \\ -3 \\ -4 \end{bmatrix} + \begin{bmatrix} -2 \\ 1 \\ 0 \\ 0 \end{bmatrix} = \begin{bmatrix} -2 \\ 1 \\ -3 \\ -4 \end{bmatrix}$$

> **验证其等价性：**
> 如果我们用传统 $2 \times 2$ 矩阵旋转去算：
> 第 0 组平面 `[1, 2]` 转 $90^\circ$，落在横轴负半区、纵轴正半区，变为 `[-2, 1]`。
> 第 1 组平面 `[3, 4]` 转 $180^\circ$，相当于绕原点对称，变为 `[-3, -4]`。
> 拼合后恰好也是 `[-2, 1, -3, -4]`。
> **结论证明了：**通过 $x_{rot}$ 与 $v_{\sin}$、$v_{\cos}$ 的逐元素乘法，不仅绕开了 $O(d^2)$ 的庞大显存占用，更利用 GPU 的一维数组对齐乘加操作（FMA），实现了完美的等价替代。
---

## 5. 关键特性总结

| 核心特性 | 机制说明 | 带来的实际收益 |
| :--- | :--- | :--- |
| **范数保持（Norm Preserving）** | 旋转矩阵属于正交矩阵（ $\Vert \boldsymbol{R}_m x \Vert = \Vert x \Vert$ ） | 不会放大或缩小词向量的模长，保障注意力分布计算数值稳定 |
| **远程衰减（Long-term Decay）** | 多频震荡波叠加干涉 | 相对距离越远，内积期望值趋于衰减，契合语言距离越远关联越弱的先验 |
| **相对位置内积不变性** | 矩阵乘法角度抵消：`R_m^T * R_n = R_{n - m}` | 无论在句首还是句尾，相同句式的相对注意力分数严格一致 |
| **长度外推与插值潜力** | 基于角频率 $\theta_i$ 参数化控制 | 可方便引入 NTK-Aware、Linear Scaling、YaRN 等位置外推算法扩展模型上下文 |

在涉及 **RoPE（旋转位置编码）** 的面试、研讨或深度技术交流中，开发者和面试官最常追问的核心问题通常集中在以下几个方面：

---

### 核心原理与数学推导

1. **既然 RoPE 叫“相对位置编码”，为什么实现时却直接乘在绝对位置 $m$ 的 $q_m$ 和 $k_n$ 上？**
* **考点**：考察是否理解“绝对操作实现相对效果”。
* **核心要点**：利用正交旋转矩阵的代数性质：

$$
\boldsymbol{R}_m^\top \boldsymbol{R}_n = \boldsymbol{R}_{n - m}
$$

在计算 Query 与 Key 的内积 $\tilde{q}_m^\top \tilde{k}_n$ 时，绝对位置 $m$ 和 $n$ 自然相减抵消，仅保留相对位移差 $(n - m)$。


2. **为什么 RoPE 只对 Query 和 Key 施加旋转，而不对 Value 向量施加？**
* **考点**：Attention 的权重计算机制与信息聚合。
* **核心要点**：位置信息是为了计算 token 之间的**相关度权重（Attention Map）**。一旦权重矩阵通过 $q$ 和 $k$ 算出了相对位置关联，再去旋转 $v$ 不仅多余，还会破坏 $v$ 聚合加权求和后的线性空间表达（旋转不具备对加法的线性分配律）。


3. **RoPE 是如何实现“远程衰减（Long-term Decay）”特性的？**
* **考点**：多频震荡与数学性质。
* **核心要点**：由于基底底数（如 10000）将不同的特征切片映射到了不同周期的旋转频率 $\theta_i$。在内积求和时，高频与低频分量相互干涉（类似于黎曼-勒贝格引理），随着相对距离 $\vert{}m - n\vert{}$ 增大，内积期望值天然具有衰减趋势。



---

### 上下文长度外推与扩展（Long Context）

4. **直接把预训练 4k 上下文的模型拿到 32k 上推理，RoPE 会遇到什么问题？**
* **考点**：位置外推失效原因（Out-of-Distribution）。
* **核心要点**：高频分量旋转圈数过多导致相位混叠，而超低频分量在预训练时只转了极小角度（例如只经历过 $0^\circ \sim 20^\circ$ ），当位置拉长到 32k 时，低频分量见到了从未见过的绝对角度，注意力机制无法正确建模。


5. **Linear Scaling（位置线性插值）和 NTK-Aware Scaled RoPE 的区别是什么？**
* **考点**：长文本扩展方案的演进。
* **核心要点**：
* **Linear Scaling**：将位置索引 $m$ 整体缩放（ $m \to m/s$ ）。缺点是对所有频率一刀切，高频分量被过度压缩，损害近距离精确语法建模。
* **NTK-Aware**：不压缩位置索引，而是通过**缩放基底底数 base**（例如将 10000 放大到 500000），让高频分量尽量少插值（保持局部精度），对低频分量大幅插值（扩展全局视野）。





---

### 工程实现与性能优化

6. **为什么 RoPE 代码实现中不用构建 $d \times d$ 的稀疏分块旋转矩阵？**
* **考点**：计算复杂度与算子优化。
* **核心要点**：分块对角矩阵大部分为 0，直接做矩阵乘法复杂度为 $O(d^2)$。工程上改用向量的逐元素相乘（Hadamard Product）与交错维度翻转（`rotate_half`），将显存和时间复杂度降到线性的  $O(d)$ 。


7. **在 FlashAttention 或 KV Cache 中，RoPE 是在什么节点执行的？**
* **考点**：推理流水线与缓存管理。
* **核心要点**： $q$ 和 $k$ 经由线性层（Linear Projection）映射出来后，立即应用 RoPE。将**已经旋转过的 $k$** 存入 KV Cache。这样后续生成新的 token 时，直接从 Cache 中取出现成的 $k$ 即可，无需重复计算。



---
