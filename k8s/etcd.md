# ETCD

## 选举要点：
- 心跳超时，则置自己为候选者，向其他节点索要投票
- 半数节点通过则成为leader
## 日志复制
- leader接收客户端请求，生成日志，并复制给其他follwer
- 超过半数回复ACK，则设置为已提交，并存储到本地
- 在下一个心跳周期，通知follower持久化

## 数据一致性
- term任期概念，只有任期较大的才能成为leader
- log被提交后，新的leader必须包含该log才能作为leader

## WAL日志
- type：nomal和ConfChange（配置变更日志）
- term：任期
- index：严格递增的
- data

## 版本对比
### v2
- 数据以树结构，存储在内存，每个节点都存了过期时间，定期清理
- currentIndex：全局唯一索引，每次变更都加1
- Watchhub管理key和订阅列表；watcher的chan缓存100个数据，阻塞则该watcher会被删除，需要重新连接；每个watcher只能watch一个key不能多key watch
- EventHistory：管理事件队列和起止索引；最多存储1000个事件，超过会丢失
### v3
- 存储：基于kvindex（btree的go开源实现）的内存索引；基于backend的后端存储
- backend：存储接口，支持多种持久化数据库，默认使用boltdb
- boltdb：kv数据库，etcd的事务基于boltdb事务的实现；存储方式：k：{main revsion（事务加1），sub rev（同一事务操作加1）} v：etcd的kv
- 内存中仅存储key
- watchGroup：包含两种订阅方式：1.基于key订阅；2.基于范围的订阅（基于IntervalTree实现），便于查询某段区间的内的订阅
- WatchableStore： 包含两种watcherGroup，synced：表示watcher的数据已经全部推送完毕；unsynced：watcher还在追赶最新数据
- 后台线程负责将unsync转移到sync
- 事件数量没有限制，同时维护一个推送时阻塞的watcher队列，使用另外的协程重试
- lease：支持多个lease关联多个key

- revision：全局唯一版本号，分为create revision和mod revision等
- version：一个修改次数计算器
- 
