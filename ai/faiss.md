
# faiss 介绍和原理
facebook开源：
  https://github.com/facebookresearch/faiss
- 向量k-NN搜索的计算库，其作用主要在保证高准确度的前提下大幅提升搜索速度

## 基本概念

kNN：k-Nearest Neighboors k临近算法
- 解决分类问题
- 机器学习唯一无模型算法

Product Quantizer，简称PQ
 - 将矢量编码或解码为代码
 - 优化距离计算的速度
 - 把连续的空间离散化
 
Inverted File System，简称IVF，基于kmeans


## 依赖

MKL ： Intel数学核心函数库（MKL）是一套高度优化、线程安全的数学例程、函数，面向高性能的工程、科学与财务应用
OpenBlas ： 基础线性代数子程序库 c++的numpy
OpenMP ： 并行描述的高层抽象降低了并行编程的难度和复杂度，解决线程粒度和负载平衡，只适合共享内存，不支持集群


