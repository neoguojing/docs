# meta 技术报告

## 开发主要阶段：
### 预训练： 预测和总结
（1）大规模训练语料库的策划和过滤，（2）模型架构的开发以及确定模型规模的对应扩展规律，（3）大规模高效预训练技术的开发，以及（4）预训练方法的开发
### 后训练： 指令跟随、人类对齐、推理等

## 关键点：
### 数据： 提升数据质量
#### 网络数据处理： 
1.敏感信息、成人信息过滤
2.html数据抽取和清洗
3.数据去重：MinHash，n-gram行数据去重，Kullback-Leibler (KL) divergence，简称KL散度，是一种衡量两个概率分布之间差异的非对称性度
4.基于模型的数据过滤,去除低质量文本:fasttext,RoBERTa,llama2,DistilRoBERTa
#### 数据混合：不同知识的比例 50%的通用知识，25的数学和推理，17的code和8的多语言
1.分类模型知识分类；
2.训练小模型评估数据混合的质量
#### 数据退火
对特定的数据集进行上采样，可以提高相关的性能
### 大规模训练：g 3.8 × 1025 FLOPs
### 复杂性控制：使用SFT、RS 和DPO，而不是混合专家模型

## 模型架构
### GQA：增加推理速度，减少缓存
### 词典为128k token 100k英语token和28k非英语token
### RoPE 的频超参数调整为500,000
### 小模型和大模型的区别： 层数不同，模型纬度不同，前馈网络纬度和注意力头数不同

## Scaling Laws
### 寻找token数量和计算量直接的关系

## 训练效率提升
### 16k GPU，每机8卡，显存80GB
### 核心：算力、存储和网络
### 技术术语： HBM3，RDMA over Converged Ethernet， Equal-Cost Multi-Path， Enhanced-ECMP 
### 并行化训练：tensor parallelism，pipeline parallelism，context parallelism，data parallelism
### tensor parallelism： 分割独立的权重到不同机器的不同chuck
### pipeline parallelism：分割模型的不同层到不同的机器
### context parallelism： 将输入分段，减少内存瓶颈

## 预训练：
### 初始化
warm up 8000步
120万训练步骤
逐步增加batch
### 长上下文训练
逐步的增大输入序列从8k -> 128k，经过6轮
### 淬炼：提高数据质量，降低学习率

## 后训练
主干是一个奖励模型和语言模型
通过人工标注数据训练奖励模型
SFT微调模型
DPO进行模型对齐
