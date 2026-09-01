
# R-CNN 家族架构演进与核心代码解析

目标检测的 R-CNN 系列可以看作是一个高度分工的**“找茬流水线”**。从 Faster R-CNN 奠定基础，到 Mask R-CNN 增加像素级描边，再到 Cascade R-CNN 引入多级严格质检，整个架构通过灵活的模块插拔，实现了对图像中物体的精准定位与识别。

## 核心角色总览

| 模块名称 | 隐喻 | 职责 | 核心痛点解决 |
| :--- | :--- | :--- | :--- |
| **RPN** | 侦察兵 | 快速扫视全图，框出可能有物体的区域 (Proposals) | 替代了以前极慢的滑动窗口算法 |
| **RoI Align** | 裁剪台 | 把尺寸不一的候选框，统一裁剪成标准大小 | 解决了老版本四舍五入导致的像素不对齐问题 |
| **RoI Head** | 鉴定专家 | 看裁剪出的特征块，判定具体类别并微调框坐标 | 给出最终的矩形框和类别概率 |
| **Mask Head** | 描边画师 | (Mask R-CNN专属) 在鉴定完毕后，画出物体的像素级轮廓 | 填补了从“矩形框”到“多边形轮廓”的空白 |
| **Cascade结构** | 质检委员会 | (Cascade R-CNN专属) 多个鉴定专家串联，标准逐级递增 | 解决了单一及格线导致的“框不够准”或“样本太少”问题 |

---

## 核心组件全代码解析（附严格维度演变）

设定我们的输入是一张 `[1, 3, 800, 800]` 的图片（1张图，3通道RGB，宽800高800），骨干网络（如 ResNet）将其浓缩 16 倍，输出特征图尺寸为 `[1, 256, 50, 50]`。

### 1. RPN (侦察兵)

```python
import torch
import torch.nn as nn
import torchvision.ops as ops

class RPN(nn.Module):
    def __init__(self, in_channels=256, num_anchors=9):
        super().__init__()
        # 3x3 卷积，融合感受野，维度不变
        self.conv = nn.Conv2d(in_channels, in_channels, kernel_size=3, padding=1)
        
        # 判断是否有物体（二分类：前景/背景），每个位置预测 9 个框
        self.objectness = nn.Conv2d(in_channels, num_anchors, kernel_size=1)
        # 预测框的初步微调量 (dx, dy, dw, dh)，每个框 4 个值
        self.bbox_pred = nn.Conv2d(in_channels, num_anchors * 4, kernel_size=1)

    def forward(self, features):
        # features 进场形状: [1, 256, 50, 50]
        x = torch.relu(self.conv(features)) # 形状: [1, 256, 50, 50]
        
        # 1. 给出每个预设框里有物体的概率
        # 形状: [1, 9, 50, 50] -> 对应 50*50 个位置，每个位置 9 个概率
        scores = self.objectness(x) 
        
        # 2. 给出每个预设框的微调量
        # 形状: [1, 36, 50, 50] -> 对应 50*50 个位置，每个位置 9*4 个微调量
        deltas = self.bbox_pred(x)
        
        # 【省略解码过程】系统会将 scores 和 deltas 结合预设的 Anchors，
        # 过滤掉重合度太高的框（NMS），最终挑选出 2000 个最靠谱的候选框。
        # proposals 形状: [2000, 5] (第0列是图片batch索引，后4列是 x1, y1, x2, y2)
        proposals = decode_and_filter(scores, deltas) 
        
        return proposals
```

### 2. RoI Head (基础鉴定专家 - Faster R-CNN核心)

```python
class RoIHead(nn.Module):
    def __init__(self, in_channels=256, num_classes=80):
        super().__init__()
        # 经过 RoI Align 后，特征固定为 7x7，展平后长度为 256 * 7 * 7
        flatten_size = in_channels * 7 * 7 # 12544
        
        # 共享全连接层，提取高级语义特征
        self.fc = nn.Sequential(
            nn.Linear(flatten_size, 1024),
            nn.ReLU(),
            nn.Linear(1024, 1024),
            nn.ReLU()
        )
        
        # 分支1：类别鉴定
        self.cls_score = nn.Linear(1024, num_classes)
        # 分支2：坐标极致微调 (每个类别都有自己专属的4个调整量)
        self.bbox_pred = nn.Linear(1024, num_classes * 4)

    def forward(self, roi_features):
        # roi_features 进场形状: [2000, 256, 7, 7] 
        
        # 展平: [2000, 12544]
        x = roi_features.view(roi_features.size(0), -1) 
        # 经过两层全连接: [2000, 1024]
        x = self.fc(x)
        
        # 类别得分形状: [2000, 80]
        scores = self.cls_score(x)      
        # 坐标微调形状: [2000, 320] (80类 * 4)
        box_deltas = self.bbox_pred(x)  
        
        return scores, box_deltas
```

