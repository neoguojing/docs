# pytorch

## 张量
- 任何张量的最后一维，都可以理解为「行的长度（每一行的元素个数）
- 向量写法 (3,) 是最合理的写法，因为它忠实反映了数据的维度和结构：
--它不是二维矩阵
--它只有一个轴
--这个轴的长度是 3
### 广播
- 广播就是让“缺省/维度为 1”的地方自动扩展，匹配另一个张量的形状，从而能做逐元素计算
- 如果两个维度相同 → 直接匹配。
- 如果某个维度是 1 → 就会广播（扩展成另一个的大小）。
- 其他情况 → 报错（形状不兼容）。
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
- vmap (vectorized map) = 把一个作用在单个样本上的函数，自动扩展成能在 batch 上并行计算的函数。
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
- nn.Module 内部通过 _modules 管理子模块
- 组合方式：
-- 顺序：nn.Sequential
-- 列表：nn.ModuleList
-- 字典：nn.ModuleDict
- 优势：
-- 参数递归管理
-- forward 递归调用
-- state_dict 自动保存子模块参数
-- 可复用、可扩展
### 前向传播由 forward
### 内置网络层
#### 全连接层：nn.Linear
```
nn.Linear 内部就是用 torch.nn.functional.linear 实现的矩阵乘法。
本质就是 torch.matmul(x, W.T) + b
参数 W 和 b 会自动注册为模型参数，并且能被优化器更新。
```
- 卷积层：nn.Conv 支持1-3维的卷积
- 池化层：
- 激活函数
- 正则化层：LayerNorm
- Dropout：随机失活
- 注意力，LSTM等
#### nn.Embedding 是一个 可训练的查找表，把离散的整数 id 转换成稠密向量，在训练过程中不断调整 embedding，使其更好地表达这些 id 的语义或特征
```
nn.Embedding(
    num_embeddings,   # 词表大小，比如 50000
    embedding_dim,    # 向量维度，比如 768
    padding_idx=None, # 如果设置，这个 index 的 embedding 永远是 0
    max_norm=None,    # 如果设置，embedding 会被约束在最大范数内
    norm_type=2.0,    # max_norm 的范数类型 (默认 L2 范数)
    scale_grad_by_freq=False, # 是否根据词频缩放梯度（稀有词梯度更大）
    sparse=False      # 是否使用稀疏更新（节省内存，适合大词表）
)
```
## 损失函数
> transfomer的损失函数多是CrossEntropyLoss
> 自监督学习：将预测的token和实际训练数据的token做输入，进行损失计算
| 类型    | 函数                  | 适用场景     | 输入特点                    |
| ----- | ------------------- | -------- | ----------------------- |
| 分类    | CrossEntropyLoss    | 多分类      | logits, label           |
| 分类    | BCEWithLogitsLoss   | 二分类/多标签  | logits, 0/1             |
| 分类    | NLLLoss             | 多分类      | log-prob, label         |
| 回归    | MSELoss             | 回归       | y\_pred, y\_true        |
| 回归    | L1Loss              | 回归       | y\_pred, y\_true        |
| 回归    | SmoothL1Loss        | 回归/异常值鲁棒 | y\_pred, y\_true        |
| 对比/嵌入 | CosineEmbeddingLoss | 相似度学习    | x1, x2, label           |
| 对比/嵌入 | TripletMarginLoss   | 嵌入学习     | anchor, pos, neg        |
| 分布    | KLDivLoss           | 分布匹配/蒸馏  | log\_prob, target\_prob |

