# client.go
- k8s.io/client-go
[![struct](https://github.com/neoguojing/docs/blob/main/k8s/modb_20211019_c1fed00a-30e6-11ec-8b07-fa163eb4f6be.png)]
## 流程
### 启动流程
- 创建client：ctx.ClientBuilder.ClientOrDie("deployment-controller")
- 创建informer：NewSharedIndexInformer
- 创建：sharedProcessor
- 创建indexer：NewIndexer
- 创建cacheMutationDetector：NewCacheMutationDetector
- 创建：record.NewBroadcaster()和eventBroadcaster.NewRecorder
- 创建workQueue：workqueue.NewNamedRateLimitingQueue
- 注册ResourceEventHandler: 为sharedProcessor注册processorListener，processorListener中绑定了ResourceEventHandler
- 注册状态处理函数：syncDeployment和监听器等
### 运行
- informer启动：sharedIndexInformer.Run
- cacheMutationDetector启动：cacheMutationDetector.Run: 将addedObjs复制给cachedObjs，addedObjs=nil；遍历cachedObjs，比较obj和copyObj是否相等，不相等则触发错误处理
- sharedProcessor启动：sharedProcessor.Run：遍历listeners，调用run函数使用注册的ResourceEventHandler向worker队列添加数据;仅添加key到工作队列;
- > 调用pop将分发的事件注入nextCh
- Reflector启动：数据类型为runtime.Object
- > 执行list，调用syncWith，更新runtime.Object和版本号到DeltaFIFO
- > 启动重新同步定时器，定期调用DeltaFIFO.Resync从localCache中同步数据到DeltaFIFO
- > 启动watch循环，从指定version开始watch，调用watchHandler，调用Add/Update/Delete向DeltaFIFO写入数据；BookMark则更新数据版本
- Controller启动：[]Delta
- > 从DeltaFIFO出队，调用HandleDeltas处理[]Delta数组；失败则放入队列重试
- > HandleDeltas遍历所有Delta数据，更新localCache，并调用processor.distribute向所有listener分发事件；对于Sync, Replaced, Added, Updated事件，则添加到cacheMutationDetector,对对象做深copy
- 业务启动，以deployment为例：
- WaitForNamedCacheSync调用deltaFIFO接口判断是否同步完毕；
- 启动n个worker线程，从work队列出对key值，调用syncDeployment处理相关
- syncDeployment：
- > 从localCache，根据key获取Deployment对象
- > 获取replicate和pod对象
- > 处理暂停条件并更新对象
- > 处理暂停事件
- > 处理回滚事件
- > 处理扩容/缩容事件
- > 根据策略处理deployment创建事件
## 关键模块
### 关键概念
- DeletedFinalStateUnknown： 对象被删除，但是watch deletion时间丢失；此时对象的状态为这个
- ExplicitKey： 只有key，没有obj
- metav1.Object ： 对象接口，如StatefulSet和Deployment均实现了该接口
- api/core/v1/types.go：定义了Node，Volume等对象；这些类均实现了runtime.Object接口；Informer传入这些对象用于分类
### Informer :
- 从从 DeltaFIFO 中 pop 相应对象，然后通过 Indexer 将对象和索引丢到本地 cache 中，再触发相应的事件处理函数（Resource Event Handlers）运行；
- 包含三个部分：Reflector、DeltaFIFO、LocalStore
- 实际开发中使用SharedIndexInformer
```
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
- NewSharedIndexInformer ： deploy等Infomer的基本实现
- > NewIndexer
- > 
- NewListWatchFromClient : 返回ListWatch对象；入参需要传入封装好的clientset.CoreV1().RESTClient()
- NewInformer : 返回一个本地cache和controller
- > NewStore ： 基于threadSafeMap实现的本地缓存
- > newInformer：
- newInformer：需要ListerWatcher监听对象变化；objType过滤key集合；ResourceEventHandler资源处理函数；Store一个本地缓存

#### Reflector
- list and watch 资源，然后同步到DeltaFIFO
- Reflector 向 apiserver watch 特定类型的资源，拿到变更通知后将其丢到 DeltaFIFO 队列中
```
type Reflector struct {
	// name identifies this reflector. By default it will be a file:line if possible.
	name string
	//期望放入store的key
	expectedTypeName string
	expectedType reflect.Type
	//期望的GVK
	expectedGVK *schema.GroupVersionKind
	// 目标存储
	store Store
	// 执行list和watcher
	listerWatcher ListerWatcher
	// 回退管理
	backoffManager wait.BackoffManager
	initConnBackoffManager wait.BackoffManager
	resyncPeriod time.Duration
	ShouldResync func() bool
	clock clock.Clock
	paginatedResult bool
	//最新的资源版本
	lastSyncResourceVersion string
	//遇到401，表示最后的版本无效
	isLastSyncResourceVersionUnavailable bool
	lastSyncResourceVersionMutex sync.RWMutex
	WatchListPageSize int64
	// Called whenever the ListAndWatch drops the connection with an error.
	watchErrorHandler WatchErrorHandler
}

```

- > Expired 或 Gone则设置资源不可用标志，pager.List从最后的资源版本开始list
- > 将获得的object调用Replace放入DeltaFIFO；
- > 调用Watch监听对象，设置最后版本、超时时间和bookmark；
- > 调用watchHandler处理watch
- watchHandler：
- > 将数据放入DeltaFIFO
- > bookmark 不断更新last resource 版本

#### ListAndWatch： 为Reflector提供能力
- 创建：NewListWatchFromClient，通过RESTClient实现
```
type Lister interface {
   // List 的返回值应该是一个 list 类型对象，也就是里面有 Items 字段，里面的 ResourceVersion 可以用来 watch
   List(options metav1.ListOptions) (runtime.Object, error)
}

type Watcher interface {
   // 从指定的资源版本开始 watch
   Watch(options metav1.ListOptions) (watch.Interface, error)
}

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

```
#### DeltaFIFO  维护一个事件队列
- 保证每个对象被处理的唯一性
- 可以查看某个对象的上一个操作和所有操作
- 维护了对象key的顺序性，同时维护了key对应对象的所有操作
- pop操作会阻塞，同时传入回调函数，处理失败之后重新加入队列
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
- > knownObjects： 是本地localcache中的值
- Resync：从localCache加载所有值到DeltaFIFO中，类型为SYNC

#### Indexer
- Indexer在threadSafeMap基础上拓展了对象检索功能
···
//给定一个对象，返回其key
type KeyFunc func(obj interface{}) (string, error)
MetaNamespaceKeyFunc实现之一：返回 namespace/name作为key
// 给定对象，返回一堆key
type IndexFunc func(obj interface{}) ([]string, error)
MetaNamespaceIndexFunc：返回ns作为key

type Indexer interface {
   Store
   Index(indexName string, obj interface{}) ([]interface{}, error) // 根据索引名和给定的对象返回符合条件的所有对象
   IndexKeys(indexName, indexedValue string) ([]string, error)     // 根据索引名和索引值返回符合条件的所有对象的 key
   ListIndexFuncValues(indexName string) []string                  // 列出索引函数计算出来的所有索引值
   ByIndex(indexName, indexedValue string) ([]interface{}, error)  // 根据索引名和索引值返回符合条件的所有对象
   GetIndexers() Indexers                     // 获取所有的 Indexers，对应 map[string]IndexFunc 类型
   AddIndexers(newIndexers Indexers) error    // 这个方法要在数据加入存储前调用，添加更多的索引方法，默认只通过 namespace 检索
}
···
- Indexer 主要提供一个对象根据一定条件检索的能力，典型的实现是通过 namespace/name 来构造 key ，通过 Thread Safe Store 来存储对象
- threadSafeMap上 一个本地缓存：使用lock和map
- > Indexers: 分类器：在原始数据上再构建一层map，相当于三级map
- > indices: 存储分类之后的数据
- > updateIndices: 删除indices老数据，并为新数据建立索引;
```
type threadSafeMap struct {
   lock  sync.RWMutex
   items map[string]interface{}
   indexers Indexers
   indices Indices
}

type Index map[string]sets.String  //存储namespace：object集合
type Indexers map[string]IndexFunc //存储不同类型的索引函数 namespace：func
type Indices map[string]Index //namespace:Index的集合
```
### listener
### record
- NewBroadcaster
### util/workqueue 只保存key
- Workqueue 一般使用的是延时队列实现，在 Resource Event Handlers 中会完成将对象的 key 放入 workqueue 的过程，然后我们在自己的逻辑代码里从 workqueue 中消费这些 key
- 普通队列： 通过cond条件等待和唤醒；queue队列保证消息处理的顺序性；dirty防止重复处理和进行事件聚合；processing保证真在处理的事件不被重复处理，结合Done函数标记事件处理完成
- 延时队列： 实现在指定的事件之后将元素加入队列；如何实现: 元素时间未到，则加入小顶堆；判断堆顶，未超时则建立定时器，等待超时事件
- 限速队列： 通过RateLimiter限速器的When方法，获取元素下一次加入队列需要的时间，利用延时队列接口，加入工作队列
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
	Forget(item interface{}) //结束重试

	// NumRequeues returns back how many times the item was requeued
	NumRequeues(item interface{}) int //重试的次数
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
	//正在被处理的元素，queue出队，并从dirty中删除
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
- Done(item)：锁定crond，从processing删除；若dirty中依然存在（再次处理），则插入到queue末尾，并发送信号唤醒一个Get函数
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
- RateLimiter接口的实现: When: 获取元素要等待的时长；Forget： 标识元素结束重试；NumRequeues： 元素被处理了多少次
- > ItemFastSlowRateLimiter: 超过阈值后调长重试时间
- > WithMaxWaitRateLimiter: 设置一个最大延时，超过则返回最大时长

### worker
- Worker 指的是我们自己的业务代码处理过程，在这里可以直接接收到 workqueue 里的任务，可以通过 Indexer 从本地缓存检索对象，通过 Clientset 实现对象的增删改查逻辑。
### ClientSet
- Clientset 提供的是资源的 CURD 能力，和 apiserver 交互
### Resource Event Handlers
- 一般是添加一些简单的过滤功能，判断哪些对象需要加到 workqueue 中进一步处理；对于需要加到 workqueue 中的对象，就提取其 key，然后入队
