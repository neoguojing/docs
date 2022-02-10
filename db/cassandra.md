# cassandra
Dynamo ：分布式k-v存储系统
## 特性
- 融合hbase和Dynamo
- 多主copy
- 全局的可用性和第延迟
- 性能与处理器线性增长
- 实时负载均衡和集群扩容
- kv访问

## 概念
- keyspace: 类似db
- table：包含partition
- partition： 包含row
- row:包含collume
- column：

## 架构
- 数据集分割采用：一致性hash；每个分片的副本被存储在多个节点上；
- 磁盘存储结构：采用LSM
- 多主副本技术：数据版本 和可调一致性
- 集群发现和失败检测：gossip
- 
