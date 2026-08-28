# CRAFT 算法核心要点整理

> **论文全称**: Character Region Awareness for Text Detection (CVPR 2019)  
> **核心定位**: 自底向上（Bottom-up）、基于字符定位与连接关系的任意形状场景文本检测算法。

---

## 1. 核心思想 (Core Concept)
* **检测策略**: 将文本检测转化为**像素级的回归/分割任务**，不直接预测整行/整词包围框，而是先检测单个字符，再根据字符间关联合成文本。
* **双通道输出 (Heatmaps)**:
  - **Region Score（区域分数）**: 预测像素为**字符中心**的概率。
  - **Affinity Score（亲和度分数）**: 预测像素为**相邻两个字符连接中心**的概率。

---

## 2. 网络结构 (Architecture)
* **Backbone**: VGG-16（带 Batch Normalization）。
* **Decoder**: 类似 U-Net / FCN 架构，引入特征金字塔与跳跃连接（Skip Connections），融合深层语义特征与浅层高分辨率特征。
* **Output**: 双通道特征图（分别对应 Region Map 与 Affinity Map）。

---

## 3. 标签生成机制 (Ground Truth Generation)
* **2D 高斯热力图**: 在字符框（Region）和字符间距框（Affinity）内填充 2D 高斯分布，形成“中心高、四周低”的平滑标签（非硬边界二值标签）。
* **透视变换**: 计算坐标框的透视变换矩阵，将标准 2D 高斯分布映射至变形多边形框内。

---

## 4. 弱监督学习 (Weakly-Supervised Learning)
* **痛点**: 真实场景数据集（如 ICDAR）大多只有**单词级（Word-level）**标注，缺少字符级标注。
* **解决方案**:
  1. **预训练**: 先在带有字符级标注的合成数据集（SynthText）上训练初始模型。
  2. **伪标签生成**: 将真实图像输入模型获取初始预测，结合单词级框，利用 **分水岭算法 (Watershed)** 将单词框切分成单个字符框，生成伪标签（Pseudo-Truths）。
  3. **置信度加权 ($S_c$)**: 若切分出的字符数等于单词实际长度，则置信度高；否则降低置信度权重。

---

## 5. 损失函数 (Loss Function)
采用像素级带**置信度加权**的 L2 损失函数 (MSE)：

$$L = \sum_{p} S_c(p) \cdot \left( \|S_r(p) - S_r^*(p)\|^2 + \|S_a(p) - S_a^*(p)\|^2 \right)$$

* $S_c(p)$: 像素 $p$ 的置信度（合成数据恒为 1，真实数据由弱监督模块计算）
* $S_r(p), S_a(p)$: 区域与亲和度的预测分数
* $S_r^*(p), S_a^*(p)$: 对应的 Ground Truth（真实值或伪标签）

---

## 6. 后处理流程 (Post-Processing)
1. **阈值二值化**: 对预测的 Region Map 和 Affinity Map 分别设置阈值（如 0.4）得到二值图。
2. **连通域提取**: 结合两张二值图，利用 OpenCV 获取**连通区域 (Connected Components)**。
3. **文本框生成**: 对连通区域计算最小外接矩形 (MinAreaRect)；或沿连通域上下边缘求极值点，生成可适应弯曲文本的**多边形框 (Polygon)**。

---

## 7. 优缺点对比 (Pros & Cons)

| 维度 | 说明 |
| :--- | :--- |
| **优势** | 1. **强适应性**: 对弯曲、倾斜、艺术字、超长文本检测效果极佳。<br>2. **低感受野依赖**: 仅关注局部字符形态与邻近关系，无需超大感受野即可处理长文本。 |
| **劣势** | 1. **推理开销**: 后处理依赖二值化与连通域提取，推理速度慢于 DBNet 等单阶段算法。<br>2. **超大间距敏感**: 若文本字符间距天然过大（如稀疏排版）或极度模糊，易出现断连。 |

