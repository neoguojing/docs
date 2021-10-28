# timer
runtime.time.go

## 总结
- 用户创建一个timer，则绑定到p上
- time函数不能阻塞
- 何时执行超时检查？：每一轮调度时都会检查
- timer函数的执行，会被调度器转换为一个g
- 大量的timer在忙的p中如何避免超时的？1.异步抢占，超过10ms的g被强行抢占；2.过多的g会被其他p偷走，分散p压力
- 如何检查超时？ 当前时间小于第一个timer的超时时间，则表示时间未到，否则执行之；
- timer在堆上已小顶堆方式存储。
## 参数 
```
type timer struct {
	pp puintptr //指向p

	// Timer wakes up at when, and then at when+period, ... (period > 0 only)
	// each time calling f(arg, now) in the timer goroutine, so f must be
	// a well-behaved function and not block.
	// when must be positive on an active timer.
	when   int64
	period int64
	f      func(interface{}, uintptr)
	arg    interface{} //指向一个g
	seq    uintptr
	// What to set the when field to in timerModifiedXX status.
	nextwhen int64
	// The status field holds one of the values below.
	status uint32
}

	timerWaiting  //等待执行
	timerRunning  //正在执行
	timerDeleted  //被删除，应该清理
	timerRemoving //正在清理
	timerRemoved //已经清理
	timerModifying
	timerModifiedEarlier  //定时器被提前
	timerModifiedLater  //定时器被置后或相等
	timerMoving  //被改变正在移动位置

```
### p上的timer参数
```
	timersLock mutex //锁
	timers []*timer   // 用户创建一个timer之后，会绑定到p上
	numTimers uint32 //timer计数
	deletedTimers uint32 //删除的timer
	// Race context used while executing timer functions.
	timerRaceCtx uintptr
	timer0When uint64 //p堆上第一个timer的when值，即最早将要触发的timer
	// The earliest known nextwhen field of a timer with
	// timerModifiedEarlier status. Because the timer may have been
	// modified again, there need not be any timer with this value.
	// This is updated using atomic functions.
	// This is 0 if there are no timerModifiedEarlier timers.
	timerModifiedEarliest uint64
```

## 函数
- checkTimers:在schedule和findrunnable中调用；检查p中将要超时的定时器
- > timer0When为0，则p上没有timer，返回传入的当前时间
- > now为0，则用nanotime赋值
- > now小于timer0When，若deletedTimers < numTimers/4 则返回，否则继续
- > adjusttimers 调整p上的所有timer
- > for循环调用runtimer，若还没到时间或没有timer，则break，否则继续执行
- > p为当前p，deletedTimers > numTimers/4,调用clearDeletedTimers清理timer
- > 返回当前时间，下一个需要执行的时间和是否有timer被成功执行
- adjusttimers: 遍历p的timer，处理提前的/滞后的和需要删除的timer
- runtimer： 执行堆上第一个timer，并删除或更新
- > 循环：
- > 获取p的第一个timer，执行状态机
- > timerWaiting：t.when > now 未超时，返回；切换状态为timerRunning，runOneTimer
- >timerDeleted:切换timerRemoving，清理dodeltimer0
- > timerModifiedEarlier/timerModifiedLater： 切换状态为timerMoving，t.when = t.nextwhen，dodeltimer0和dodeltimer
- > timerModifying:osyield
- > 其他则调用：badTimer
- runOneTimer：执行一个timer，调用f函数
- dodeltimer0:清理第一个timer
- clearDeletedTimers
## 引用

- https://medium.com/a-journey-with-go/go-timers-life-cycle-403f3580093a
