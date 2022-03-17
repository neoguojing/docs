# client.go
- k8s.io/client-go


## 关键模块
### 关键概念
- DeletedFinalStateUnknown： 对象被删除，但是watch deletion时间丢失；此时对象的状态为这个
- ExplicitKey： 只有key，没有obj
- metav1.Object ： 对象接口，如StatefulSet和Deployment均实现了该接口
- api/core/v1/types.go：定义了Node，Volume等对象；这些类均实现了runtime.Object接口；Informer传入这些对象用于分类
### Informer
- 包含三个部分：Reflector、DeltaFIFO、LocalStore
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

type SharedInformer interface {
	// AddEventHandler adds an event handler to the shared informer using the shared informer's resync
	// period.  Events to a single handler are delivered sequentially, but there is no coordination
	// between different handlers.
	AddEventHandler(handler ResourceEventHandler)
	// AddEventHandlerWithResyncPeriod adds an event handler to the
	// shared informer with the requested resync period; zero means
	// this handler does not care about resyncs.  The resync operation
	// consists of delivering to the handler an update notification
	// for every object in the informer's local cache; it does not add
	// any interactions with the authoritative storage.  Some
	// informers do no resyncs at all, not even for handlers added
	// with a non-zero resyncPeriod.  For an informer that does
	// resyncs, and for each handler that requests resyncs, that
	// informer develops a nominal resync period that is no shorter
	// than the requested period but may be longer.  The actual time
	// between any two resyncs may be longer than the nominal period
	// because the implementation takes time to do work and there may
	// be competing load and scheduling noise.
	AddEventHandlerWithResyncPeriod(handler ResourceEventHandler, resyncPeriod time.Duration)
	// GetStore returns the informer's local cache as a Store.
	GetStore() Store
	// GetController is deprecated, it does nothing useful
	GetController() Controller
	// Run starts and runs the shared informer, returning after it stops.
	// The informer will be stopped when stopCh is closed.
	Run(stopCh <-chan struct{})
	// HasSynced returns true if the shared informer's store has been
	// informed by at least one full LIST of the authoritative state
	// of the informer's object collection.  This is unrelated to "resync".
	HasSynced() bool
	// LastSyncResourceVersion is the resource version observed when last synced with the underlying
	// store. The value returned is not synchronized with access to the underlying store and is not
	// thread-safe.
	LastSyncResourceVersion() string

	// The WatchErrorHandler is called whenever ListAndWatch drops the
	// connection with an error. After calling this handler, the informer
	// will backoff and retry.
	//
	// The default implementation looks at the error type and tries to log
	// the error message at an appropriate level.
	//
	// There's only one handler, so if you call this multiple times, last one
	// wins; calling after the informer has been started returns an error.
	//
	// The handler is intended for visibility, not to e.g. pause the consumers.
	// The handler should return quickly - any expensive processing should be
	// offloaded.
	SetWatchErrorHandler(handler WatchErrorHandler) error
}
type SharedIndexInformer interface {
	SharedInformer
	// AddIndexers add indexers to the informer before it starts.
	AddIndexers(indexers Indexers) error
	GetIndexer() Indexer
}

