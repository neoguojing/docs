
# 深度解读：FCN (全卷积网络) 与 PyTorch 代码实现

## 一、 核心原理解析

### 1. 全卷积（Fully Convolutional）
在传统的 CNN（如 VGG、AlexNet）中，网络前段是卷积层，末端连接全连接层（FC）。
*   **传统局限**：全连接层要求输入尺寸固定，且将二维特征图展平为一维向量，**彻底破坏了图像的空间位置信息**。
*   **FCN 的改造**：将全连接层替换为 **$1\times1$ 的卷积层**。
    *   *对比*：若池化后特征图尺寸为 $7 \times 7 \times 512$。传统 FC 层将其展平为 $25088$ 维的向量；FCN 使用通道数为 4096 的 $1\times1$ 卷积，输出变为 $7 \times 7 \times 4096$ 的**二维特征图**。
    *   *优势*：保留了特征所在的空间位置（即热力图 Heatmap），且网络不再受限，可接收**任意尺寸的图像**。

### 2. 转置卷积（Transposed Convolution）
语义分割需输出与原图同大的预测图，但深度特征经过多次池化后尺寸极小（如原图的 $1/32$），需要“放大”回原图大小。
*   **传统上采样**：如双线性插值，是固定数学公式，无学习参数。
*   **转置卷积（反卷积）**：**带有可学习参数的上采样方法**。它将一个特征值映射回更大的像素区域，通过在像素间填充 0 并卷积，实现尺寸放大。网络可自动学习如何最合理地还原图像边缘和细节。

---

## 二、 空间尺寸演进与跳跃连接

设输入图像尺寸为 $H \times W$。在骨干网络（如 VGG16）中，图像经过 5 次池化，特征图尺寸递减，我们用 C1-C5 表示：

*   **输入**: $H \times W$
*   **C1**: $H/2 \times W/2$
*   **C2**: $H/4 \times W/4$
*   **C3**: $H/8 \times W/8$
*   **C4**: $H/16 \times W/16$
*   **C5**: $H/32 \times W/32$ （感受野最大、语义最强，但空间细节丢失最严重）

为找回空间细节，FCN 引入**跳跃连接（Skip Architecture）**：

1.  **FCN-32s**：直接将 **C5** 降维后进行 **32倍** 上采样，恢复到 $H \times W$。预测边缘非常粗糙。
2.  **FCN-16s**：将 **C5** 上采样 **2倍**（尺寸变为 $H/16$），与同尺寸的 **C4** 按位相加。融合后进行 **16倍** 上采样。边缘有所改善。
3.  **FCN-8s**（最优结构）：将上一步融合的特征再次上采样 **2倍**（尺寸变为 $H/8$），与同尺寸的 **C3** 按位相加。最后进行 **8倍** 上采样，恢复到 $H \times W$。由于融合了 C3 的精细细节，分割边缘最为准确。

---

## 三、 FCN-8s 与 Loss 计算 PyTorch 实现

