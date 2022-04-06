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
- Watch：向watchers 注册一个broadcasterWatcher，并返回
- Action： 向incoming注册一个事件
- loop：消费incoming，并向所有watchers，发布该消息