```
import torch
import torch.nn as nn
import torch.nn.functional as F
from torchvision import models

# =====================================================================
# 1. 解码器基础模块 (DoubleConv)
# =====================================================================

class DoubleConv(nn.Module):
    """CRAFT Decoder 卷积块：1x1 降维 -> 3x3 特征提取"""
    def __init__(self, in_ch, mid_ch, out_ch):
        super(DoubleConv, self).__init__()
        self.conv = nn.Sequential(
            nn.Conv2d(in_ch, mid_ch, kernel_size=1),
            nn.BatchNorm2d(mid_ch),
            nn.ReLU(inplace=True),
            nn.Conv2d(mid_ch, out_ch, kernel_size=3, padding=1),
            nn.BatchNorm2d(out_ch),
            nn.ReLU(inplace=True)
        )

    def forward(self, x):
        # 示例：输入 [B, in_ch, H, W] -> 输出 [B, out_ch, H, W]
        return self.conv(x)


# =====================================================================
# 2. CRAFT 主体网络架构
# =====================================================================

class CRAFT(nn.Module):
    def __init__(self, pretrained=False):
        super(CRAFT, self).__init__()
        
        # Backbone: VGG16 with BatchNorm
        vgg = models.vgg16_bn(pretrained=pretrained).features
        
        # 截取不同尺度的特征层
        self.slice1 = vgg[:6]   # Conv1_2
        self.slice2 = vgg[6:13]  # Conv2_2
        self.slice3 = vgg[13:23] # Conv3_3
        self.slice4 = vgg[23:33] # Conv4_3
        self.slice5 = vgg[33:43] # Conv5_3
        
        # Decoder 融合模块
        self.upconv1 = DoubleConv(1024, 512, 256)
        self.upconv2 = DoubleConv(512, 256, 128)
        self.upconv3 = DoubleConv(256, 128, 64)
        self.upconv4 = DoubleConv(64, 32, 32)
        
        # 预测头 (输出 Region Score 和 Affinity Score)
        self.conv_cls = nn.Sequential(
            nn.Conv2d(32, 32, kernel_size=3, padding=1),
            nn.ReLU(inplace=True),
            nn.Conv2d(32, 32, kernel_size=3, padding=1),
            nn.ReLU(inplace=True),
            nn.Conv2d(32, 16, kernel_size=3, padding=1),
            nn.ReLU(inplace=True),
            nn.Conv2d(16, 16, kernel_size=1),
            nn.ReLU(inplace=True),
            nn.Conv2d(16, 2, kernel_size=1)
        )

    def forward(self, x):
        # 假设输入图像 x 维度为: [Batch=2, Channels=3, Height=640, Width=640]

        # -----------------------------------------------------------------
        # 1. 特征提取 (Backbone: VGG16)
        # -----------------------------------------------------------------
        h1 = self.slice1(x)   # Output Shape: [2, 64, 640, 640]   (原图 1/1 尺度)
        h2 = self.slice2(h1)  # Output Shape: [2, 128, 320, 320]  (原图 1/2 尺度)
        h3 = self.slice3(h2)  # Output Shape: [2, 256, 160, 160]  (原图 1/4 尺度)
        h4 = self.slice4(h3)  # Output Shape: [2, 512, 80, 80]    (原图 1/8 尺度)
        h5 = self.slice5(h4)  # Output Shape: [2, 512, 40, 40]    (原图 1/16 尺度)

        # -----------------------------------------------------------------
        # 2. 特征上采样与多尺度融合 (Decoder)
        # -----------------------------------------------------------------
        # Stage 1: 融合 h5(1/16) 与 h4(1/8)
        h5_up = F.interpolate(h5, size=h4.shape[2:], mode='bilinear', align_corners=True) 
        # Output Shape: [2, 512, 80, 80]
        
        cat1 = torch.cat([h5_up, h4], dim=1) 
        # Output Shape: [2, 1024, 80, 80]
        
        y = self.upconv1(cat1) 
        # Output Shape: [2, 256, 80, 80]

        # Stage 2: 融合 y(1/8) 与 h3(1/4)
        y_up1 = F.interpolate(y, size=h3.shape[2:], mode='bilinear', align_corners=True) 
        # Output Shape: [2, 256, 160, 160]
        
        cat2 = torch.cat([y_up1, h3], dim=1) 
        # Output Shape: [2, 512, 160, 160]
        
        y = self.upconv2(cat2) 
        # Output Shape: [2, 128, 160, 160]

        # Stage 3: 融合 y(1/4) 与 h2(1/2)
        y_up2 = F.interpolate(y, size=h2.shape[2:], mode='bilinear', align_corners=True) 
        # Output Shape: [2, 128, 320, 320]
        
        cat3 = torch.cat([y_up2, h2], dim=1) 
        # Output Shape: [2, 256, 320, 320]
        
        y = self.upconv3(cat3) 
        # Output Shape: [2, 64, 320, 320]

        # Stage 4: 细化卷积
        y = self.upconv4(y) 
        # Output Shape: [2, 32, 320, 320]

        # -----------------------------------------------------------------
        # 3. 预测输出 (Prediction Head)
        # -----------------------------------------------------------------
        out = self.conv_cls(y) 
        # Output Shape: [2, 2, 320, 320] (双通道预测热图: Region Score 和 Affinity Score)
        
        return out


# =====================================================================
# 3. 带像素级置信度加权的 MSE 损失函数
# =====================================================================

class MapMSELoss(nn.Module):
    def __init__(self):
        super(MapMSELoss, self).__init__()

    def forward(self, y_pred, y_true, confidence):
        """
        输入维度参数:
        - y_pred:     [2, 2, 320, 320]  (模型预测热图)
        - y_true:     [2, 2, 320, 320]  (标注/伪标签热图)
        - confidence: [2, 320, 320]     (像素级置信度权重矩阵)
        """
        # 1. 计算未加权 L2 损失
        loss = (y_pred - y_true) ** 2  
        # Output Shape: [2, 2, 320, 320]
        
        # 2. 扩展 confidence 匹配通道维度
        confidence = confidence.unsqueeze(1).expand_as(loss) 
        # Output Shape: [2, 2, 320, 320]
        
        # 3. 像素级加权点乘
        weighted_loss = loss * confidence 
        # Output Shape: [2, 2, 320, 320]
        
        # 4. 标量化规约求平均
        total_loss = weighted_loss.sum() / (confidence.sum() + 1e-8) 
        # Output Shape: torch.Size([]) (标量/Scalar 0维 Tensor)
        
        return total_loss


# =====================================================================
# 4. 前向传播与维度测试脚本
# =====================================================================

if __name__ == "__main__":
    model = CRAFT(pretrained=False)
    criterion = MapMSELoss()
    model.eval()

    # 1. 创建虚拟输入图像
    img_input = torch.randn(2, 3, 640, 640)
    # Tensor Shape: [2, 3, 640, 640]

    # 2. 前向推理计算
    outputs = model(img_input)
    # Tensor Shape: [2, 2, 320, 320]

    # 拆分预测图
    region_scores = outputs[:, 0:1, :, :]
    # Tensor Shape: [2, 1, 320, 320]
    
    affinity_scores = outputs[:, 1:2, :, :]
    # Tensor Shape: [2, 1, 320, 320]

    # 3. 创建虚拟 Target 标签与置信度矩阵
    targets = torch.randn(2, 2, 320, 320)
    # Tensor Shape: [2, 2, 320, 320]
    
    confidence = torch.ones(2, 320, 320)
    # Tensor Shape: [2, 320, 320]

    # 4. 计算 Loss
    loss_val = criterion(outputs, targets, confidence)
    # Tensor Shape: torch.Size([]) (Scalar)

    print(f"输入图像尺寸:      {img_input.shape}")
    print(f"模型输出尺寸:      {outputs.shape}")
    print(f"Region Score 尺寸: {region_scores.shape}")
    print(f"Loss 结果数值类型: {loss_val.shape} (标量值: {loss_val.item():.4f})")
```
