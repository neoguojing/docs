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
- > commit-log : 每次写之前都先写，可以分段，压缩、删除和滚动
- > memtale:内存写缓存
- > sstable：LSM文件结构，支持文件压缩；包含数据，索引，boomfilter等
- 多主副本技术：数据版本 和可调一致性；每条数据又个版本，采用last-win策略，解决数据冲突保证最终一致性；数据通过一定的copy策略，copy到不同的物理机器上；
- > 通过https://www.cnblogs.com/fengzhiwu/p/5524324.html hash数校验每个副本的完整性
- > 可调节的一致性：类似kafka，是等待所有副本回应还是部分副本回应
- 集群发现和失败检测：gossip
- > 失败检测不会删除节点
- 只能保证最终一致性
- 

### 副本copy策略
- NetworkTopologyStrategy
- SimpleStrategy
