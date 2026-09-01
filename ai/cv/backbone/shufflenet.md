
# ShuffleNet 核心技术解析

## 一、ShuffleNet 概述

ShuffleNet 是一种专为计算力受限的移动设备设计的轻量级卷积神经网络。它的核心创新在于解决了传统网络中 $1\times1$ 卷积计算代价过高，以及分组卷积带来的信息不流通问题。

---

## 二、核心特性

### 2.1 逐点分组卷积 (Pointwise Group Convolution)

在 ResNeXt 等网络中，$3\times3$ 卷积被替换为了深度可分离卷积（DWConv），这使得 $1\times1$ 卷积成为了计算量的瓶颈。ShuffleNet 将 $1\times1$ 卷积也进行了分组（Group），极大地减少了参数量和计算量（FLOPs）。

### 2.2 通道洗牌 (Channel Shuffle)

纯粹的分组卷积会导致各个通道组之间的信息"隔离"（即某个输出通道只包含了某个输入通道组的信息）。Channel Shuffle 通过对分组后的通道进行重组（Reshape $\rightarrow$ Transpose $\rightarrow$ Flatten），打乱通道顺序，使得后续的卷积层能够跨组提取特征。

### 2.3 两种基本单元 (ShuffleNet Unit)

- **步长为1 (Stride=1)**：用于加深网络。采用残差连接（Add），输入和输出的特征图尺寸和通道数完全一致。
- **步长为2 (Stride=2)**：用于降采样（Downsampling）。捷径分支（Shortcut）使用 $3\times3$ 平均池化，主分支的最后输出不经过 ReLU 直接与捷径分支进行通道拼接（Concat），从而在降低空间尺寸的同时加倍通道数。

---

## 三、核心代码实现与维度推导 (PyTorch)

通用维度计算公式为：

$$Output = \left\lfloor \frac{Input - Kernel + 2 \times Padding}{Stride} \right\rfloor + 1$$

### 3.1 通道洗牌操作 (Channel Shuffle)

```python
import torch
import torch.nn as nn

def channel_shuffle(x, groups):
    """
    通道洗牌 (Channel Shuffle) 操作
    作用：打乱分组卷积后的通道顺序，促进不同组之间的特征信息交流。
    """
    # 1. 获取输入特征图的维度信息
    # x 维度: [B, C, H, W]
    # B: Batch size (批次大小)
    # C: Channels (通道数)
    # H: Height (高度)
    # W: Width (宽度)
    batch_size, num_channels, height, width = x.size()

    # 确保总通道数可以被组数整除
    channels_per_group = num_channels // groups

    # 2. 维度重塑 (Reshape)
    # 将一维的通道 C 拆分为两维：(groups, channels_per_group)
    # 维度变化: [B, C, H, W] -> [B, groups, channels_per_group, H, W]
    # 为什么这样算：逻辑上把通道分成了 groups 个堆，每堆有 channels_per_group 个通道
    x = x.view(batch_size, groups, channels_per_group, height, width)

    # 3. 维度转置 (Transpose / Permute)
    # 交换 groups 和 channels_per_group 的维度位置
    # 维度变化: [B, groups, channels_per_group, H, W] -> [B, channels_per_group, groups, H, W]
    # 为什么转置：这是打乱通道顺序的核心步。相当于把原来的行读取变成列读取。
    x = torch.transpose(x, 1, 2).contiguous()

    # 4. 展平恢复 (Flatten)
    # 将交换后的前两维重新合并为一个维度 C
    # 维度变化: [B, channels_per_group, groups, H, W] -> [B, C, H, W]
    # 为什么这样算：恢复到标准的 4D 张量格式，供后续卷积层使用，此时通道内的信息已经跨组混合。
    x = x.view(batch_size, -1, height, width)

    return x
```

### 3.2 ShuffleNet 基本单元 (ShuffleNet Unit)

