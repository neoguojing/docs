# 人脸识别 Pipeline

## 1. 人脸检测 Face Detection

### 做什么
定位图片中的人脸。

### 输入
原始图片 / 视频帧

### 输出
bbox、score、class

### 常见模型
Faster R-CNN、RetinaNet、YOLO、RetinaFace、SCRFD
两阶段检测：RPN
---

## 2. 人脸关键点 Landmark

### 做什么
定位眼睛、鼻子、嘴角等关键点。

### 输入
人脸 bbox

### 输出
5/68/106 等 landmark

### 用途
- 人脸对齐
- 姿态估计
- 质量评估

---

## 3. 人脸质量评估

### 做什么
判断当前人脸是否适合识别。

### 常见指标
- 清晰度
- 人脸大小
- 姿态角
- 遮挡
- 光照
- 完整性

### 模糊度
传统：
- Laplacian
- Sobel
- FFT

深度学习：
- 分类模型
- Quality Score 模型

---

## 4. 人脸姿态估计

### 输出
- Yaw：左右转头
- Pitch：抬头/低头
- Roll：左右歪头

### 方法
- landmark + 几何方法
- 深度学习回归

---

## 5. 人脸对齐 Face Alignment

### 输入
人脸 + landmark

### 做什么
通过旋转、缩放、平移，将人脸统一到标准位置。

### 常见方法
Similarity Transform / Affine Transform

### 输出
例如 112×112 标准人脸

---

## 6. 帧选择

主要用于视频。

```text
多帧人脸
 ↓
质量评估
 ↓
选择质量最高的人脸
 ↓
进入特征提取
