# client.go
- k8s.io/client-go


## 关键模块
###  cache
- NewIndexerInformer
- NewSharedIndexInformer
- NewListWatchFromClient
- NewInformer 
### listener
### record
- NewBroadcaster
### util/workqueue
- 接口定义
```
type Interface interface {
	Add(item interface{})
	Len() int
	Get() (item interface{}, shutdown bool)
	Done(item interface{})
	ShutDown()
	ShuttingDown() bool
}

type DelayingInterface interface {
	Interface
	// AddAfter adds an item to the workqueue after the indicated duration has passed
	AddAfter(item interface{}, duration time.Duration)
}

type RateLimitingInterface interface {
	DelayingInterface

	// AddRateLimited adds an item to the workqueue after the rate limiter says it's ok
	AddRateLimited(item interface{})

	// Forget indicates that an item is finished being retried.  Doesn't matter whether it's for perm failing
	// or for success, we'll stop the rate limiter from tracking it.  This only clears the `rateLimiter`, you
	// still have to call `Done` on the queue.
	Forget(item interface{})

	// NumRequeues returns back how many times the item was requeued
	NumRequeues(item interface{}) int
}

```
- 底层结构
```
type set map[t]empty
type Type struct {
	//元素的顺序表达；和dirty等价
	queue []t
	//需要被处理的元素
	dirty set
	//正在被处理的元素
	processing set
	cond *sync.Cond
	shuttingDown bool
	metrics queueMetrics
	unfinishedWorkUpdatePeriod time.Duration
	clock                      clock.Clock
}
```
- Add:锁定crond;shuttingDown则返回；dirty中以及存在则返回；不存在则放入dirty；processing存在则返回；将item加入queue；发送信号唤醒一个Get；
- Get(item)操作：特性：必定会获取特定值：锁定crond；若队列为空，则在条件变量上等待被唤醒；否则返回queue[0],并重新赋值queue，插入processing和在dirty中删除元素
- Done(item)：锁定crond，从processing删除；若dirty中依然存在，则插入到queue末尾，并发送信号唤醒一个Get函数
- ShutDown：设置shuttingDown标记，并使用cond的广播唤醒所有阻塞的线程
- ShuttingDown： 获取ShuttingDown的标记
- AddAfter ： 有duration；小于等于0则直接放入队列；否则发送到waitingForAddCh一个channel中等待waitingLoop处理
- waitingLoop：一个大循环
- > 监听waitingForAddCh，若时间未到，则放入到优先队列（一个小顶堆）和一个map中（循环尝试将channel里数据读完）；
- > 遍历优先队列：堆顶超时则出队加入队列，否则跳出当前循环；
- > 计算堆顶的超时时间，并建立定时器；使用select监听定时器，直到超时
- AddRateLimited：调用AddAfter，并用rateLimiter计算加入时间
- NumRequeues：
- newQueue ： 真正的队列创建接口
- NewNamedRateLimitingQueue
- NewDelayingQueue
- 