### 3. Mask Head (描边画师 - Mask R-CNN 拓展)

```python
class MaskHead(nn.Module):
    def __init__(self, in_channels=256, num_classes=80):
        super().__init__()
        # 特征提取层
        self.conv = nn.Sequential(
            nn.Conv2d(in_channels, 256, kernel_size=3, padding=1),
            nn.ReLU()
            # 实际网络会有4层连续Conv2d
        )
        # 【核心魔法】：反卷积放大分辨率 (14x14 放大至 28x28)
        self.deconv = nn.ConvTranspose2d(256, 256, kernel_size=2, stride=2)
        
        # 输出掩码 (每个类生成一张 28x28 的黑白图)
        self.mask_predictor = nn.Conv2d(256, num_classes, kernel_size=1)

    def forward(self, roi_mask_features):
        # 画师要看更细致的图，进场形状: [2000, 256, 14, 14]
        
        x = self.conv(roi_mask_features)  # 形状不变: [2000, 256, 14, 14]
        x = torch.relu(self.deconv(x))    # 分辨率翻倍: [2000, 256, 28, 28]
        
        mask_logits = self.mask_predictor(x) # 形状: [2000, 80, 28, 28]
        # 通过 Sigmoid 将数值压缩到 0~1 之间，代表像素属于该物体的概率
        return torch.sigmoid(mask_logits)
```

---

## 终极组装：Cascade Mask R-CNN (全宇宙最强拼图)

这段代码将展示整个系统是如何跑起来的。它包含了 Cascade 的“循环套娃”以及 Mask 的“旁支画图”。

```python
class UltimateRCNN(nn.Module):
    def __init__(self):
        super().__init__()
        self.backbone = resnet50_fpn()
        self.rpn = RPN()
        
        # 【Cascade R-CNN 特性】建立三级质检委员会
        self.head_stage1 = RoIHead(num_classes=80)
        self.head_stage2 = RoIHead(num_classes=80)
        self.head_stage3 = RoIHead(num_classes=80)
        
        # 【Mask R-CNN 特性】画师在终点等候
        self.mask_head = MaskHead(num_classes=80)

    def apply_deltas(self, boxes, deltas):
        # 伪方法：将预测的偏移量加到框上，生成新框
        return boxes + deltas 

    def forward(self, image):
        # image 形状: [1, 3, 800, 800]
        
        # 1. 骨干提取特征: [1, 256, 50, 50]
        features = self.backbone(image)
        
        # 2. 侦察兵找初始框: [2000, 5] (包含batch_index)
        proposals = self.rpn(features)

        # ================== Cascade 质检流水线 ==================
        
        # 【专家1】
        # RoI Align 抠图 (7x7): [2000, 256, 7, 7]
        roi_feat_1 = ops.roi_align(features, proposals, output_size=(7, 7), spatial_scale=1/16.0)
        scores_1, deltas_1 = self.head_stage1(roi_feat_1)
        # 用微调量修正框，进入下一关
        boxes_2 = self.apply_deltas(proposals, deltas_1)

        # 【专家2】
        # 拿更准的 boxes_2 重新抠图: [2000, 256, 7, 7]
        roi_feat_2 = ops.roi_align(features, boxes_2, output_size=(7, 7), spatial_scale=1/16.0)
        scores_2, deltas_2 = self.head_stage2(roi_feat_2)
        # 再次修正框
        boxes_3 = self.apply_deltas(boxes_2, deltas_2)

        # 【专家3】
        # 拿极其准的 boxes_3 最终抠图: [2000, 256, 7, 7]
        roi_feat_3 = ops.roi_align(features, boxes_3, output_size=(7, 7), spatial_scale=1/16.0)
        final_scores, final_deltas = self.head_stage3(roi_feat_3)
        
        # 获得终极精准框: [2000, 5]
        final_boxes = self.apply_deltas(boxes_3, final_deltas)

        # ================== Mask 画师工坊 ==================
        
        # 画师出场！拿着终极精准框，去抠一张更大的图 (14x14)
        # 形状: [2000, 256, 14, 14]
        roi_mask_feat = ops.roi_align(features, final_boxes, output_size=(14, 14), spatial_scale=1/16.0)
        
        # 描绘出像素级掩码
        # 形状: [2000, 80, 28, 28]
        final_masks = self.mask_head(roi_mask_feat)

        return final_scores, final_boxes, final_masks
```
