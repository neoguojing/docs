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
## 自动微分
## 模块化神经网络
## 损失函数
## 优化器
## 数据加载
## 设备管理
## 保存与加载模型
