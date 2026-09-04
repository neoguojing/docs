# crowd

## 人群计数和位置估计
- 输入图片 ，输出 密度信息
- 基于目标检测的人数统计方案
- 基于密度估计：MCNN 、CSRNet 生成密度图； 基于贝叶斯损失优化密度图估计统计图中像素
- 基于人头关键点的人群计数

# 基于目标检测的方案

## 核心逻辑：利用 YOLO、Faster R-CNN 等检测器直接回归人头或人体的边界框（Bounding Box）。

### 适用性：提供高精度的位置和尺度信息，适用于稀疏人群或低视角场景。但在高密度、严重遮挡的场景下，感受野内特征重叠会导致 NMS（非极大值抑制）误删大量目标，召回率急剧下降。

# CSRNet 人群计数算法核心说明文档

CSRNet（Congested Scene Recognition Network）是基于密度图估计（Density Map-Based）的人群计数经典范式。其核心突破在于：利用预训练的 VGG 网络提取前端语义特征，并在后端引入**空洞卷积（Dilated Convolution）**，在不降低特征图空间分辨率的前提下，大幅扩展感受野，从而精准捕获极度拥挤场景下的人群密度分布。

本文档将从底层数学转换、数值推演、网络架构到 PyTorch 代码实现，提供端到端的全景解析。

---

## 1. 核心数学基础：点标注到密度图 (Ground Truth)

深度学习模型无法直接学习离散的坐标点，必须在数据预处理阶段，将人工标注的“人头点坐标”转换为连续的二维“密度图”，作为网络学习的监督信号（Ground Truth）。

### 1.1 数学公式
$$D(x) = \sum_{i=1}^{N} \delta(x - x_i) * G_{\sigma}(x)$$

*   $N$: 图像中的总人数。
*   $\delta(x - x_i)$: 冲激函数。在一个全零矩阵中，仅在人头真实坐标 $x_i$ 处赋值为 $1$。
*   $G_{\sigma}$: 方差为 $\sigma^2$ 的二维高斯核（总和归一化为 1）。
*   $*$: 二维空间卷积操作。本质上是将离散的点平滑扩散为高斯分布。

### 1.2 极简数值示例

假设图像大小为 $5 \times 5$，包含 **2 个人**，坐标分别在 $(1, 1)$ 和 $(3, 3)$（索引从 $0$ 开始）。

**步骤一：构建点标注矩阵 (Point Map)**

$$
\begin{bmatrix} 
0 & 0 & 0 & 0 & 0 \\ 
0 & 1 & 0 & 0 & 0 \\ 
0 & 0 & 0 & 0 & 0 \\ 
0 & 0 & 0 & 1 & 0 \\ 
0 & 0 & 0 & 0 & 0 
\end{bmatrix}
$$

**步骤二：定义归一化高斯核 (Gaussian Kernel)**

假设使用一个 $3 \times 3$ 的高斯核，内部权重和为 1：

$$
\begin{bmatrix} 
0.05 & 0.10 & 0.05 \\ 
0.10 & 0.40 & 0.10 \\ 
0.05 & 0.10 & 0.05 
\end{bmatrix}
$$

**步骤三：空间卷积平滑 (Density Map)**

将高斯核中心对准值为 $1$ 的点进行滑动。注意坐标 $(2, 2)$ 处是两个分布的**重叠区**，其值为叠加求和（$0.05 + 0.05 = 0.10$）。密度图全矩阵求和严格等于 **2.0**。

$$
\begin{bmatrix} 
0.05 & 0.10 & 0.05 & 0 & 0 \\ 
0.10 & 0.40 & 0.10 & 0 & 0 \\ 
0.05 & 0.10 & \mathbf{0.10} & 0.10 & 0.05 \\ 
0 & 0 & 0.10 & 0.40 & 0.10 \\ 
0 & 0 & 0.05 & 0.10 & 0.05 
\end{bmatrix}
$$
---

## 2. CSRNet 网络架构设计

为了拟合上述数学过程生成的密度图，CSRNet 采用了“前端降采样 + 后端空洞卷积”的非对称架构。

1.  **前端特征提取 (Frontend - VGG16)**：
    *   截取预训练的 VGG-16 网络的前 10 个卷积层。
    *   包含 3 次 $2 \times 2$ 的最大池化操作（Max Pooling）。
    *   **空间影响**：特征图的高和宽被缩小为原图的 $1/8$（降采样 8 倍）。
2.  **后端密度估计 (Backend - Dilated CNN)**：
    *   包含 6 层空洞卷积（`kernel_size=3`, `dilation=2`, `padding=2`）。
    *   **空间影响**：由于 `padding == dilation`，这 6 层卷积**不会**改变特征图的分辨率，始终保持在原图的 $1/8$ 大小。但感受野成倍扩大，使其能够理解人头周边的拥挤上下文。
3.  **输出层 (Output)**：
    *   使用 $1 \times 1$ 卷积将通道数压缩为 $1$。输出最终的预测密度图。

---

## 3. 完整 PyTorch 代码实现

以下代码实现了从 Ground Truth 生成到网络前向传播、再到损失计算与人数统计的完整闭环，并包含详细的张量维度追踪。

