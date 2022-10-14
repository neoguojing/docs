# frame
- input_shape ： 张量的纬度信息，每一维的纬度
- input_dim：张量的纬度
- channel ：图片的通道数，RBG为3通道

## 特征工程
- 特征提取
## 神经网络
- 卷积神经网络提升精度： 1. 增加层数；2，增加宽度，即提升每层能够提取的特征数；3.增加图像分辨率

## pytorch
- EfficientNet:平衡深度、宽度和分辨率这三个维度，通过一组固定的缩放系数统一缩放这三个维度
- state_dict： torch.nn.Module模型的可学习参数（即权重和偏差）包含在模型的参数中
### checkpoint
- checkpoint检查点：不仅保存模型的参数，优化器参数，还有loss，epoch等（相当于一个保存模型的文件夹）
- 
