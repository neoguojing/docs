# 存储&&数据库

## 统计
- Cassandra
- mysql
- mongo
- HDFS
- HBASE
- level DB
- Redis
- clickhose
- kafka
- ElasticSearch
- InfluxDB

|       | 内存数据库 |  磁盘数据页  | 顺序写  | copy on write |索引|锁|高可用|分布一致性|
|  ---- |  ----  | ----  | ----  | ----  | ----  | ----  | ----  | ----  | 
| mysql | buffer pool LRU缓存已访问数据| B+树 | change buffer使得二级索引修改不是随机写| | B+| 行锁，表锁，意向锁、间隙索| binlog/redo log|隔离级别|
| level DB | 1.memtable跳跃列表 2.immutable memtable只读 | sstable lsm-tree manifest保存文件分布信息 不活跃数据下沉| log文件顺序写| | | |通过log文件记录操作用于恢复| ||
| ElasticSearch  |  | 文档id增量压缩技术和（商，余数）压缩技术 | | | term index 前缀树，记录关键字典的起始位置，term dict关键字排序二分查找||||
| HDFS  |   | 文件拆分（128M）存储，多副本，gossip数据复制|||NameNode 存储文件分布信息||||
| HBASE  | memstore 跳跃列表数据有序；blockcache：LRU 读缓存 | HFile负责存储数据：包含索引系统，bloomfilter| memstore数据有序，写入磁盘顺序写||行锁|分布在每个region的WAL日志，用于崩溃恢复；数据文件使用HDFS，有多个副本|事务的强一致性：通过锁和MVCC实现并发控制；行锁和两阶段提交协议（获取所有行锁才写入）；|
| Cassandra  | 单元格 |
| mongo  | 单元格 |

| clickhose  | 单元格 |
| kafka  | 单元格 |
| InfluxDB  | 单元格 |
| Redis  | 单元格 |
## 常用技术

- 顺序写
- 写时复制
- 索引
- 分布式
- 锁
- 高可用
