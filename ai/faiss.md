
# faiss 介绍和原理
facebook开源：
  https://github.com/facebookresearch/faiss
- 向量k-NN搜索的计算库，其作用主要在保证高准确度的前提下大幅提升搜索速度
- 本质上是一个向量（矢量）数据库
- 主要流程包括train，add
- faiss 索引基本上是基于PQ或IVF
- faiss PQ均基于L2（欧式距离）
- 高效的k-means实现
- PCA降维算法


## 基本概念

Embedding嵌入：
 - 使得高维的原始数据被映射到低维流形之后变得可分
 - 低维流形的表征向量叫做Embedding
 - 原始数据提取出来的Feature
 - 神经网络映射之后的低维向量

线性变化：
- 不会改变原始数据的数值排序
- 把一个向量空间里的向量映射到了另一个向量空间里的另一个向量
- 操作：投影、伸缩、旋转

归一化：
 - 数据变换到一定的空间（0,1）或（-1,1）
 - 本质上是数据的线性变化
 - 提升模型的收敛速度，优化梯度下降，方便KNN算法，避免数值问题
 - 常用方法：Min-Max、平均值、对数等
 

相似度度量：
- 余弦consine ： 计算两个向量的夹角余弦值，值位于（-1,1）,越大则相似度越高；衡量的是向量在方向上的差异；
- 点积 ： 余弦的另外一种表达方式，方便计算；夹角为0时，点积最大；90度时，点积为0；
- 欧拉距离： 空间距离计算公式，值越小越相似；衡量空间上的绝对距离；

距离计算:
x 为待检索向量 
y 为库中向量

q(x) 为x对应的中心点
q(y) 为y对应的中性点

- 对称距离  q(x) 与 q(y)的距离
- 非对称距离 x 到 q(y)的距离

faiss 使用非对称距离

kNN：k-Nearest Neighboors k临近算法
- 解决分类问题
- 机器学习唯一无模型算法
- 将待分类样本，计算与其他样本的距离，排序，选出距离近的K个点，选出K个点中最多的分类，将数据归类

Product Quantizer，简称PQ，乘积量化，计算距离的方法和向量压缩
 - 将矢量编码为代码
 - 优化距离计算的速度
 - http://www.fabwrite.com/productquantization
 - 向量压缩：高维向量d切分为m段，每段用k-means算法生成k（一般为256）个中心点id（一般为8bit），d维向量被转换为m*8bit的向量码本（压缩）
 - 距离计算： x查询向量压缩得到x1，x与x1比较，得到簇心id表示的压缩向量x2，查询码表
 - 流程：
 - > 切分向量为m段
 - > 计算所有向量每段（共m段）的簇心，共256个
 - > 每组的簇心用id标识,2^8=256,每个id用8bit
 - > 每个向量的每段用簇心id表示，m个id正好可以表示一个向量；称为向量压缩
 - > 向量查询：
 - > 查询向量切分为m段；
 - > 计算查询向量每段到256个簇心的距离，每段的最近簇心id；组合成为该向量的编码；
 - > 查询码表，计算距离最短的码表对应的向量即最相似的向量
 
 ![训练](https://github.com/neoguojing/docs/blob/main/ai/PQ-train.png)
 
 ![检索](https://github.com/neoguojing/docs/blob/main/ai/PQ-search.png)
 
Inverted File System，简称IVF，基于kmeans
- 减少需要计算距离的目标向量个数
- 对库里所有向量做kmeans聚簇
- 向量转换为与簇心的残差
- 流程：
- > 为减少距离计算，先对所有向量分类，即计算簇心；查询向量则是计算到所有簇心的距离，然后再逐一计算到簇心下每个向量的距离；
- > 归一化，减小误差，则用向量减去簇心向量；
- > 可以配合使用PQ算法，进一步提升性能
![IVF](https://github.com/neoguojing/docs/blob/main/ai/IVF-IVF.png)

![IVF-PQ](https://github.com/neoguojing/docs/blob/main/ai/IVF-IVF-PQ.png)

维诺空间：

https://blog.csdn.net/kevin_darkelf/article/details/81455687

PCA降维： principal component analysis ( 主成分分析)
- 减少内存或者硬盘的使用，加快机器学习的速度
  


## 依赖

- MKL ： Intel数学核心函数库（MKL）是一套高度优化、线程安全的数学例程、函数，面向高性能的工程、科学与财务应用
- OpenBlas ： 基础线性代数子程序库 c++的numpy
- OpenMP ： 并行描述的高层抽象降低了并行编程的难度和复杂度，解决线程粒度和负载平衡，只适合共享内存，不支持集群

## faiss使用

### 索引构建

- 全量构建: 原始向量文件 -> train -> add -> faiss索引文件
- 增量构建: faiss索引文件 -> add ->  faiss索引文件   
- train 阶段: 在PQ索引中包含 : 1 Clustering 聚类  2.Asign 原始向量 转换为编码的过程

### 参数说明
- d ： 指定向量的维度
- nlist ： IVF划分的子搜索空间
- m：PQ算法降维参数，d是m的倍数
- nprobe ： 需要检索的聚类中心数量，控制精度/速度
- quantizer， 来给 Voronoi 单元格分配向量。 每个单元格由质心（centroid）定义。 找某个向量落在哪个 Voronoi 单元格的任务，就是一个在质心集合中找这个向量最近邻居的任务。 这是另一个索引的任务，通常是 IndexFlatL2
### 索引类型

FLAT 表示不压缩索引

- IndexFlatL2 欧式距离计算，暴力搜索 精确
- IndexFlatIP 点积计算    精确
- IndexIVFFLat 加聚类的索引  精确检索
- IndexPQ 向量压缩，特征编码，降低内存使用
- IndexIVFPQ 向量压缩+聚类   非精确 nprobe参数 调节精度和速度 不支持增量构建

### 使用
- add_with_ids 为每个向量建立一个64bit id(索引)
- reconstruct 取出原始特征
- add 添加索引
- train 训练索引
- search 检索

### GPU

- index_gpu_to_cpu可以将索引从GPU复制到CPU，
- index_cpu_to_gpu 和 index_cpu_to_gpu_multiple可以从CPU复制到GPU
- GpuIndexFlat, GpuIndexIVFFlat 和 GpuIndexIVFPQ
- GpuResources 资源对象,来避免无效的数据交互

