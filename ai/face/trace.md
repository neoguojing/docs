# trace

## 概念
- 卡尔曼滤波器（Kalman Filter, KF）**是一种用于线性动态系统状态估计的递归算法，特别适合噪声和不确定性环境下的状态预测和估计
### 追踪
- 目标轨迹：一般需要包含，位置信息、速度信息、时间和置信度等信息
- 输入:
- - 多帧连续的视频帧。
- - 每帧检测出的目标的边界框（bounding box）、特征（align feature）。
- 输出:
- 每个目标在多帧之间的轨迹。一般有个跟踪id
- 核心组件:
- - 目标检测模块：检测目标并输出其边界框和特征。
- - 轨迹预测模块：基于卡尔曼滤波器预测目标位置。
- - 相似度计算模块：通过特征相似性、边界框距离、大小变化计算匹配分数。
- - 目标分配模块：采用匈牙利算法（KM算法）完成目标与轨迹的关联。
- - 轨迹管理模块：处理轨迹的创建、更新、删除。
- 卡夫慢滤波器：
- - 平滑目标的运动轨迹并降低噪声影响
- - 对目标在当前帧的位置进行预测，以便在下一帧中快速和检测结果进行匹配
  - 在目标遮挡等情况下可以正确预测合理位置
  - 输入： 状态向量：坐标、速度、大小，测量值，协方差矩阵
  - 输出： 预测状态，修正状态和误差协方差矩阵
- 匈牙利算法（最大二分图）：
- - 解决轨迹和目标的匹配问题
  - 输入： 轨迹的集合，目标集合，计算一个代价矩阵
  - 输出： 最优匹配结果，矩阵中，是的代价最小
### 选帧
- 选帧的目的：基本假设：目标出现5s，视频帧每秒25帧，图片相似度高，浪费算力；一般选帧5帧
- 如何选帧？ 角度、模糊度
- 问题1：不能按质量分排序，因为相邻的图片相似度高
- 问题2: 抽样不同位置的帧
- 选帧时机：目标离开之后，从全局选帧；设置返回时间，超时则立即返回
- 缓存：缓存目标信息，目标小图和场景大图；可以设计GPU缓存池，降低图像分辨率
- 选帧逻辑：动态选帧（每一帧都更新缓存和选帧），强制选帧：跟踪结束或者超时
- 选帧策略：排序选帧topk；分段选帧，基于分段再选帧
- 必要参数：最大跟踪目标数、各个跟踪目标的数量、进入画面到选帧的时间（用于快速选帧）、分段选帧的触发时间
- 架构：每个流绑定一个选帧器，每个选帧器需要缓存不同的目标的抓拍