// 向Infomer注册自定义数据处理函数
type ResourceEventHandlerFuncs struct {
	AddFunc    func(obj interface{})
	UpdateFunc func(oldObj, newObj interface{})
	DeleteFunc func(obj interface{})
}
```
- NewIndexerInformer
- > NewIndexer：使用threadSafeMap存储，一般使用DeletionHandlingMetaNamespaceKeyFunc索引器（获取对象的Name作为key）
- > newInformer
- NewSharedIndexInformer
- > NewIndexer
- > 
- NewListWatchFromClient : 返回ListWatch对象；入参需要传入封装好的clientset.CoreV1().RESTClient()
- NewInformer : 返回一个本地cache和controller
- > NewStore ： 基于threadSafeMap实现的本地缓存
- > newInformer：
- newInformer：需要ListerWatcher监听对象变化；objType过滤key集合；ResourceEventHandler资源处理函数；Store一个本地缓存

#### Reflector
```
type Reflector struct {
	// name identifies this reflector. By default it will be a file:line if possible.
	name string

	// The name of the type we expect to place in the store. The name
	// will be the stringification of expectedGVK if provided, and the
	// stringification of expectedType otherwise. It is for display
	// only, and should not be used for parsing or comparison.
	expectedTypeName string
	// An example object of the type we expect to place in the store.
	// Only the type needs to be right, except that when that is
	// `unstructured.Unstructured` the object's `"apiVersion"` and
	// `"kind"` must also be right.
	expectedType reflect.Type
	// The GVK of the object we expect to place in the store if unstructured.
	expectedGVK *schema.GroupVersionKind
	// The destination to sync up with the watch source
	store Store
	// listerWatcher is used to perform lists and watches.
	listerWatcher ListerWatcher

	// backoff manages backoff of ListWatch
	backoffManager wait.BackoffManager
	// initConnBackoffManager manages backoff the initial connection with the Watch calll of ListAndWatch.
	initConnBackoffManager wait.BackoffManager

	resyncPeriod time.Duration
	// ShouldResync is invoked periodically and whenever it returns `true` the Store's Resync operation is invoked
	ShouldResync func() bool
	// clock allows tests to manipulate time
	clock clock.Clock
	// paginatedResult defines whether pagination should be forced for list calls.
	// It is set based on the result of the initial list call.
	paginatedResult bool
	// lastSyncResourceVersion is the resource version token last
	// observed when doing a sync with the underlying store
	// it is thread safe, but not synchronized with the underlying store
	lastSyncResourceVersion string
	// isLastSyncResourceVersionUnavailable is true if the previous list or watch request with
	// lastSyncResourceVersion failed with an "expired" or "too large resource version" error.
	isLastSyncResourceVersionUnavailable bool
	// lastSyncResourceVersionMutex guards read/write access to lastSyncResourceVersion
	lastSyncResourceVersionMutex sync.RWMutex
	// WatchListPageSize is the requested chunk size of initial and resync watch lists.
	// If unset, for consistent reads (RV="") or reads that opt-into arbitrarily old data
	// (RV="0") it will default to pager.PageSize, for the rest (RV != "" && RV != "0")
	// it will turn off pagination to allow serving them from watch cache.
	// NOTE: It should be used carefully as paginated lists are always served directly from
	// etcd, which is significantly less efficient and may lead to serious performance and
	// scalability problems.
	WatchListPageSize int64
	// Called whenever the ListAndWatch drops the connection with an error.
	watchErrorHandler WatchErrorHandler
}

```
#### DeltaFIFO  维护一个事件队列
- 保证每个对象被处理的唯一性
- 可以查看某个对象的上一个操作和所有操作
```
Added   DeltaType = "Added"
Updated DeltaType = "Updated"
Deleted DeltaType = "Deleted"
// Replaced is emitted when we encountered watch errors and had to do a
// relist. We don't know if the replaced object has changed.
//
// NOTE: Previous versions of DeltaFIFO would use Sync for Replace events
// as well. Hence, Replaced is only emitted when the option
// EmitDeltaTypeReplaced is true.
Replaced DeltaType = "Replaced"
// Sync is for synthetic events during a periodic resync.
Sync DeltaType = "Sync"

type Delta struct {
	Type   DeltaType
	Object interface{}
}
// 聚合了一个key的所有操作，按照顺序
type Deltas []Delta
type DeltaFIFO struct { // 为每个key维护一个队列，key之间也有先后顺序
	// lock/cond protects access to 'items' and 'queue'.
	lock sync.RWMutex
	cond sync.Cond
	//保存key 到Deltas的映射
	items map[string]Deltas

	//保存items种的key，维持key的顺序性；
	queue []string
	populated bool
	initialPopulationCount int
	//key获取函数
	keyFunc KeyFunc
	//affecting Delete(), Replace(), and Resync()
	knownObjects KeyListerGetter
	closed bool
	emitDeltaTypeReplaced bool
}
```
- queueActionLocked ： 插入新的delta；1.items不存在才会插入queue；2.然后更新items的值；其中会调用dedupDeltas，合并两个连续的delete事件；
- Pop： 若队列无数据；则挂起；从queue出队，从items获取Deltas；然后从items删除；若process处理失败，则重新放入队列；返回一个Deltas，包含一个key的所有事件；process是pop的入参，由调用方指定
- Replace:1.使用sync或replace添加对象；2.执行删除操作
- > 批量添加对象到队列；并从队列中删除新添加对象中不存在的对象；
- > knownObjects 是什么
- 

#### LocalStore
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

### worker
