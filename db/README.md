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
| ElasticSearch  | 单元格 | | | | term index 前缀树，记录关键字典的起始位置，term dict关键字排序二分查找||||
| Cassandra  | 单元格 |
| mongo  | 单元格 |
| HDFS  | 单元格 |
| HBASE  | 单元格 |
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
