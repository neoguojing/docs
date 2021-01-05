
# faiss 介绍和原理

## 基本概念
Product Quantizer，简称PQ，将矢量编码或解码为代码，PQ优化了向量距离计算的过程
Inverted File System，简称IVF，给予kmeans


## 依赖

MKL ： Intel数学核心函数库（MKL）是一套高度优化、线程安全的数学例程、函数，面向高性能的工程、科学与财务应用
OpenBlas ： 基础线性代数子程序库 c++的numpy
OpenMP ： 并行描述的高层抽象降低了并行编程的难度和复杂度，解决线程粒度和负载平衡，只适合共享内存，不支持集群
