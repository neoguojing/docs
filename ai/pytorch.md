# pytorch

## 张量
- 任何张量的最后一维，都可以理解为「行的长度（每一行的元素个数）
- 向量写法 (3,) 是最合理的写法，因为它忠实反映了数据的维度和结构：
--它不是二维矩阵
--它只有一个轴
--这个轴的长度是 3
### 三要素
- shape（形状）：各维度长度，如 (B, C, H, W)。
- dtype（数据类型）：如 float32、bfloat16、int64、bool、complex64 等。
- device（设备）：cpu 或 cuda（GPU）。
### 底层实现
- PyTorch 的核心是 ATen 库（C++），所有张量操作最终调用 ATen 的函数。
- Tensor 在 C++ 层是一个 TensorImpl 对象，Python 只是一个封装（PyObject）。
- Tensor 的数据和元信息分开管理：
-- 数据（storage）：连续内存块，存储实际数字（float32, int64 等）
-- 元信息（TensorImpl）：shape、stride、dtype、device、requires_grad 等
### contiguous
- 由于底层存储是连续内存，大部分情况下知识改变了元数据
- 某些情况下，CUDA等要求内存是连续的
- contiguous 重新分配内存和元素重排
- is_contiguous 判断内存是否连续
### 基于轴的操作
- 沿着 axis 的方向“压扁”它，剩下的方向就保留下来
- 矩阵：axis=0，即沿着行的方向（上下）压缩，axis=1，沿着列的方向压缩
### 张量的算子，对张亮进行的数学运算
- +, -, *, / 对张量元素做逐元素运算
- matmul, t()做矩阵乘法或转置
- sum, mean, max对张量沿指定维度求和、
- reshape： 改变shape，总元素不变
- transpose： 转置，和.T一致
- squeeze：删除张量中长度为1的维度
- unsqueeze：在指定位置增加长度度为1的纬度
- 逻辑/比较运算>, <, ==
- 激活函数ReLU, Sigmoid, Tanh非线性映射，用于神经网络
- 索引/切片x[0,:], gather获取张量的子集或重排
- gather → 类似“根据索引表挑选元素”，特别适合 batch 处理或神经网络输出重排
```
x[0, :]  # 第0行 → tensor([1,2,3])
x[:, 1]  # 第1列 → tensor([2,5,8])
x[1:3, 0:2]  # 第1、2行，第0、1列 → tensor([[4,5],[7,8]])
```
## 自动微分张量 
- tensor 的属性 requires_grad=True 表示需要跟踪梯度。
- 张量 .grad 存储了该张量的梯度（在反向传播后）。
- 导数：就是函数变化的“斜率”或“敏感度“
- 链式法则：输出对输入的变化率 = 输出对中间变量的变化率 × 中间变量对输入的变化率
- 多元函数的偏导数：多元函数即输入有多个，偏导数：表示输出对某一个输入的变化率
- 梯度（Gradient）**是所有偏导数组成的向量
- 残差连接 = 恒等梯度 + 非线性梯度 保证深层网络梯度不消失 这是 Transformer 可以训练非常深层的根本原因
- 损失函数输出：总是一个标量，衡量整个样本或 batch 的误差
- 损失函数对输出求导：得到与网络输出相同形状的梯度，向量或者矩阵
- 对 输入 X 求导 → 保证梯度能继续往前传
- 对 参数 W 求导 → 保证参数能更新
- 参数更新：通常是用w-学习率*w的导数
### 参数和方法
- requires_grad	是否追踪梯度
- grad	存储梯度值
- backward()	反向传播计算梯度
- - create_graph=True 让反向传播也构建计算图，从而支持二阶导数
  - 默认 retain_graph=False，会释放中间节点，节省内存
- no_grad() / detach()	停止追踪，节省内存，常用于推理
### 计算图与求导过程
> 在前向计算时动态构建计算图（记录每个算子的输入输出关系）。
> 在反向传播时，基于链式法则逐节点调用反向函数，把梯度从输出传播到输入。
> 底层由 C++ Autograd Engine + ATen 张量库 驱动，Python 只是调用接口。
- 每个操作（如加、乘、矩阵乘法）都会在图中形成一个节点。
- 图的叶子节点是原始张量（requires_grad=True）：Tensor
- 每个 Tensor 有一个 .grad_fn 指针，指向它是由哪个函数计算出来的
- 图的边是函数：封装在 Function 对象中
- 输出节点是最终的函数值。
- backward 从输出节点开始，递归调用每个节点的 backward() 函数；而每个Function都实现了前向和后向函数
- 累加梯度到后面的节点的.grad
### 矩阵求导原理：
- Jacobian 矩阵，输出对每个矩阵元素求导组成的矩阵，计算量和存储量大
- 向量-雅可比积 (Vector-Jacobian Product, VJP)：拿一行权重（上游梯度）去乘这个表格，只算需要的那一部分
## 模块化神经网络
### nn.Module
- 参数注册：通过 nn.Parameter 自动加入 _parameters
- 子模块注册：通过 _modules 管理，支持递归访问
- 前向传播：forward() 用户定义，__call__ 负责调用并处理 hooks
- 缓冲区：register_buffer 管理非训练张量
- 训练模式管理：self.training 控制 Dropout/BatchNorm 行为
- 模块序列化：state_dict() / load_state_dict() 保存和加载权重
-- 收集本模块参数
-- 收集 buffer
-- 递归收集子模块
-- OrderedDict：保持层级顺序，方便加载到原模型
  
### 子模块组合
- 子模块本质上是 另一个 Module 实例
- 
### 自动管理参数
### 前向传播由 forward
### 内置网络层
- 全连接层：nn.Linear
- 卷积层：nn.Conv 支持1-3维的卷积
- 池化层：
- 激活函数
- 正则化层：LayerNorm
- Dropout：随机失活
- 注意力，LSTM等
## 损失函数
## 优化器
## 数据加载
## 设备管理
## 保存与加载模型
