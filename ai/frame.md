# frame
- input_shape ： 张量的纬度信息，每一维的纬度
- input_dim：张量的纬度
- channel ：图片的通道数，RBG为3通道

## 工具：
- pip install netron ： 查看网络的输入和输出
## CV算法
- 计算机视觉的三大任务
- > 图片分类，物体定位（roi），目标检测（位置和类别）

## 特征工程
- 特征提取
## 神经网络
- 卷积神经网络提升精度： 1. 增加层数；2，增加宽度，即提升每层能够提取的特征数；3.增加图像分辨率
- ResNet: 残差网络，在普通网络上加如跳跃层，主要解决： 增加网络深度的退化问题：深度达到一定成都，精度饱和之后快速下降
- YOLOP： 全景驾驶感知，同时处理目标检测、可行驶区域分割、车道线检测 
- > 编码器： Backbone网络 提取图片特征，Neck网络：生成多尺度特征
- > Decoders 解码器: 目标检测部分、可行驶区域分割、车道线分割
- Backbone网络:提取特征的网络，通常如resnet VGG，提取特征能力较强
- head网络： 是获取网络输出内容的网络，head利用这些特征，做出预测
- Neck网络：放在backbone和head之间的
- bottleneck:瓶颈的意思，通常指的是网网络输入的数据维度和输出的维度不同，输出的维度比输入的小了许多，就像脖子一样，变细了。经常设置的参数 bottle_num=256，指的是网络输出的数据的维度是256 ，可是输入进来的可能是1024维度的
- GAP：在设计的网络中经常能够看到gap这个层，我之前不知道是干啥的，后了解了，就是Global Average Pool全局平均池化，就是将某个通道的特征取平均值，经常使用
- Embedding: 深度学习方法都是利用使用线性和非线性转换对复杂的数据进行自动特征抽取，并将特征表示为“向量”（vector），这一过程一般也称为“嵌入”（embedding）
- End-To-End的方案，即输入一张图，输出最终想要的结果，算法细节和学习过程全部丢给了神经网络
- YOLOv4:
##  目标检测
- 通用做法：通常是利用滑动窗口进行选取目标候选框，然后利用一些算法进行特征提取，最后再扔到分类器中去检测分类
- FCN： 全卷积神经网络（FCN）；让已经设计好的网络可以输入任意大小的图片
- > CNN中的全连接层换成了卷积操作,1x1的卷积其实就相当于全连接操作
- > 用于目标检测加速：
## pytorch
- EfficientNet:平衡深度、宽度和分辨率这三个维度，通过一组固定的缩放系数统一缩放这三个维度
- state_dict： torch.nn.Module模型的可学习参数（即权重和偏差）包含在模型的参数中
- torchvision: 提供视觉模型和相关工具
### checkpoint
- checkpoint检查点：不仅保存模型的参数，优化器参数，还有loss，epoch等（相当于一个保存模型的文件夹）
- 