## 优化器
> 优化器负责 根据梯度更新模型参数，是训练神经网络的核心
> PyTorch 中优化器位于 torch.optim 模块，常用类继承自 Optimizer 基类
```
optimizer = torch.optim.Adam(model.parameters(), lr=1e-3)

for x, y in dataloader:
    optimizer.zero_grad()         # 清空梯度
    y_pred = model(x)
    loss = loss_fn(y_pred, y)
    loss.backward()               # 反向传播计算梯度
    optimizer.step()              # 更新参数

```
### 核心工作：
- 接收模型参数
- 根据梯度和优化算法更新参数
- 支持动量、权重衰减、学习率调度等
### 核心成员：
- param_groups：参数组，每组可设置不同学习率或权重衰减
- state：存储优化器状态（如动量、Adam 的一阶/二阶矩）
### 核心方法：
- step()：执行一次参数更新
- zero_grad()：清空梯度（一般在反向传播前调用）
### 核心流程：
- 遍历 param_groups
- 对每个参数 p：
-- 获取梯度 g = p.grad
-- 更新状态 state[p]
-- 计算参数更新量 delta：参数更新量 = 学习率 × 梯度 × 其他修正因子（如动量、自适应系数）
-- 执行 p.data.add_(delta)：等价于 p.data = p.data + delta；
### 常见优化器
| 优化器          | 特点             | 公式（参数更新）                                                                                                                                                                                                                                          |
| ------------ | -------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| SGD          | 随机梯度下降，简单，收敛慢  | $\theta_{t+1} = \theta_t - \eta \nabla_\theta L$                                                                                                                                                                                                  |
| SGD+Momentum | 引入动量，缓冲梯度      | $v_{t} = \mu v_{t-1} + \nabla_\theta L, \theta_{t+1} = \theta_t - \eta v_t$                                                                                                                                                                       |
| SGD+Nesterov | 提前梯度更新         | $v_{t} = \mu v_{t-1} + \nabla_\theta L(\theta_t - \eta \mu v_{t-1})$                                                                                                                                                                              |
| Adam         | 自适应学习率，一阶矩和二阶矩 | $ m_t = \beta_1 m_{t-1} + (1-\beta_1) g_t$ <br> $v_t = \beta_2 v_{t-1} + (1-\beta_2) g_t^2$ <br> $\hat{m}_t = m_t/(1-\beta_1^t)$, $\hat{v}_t = v_t/(1-\beta_2^t)$ <br> $\theta_{t+1} = \theta_t - \eta \hat{m}_t / (\sqrt{\hat{v}_t} + \epsilon)$ |
| AdamW        | Adam + 正则化分离   | 权重衰减直接作用于参数而不是梯度                                                                                                                                                                                                                                  |
| RMSprop      | 自适应学习率         | $E[g^2]_t = \gamma E[g^2]_{t-1} + (1-\gamma) g_t^2$ <br> $\theta_{t+1} = \theta_t - \eta g_t / (\sqrt{E[g^2]_t} + \epsilon)$                                                                                                                      |

### 概念：
| 概念       | 作用         | PyTorch 表达           |
| -------- | ---------- | -------------------- |
| 学习率 η    | 控制更新步长     | lr=0.01              |
| 动量 μ     | 利用历史梯度平滑更新 | momentum=0.9         |
| Nesterov | 提前修正梯度方向   | nesterov=True        |
| 自适应学习率   | 对不同参数调节步长  | Adam / RMSprop       |
| 权重衰减 λ   | 防止过拟合      | weight\_decay=0.01   |
| 梯度裁剪     | 防止梯度爆炸     | clip\_grad\_norm\_   |
| 梯度累积     | 多步累积梯度再更新  | 手动控制 backward / step |

## 数据加载
### Dataset
- 抽象类：torch.utils.data.Dataset
- 核心方法：
```
__len__()     # 返回样本数量
__getitem__(idx)  # 返回 idx 对应的样本 (x, y)
```
### DataLoader
- 作用：批量读取、打乱顺序、并行加载
- 常用参数：
-- batch_size：每个 batch 样本数量
-- shuffle：是否打乱顺序
-- num_workers：多进程加载数量
-- collate_fn：自定义 batch 合并方法
## 设备管理
- CUDACachingAllocator 管理gpu内存
## 自定义算子
```
import torch

class SwishFn(torch.autograd.Function):
    @staticmethod
    def forward(ctx, x):
        sig = x.sigmoid()
        y = x * sig
        ctx.save_for_backward(x, sig)   # 反向会用到
        return y

    @staticmethod
    def backward(ctx, grad_y):
        x, sig = ctx.saved_tensors
        grad_x = grad_y * (sig + x * sig * (1 - sig))
        return grad_x

def swish(x):
    return SwishFn.apply(x)

# 使用 & 梯度检验（双精度 + 小扰动）
x = torch.randn(5, requires_grad=True, dtype=torch.double)
torch.autograd.gradcheck(SwishFn.apply, (x,), eps=1e-6, atol=1e-4, rtol=1e-3)

```
