# Broadcaster

## 概念
```
type Interface interface {
	Stop()
	ResultChan() <-chan Event
}

type EventType string
const (
	Added    EventType = "ADDED"
	Modified EventType = "MODIFIED"
	Deleted  EventType = "DELETED"
	Bookmark EventType = "BOOKMARK"
	Error    EventType = "ERROR"
)

type Event struct {
	Type EventType
	Object runtime.Object
}
```

## Broadcaster
- 将incoming的数据，分发到watchers的每个broadcasterWatcher
```
type Broadcaster struct {
	watchers     map[int64]*broadcasterWatcher
	nextWatcher  int64
	distributing sync.WaitGroup

	incoming chan Event  //进入的chan
	stopped  chan struct{}

	// How large to make watcher's channel.
	watchQueueLength int
	fullChannelBehavior FullChannelBehavior
}

type broadcasterWatcher struct {
	result  chan Event
	stopped chan struct{}
	stop    sync.Once
	id      int64
	m       *Broadcaster
}
```
- Watch：向watchers 注册一个broadcasterWatcher，并返回；注册时实际向incoming发送一个注册event，等待直到注册成功才返回；
- Action： 向incoming注册一个事件
- loop：消费incoming，并向所有watchers，发布该消息

## eventBroadcasterImpl 继承Broadcaster
```
type eventBroadcasterImpl struct {
	*watch.Broadcaster
	sleepDuration time.Duration
	options       CorrelatorOptions
}
```
- NewRecorder：构建EventRecorder

## EventRecorder 构建事件放入Broadcaster做分发
- recorderImpl：实现EventRecorder；并继承Broadcaster
```
type recorderImpl struct { //实现EventRecorder
	scheme *runtime.Scheme
	source v1.EventSource
	*watch.Broadcaster
	clock clock.Clock
}
```
- Event
- Eventf：
- AnnotatedEventf ： 和Eventf一样，只是添加了注释信息
