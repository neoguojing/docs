
# MobileNet V1 与 V2 技术对比

MobileNet V1 和 V2 是轻量级神经网络发展史上的两个核心里程碑，它们的演进代表了移动端视觉架构从“减少计算量”向“兼顾高维特征与低维表达”的跨越。

> MobileNet V1 vs V2 Block Evolution. 来源：Medium

## 核心架构特性对比

| 特性维度 | MobileNet V1 | MobileNet V2 |
|:---------|:-------------|:-------------|
| 核心基石 | 深度可分离卷积 (DW + PW) | 深度可分离卷积 (DW + PW) |
| 基础模块 | 直筒型序列：3x3 DW -> 1x1 PW | 倒残差结构：1x1 升维 -> 3x3 DW -> 1x1 降维 |
| 残差连接 | 无（类似 VGG 的直连结构） | 有（类似 ResNet，仅在步长为 1 且维度相同时启用） |
| 激活函数 | ReLU | ReLU6（限制最大值为 6，防溢出，利于量化） |
| 瓶颈层设计 | 降维后依然使用 ReLU，导致低维信息损失 | 线性瓶颈（Linear Bottleneck）：降维后直接输出，无激活函数 |

---

## 一、MobileNet V1：深度可分离卷积

**设计哲学：** 将标准卷积拆分为“空间特征提取 (Depthwise)”和“通道特征融合 (Pointwise)”，在保持特征提取能力的同时，将计算量骤降 8~9 倍。

```python
import torch
import torch.nn as nn

class DepthwiseSeparableConvV1(nn.Module):
    """MobileNet V1 核心模块"""
    def __init__(self, in_channels, out_channels, stride):
        super().__init__()
        
        # 1. 深度卷积 (Depthwise Conv)
        # 核心：groups=in_channels，强迫每个输入通道分配独立的 3x3 卷积核
        self.dw = nn.Sequential(
            nn.Conv2d(in_channels, in_channels, kernel_size=3, stride=stride, 
                      padding=1, groups=in_channels, bias=False),
            nn.BatchNorm2d(in_channels),
            nn.ReLU(inplace=True)
        )
        
        # 2. 逐点卷积 (Pointwise Conv)
        # 核心：使用 1x1 卷积核，跨通道融合特征，并改变通道数为 out_channels
        self.pw = nn.Sequential(
            nn.Conv2d(in_channels, out_channels, kernel_size=1, stride=1, 
                      padding=0, bias=False),
            nn.BatchNorm2d(out_channels),
            nn.ReLU(inplace=True) 
            # 注意：V1 在这里使用了 ReLU，这在低维状态下会造成信息流失
        )

    def forward(self, x):
        """
        维度变化推演 (假设输入为 [B, 32, 112, 112], stride=2, out_channels=64):
        1. dw(x): 
           - 空间公式: H_out = (112 + 2*1 - 3)//2 + 1 = 56
           - 通道公式: groups=32, 输出通道不变，仍为 32。
           - 输出维度 -> [B, 32, 56, 56]
        2. pw(x):
           - 空间公式: H_out = (56 + 2*0 - 1)//1 + 1 = 56 (1x1 卷积不改变空间尺寸)
           - 通道公式: 映射至 out_channels (64)
           - 最终输出 -> [B, 64, 56, 56]
        """
        out = self.dw(x)
        out = self.pw(out)
        return out
```

---

## 二、MobileNet V2：倒残差与线性瓶颈

**设计哲学：** V1 证明了深度卷积的高效，但它在低维空间中进行，特征提取受限；同时 ReLU 在低维降维时会“抹杀”大量负值特征。V2 通过“先升维、后卷积、再降维（且不用 ReLU）”完美解决了这个痛点。

```python
class InvertedResidualV2(nn.Module):
    """MobileNet V2 核心模块：倒残差 + 线性瓶颈"""
    def __init__(self, in_channels, out_channels, stride, expand_ratio):
        super().__init__()
        self.stride = stride
        
        # 计算升维后的中间通道数 (通常 expand_ratio = 6)
        hidden_dim = int(round(in_channels * expand_ratio))
        
        # 仅当步长为 1 且输入输出通道数相同时，才使用残差相加
        self.use_res_connect = self.stride == 1 and in_channels == out_channels

        layers = []
        # 1. 逐点卷积升维 (Expansion)
        if expand_ratio != 1:
            layers.extend([
                nn.Conv2d(in_channels, hidden_dim, kernel_size=1, stride=1, padding=0, bias=False),
                nn.BatchNorm2d(hidden_dim),
                nn.ReLU6(inplace=True) # 使用 ReLU6
            ])
            
        layers.extend([
            # 2. 深度卷积提取特征 (Depthwise) - 在高维空间中进行
            nn.Conv2d(hidden_dim, hidden_dim, kernel_size=3, stride=stride, 
                      padding=1, groups=hidden_dim, bias=False),
            nn.BatchNorm2d(hidden_dim),
            nn.ReLU6(inplace=True),
            
            # 3. 逐点卷积降维 (Linear Projection)
            nn.Conv2d(hidden_dim, out_channels, kernel_size=1, stride=1, padding=0, bias=False),
            nn.BatchNorm2d(out_channels)
            # 核心创新：此处直接结束，不添加任何非线性激活函数，保护低维特征流形
        ])
        
        self.conv = nn.Sequential(*layers)

    def forward(self, x):
        """
        维度变化推演 (假设输入为 [B, 24, 56, 56], stride=1, out_channels=24, expand_ratio=6):
        1. 升维: 24 * 6 = 144
           - [B, 24, 56, 56] -> 1x1 Conv -> [B, 144, 56, 56]
        2. DW特征提取: 
           - [B, 144, 56, 56] -> 3x3 DW Conv (stride=1) -> [B, 144, 56, 56]
        3. 降维(线性):
           - [B, 144, 56, 56] -> 1x1 Conv -> [B, 24, 56, 56]
        4. 残差相加:
           - 满足 use_res_connect，将步骤 3 的输出与原始输入 x 相加。
           - 最终输出 -> [B, 24, 56, 56]
        """
        if self.use_res_connect:
            return x + self.conv(x)
        else:
            return self.conv(x)
```

---

## 三、维度计算通用法则 (The “Why”)

无论在哪一代网络中，维度的变化严格遵循两个维度的计算规则：

### 空间分辨率（高 H 和宽 W）

H_out = floor((H_in - K + 2*P) / S) + 1

- **S=1（步长为 1）：** 搭配 3x3 卷积核（K=3）和 padding=1（P=1），公式抵消，尺寸完全不变。
- **S=2（步长为 2）：** 尺寸直接折半，实现下采样（Downsampling）。MobileNet 系列从不使用 Pooling 层降维，全程靠步长控制。

### 通道维度（Channel C）

- **DW 卷积：** `groups` 参数强制设为等于 `in_channels`，这意味着每个卷积核只看一张特征图，因此输出通道必定等于输入通道（不改变厚度）。
- **PW 卷积 (1x1)：** 专门负责重新混合特征，输出通道完全由代码中设定的 `out_channels` 决定（直接压缩或放大厚度）。
