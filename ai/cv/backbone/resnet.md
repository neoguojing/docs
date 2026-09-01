
# ResNet-50 核心特性与完整源码解析

## 一、ResNet-50 的三大核心特性

### 1. 残差学习 (Residual Learning)

通过建立捷径（Shortcut），将网络结构变为 H(x) = F(x) + x。如果该层不需要提取新特征，网络只需将主路 F(x) 的权重学为 0，即可实现恒等映射，彻底解决了网络加深导致的梯度消失和退化问题。

### 2. 瓶颈结构 (Bottleneck) 的极限降参

在 50 层以上的深层网络中，摒弃了两个 3×3 卷积的堆叠，强制使用 1×1→3×3→1×1 的三明治结构。

### 3. 降采样与通道膨胀的同步发生

除了 Stage 1，每个 Stage 的首个模块都会通过步长（stride=2）让图像长宽减半，同时将通道数翻倍。

---

## 二、重点展开：1×1 卷积在 ResNet-50 中的两大"灵魂用途"

在 ResNet-50 中，1×1 卷积不仅用于主路的特征融合，更是解决张量维度冲突的唯一手段。

### 用途一：主路 (Main Path) 的"降维与升维"

**降维 (压缩层)**：假设输入特征图为 256 通道，先用 64 个 1×1 卷积核将其压扁为 64 通道。这一步极大地减少了后续 3×3 卷积的计算参数。

**升维 (膨胀层)**：在 3×3 卷积提取完空间特征后（此时仍为 64 通道），用 256 个 1×1 卷积核将其重新撑厚到 256 通道。

**特性本质**：这里的 1×1 不提取任何空间几何特征，它纯粹在深度方向上对通道进行加权融合，充当了算力控制的"阀门"。

### 用途二：捷径 (Shortcut) 的"维度对齐转换头"

当网络进入一个新的 Stage（例如从 Stage 1 进入 Stage 2）时，主路会发生降采样：

- **主路 F(x)**：输出尺寸减半（56→28），通道翻倍（256→512）。
- **原图 x**：依然是 56×56 大小，256 通道。

此时 F(x) + x 无法相加。必须在捷径上强行插入一个 1×1 卷积：

- 设置其卷积核数量为 512（解决通道不匹配）。
- 设置其步长 stride = 2（解决空间尺寸不匹配）。

这个 1×1 卷积就像一个转换接头，瞬间把原图 x 的形状调整得和 F(x) 一模一样，确保残差相加顺利执行。

---

## 三、ResNet-50 完整源码与硬核维度注释

下面的代码基于最权威的 PyTorch 官方实现逻辑重构，注释中给出了每一步形状变化的精确推导公式：

> W_out = ⌊(W_in − K + 2P) / S⌋ + 1

