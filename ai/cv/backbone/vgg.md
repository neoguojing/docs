# 骨干网路
> Backbone（骨干网络）是负责从输入图像中提取语义特征的基础网络，其本质是将高维的“空间像素”逐步压缩，并转化为深度的“通道特征”。面试的核心考点集中在架构设计的演进动机、计算复杂度（FLOPs/参数量）的量化拆解，以及特征张量在网络中的流转与对齐。

# VGG 网络详解

## 一、VGG 网络的 3 个核心特性

### 1. 小卷积核的绝对统治

彻底摒弃了 AlexNet 中的 $11\times11$ 和 $5\times5$ 大卷积核，整个网络的所有卷积层全都是 $3\times3$ 尺寸。

### 2. "维度不变"与"尺寸减半"的规律交替

- 所有的 $3\times3$ 卷积层都设置了 padding=1 和 stride=1，这保证了卷积操作不改变图像的空间尺寸，只负责加深通道。
- 所有的降采样都交给 $2\times2$（stride=2）的 MaxPool 最大池化层，这保证了每次池化后，图像空间尺寸精确缩小一半。

### 3. 通道数翻倍策略

每经过一次池化层（图像尺寸减半），下一个阶段的卷积核通道数就会翻倍（64 → 128 → 256 → 512），完美诠释了"空间压缩，通道膨胀"的设计美学。

---

## 二、维度计算通用公式（代码注释的依据）

在看代码前，牢记决定图像宽/高（$W_{out}$）变化的通用公式：

$$W_{out} = \left\lfloor \frac{W_{in} - K + 2P}{S} \right\rfloor + 1$$

其中：

- $W_{in}$: 输入尺寸
- $K$: 卷积核大小 (Kernel Size)
- $P$: 填充 (Padding)
- $S$: 步长 (Stride)

**VGG 的标准操作：**

- **VGG 卷积**：$K=3, P=1, S=1$ → $W_{out} = (W_{in} - 3 + 2)/1 + 1 = W_{in}$（尺寸不变）
- **VGG 池化**：$K=2, P=0, S=2$ → $W_{out} = (W_{in} - 2 + 0)/2 + 1 = W_{in}/2$（尺寸减半）

---

## 三、VGG-16 PyTorch 源码实现与硬核维度追踪

以下是经典的 VGG-16 架构（包含 13 个卷积层 + 3 个全连接层）。代码按网络的前向传播阶段进行了模块化拆解。