```python
class ShuffleNetUnit(nn.Module):
    def __init__(self, in_channels, out_channels, stride, groups=3):
        """
        ShuffleNet 基本单元
        :param in_channels: 输入通道数
        :param out_channels: 最终输出通道数
        :param stride: 步长 (1 或 2)
        :param groups: 分组卷积的组数 (通常设为3或8)
        """
        super(ShuffleNetUnit, self).__init__()
        self.stride = stride
        self.groups = groups

        # 瓶颈结构中的中间通道数，通常为输出通道数的 1/4
        mid_channels = out_channels // 4

        # 如果是降采样模块 (stride=2)，最终是用 Concat，所以主分支输出的通道数要减去输入的通道数
        # 如果是特征提取模块 (stride=1)，是用 Add 融合，主分支输出通道数等于最终通道数
        branch_out_channels = out_channels - in_channels if stride == 2 else out_channels

        # 1. 1x1 逐点分组卷积 (降维)
        # 维度计算: Kernel=1, Pad=0, Stride=1
        # H_out = (H_in - 1 + 2*0)/1 + 1 = H_in (高宽不变)
        # 维度变化: [B, in_channels, H, W] -> [B, mid_channels, H, W]
        self.gconv1 = nn.Sequential(
            nn.Conv2d(in_channels, mid_channels, kernel_size=1, groups=groups, bias=False),
            nn.BatchNorm2d(mid_channels),
            nn.ReLU(inplace=True)
        )

        # 2. 3x3 深度卷积 (Depthwise Conv)
        # groups 设置为通道数 (mid_channels)，即每个通道独立卷积
        # 维度计算: Kernel=3, Pad=1, Stride=S (S=1 或 2)
        # 假设 stride=1: H_out = (H_in - 3 + 2*1)/1 + 1 = H_in (高宽不变)
        # 假设 stride=2: H_out = (H_in - 3 + 2*1)/2 + 1 = H_in / 2 (高宽减半，向下取整)
        # 维度变化: [B, mid_channels, H, W] -> [B, mid_channels, H/S, W/S]
        self.dwconv = nn.Sequential(
            nn.Conv2d(mid_channels, mid_channels, kernel_size=3, stride=stride, 
                      padding=1, groups=mid_channels, bias=False),
            nn.BatchNorm2d(mid_channels)
            # 注意：ShuffleNet 中 DWConv 之后不加 ReLU
        )

        # 3. 1x1 逐点分组卷积 (升维)
        # 维度计算: Kernel=1, Pad=0, Stride=1
        # H_out = (H_in/S - 1 + 2*0)/1 + 1 = H_in/S (高宽维持上一层)
        # 维度变化: [B, mid_channels, H/S, W/S] -> [B, branch_out_channels, H/S, W/S]
        self.gconv2 = nn.Sequential(
            nn.Conv2d(mid_channels, branch_out_channels, kernel_size=1, groups=groups, bias=False),
            nn.BatchNorm2d(branch_out_channels)
        )

        # 4. 捷径分支 (Shortcut) 的平均池化，仅在降采样 (stride=2) 时使用
        if self.stride == 2:
            # 维度计算: Kernel=3, Pad=1, Stride=2
            # H_out = (H_in - 3 + 2*1)/2 + 1 = H_in / 2
            # 维度变化: [B, in_channels, H, W] -> [B, in_channels, H/2, W/2]
            self.shortcut = nn.AvgPool2d(kernel_size=3, stride=2, padding=1)
            
        self.relu = nn.ReLU(inplace=True)

    def forward(self, x):
        # x 初始维度: [B, in_channels, H, W]

        # 保存捷径分支特征
        residual = x

        # 步骤 1: 1x1 组卷积 -> 输出维度 [B, mid_channels, H, W]
        out = self.gconv1(x) 

        # 步骤 2: Channel Shuffle -> 输出维度不变 [B, mid_channels, H, W]
        out = channel_shuffle(out, self.groups)

        # 步骤 3: 3x3 深度可分离卷积 -> 输出维度 [B, mid_channels, H/S, W/S]
        out = self.dwconv(out)

        # 步骤 4: 1x1 组卷积 -> 输出维度 [B, branch_out_channels, H/S, W/S]
        out = self.gconv2(out)

        # 步骤 5: 融合输出
        if self.stride == 2:
            # 捷径分支通过 3x3 均值池化降采样 -> 维度: [B, in_channels, H/2, W/2]
            residual = self.shortcut(x)

            # 沿通道维度 (dim=1) 拼接 (Concat)
            # 主分支: [B, out_channels - in_channels, H/2, W/2]
            # 捷径分支: [B, in_channels, H/2, W/2]
            # 拼接后维度计算: (out_channels - in_channels) + in_channels = out_channels
            # 最终维度变化: -> [B, out_channels, H/2, W/2]
            out = torch.cat((residual, out), dim=1)
        else:
            # 步长为1时，直接相加 (Add)
            # 主分支和捷径分支的维度均保持为: [B, in_channels, H, W]
            # 相加后维度变化: -> [B, out_channels, H, W] (因为在 stride=1 时 in_channels == out_channels)
            out = out + residual

        # 最后的 ReLU 激活
        return self.relu(out)
```

---

## 四、空间维度与通道维度的设计哲学

### 4.1 1x1 降维与升维

在主干路中，特征先经过 $1\times1$ 卷积将其通道数压缩至目标输出数量的 1/4（即 mid_channels）。这大幅削减了后续 $3\times3$ 深度卷积的计算负担。完成特征提取后，再用一层 $1\times1$ 卷积将其升维回目标通道数。

### 4.2 Concat vs Add

**为什么 stride=2 时用 Concat 而 stride=1 用 Add？**

- **stride=2（下采样阶段）**：分辨率缩小（$H/2 \times W/2$），导致空间信息丢失。通过 Concat 将原特征（池化后）和新提取的特征直接拼在一起，可以补偿信息损失并低成本地扩充通道数。
- **stride=1（特征提取阶段）**：空间尺寸没变，用 Add（如同 ResNet）可以在不增加模型计算规模的前提下有效缓解梯度消失，促使网络可以堆叠得更深。