```python
import torch
import torch.nn as nn

class Bottleneck(nn.Module):
    \"\"\"
    ResNet-50 的核心积木：瓶颈层。
    无论输入通道是多少，主路输出的通道数永远是中间 3x3 卷积通道数的 4 倍 (expansion = 4)。
    \"\"\"
    expansion = 4

    def __init__(self, in_channels, mid_channels, stride=1):
        super(Bottleneck, self).__init__()
        
        # -------------------------------------------------------------
        # 主路 第一步：1x1 卷积 (降维)
        # 作用：跨通道特征融合，大幅压缩通道数，降低后续计算量
        # -------------------------------------------------------------
        self.conv1 = nn.Conv2d(in_channels, mid_channels, kernel_size=1, stride=1, bias=False)
        self.bn1 = nn.BatchNorm2d(mid_channels)
        
        # -------------------------------------------------------------
        # 主路 第二步：3x3 卷积 (空间特征提取)
        # 作用：提取边缘、纹理等空间特征。整个模块的"空间降采样"发生在这里！
        # 如果传入的 stride=2，则图像宽和高在此处减半。
        # padding=1 的精妙之处：当 stride=1 时，(W - 3 + 2*1)/1 + 1 = W，尺寸不变。
        # -------------------------------------------------------------
        self.conv2 = nn.Conv2d(mid_channels, mid_channels, kernel_size=3, 
                               stride=stride, padding=1, bias=False)
        self.bn2 = nn.BatchNorm2d(mid_channels)
        
        # -------------------------------------------------------------
        # 主路 第三步：1x1 卷积 (升维)
        # 作用：将通道数放大 4 倍，恢复特征的厚度，准备输出
        # -------------------------------------------------------------
        out_channels = mid_channels * self.expansion
        self.conv3 = nn.Conv2d(mid_channels, out_channels, kernel_size=1, stride=1, bias=False)
        self.bn3 = nn.BatchNorm2d(out_channels)
        
        self.relu = nn.ReLU(inplace=True)
        
        # -------------------------------------------------------------
        # 捷径 (Shortcut) 的 1x1 转换头
        # 触发条件：步长不为1 (空间尺寸缩水了) 或 输入通道数 != 输出通道数
        # -------------------------------------------------------------
        self.shortcut = nn.Sequential()
        if stride != 1 or in_channels != out_channels:
            self.shortcut = nn.Sequential(
                # 利用 1x1 卷积同时完成"扩充通道"和"缩小尺寸(stride=2)"
                # 尺寸变化公式: (W - 1 + 2*0)/2 + 1 = W/2
                nn.Conv2d(in_channels, out_channels, kernel_size=1, stride=stride, bias=False),
                nn.BatchNorm2d(out_channels)
            )

    def forward(self, x):
        identity = self.shortcut(x)  # 捷径数据流

        out = self.relu(self.bn1(self.conv1(x)))
        out = self.relu(self.bn2(self.conv2(out)))
        out = self.bn3(self.conv3(out))  # 第三步卷积后不立即过 ReLU
        
        out += identity              # 残差矩阵元素级相加 (Element-wise Addition)
        out = self.relu(out)         # 相加后再过 ReLU，保持非线性
        return out


class ResNet50(nn.Module):
    def __init__(self, num_classes=1000):
        super(ResNet50, self).__init__()
        
        self.in_channels = 64  # Stem 层输出的初始通道数
        
        # -------------------------------------------------------------
        # 1. Stem 层 (快速预处理)
        # 输入张量: [B, 3, 224, 224]
        # -------------------------------------------------------------
        # 空间计算: (224 - 7 + 2*3)/2 + 1 = 112
        # 输出张量: [B, 64, 112, 112]
        self.conv1 = nn.Conv2d(3, 64, kernel_size=7, stride=2, padding=3, bias=False)
        self.bn1 = nn.BatchNorm2d(64)
        self.relu = nn.ReLU(inplace=True)
        
        # 空间计算: (112 - 3 + 2*1)/2 + 1 = 56
        # 输出张量: [B, 64, 56, 56]
        self.maxpool = nn.MaxPool2d(kernel_size=3, stride=2, padding=1)
        
        # -------------------------------------------------------------
        # 2. Stage 1 (包含 3 个 Bottleneck)
        # -----------------------------------------
        # mid_channels=64, 最终输出 out_channels = 64*4 = 256
        # 此时输入是 64 通道，尺寸 56x56。此阶段不需要降采样 (stride=1)
        # 最终张量: [B, 256, 56, 56]
        self.layer1 = self._make_layer(Bottleneck, mid_channels=64, num_blocks=3, stride=1)
        
        # -------------------------------------------------------------
        # 3. Stage 2 (包含 4 个 Bottleneck)
        # -----------------------------------------
        # 此时输入是 256 通道。该阶段首个 Block 设置 stride=2。
        # 空间计算: 56 -> 28; 通道计算: 128*4 = 512
        # 最终张量: [B, 512, 28, 28]
        self.layer2 = self._make_layer(Bottleneck, mid_channels=128, num_blocks=4, stride=2)
        
        # -------------------------------------------------------------
        # 4. Stage 3 (包含 6 个 Bottleneck)
        # -----------------------------------------
        # 空间计算: 28 -> 14; 通道计算: 256*4 = 1024
        # 最终张量: [B, 1024, 14, 14]
        self.layer3 = self._make_layer(Bottleneck, mid_channels=256, num_blocks=6, stride=2)
        
        # -------------------------------------------------------------
        # 5. Stage 4 (包含 3 个 Bottleneck)
        # -----------------------------------------
        # 空间计算: 14 -> 7; 通道计算: 512*4 = 2048
        # 最终张量: [B, 2048, 7, 7]
        self.layer4 = self._make_layer(Bottleneck, mid_channels=512, num_blocks=3, stride=2)
        
        # -------------------------------------------------------------
        # 6. 分类头 (Classification Head)
        # -----------------------------------------
        # 全局平均池化，强制把 7x7 的空间分辨率压缩成 1x1
        # 张量变为: [B, 2048, 1, 1]
        self.avgpool = nn.AdaptiveAvgPool2d((1, 1))
        # 展平后接入全连接层输出分类概率
        self.fc = nn.Linear(2048, num_classes)

    def _make_layer(self, block, mid_channels, num_blocks, stride):
        # 构建一个 Stage。精髓在于：只有第一个 block 的步长是传入的 stride，其余全为 1。
        strides = [stride] + [1] * (num_blocks - 1)
        layers = []
        for s in strides:
            # 动态传入当前的 self.in_channels
            layers.append(block(self.in_channels, mid_channels, s))
            # 每次建完一个 Block，立刻更新网络当前的通道数，供下一个 Block 读取
            self.in_channels = mid_channels * block.expansion
        return nn.Sequential(*layers)

    def forward(self, x):
        x = self.relu(self.bn1(self.conv1(x)))
        x = self.maxpool(x)
        
        x = self.layer1(x)
        x = self.layer2(x)
        x = self.layer3(x)
        x = self.layer4(x)
        
        x = self.avgpool(x)
        x = torch.flatten(x, 1)
        x = self.fc(x)
        
        return x
```
