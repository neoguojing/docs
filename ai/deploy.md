# 模型部署

- 训练（training）包含了前向传播和后向传播两个阶段，针对的是训练集。训练时通过误差反向传播来不断修改网络权值（weights）。
- 推理（inference）只包含前向传播一个阶段，针对的是除了训练集之外的新数据。可以是测试集，但不完全是，更多的是整个数据集之外的数据。其实就是针对新数据进行预测，预测时，速度是一个很重要的因素。
## 部署问题
- 必须使用训练的平台和环境：一般的深度学习项目，训练时为了加快速度，会使用多GPU分布式训练。但在部署推理时，为了降低成本，往往使用单个GPU机器甚至嵌入式平台（比如 NVIDIA Jetson）进行部署，部署端也要有与训练时相同的深度学习环境，如caffe，TensorFlow等。
- 占用空间大速度慢：由于训练的网络模型可能会很大（比如，inception，resnet等），参数很多，而且部署端的机器性能存在差异，就会导致推理速度慢，延迟高。这对于那些高实时性的应用场合是致命的，比如自动驾驶要求实时目标检测，目标追踪等。

## 解决方案：
- squeezenet，mobilenet，shufflenet等：基于现有的经典模型提出一种新的模型结构，然后用这些改造过的模型重新训练，再重新部署
- tensorRT： 推理优化器。当你的网络训练完之后，可以将训练模型文件直接丢进tensorRT中，而不再需要依赖深度学习框架（Caffe，TensorFlow等）

## tensorRT：
- 直接解析caffe和TensorFlow的网络模型
- caffe2，pytorch，mxnet，chainer，CNTK等框架：转换为ONNX，对ONNX模型做解析
- 基本上比较经典的层比如，卷积，反卷积，全连接，RNN，softmax等，在tensorRT中都是有对应的实现方式的，tensorRT是可以直接解析的。
- tensorRT中有一个 Plugin 层，这个层提供了 API 可以由用户自己定义tensorRT不支持的层
### 如何优化的
- 层间融合或张量融合：横向合并可以把卷积、偏置和激活层合并成一个CBR结构，只占用一个CUDA核心；纵向合并可以把结构相同，但是权值不同的层合并成一个更宽的层，也只占用一个CUDA核心
- 数据精度校准：训练神经网络时网络中的张量（Tensor）都是32位浮点数的精度；一旦网络训练完成，在部署推理的过程中由于不需要反向传播，完全可以适当降低数据精度，比如降为FP16或INT8的精度。更低的数据精度将会使得内存占用和延迟更低，模型体积更小。
- Kernel Auto-Tuning：对不同的计算进行优化
- Dynamic Tensor Memory：在每个tensor的使用期间，TensorRT会为其指定显存，避免显存重复申请，减少内存占用和提高重复使用效率。
- Multi-Stream Execution
### 具体流程：
- 输入：模型文件，权值文件（为了解析模型时使用），标签文件（推理输出时将数字映射为有意义的文字标签）
- 编译：将caffe，tensor模型编译为tensorRT，输出plan file
- 部署：编写代码，加载模型，输入图片，得到结果；需要配合cuda编程完成工作
## ONNX 是微软和Facebook携手开发的开放式神经网络交换工具 开放神经网络交换
- https://github.com/onnx/
- 存储了神经网络模型的权重
- 存储了模型的结构信息以及网络中每一层的输入输出和一些其它的辅助信息
- protobuf存储模型信息，存储结构定义在protobuf中
### android环境
- NCNN 转换工具转换onnx

## 推理加速
```
下面是关于OpenBLAS、BLIS、MKL和cuBLAS的简要介绍：

OpenBLAS（Open Basic Linear Algebra Subprograms）：OpenBLAS是一个开源的基础线性代数子程序库，提供了高性能的基本线性代数运算函数，如矩阵乘法、矩阵-向量操作等。它被广泛用于科学计算、机器学习和深度学习等领域。OpenBLAS针对多核CPU进行了优化，支持多线程并行计算。它是一个跨平台的库，可以在Linux、Windows和其他操作系统上使用。

BLIS（BLAS-like Library Instantiation Software）：BLIS是一个可实例化的类BLAS（Basic Linear Algebra Subprograms）库框架。它提供了一组C语言接口，用于执行基本线性代数运算，如矩阵乘法、矩阵-向量操作等。BLIS的设计目标是提供高性能、可移植和可扩展的线性代数操作。它支持多种架构和处理器，并使用优化技术来提高运算速度。BLIS可以作为一个库被链接到应用程序中，也可以通过编译器支持进行自动实例化。

MKL（Intel Math Kernel Library）：MKL是英特尔提供的数学核心库，旨在优化数值计算和科学计算应用的性能。MKL提供了广泛的数学函数和线性代数子程序，包括BLAS、LAPACK、FFT、向量数学函数等。MKL针对英特尔处理器进行了高度优化，利用了硬件特性和并行计算能力，以提供高性能的数学计算功能。MKL支持多个操作系统和编程语言，包括C/C++、Fortran、Python等。

cuBLAS（CUDA Basic Linear Algebra Subroutines）：cuBLAS是NVIDIA CUDA平台上的基本线性代数子程序库。它提供了一组GPU加速的线性代数操作，如矩阵乘法、矩阵-向量操作等。cuBLAS是专门为在CUDA设备上进行高性能并行计算而设计的，利用了GPU的并行计算能力来加速线性代数运算。它可以与CUDA C/C++编程一起使用，并通过CUDA API进行调用。
```