```python
import torch
import torch.nn as nn
import torch.nn.functional as F
from torchvision import models

# ==============================================================================
# 模块 1: 数据预处理 (离线或 DataLoader 中执行)
# ==============================================================================
def generate_ground_truth(points, h, w, downsample=8):
    """
    将离散坐标转换为真实密度图。
    因 CSRNet 网络经过 3 次池化，输出特征图缩小了 8 倍，
    故此处生成的 GT 密度图尺寸也必须对应缩小 8 倍以进行 Loss 对齐。
    """
    gt_h, gt_w = h // downsample, w // downsample
    # 初始 GT 维度: [1, 1, H/8, W/8]
    density_map = torch.zeros((1, 1, gt_h, gt_w))
    
    if len(points) == 0:
        return density_map
        
    # 1. 坐标降采样 (与特征图尺度对齐)
    points = points / downsample 
    
    # 2. 构造冲激函数
    for point in points:
        x, y = int(point[0]), int(point[1])
        # 边界保护
        if 0 <= x < gt_w and 0 <= y < gt_h:
            density_map[0, 0, y, x] += 1.0
            
    # 3. 构造 15x15 的二维高斯核 (固定方差 sigma=4.0)
    kernel_size, sigma = 15, 4.0
    grid = torch.arange(kernel_size, dtype=torch.float32) - kernel_size // 2
    xx, yy = torch.meshgrid(grid, grid, indexing='ij')
    gaussian_kernel = torch.exp(-(xx**2 + yy**2) / (2 * sigma**2))
    gaussian_kernel = gaussian_kernel / torch.sum(gaussian_kernel)
    weight = gaussian_kernel.view(1, 1, kernel_size, kernel_size)
    
    # 4. 卷积平滑
    density_map = F.conv2d(density_map, weight, padding=kernel_size//2)
    return density_map

# ==============================================================================
# 模块 2: 网络模型定义 (CSRNet)
# ==============================================================================
class CSRNet(nn.Module):
    def __init__(self):
        super(CSRNet, self).__init__()
        
        # 前端: VGG-16 特征提取 (降采样 8 倍)
        vgg = models.vgg16(weights=models.VGG16_Weights.IMAGENET1K_V1)
        self.frontend = nn.Sequential(*list(vgg.features.children()))
        
        # 后端: 空洞卷积 (保持分辨率，扩大感受野)
        self.backend = nn.Sequential(
            nn.Conv2d(512, 512, kernel_size=3, padding=2, dilation=2), nn.ReLU(inplace=True),
            nn.Conv2d(512, 512, kernel_size=3, padding=2, dilation=2), nn.ReLU(inplace=True),
            nn.Conv2d(512, 512, kernel_size=3, padding=2, dilation=2), nn.ReLU(inplace=True),
            nn.Conv2d(512, 256, kernel_size=3, padding=2, dilation=2), nn.ReLU(inplace=True),
            nn.Conv2d(256, 128, kernel_size=3, padding=2, dilation=2), nn.ReLU(inplace=True),
            nn.Conv2d(128, 64,  kernel_size=3, padding=2, dilation=2), nn.ReLU(inplace=True),
            
            # 输出层: 通道压缩
            nn.Conv2d(64, 1, kernel_size=1)
        )
        self._initialize_weights()

    def forward(self, x):
        # 输入维度: [Batch, 3, H, W]
        x = self.frontend(x) 
        # 前端输出维度: [Batch, 512, H/8, W/8]
        x = self.backend(x)  
        # 后端输出维度: [Batch, 1, H/8, W/8]
        return x

    def _initialize_weights(self):
        for m in self.backend.modules():
            if isinstance(m, nn.Conv2d):
                nn.init.normal_(m.weight, std=0.01)
                if m.bias is not None:
                    nn.init.constant_(m.bias, 0)

# ==============================================================================
# 模块 3: 训练与推理 Pipeline 模拟
# ==============================================================================
if __name__ == "__main__":
    # 设定模拟图像分辨率
    H, W = 768, 1024
    image = torch.randn(1, 3, H, W) 
    
    # 模拟真实标注数据: 图像中存在 3 个人头
    annotated_points = torch.tensor([[100, 200], [500, 600], [700, 800]], dtype=torch.float32)

    # 1. 生成 Ground Truth 密度图 (尺寸将变为 96 x 128)
    target_density = generate_ground_truth(annotated_points, h=H, w=W, downsample=8)

    # 2. 网络前向传播
    model = CSRNet()
    pred_density = model(image)

    # 3. 计算损失 (训练态) - 逐像素的均方误差
    criterion = nn.MSELoss(reduction='sum')
    loss = criterion(pred_density, target_density)

    # 4. 人数统计 (推理态) - 对预测密度图的所有像素求和
    estimated_count = torch.sum(pred_density).item()
    gt_count = torch.sum(target_density).item()

    print(f"Ground Truth 形状: {target_density.shape}")
    print(f"预测密度图 形状: {pred_density.shape}")
    print("-" * 30)
    print(f"真实标签总人数: {gt_count:.2f}")
    print(f"网络预测总人数: {estimated_count:.2f}")
    print(f"当前 MSE Loss : {loss.item():.4f}")
