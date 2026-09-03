# 计算机视觉中的输入对齐技术：从几何矩阵到代码实现

在计算机视觉（特别是 OCR 和人脸识别）中，输入对齐（Alignment）的核心目的是消除平移、旋转、缩放以及形变带来的几何干扰，降低下游模型（如特征提取或序列识别）的学习难度。对齐通常发生在检测（Detection）之后、识别（Recognition）之前。

---

## 一、 主流对齐方法分类

对齐方法主要分为**显式几何变换**与**网络内隐式特征对齐**两大类：

### 1. 显式几何变换（基于图像级的物理对齐）
*   **相似变换 (Similarity Transformation)：** 主要用于**人脸对齐**。仅允许平移、旋转和等比例缩放，不允许剪切（Shear），保证对齐后比例不失调。
*   **透视变换 (Perspective Transformation)：** 主要用于**OCR文档分析/文本行截取**。将倾斜视角下呈现为任意四边形的文档或文本，投影映射并拉伸为水平的标准矩形。
*   **薄板样条插值 (Thin Plate Spline, TPS)：** 应对**非刚性形变**（如弯曲文本、褶皱文档），通过匹配控制点构建非线性映射场，将目标“拉直”。

### 2. 隐式特征对齐（基于深度学习的端到端对齐）
*   **空间变换网络 (STN)：** 即插即用模块，动态预测变换参数，使用可微网格采样器重采样特征图，无需关键点标注自适应对齐。
*   **RoIAlign / RoIRotate：** 采用双线性插值在连续坐标上提取特征（消除量化误差）。RoIRotate 直接在特征图上带角度采样以对齐倾斜文本。
*   **可变形卷积 (DCN)：** 学习每个卷积核采样点的 2D 偏移量，使感受野自适应不规则目标的形状。

---

## 二、 核心数学概念与数值推演

对齐过程本质上是：**用最小二乘法寻找变换矩阵，并通过插值对原图进行 Warp 重采样**。

### 1. 相似变换矩阵：死板的“刚性”变换规则
相似变换矩阵通过控制4个内部变量（缩放 $s$、旋转角 $\theta$、X平移 $tx$、Y平移 $ty$），确保目标只被平移、放大和旋转，绝对不会被拉伸变形。矩阵公式为：
$$M = \begin{bmatrix} s\cos\theta & -s\sin\theta & tx \\ s\sin\theta & s\cos\theta & ty \end{bmatrix}$$

**数值推演：**
假设让人脸**放大 2 倍，逆时针旋转 90度（$\theta=90^\circ$），向右平移 10，向下平移 5**。
因为 $\cos(90^\circ)=0$ 且 $\sin(90^\circ)=1$，矩阵计算如下：
$$M = \begin{bmatrix} 2 \times 0 & -2 \times 1 & 10 \\ 2 \times 1 & 2 \times 0 & 5 \end{bmatrix} = \begin{bmatrix} 0 & -2 & 10 \\ 2 & 0 & 5 \end{bmatrix}$$

若原图有一关键点坐标 $(x=1, y=3)$。代入矩阵：
*   $X' = 0 \times 1 + (-2) \times 3 + 10 = 4$
*   $Y' = 2 \times 1 + 0 \times 3 + 5 = 7$
该点被成功转移到了新图像的 $(4, 7)$。

### 2. 最小二乘法：在充满误差的现实中“找折中”
现实中检测出的关键点存在误差，无法找到一个能让所有点完美重合的矩阵。**最小二乘法用于计算出一个“总误差最小的妥协矩阵”**。

**数值推演：**
假设要解极简缩放公式：$Y = a \cdot X$。检测到 3 个带有误差的点：
*   $X=1 \rightarrow Y=2$ （暗示 $a=2$）
*   $X=2 \rightarrow Y=4.2$ （暗示 $a=2.1$）
*   $X=3 \rightarrow Y=5.8$ （暗示 $a=1.933$）

1.  定义总误差方程（差值平方和）：
    $$Error = (a \cdot 1 - 2)^2 + (a \cdot 2 - 4.2)^2 + (a \cdot 3 - 5.8)^2$$
2.  展开并合并：
    $$Error = 14a^2 - 55.6a + 55.28$$
3.  求导找最低点（令其等于 0）：
    $$28a - 55.6 = 0 \implies a = 1.985$$
最小二乘法算出了最优解 $a = 1.985$，这正是 OpenCV 中 `estimateAffinePartial2D` 寻找最优矩阵的底层逻辑。

### 3. 透视变换矩阵：实现“近大远小”的三维降维打击
透视矩阵是 $3 \times 3$ 的。其核心在**第三行**，计算时会生成深度变量 $W$，最后一步**强制把算出的 $X$ 和 $Y$ 坐标除以 $W$**，实现梯形畸变校正。

**数值推演：**
假设有一透视矩阵（注意第三行的 $0.1$）：
$$M = \begin{bmatrix} 2 & 0 & 0 \\ 0 & 2 & 0 \\ 0.1 & 0 & 1 \end{bmatrix}$$

考察倾斜文本行同一高度的两个点：
*   **左侧点 A $(x=0, y=10)$：**
    *   $X' = 0, Y' = 20$
    *   深度 $W' = 0.1(0) + 0(10) + 1(1) = 1$
    *   **最终坐标：** $(0/1, 20/1) = \mathbf{(0, 20)}$。
