# pool

- 1.13之后sync.pool 不会被垃圾回收，替代策略是：回收两个gc之内未被使用的实例

## 实现
```
func init() {
   runtime_registerPoolCleanup(poolCleanup)
}

func gcStart(trigger gcTrigger) {
   [...]
   // clearpools before we start the GC
   clearpools()
```
- 每个p创建一个poolLocal ,包含private和share两个缓存
- 优先从private找对象，找不到，去shared找（加锁）
- 没有则调用注册的create函数创建一个
- put: 若private没有，则放入private，否则放入shared
- shard无锁设计：双向列表：local p从队列头插入和删除，其他p从末尾删除；若需要新创建item，则基于原来个数翻倍，第一次8，第二次16
- 回收设计：两个pool集合：1.活跃的；2.归档的
