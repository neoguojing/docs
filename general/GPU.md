# GPU

图形绘制 

物理模拟：GPU硬件集成的物理引擎（PhysX、Havok）

海量计算:计算着色器及流输出的出现，为各种可以并行计算的海量需求得以实现，CUDA就是最好的例证

AI运算:

NVIDIA:

GeForce
GTX
RTX

AMD:
Radeon


显卡构成：
GPU，显卡还有扇热器、通讯元件、与主板和显示器连接的各类插槽

架构：
GPCs(Graphics Processing Cluster)
TPC（Texture/Processor Cluster，纹理处理簇）
SM（Stream Multiprocessor，流多处理器）
SP（Streaming Processor，流处理器）
L1缓存、MT Issue（多线程指令获取）、C-Cache（常量缓存）、共享内存
SFU（Special Function Unit，特殊函数单元）
FPU（浮点数单元）
ALU（逻辑运算单元）
Warp（线程束）
ROP(render output unit，渲染输入单元)
SIMD（Single Instruction Multiple Data）是单指令多数据
SIMT（Single Instruction Multiple Threads，单指令多线程）是SIMD的升级版，可对GPU中单个SM中的多个Core同时处理同一指令，并且每个Core存取的数据可以是不同的。

级缓存结构：寄存器、L1缓存、L2缓存、GPU显存、系统显存

分离式架构，CPU和GPU各自有独立的缓存和内存，它们通过PCI-e等总线通讯。这种结构的缺点在于 PCI-e 相对于两者具有低带宽和高延迟，数据的传输成了其中的性能瓶颈。目前使用非常广泛，如PC、智能手机
耦合式架构，CPU 和 GPU 共享内存和缓存。AMD 的 APU 采用的就是这种结构，目前主要使用在游戏主机中

GPU Context

GPU Context代表了GPU计算的状态。
在GPU中拥有自己的虚拟地址。
GPU 中可以并存多个活跃态下的Context。

CPU-GPU数据流
下图是分离式架构的CPU-GPU的数据流程图：

1、将主存的处理数据复制到显存中。

2、CPU指令驱动GPU。

3、GPU中的每个运算单元并行处理。此步会从显存存取数据。

4、GPU将显存结果传回主存。

CPU将计算好显示内容提交至 GPU，GPU 渲染完成后将渲染结果存入帧缓冲区，视频控制器会按照 VSync 信号逐帧读取帧缓冲区的数据，经过数据转换后最终由显示器进行显示。