```python
import torch
import torch.nn as nn
from torchvision import models

class FCN8s(nn.Module):
    def __init__(self, num_classes=21):
        super(FCN8s, self).__init__()
        
        # 1. 加载预训练的 VGG16
        vgg16 = models.vgg16(pretrained=True)
        features = list(vgg16.features.children())
        
        # 2. 截取 VGG 的多尺度特征
        # 输入: (B, 3, H, W) -> 输出 C3: (B, 256, H/8, W/8)
        self.C3_features = nn.Sequential(*features[:17])
        # 输入 C3 -> 输出 C4: (B, 512, H/16, W/16)
        self.C4_features = nn.Sequential(*features)
        # 输入 C4 -> 输出 C5: (B, 512, H/32, W/32)
        self.C5_features = nn.Sequential(*features[24:])
        
        # 3. 全卷积改造：用 1x1 卷积替代原有的全连接层
        # 输入 C5: (B, 512, H/32, W/32) -> 输出: (B, num_classes, H/32, W/32)
        self.fc_to_conv = nn.Sequential(
            nn.Conv2d(512, 4096, kernel_size=1),
            nn.ReLU(inplace=True),
            nn.Dropout(),
            nn.Conv2d(4096, 4096, kernel_size=1),
            nn.ReLU(inplace=True),
            nn.Dropout(),
            nn.Conv2d(4096, num_classes, kernel_size=1)
        )
        
        # 4. 浅层特征降维（为按位相加做准备）
        self.score_C4 = nn.Conv2d(512, num_classes, kernel_size=1)
        self.score_C3 = nn.Conv2d(256, num_classes, kernel_size=1)
        
        # 5. 转置卷积（上采样）
        # 将深层语义图放大 2 倍
        self.upsample_2x_1 = nn.ConvTranspose2d(
            num_classes, num_classes, kernel_size=4, stride=2, padding=1, bias=False)
        # 将融合图再放大 2 倍
        self.upsample_2x_2 = nn.ConvTranspose2d(
            num_classes, num_classes, kernel_size=4, stride=2, padding=1, bias=False)
        # 最终放大 8 倍，恢复至原图分辨率
        self.upsample_8x = nn.ConvTranspose2d(
            num_classes, num_classes, kernel_size=16, stride=8, padding=4, bias=False)

    def forward(self, x):
        # --- 特征提取 ---
        C3 = self.C3_features(x)        # 尺寸: H/8
        C4 = self.C4_features(C3)       # 尺寸: H/16
        C5 = self.C5_features(C4)       # 尺寸: H/32
        
        # --- 全卷积映射 ---
        score_C5 = self.fc_to_conv(C5)  # 尺寸: H/32
        
        # --- 第一次跳跃连接 (FCN-16s) ---
        up_C5 = self.upsample_2x_1(score_C5)  # H/32 上采样到 H/16
        score_C4 = self.score_C4(C4)          # C4 通道降维
        fuse_16s = up_C5 + score_C4           # 尺寸 H/16 融合相加
        
        # --- 第二次跳跃连接 (FCN-8s) ---
        up_fuse_16s = self.upsample_2x_2(fuse_16s) # H/16 上采样到 H/8
        score_C3 = self.score_C3(C3)               # C3 通道降维
        fuse_8s = up_fuse_16s + score_C3           # 尺寸 H/8 融合相加
        
        # --- 最终上采样输出 ---
        out = self.upsample_8x(fuse_8s)            # H/8 上采样 8 倍到 H
        return out


# ================= 训练与 Loss 计算演示 =================
if __name__ == '__main__':
    batch_size, num_classes, H, W = 4, 21, 224, 224
    model = FCN8s(num_classes=num_classes)
    
    # 1. 模拟输入与标签
    x = torch.randn(batch_size, 3, H, W)
    # 重点：标签 Target 的形状是 (B, H, W)，没有通道维度！数据类型必须是 Long。
    # 标签内每个数值代表该像素所属的类别索引 (范围 0 ~ num_classes-1)
    target = torch.randint(0, num_classes, (batch_size, H, W), dtype=torch.long)
    
    # 2. 前向传播
    outputs = model(x)  # outputs 形状: (B, 21, H, W)
    
    # 3. 计算损失
    # CrossEntropyLoss 会自动在 outputs 的通道维度 (dim=1) 上计算 Softmax 并与 target 索引比对
    criterion = nn.CrossEntropyLoss()
    loss = criterion(outputs, target)
    loss.backward()
    
    print(f"输入图像尺寸: {x.shape}")
    print(f"预测输出尺寸: {outputs.shape}")
    print(f"真实标签尺寸: {target.shape}")
    print(f"当前 Batch Loss: {loss.item():.4f}")
```

## 四、 代码执行流程说明
### 1. 初始化阶段 (__init__)特征截取：
将 VGG16 按池化层的位置拆分，划分为三个模块，分别负责提取原图 $1/8$（C3）、$1/16$（C4）和 $1/32$（C5）尺寸的特征图。结构转换与降维：使用 $1\times1$ 卷积替代全连接层，突破尺寸限制。同时定义 $1\times1$ 卷积将浅层特征 C3 和 C4 的通道数压缩至目标类别数，为跳跃连接做好维度匹配。配置转置卷积：设定三组转置卷积权重矩阵，分别执行 2倍、2倍和 8倍的尺寸放大操作。
### 2. 前向传播阶段 (forward)提取多尺度特征：
输入图像穿过骨干网络，在不同深度截获尺寸递减、语义递增的张量（C3、C4、C5）。逐层跳跃融合：深层语义图（尺寸 $1/32$）经 2倍转置卷积放大后，与降维后的中层细节 C4（尺寸 $1/16$）按位相加；该融合特征再放大 2倍，与降维后的浅层细节 C3（尺寸 $1/8$）再次相加。还原分辨率：最终的融合特征通过 8 倍转置卷积，输出与原始图像宽高一致的四维预测张量 (Batch, num_classes, Height, Width)。
### 3. 损失计算阶段标签格式约束：
语义分割的掩码数据形状必须严格为 (Batch, Height, Width)，内部数据需为整型（Long），直接表示像素所属类别的索引。交叉熵比对：nn.CrossEntropyLoss() 接收四维预测和三维标签后，自动在预测张量的通道维度（dim=1）对每个像素计算 Softmax 概率，并汇总计算出全图的平均交叉熵损失用于梯度反向传播。
