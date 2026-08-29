# CRNN 核心原理：像阅读一样识别文本

CRNN 的核心思想是将图像识别问题转化为序列识别问题。它通过将图像切割成密集的垂直特征序列，让神经网络像"从左到右阅读"一样识别变长文本，整个过程无需任何字符级别的边界框标注。

## 直观示例：识别图像中的 "CAT"

假设输入是一张高度为 32 像素、包含单词 "CAT" 的细长图片：

### 1. CNN 特征切片

CNN 将这张图像压缩为一个特征图（高度变为 1）。你可以将其想象为把原图沿水平方向切成了 20 个小竖条，每个竖条被编码为一个 512 维的向量。

### 2. RNN 时序预测

双向 LSTM 按顺序审视这 20 个竖条。由于字符有宽度，一个 "C" 可能会占据连续的 4 个竖条，且字符间存在缝隙。RNN 会输出类似这样的 20 步预测序列（`-` 代表 CTC 的空白占位符）：

```
[C, C, C, -, -, A, A, -, -, -, T, T, T, -, -, -, -, -, -, -]
```

### 3. CTC 折叠解码

CTC 算法应用两条极简规则：首先合并相邻的重复字符变为 `[C, -, A, -, T, -]`，然后强行剔除所有空白符 `-`，最终输出完美的识别结果 CAT。

---

## 模型架构设计

CRNN 由三个核心组件构成，形成一个端到端的流水线：

| 组件 | 功能 | 关键技术 |
| :--- | :--- | :--- |
| **CNN 主干网络** | 提取输入图像的局部视觉特征，输出特征图 | VGG/ResNet 变体、BatchNorm |
| **BiLSTM 序列建模层** | 将特征图按列展开为序列，双向建模字符间上下文依赖 | 双向 LSTM、多层堆叠 |
| **CTC 损失与解码器** | 实现输入图像与输出标签之间的对齐，支持变长预测 | 空白符(Blank)、动态规划 |

---

## PyTorch 完整代码实现

以下是使用 PyTorch 实现的标准 CRNN 完整代码，包含了模型定义、前向传播以及 CTC Loss 的计算逻辑：

```python
import torch
import torch.nn as nn
import torch.nn.functional as F

class BidirectionalLSTM(nn.Module):
    def __init__(self, input_size, hidden_size, output_size):
        super(BidirectionalLSTM, self).__init__()
        # RNN 期望输入形状为 (Seq_Length, Batch_Size, Features)
        self.rnn = nn.LSTM(input_size, hidden_size, bidirectional=True, batch_first=False)
        # 双向 LSTM 输出维度是 hidden_size * 2
        self.linear = nn.Linear(hidden_size * 2, output_size)

    def forward(self, x):
        recurrent, _ = self.rnn(x)
        # recurrent 形状: (Seq_Length, Batch_Size, hidden_size * 2)
        output = self.linear(recurrent)
        # output 形状: (Seq_Length, Batch_Size, output_size)
        return output

class CRNN(nn.Module):
    def __init__(self, img_channels, num_classes, hidden_size=256):
        super(CRNN, self).__init__()
        
        # 1. CNN 提取层 (类 VGG 结构，高度逐步降维至 1)
        # 假设输入图像尺寸为 (Batch, Channels, H=32, W)
        self.cnn = nn.Sequential(
            nn.Conv2d(img_channels, 64, kernel_size=3, stride=1, padding=1),
            nn.ReLU(True),
            nn.MaxPool2d(2, 2), # 尺寸变化: H(32->16), W(W->W/2)
            
            nn.Conv2d(64, 128, kernel_size=3, stride=1, padding=1),
            nn.ReLU(True),
            nn.MaxPool2d(2, 2), # 尺寸变化: H(16->8), W(W/2->W/4)
            
            nn.Conv2d(128, 256, kernel_size=3, stride=1, padding=1),
            nn.BatchNorm2d(256),
            nn.ReLU(True),
            # 注意: 这里的池化窗口 (2,2)，步长为 (2,1)。高度减半，但宽度步长为1，保留序列长度
            nn.MaxPool2d((2, 2), stride=(2, 1), padding=(0, 1)), # H(8->4)
            
            nn.Conv2d(256, 512, kernel_size=3, stride=1, padding=1),
            nn.BatchNorm2d(512),
            nn.ReLU(True),
            nn.MaxPool2d((2, 2), stride=(2, 1), padding=(0, 1)), # H(4->2)
            
            # 使用 valid 卷积将高度彻底降为 1
            nn.Conv2d(512, 512, kernel_size=2, stride=1, padding=0),
            nn.BatchNorm2d(512),
            nn.ReLU(True) # 最终尺寸: (Batch, 512, H=1, W_new)
        )
        
        # 2. RNN 序列建模层
        self.rnn = nn.Sequential(
            BidirectionalLSTM(512, hidden_size, hidden_size),
            BidirectionalLSTM(hidden_size, hidden_size, num_classes)
        )

    def forward(self, x):
        # 提取视觉特征
        conv = self.cnn(x) 
        
        # 维度转换以适配 RNN
        b, c, h, w = conv.size()
        assert h == 1, "CNN 输出的高度必须为 1"
        conv = conv.squeeze(2) # 变为 (Batch, Channels, W_new)
        
        # PyTorch RNN 默认时间步维度在最前面
        conv = conv.permute(2, 0, 1) # 变为 (W_new, Batch, Channels)
        
        # 输出分类概率 (未经过 softmax)
        output = self.rnn(conv)
        return output

# ================= 运行测试与 CTC Loss 计算 =================

if __name__ == "__main__":
    # 参数配置
    batch_size = 4
    img_channels = 1   # 灰度图
    img_height = 32
    img_width = 128
    num_classes = 37   # 26个英文字母 + 10个数字 + 1个空白符(Blank=0)
    
    # 初始化模型
    model = CRNN(img_channels, num_classes)
    
    # 生成随机测试数据: 4张 32x128 的灰度图
    dummy_input = torch.randn(batch_size, img_channels, img_height, img_width)
    
    # 前向传播
    logits = model(dummy_input)
    print(f"Logits shape: {logits.shape}") # 预期: (Seq_Length, Batch_Size, Num_Classes) -> (33, 4, 37)
    
    # 计算 CTC Loss
    # CTC Loss 期望输入为 LogSoftmax 后的对数概率
    log_probs = F.log_softmax(logits, dim=2)
    
    # 生成假标签 (假设 Batch 中 4 张图，每张图包含真实的字符索引序列)
    # 例如：标签长度分别为 3, 5, 4, 6
    targets = torch.randint(1, num_classes, (18,)) # 扁平化的一维标签张量
    
    # 记录模型输出的序列长度 (Batch 个 Seq_Length)
    input_lengths = torch.full(size=(batch_size,), fill_value=logits.size(0), dtype=torch.long)
    # 记录真实标签的序列长度
    target_lengths = torch.tensor([3, 5, 4, 6], dtype=torch.long)
    
    # PyTorch 内置 CTC Loss (默认 blank=0)
    ctc_loss_fn = nn.CTCLoss(blank=0, zero_infinity=True)
    loss = ctc_loss_fn(log_probs, targets, input_lengths, target_lengths)
    
    print(f"CTC Loss: {loss.item():.4f}")
```

