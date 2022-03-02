# client.go
- k8s.io/client-go


## 关键模块
###  cache
```
type ListerWatcher interface {
	Lister
	Watcher
}

type ListWatch struct {
	ListFunc  ListFunc
	WatchFunc WatchFunc
	// DisableChunking requests no chunking for this list watcher.
	DisableChunking bool
}
type Controller interface {
	// Run does two things.  One is to construct and run a Reflector
	// to pump objects/notifications from the Config's ListerWatcher
	// to the Config's Queue and possibly invoke the occasional Resync
	// on that Queue.  The other is to repeatedly Pop from the Queue
	// and process with the Config's ProcessFunc.  Both of these
	// continue until `stopCh` is closed.
	Run(stopCh <-chan struct{})

	// HasSynced delegates to the Config's Queue
	HasSynced() bool

	// LastSyncResourceVersion delegates to the Reflector when there
	// is one, otherwise returns the empty string
	LastSyncResourceVersion() string
}
```
- NewIndexerInformer
- NewSharedIndexInformer
- NewListWatchFromClient : 返回ListWatch对象；入参需要传入封装好的clientset.CoreV1().RESTClient()
- NewInformer : 返回一个本地cache和controller
- > NewStore
- > newInformer：
- threadSafeMap 一个本地缓存：使用lock和map
- > Indexers: 分类器：在原始数据上再构建一层map，相当于三级map
- > indices: 存储分类之后的数据
- > updateIndices: 删除indices老数据，并为新数据建立索引
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
- NewNamedRateLimitingQueue：实现将数据加入队列的功能，限制10qps的流量
- DefaultControllerRateLimiter：1.bucketlimit：10qps:限制重试的次数 
- BucketRateLimiter：使用golang/time限流包
- > When :返回拿到令牌需要的时间
- ItemExponentialFailureRateLimiter：指数限流器：完成：baseDelay * 2^元素失败的次数
- > When: 返回baseDelay * 2^元素失败的次数的值
- > NumRequeues: 返回failures map的长度
- > Forget: 从failures map 删除元素
- MaxOfRateLimiter：组合多个限流器
-  > When: 获取多个限流器中返回的等待时间最大的
-  > NumRequeues:同上
-  > Forget:依次调用所有的
- NewDelayingQueue： 实现将元素延时加入队列的功能
- 
