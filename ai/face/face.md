# face

## 一般流程
- 人脸检测：基于faster-rcnn 、retinanet; 输入图片，输出：分数、图片id、标签以及roi
- 人脸关键点提取：                   输出：关键点坐标
- - 旨在识别和定位人脸上的特定点（如眼睛、鼻子、嘴巴和脸部轮廓）
  - Cascade CNN (级联卷积神经网络)，Hourglass Network (沙漏网络)， Heatmap Regression (热力图回归)， 3D Dense Face Alignment (3D 稠密对齐)
  - 常用数据集：68关键点标注，人脸遮挡数据集、各种外部环境和姿态数据集
- 人脸模糊度评估： 输入：图片和人脸区域  输出：模糊分数
- - 图像梯度、频域、清晰度分类模型
- 人脸角度评估： 输入：关键点   输出: pitch,roll和yaw
- - 回归模型、几何方法
- 帧选择
- 结果分析
- - 人脸特征提取： 输入： 关键点  输出：分数和特征
  - - 深度学习一般基于神经网络提取特征
  - 人脸属性提取： 输入： 关键点 输出：人脸属性和分数 性别、年龄、表情
  - - 分类模型
