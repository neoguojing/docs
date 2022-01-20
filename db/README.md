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

|       |  存储结构   | 顺序写  | copy on write |索引|锁|高可用|分布一致性|
|  ---- |  ----  | ----  | ----  | ----  | ----  | ----  | 
| Cassandra  |  |||||||
| mysql  | B+树 | change buffer使得二级索引修改不是随机写| | B+| 行锁，表锁，意向锁、间隙索| binlog/redo log|隔离级别|
| mongo  | 单元格 |
| HDFS  | 单元格 |
| HBASE  | 单元格 |
| level DB  | 单元格 |
| clickhose  | 单元格 |
| kafka  | 单元格 |
| ElasticSearch  | 单元格 |
| InfluxDB  | 单元格 |
| Redis  | 单元格 |
## 常用技术

- 顺序写
- 写时复制
- 索引
- 分布式
- 锁
- 高可用
