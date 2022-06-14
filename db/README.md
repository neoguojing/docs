# 存储&&数据库
- OLTP（on-line transaction processing）翻译为联机事务处理；主要用来记录某类业务事件的发生，如ERP，OA等系统；OLTP是数据库的应用
- OLAP（On-Line Analytical Processing）翻译为联机分析处理；依赖于OLTP；OLAP是数据仓库应用
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
| kafka  | 单元格 |每个分区对应一个log文件，log文件分段存储；对应两个索引文件；文件以最开始的索引命名 |消息写入在文件后面追加/mmap直接将内容写入文件 |单元格 |单元格 |单元格 | 分区可以对topic水平扩展，存储不同的消息，因此多个分区消费是无序的；key定期压缩保留最新key;日志多副本 | leader副本响应请求；维护已同步副本；follwer定期从leader获取日志 |
| InfluxDB  | 单元格 |
| Redis  | 单元格 |
## 常用技术
### 数据分片
### 内存数据库
#### memtable 
代表：leveldb hbase Cassandra
- > 达到一定大小转为不可写
- > 一般使用跳跃列表排序；所以数据有序

### 磁盘存储
- 顺序写：kafka，hbase，leveldb的log
- mmap：内存和文件关联，无序copy：kafka
- lsm-tree: leveldb cassandra 适用于nosql数据库
- > 支持顺序写磁盘，支持文件压缩，
- B-tree ：磁盘是随机写，效率低
### 索引
- B+: mysql
- hash: Cassandra
### 分布式架构
- 一致性hash：cassandra
- 行数据分布存储：hbase 将行拆分为不同region，分布在不同的Region Server上；行拆分为列进行存储
### 高可用
#### 写操作记录
- mysql ：redo log
- redis ： AOF：会将写命令以某种方式放入AOF_BUF,然后由刷入磁盘；可以一秒一次或者每个命令一次；
- > AOF重新：fork子进程，子进程有数据库信息，依据数据库信息写入心的AOF文件；父进程继续接收请求，将命令写入老的AOF和AOF重写缓存；子进程写完发送信号，父进程收到信号，阻塞写操作，将AOF缓存的内容写入新文件，然后用新文件替换旧文件；
- level db ：WAL
- hbase：每个region一个的WAL

#### 快照
- mysql ： binlog
- redis：RDB:采用二进制方式保存数据；恢复快，但是又丢失数据风险；

### 事务