*   **右侧较远点 B $(x=10, y=10)$：**
    *   $X' = 20, Y' = 20$
    *   深度 $W' = 0.1(10) + 0 + 1 = 2$ （X变大导致 W 变大）
    *   **最终坐标：** $(20/2, 20/2) = \mathbf{(10, 10)}$。
同高度的两点，右侧点因为更远被压缩回了 10。这种基于深度的不均匀缩放拉平了倾斜文档。

### 4. 图像仿射扭曲 (Warp) 与双线性插值
Warp 是“生成新白纸，遍历每个格子，反查公式去原图抄颜色（反向映射）”的过程。当反查到小数坐标时，需使用**双线性插值**。

**数值推演：**
1. 目标白纸遍历到像素 $(10, 10)$，反查逆矩阵发现对应原图坐标 $(1.2, 2.8)$。
2. 找到原图包围 $(1.2, 2.8)$ 的4个整数邻居及灰度值：
    *   左上 $P1(1, 2)$: 100 | 右上 $P2(2, 2)$: 110
    *   左下 $P3(1, 3)$: 200 | 右下 $P4(2, 3)$: 210
3. 插值计算：当前点距左边缘 $dx = 0.2$，距上边缘 $dy = 0.8$。
    *   上边缘混合：$100 \times (1-0.2) + 110 \times 0.2 = \mathbf{102}$
    *   下边缘混合：$200 \times (1-0.2) + 210 \times 0.2 = \mathbf{202}$
    *   纵向最终混合：$102 \times (1-0.8) + 202 \times 0.8 = \mathbf{182}$
4. 目标图像 $(10, 10)$ 处被填入颜色 **182**。

---

## 三、 基于坐标对齐的代码工程实现 (Python + OpenCV)

### 1. 人脸对齐：基于 5 关键点的相似变换

```python
import cv2
import numpy as np

def align_face(image, source_keypoints):
    """
    基于5关键点的人脸相似变换对齐 (ArcFace 112x112 标准)
    :param image: 原始图像 (H, W, 3)
    :param source_keypoints: 检测器输出的5个关键点坐标，shape (5, 2)
    :return: 112x112 的对齐人脸图像
    """
    # 1. 定义标准正脸模板坐标 (InsightFace 112x112 模板)
    target_keypoints = np.array([
        [38.2946, 51.6963],  # 左眼
        [73.5318, 51.5014],  # 右眼
        [56.0252, 71.7366],  # 鼻尖
        [41.5493, 92.3655],  # 左嘴角
        [70.7299, 92.2041]   # 右嘴角
    ], dtype=np.float32)
    
    source_keypoints = np.array(source_keypoints, dtype=np.float32)
    
    # 2. 计算相似变换矩阵 (estimateAffinePartial2D 计算最优缩放、旋转和平移)
    T, _ = cv2.estimateAffinePartial2D(source_keypoints, target_keypoints, method=cv2.LMEDS)
    
    # 3. 执行仿射变换重采样 (Warp)
    aligned_face = cv2.warpAffine(
        image, 
        T, 
        (112, 112), 
        flags=cv2.INTER_LINEAR,
        borderMode=cv2.BORDER_CONSTANT,
        borderValue=(0, 0, 0)
    )
    
    return aligned_face

import cv2
import numpy as np

def order_points(pts):
    """规范化四点顺序：左上、右上、右下、左下"""
    rect = np.zeros((4, 2), dtype="float32")
    x_sorted = pts[np.argsort(pts[:, 0]), :]
    left_most = x_sorted[:2, :]
    right_most = x_sorted[2:, :]
    rect[0] = left_most[np.argsort(left_most[:, 1])[0]]
    rect[3] = left_most[np.argsort(left_most[:, 1])[1]]
    rect[1] = right_most[np.argsort(right_most[:, 1])[0]]
    rect[2] = right_most[np.argsort(right_most[:, 1])[1]]
    return rect

def align_text(image, box_points):
    """
    基于4顶点多边形的透视变换文本截取
    :param image: 原始图像
    :param box_points: 任意四边形的4个坐标，shape (4, 2)
    :return: 拉平并裁剪出的矩形文本行图像
    """
    # 1. 规范化四点顺序
    rect = order_points(np.array(box_points, dtype="float32"))
    (tl, tr, br, bl) = rect

    # 2. 计算目标矩形的宽与高 (取对应边缘最大值)
    width_top = np.linalg.norm(tr - tl)
    width_bottom = np.linalg.norm(br - bl)
    max_width = max(int(width_top), int(width_bottom))

    height_left = np.linalg.norm(tl - bl)
    height_right = np.linalg.norm(tr - br)
    max_height = max(int(height_left), int(height_right))

    # 3. 构造目标标准矩形坐标
    target_pts = np.array([
        [0, 0],
        [max_width - 1, 0],
        [max_width - 1, max_height - 1],
        [0, max_height - 1]
    ], dtype="float32")

    # 4. 计算 3x3 透视变换矩阵
    M = cv2.getPerspectiveTransform(rect, target_pts)

    # 5. 执行透视变换重采样
    aligned_text = cv2.warpPerspective(
        image, 
        M, 
        (max_width, max_height),
        flags=cv2.INTER_LINEAR,
        borderMode=cv2.BORDER_REPLICATE # 边缘复制防黑边
    )

    return aligned_text
```