```python
import torch
import torch.nn as nn

class VGG16(nn.Module):
    def __init__(self, num_classes=1000):
        super(VGG16, self).__init__()
        
        # -----------------------------------------
        # Block 1: 提取极浅层特征 (边缘、颜色)
        # 输入: [B, 3, 224, 224]
        # -----------------------------------------
        self.block1 = nn.Sequential(
            nn.Conv2d(3, 64, kernel_size=3, padding=1),
            nn.ReLU(inplace=True),
            nn.Conv2d(64, 64, kernel_size=3, padding=1),
            nn.ReLU(inplace=True),
            nn.MaxPool2d(kernel_size=2, stride=2)
        )
        
        # -----------------------------------------
        # Block 2: 提取浅层特征 (纹理)
        # -----------------------------------------
        self.block2 = nn.Sequential(
            nn.Conv2d(64, 128, kernel_size=3, padding=1),
            nn.ReLU(inplace=True),
            nn.Conv2d(128, 128, kernel_size=3, padding=1),
            nn.ReLU(inplace=True),
            nn.MaxPool2d(kernel_size=2, stride=2)
        )
        
        # -----------------------------------------
        # Block 3: 提取中层特征 (简单形状、部件)
        # 注意：从这里开始，每组包含 3 个卷积层
        # -----------------------------------------
        self.block3 = nn.Sequential(
            nn.Conv2d(128, 256, kernel_size=3, padding=1),
            nn.ReLU(inplace=True),
            nn.Conv2d(256, 256, kernel_size=3, padding=1),
            nn.ReLU(inplace=True),
            nn.Conv2d(256, 256, kernel_size=3, padding=1),
            nn.ReLU(inplace=True),
            nn.MaxPool2d(kernel_size=2, stride=2)
        )
        
        # -----------------------------------------
        # Block 4: 提取高层特征 (复杂形状、对象)
        # -----------------------------------------
        self.block4 = nn.Sequential(
            nn.Conv2d(256, 512, kernel_size=3, padding=1),
            nn.ReLU(inplace=True),
            nn.Conv2d(512, 512, kernel_size=3, padding=1),
            nn.ReLU(inplace=True),
            nn.Conv2d(512, 512, kernel_size=3, padding=1),
            nn.ReLU(inplace=True),
            nn.MaxPool2d(kernel_size=2, stride=2)
        )
        
        # -----------------------------------------
        # Block 5: 提取全局高层语义
        # -----------------------------------------
        self.block5 = nn.Sequential(
            nn.Conv2d(512, 512, kernel_size=3, padding=1),
            nn.ReLU(inplace=True),
            nn.Conv2d(512, 512, kernel_size=3, padding=1),
            nn.ReLU(inplace=True),
            nn.Conv2d(512, 512, kernel_size=3, padding=1),
            nn.ReLU(inplace=True),
            nn.MaxPool2d(kernel_size=2, stride=2)
        )
        
        # -----------------------------------------
        # 分类器 (Classifier): 全连接层
        # -----------------------------------------
        self.classifier = nn.Sequential(
            # 将展平的一维向量映射到 4096 维隐藏层
            nn.Linear(512 * 7 * 7, 4096),
            nn.ReLU(inplace=True),
            nn.Dropout(p=0.5), # 防止过拟合
            
            nn.Linear(4096, 4096),
            nn.ReLU(inplace=True),
            nn.Dropout(p=0.5),
            
            # 映射到最终的类别数
            nn.Linear(4096, num_classes)
        )

    def forward(self, x):
        # 假设输入 x 的形状: [Batch, 3, 224, 224]
        
        # 1. 经过 Block 1
        # Conv作用: (224-3+2)/1+1 = 224。通道变64。形状: [B, 64, 224, 224]
        # Pool作用: (224-2)/2+1 = 112。形状减半。
        x = self.block1(x) 
        # 当前 x 形状: [B, 64, 224, 224] -> [B, 64, 112, 112]
        
        # 2. 经过 Block 2
        # Conv作用: 空间不变，通道扩充至 128。
        # Pool作用: (112-2)/2+1 = 56。
        x = self.block2(x)
        # 当前 x 形状: [B, 128, 56, 56]
        
        # 3. 经过 Block 3
        # Conv作用: 空间不变，通道扩充至 256。
        # Pool作用: (28-2)/2+1 = 28。
        x = self.block3(x)
        # 当前 x 形状: [B, 256, 28, 28]
        
        # 4. 经过 Block 4
        # Conv作用: 空间不变，通道扩充至 512。
        # Pool作用: (28-2)/2+1 = 14。
        x = self.block4(x)
        # 当前 x 形状: [B, 512, 14, 14]
        
        # 5. 经过 Block 5
        # Conv作用: 空间不变，通道保持 512。
        # Pool作用: (14-2)/2+1 = 7。
        x = self.block5(x)
        # 当前 x 形状: [B, 512, 7, 7]
        
        # 6. 展平 (Flatten)
        # 将三维的特征图 [512, 7, 7] 拉直成一维向量，用来输入全连接层
        # 512 * 7 * 7 = 25088
        x = torch.flatten(x, 1)
        # 当前 x 形状: [B, 25088]
        
        # 7. 全连接分类
        x = self.classifier(x)
        # 当前 x 形状: [B, num_classes] (例如 [B, 1000])
        
        return x
```

---

## 四、VGG-16 维度追踪速查表

| 阶段 | 操作 | 输入形状 | 输出形状 | 维度变化说明 |
|------|------|----------|----------|-------------|
| 输入 | — | [B, 3, 224, 224] | [B, 3, 224, 224] | 原始图像 |
| Block 1 | Conv×2 + ReLU×2 | [B, 3, 224, 224] | [B, 64, 224, 224] | 通道 3→64，尺寸不变 |
| Block 1 | MaxPool | [B, 64, 224, 224] | [B, 64, 112, 112] | 尺寸减半 |
| Block 2 | Conv×2 + ReLU×2 | [B, 64, 112, 112] | [B, 128, 112, 112] | 通道 64→128，尺寸不变 |
| Block 2 | MaxPool | [B, 128, 112, 112] | [B, 128, 56, 56] | 尺寸减半 |
| Block 3 | Conv×3 + ReLU×3 | [B, 128, 56, 56] | [B, 256, 56, 56] | 通道 128→256，尺寸不变 |
| Block 3 | MaxPool | [B, 256, 56, 56] | [B, 256, 28, 28] | 尺寸减半 |
| Block 4 | Conv×3 + ReLU×3 | [B, 256, 28, 28] | [B, 512, 28, 28] | 通道 256→512，尺寸不变 |
| Block 4 | MaxPool | [B, 512, 28, 28] | [B, 512, 14, 14] | 尺寸减半 |
| Block 5 | Conv×3 + ReLU×3 | [B, 512, 14, 14] | [B, 512, 14, 14] | 通道保持 512，尺寸不变 |
| Block 5 | MaxPool | [B, 512, 14, 14] | [B, 512, 7, 7] | 尺寸减半 |
| Flatten | — | [B, 512, 7, 7] | [B, 25088] | 展平为一维向量 |
| Classifier | Linear(25088→4096) | [B, 25088] | [B, 4096] | 全连接层 |
| Classifier | Dropout(0.5) | [B, 4096] | [B, 4096] | 随机失活 |
| Classifier | Linear(4096→4096) | [B, 4096] | [B, 4096] | 全连接层 |
| Classifier | Dropout(0.5) | [B, 4096] | [B, 4096] | 随机失活 |
| Classifier | Linear(4096→K) | [B, 4096] | [B, K] | 输出类别 logits |