---

## 工程落地与优化补充

在实际项目中，除了模型本身，数据预处理和部署优化同样决定了最终效果：

### 1. 图像预处理增强

原始图像常存在光照不均、噪声干扰等问题。在送入模型前，建议使用 OpenCV 进行以下处理：

- **自适应直方图均衡化 (CLAHE)**：有效提升低对比度图像的文本清晰度。
- **尺寸归一化**：将图像高度固定为 32 像素，宽度按比例缩放，避免字符变形。
- **归一化到 [-1, 1]**：加速模型收敛。

```python
import cv2
import numpy as np

def preprocess_image(image: np.ndarray, target_height=32):
    # 自动灰度化
    if len(image.shape) == 3:
        image = cv2.cvtColor(image, cv2.COLOR_BGR2GRAY)
    # 自适应直方图均衡化
    clahe = cv2.createCLAHE(clipLimit=2.0, tileGridSize=(8,8))
    image = clahe.apply(image)
    # 尺寸归一化：保持宽高比
    h, w = image.shape
    scale = target_height / h
    new_w = int(w * scale)
    resized = cv2.resize(image, (new_w, target_height), interpolation=cv2.INTER_CUBIC)
    # 归一化到[-1,1]
    normalized = (resized.astype(np.float32) / 255.0 - 0.5) * 2
    return normalized[None, None, ...]  # (1,1,H,W)
```

### 2. 训练策略建议

- **数据增强**：对训练图像进行随机旋转（±15°）、高斯噪声、弹性形变等，能显著提升模型对手写体或倾斜文本的鲁棒性。
- **优化器选择**：推荐使用 Adam 或 SGD+Momentum，学习率可从 1e-3 开始，配合余弦退火策略。
- **正则化**：在 RNN 层之间加入 Dropout (0.3-0.5) 防止过拟合。

### 3. 部署与推理加速

- **模型转换**：将 PyTorch 模型导出为 ONNX 格式，可跨平台部署。
- **推理加速**：使用 TensorRT 或 OpenVINO 进行推理加速，通常可将 CRNN 的推理延迟从 50ms 降至 20ms 以内，非常适合 CPU 环境下的实时识别场景。
