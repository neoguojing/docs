# innodb
- 崩溃恢复
- buffer pool 缓存index和表；80%的内存用于cache
- 自适应hash索引，缓存索引页，加速热表查询
- 

## MVCC
- 解决在事务中特定隔离级别（RR）中的读一致问题
- 允许不加锁的并发读
### 如何实现
- 回滚段：保存某个事务的回滚信息；1.用于回滚；2.用于并发读的版本控制
- 数据行添加三个字段：1. DB_TRX_ID：保存该行数据最好被修改的事务id；2. DB_ROLL_PTR：指向一条undo log记录，包含恢复之前数据需要的操作；3. DB_ROW_ID：自增的行id

## 架构

### 内存架构
- Buffer Pool：缓存表和索引；以列表的方式组织页数据；采用LRU算法淘汰不被访问的内存；5/8用于保存被访问的；3/8属于old list；新加入的数据被插入交界处
- Change Buffer：避免修改二级索引（通常不连续）引起的随机写
- > 缓存不在buffer pool中的二级索引页
- > 修改的二级索引页会定期与buffer pool中的索引页合并；buffer pool则会被purge到磁盘上
- > Change Buffer 占用部分buffer pool；磁盘上页有持久化
-  Adaptive Hash Index： 自适应hash索引：加速热数据查询；在like查询时无益处；通过索引key的前缀或者b-tree的值，建立到buffer pooll中缓存页面的指针映射
-  Log Buffer：缓存即将存盘的redo log；默认16兆
### 磁盘架构
- 表格数据：.frm
- 索引数据：
- system tablespace：包含metadata for InnoDB-related objects， doublewrite buffer, the change buffer, and undo logs 和部分系统表和索引
- Undo Tablespaces ： 也额可以保存undolog
- Doublewrite Buffer
- Redo Log：
- Undo Logs：
