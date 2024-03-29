# 锁
## 总结
- 只有cpu个数大于1，才使用自旋
- 锁不能被copy
- 结构体内的锁需要用指针，保证状态一致性

## 锁的概念
- 原子操作：std::atomic::compare_exchange_weak CAS = cmpxchg(x86) ldrex/strex (ar)
- 自旋锁：循环+CAS 确定占用cpu，导致饥饿
- 自适应锁：锁获取不到则休眠线程，直到有线程解锁再唤醒休眠线程，实现：1.锁状态；2.等待队列和休眠/唤醒机制，一般使用futex
### 锁算法
- 
## go的锁
### 死锁检测
- 什么时候执行死锁检测：具体为m变空闲或退出的时候：1.sched.nmfreed增加；2.sched.nmsys增加3.sched.nmidlelocked增加，4.sched.nmidle增加
- 系统m：sysmon；templateThread
- 系统g：除了runtime.main,handleAsyncEvent和runfinq，其他有runtime前缀的都是系统g
- 正在运行的线程大于0，则不是死锁：（sched.mnext（已经创建的m） - sched.nmfreed（释放的m））- sched.nmidle（空闲的m） - sched.nmidlelocked（锁住的空闲m） - sched.nmsys（系统m）
- 如何判断死锁：在运行的非系统m = 0，所有g都处于等待或抢占状态，且没有p在等待定时器超时，p没有定时器，则发生死锁。
### 运行时锁：
- 锁状态包含:未锁定，锁定和休眠
- 加锁：m加锁，xchg抢锁，失败则自旋（4+1）+pause+osyield，尝试抢占，仍然不成功，则futex休眠
- 解锁：设置锁状态为解锁，若之前状态为休眠，则futex唤醒；
### 用户锁
- 锁标记：锁定，唤醒，饥饿
- 锁模式：正常（性能优异，g可以多次获得锁）和饥饿（解决尾端延时问题），
- 饥饿状态：超过1ms未抢到锁，锁陷入饥饿状态；锁的拥有权直接交给队列前面的等待者；新的竞争者不竞争，不自旋，直接陷入等待；
- 正常模式切换：最后一个等待者或者超时小于1ms，则切换饥饿状态为正常状态
- 加锁流程：先自旋，抢占不到则休眠
- > 1.自旋4次检测锁状态是否为空闲
- > 2.空闲且不处于饥饿：直接抢占，成功则返回，不成功则休眠
- > 3.空闲且处于饥饿，则不抢占锁，直接陷入休眠
- > 4.已加锁且非饥饿，则计数加1，陷入休眠
- > 5.已加锁且饥饿，计数加1，陷入休眠
- > 6.继续下轮循环
- 解锁流程：饥饿状态，直接运行唤醒的g；否则，无等待或者锁已经被抢占或已经唤醒一个g直接返回；否则唤醒一个g，等待调度
#### 读写锁
- 写锁同步：之间通过互斥锁同步
- 写锁阻塞读锁：将readerCount-2^30置为负数；读锁检测到负数则挂起
- 读锁阻塞写锁：写锁检测到readerCount不为0且readerWait不为0，则挂起
- 写锁何时唤醒：读锁解锁时，readerWait = 0时，唤醒写锁
- 读锁何时唤醒：写锁减锁时，会唤醒所有读锁
- 写锁会饿死吗？：readerWait是readerCount的快照，不会随着readerCount增加而变化。
## 锁类型
- 自旋锁： 自旋+osyield（多进程竞争影响效率） 自旋+sleep（时间不好控制） 自旋+futex；自旋锁不可递归（重入）；自旋锁关闭了中断和抢占
## 原子操作：
- lock指令： lock前缀的指令，会锁cpu总线
- xchg指令： 原子操作，将新值存入变量，并返回旧值；锁内存总线
- 原子类型： int类型，禁止寄存器缓存
- cmpxchg： 变量值与旧值比较，相同则用新值覆盖，并返回旧值
- barrier()：空操作，告诉编译器这里有一个内存屏障，放弃在寄存器中的暂存值，重新从内存中读入
- PAUSE：指令给处理器提了个醒：这段代码序列是个循环等待；执行一个预定义的等待，降低耗电
- osyield： 系统调用，主动释放cpu

## 系统调用或汇编
- runtime·osyield：linux系统调用sched_yield，主动放弃cpu，当前进程或者线程进入调度队列等待重新调度
- runtime·procyield： 执行30次pause指令；pause提醒cpu代码是个循环等待，避免循环代码导致的可能的内存顺序违规导致的性能下降；降低耗电；执行一个预定义的延迟；
- futex： 
  数据结构： hashbucket -》 每个bucket维护一个列表，每个列表持有一个自旋锁；减小队列长度；自旋锁保护比较和入队操作的原子性；
  
  futex_wait流程：
  ```
  加自旋锁
  检测*uaddr是否等于val，如果不相等则会立即返回
  将进程状态设置为TASK_INTERRUPTIBLE
  将当前进程插入到等待队列中
  释放自旋锁
  创建定时任务：当超过一定时间还没被唤醒时，将进程唤醒
  挂起当前进程
  ```
  futex_wake流程如下:
  ```
  找到uaddr对应的futex_hash_bucket，即代码中的hb
  对hb加自旋锁
  遍历fb的链表，找到uaddr对应的节点
  调用wake_futex唤起等待的进程
  释放自旋锁
  ```
  golang封装
```
  func futexsleep(addr *uint32, val uint32, ns int64) : if *addr==val then sleep ; sleeptime < ns; if ns < 0 永远睡眠
  func futexwakeup(addr *uint32, cnt uint32) ： 唤醒地址addr上的线程，最多唤醒cnt个
```
  

## 运行时锁

```
type mutex struct {
 // GOEXPERIMENT=staticlockranking控制是否开启静态锁排序，默认不开启，为空结构体
 lockRankStruct
  // 基于futex的实现将其看作uint32的key
 key uintptr
```

### 关闭静态锁排序 lockrank_off.go

#### lock2 加锁流程 
- 获取m.lock++,禁止抢占
- xchg指令加锁
- 加锁失败，自旋4次：用cas指令抢占，抢占失败，执行pause等待；
- 自旋失败，尝试cas获取锁，失败，系统调用sched_yield，让出cpu，重新调度
- 调度返回，xchg尝试抢占锁，失败则futexsleep，挂起线程等待唤醒


#### 解锁流程
- xchg设置锁状态为unlock
- 若之前锁状态为sleep，则使用futexwakeup 唤醒
- m.locks-- 开启抢占
- 恢复g的抢占状态

### 开启静态锁排序 lockrank_on.go


## 用户锁
```
type Mutex struct {
	state int32 // | waiters（29bit） |starv（1bit）|waken（1bit）|locked（1bit）|
	sema  uint32
}

// A Locker represents an object that can be locked and unlocked.
type Locker interface {
	Lock()
	Unlock()
}

  mutexLocked = 1 
	mutexWoken = 2  //表示是否有协程已被唤醒，0：没有协程唤醒 1：已有协程唤醒，正在加锁过程中；通知Unlock的g不要再唤醒其他阻塞的g
	mutexStarving = 4  //当g等待锁超过10ms，设置该标志，告诉其他g锁进入饥饿模式，关闭自旋，直接交付
	mutexWaiterShift = 3 //偏移位数
	
type semaRoot struct {
	lock  mutex
	treap *sudog // root of balanced tree of unique waiters.
	nwait uint32 // Number of waiters. Read w/o the lock.
}

<!-- 读写锁，读写信号量，读计数和读等待 -->
type RWMutex struct {
	w           Mutex  // held if there are pending writers
	writerSem   uint32 // semaphore for writers to wait for completing readers
	readerSem   uint32 // semaphore for readers to wait for completing writers
	readerCount int32  // number of pending readers
	readerWait  int32  // number of departing readers
}

```

![semaphore](./golang-semaphore.drawio.png)
### 函数
- Lock:
- > CompareAndSwapInt32设置锁状态，成功则返回
- > 失败，陷入lockSlow
- lockSlow：循环
- > old锁处于锁定状态且可自旋，则自旋;wake标记未设置则尝试设置mutexWoken
- > old锁状态非饥饿，则设置new为mutexLocked //若锁处于饥饿状态，则当前g不加锁
- > old锁mutexLocked|mutexStarving状态已设置，则new的计数器加一 // 入等待队列
- > 锁超时且old锁为mutexLocked，设置new的mutexStarving
- > 若已唤醒，则清空new的mutexWoken
- > cas设置锁为new状态值，成功：
- > 锁之前未处于mutexLocked|mutexStarving，则加锁成功，break
- > 调用sync_runtime_SemacquireMutex，进入休眠，若waitStartTime非0，则直接加入等待队列头部
- > 计算等待时间，判断是否需要进入饥饿
- > 锁处于饥饿状态：队列只有一个等待，则清理饥饿状态，加锁状态和计数器减一
- sync_runtime_canSpin ：不能自旋的场景
- > 已经字段超过4次
- > 单核不自旋
- > GOMAXPROCS < 1
- > 当前m有一个p，且p本地运行队列不为空 (为什么)
- sync_runtime_doSpin：调用汇编函数procyield，执行30次pause
- sync_runtime_SemacquireMutex：semacquire1
- semacquire1：抢占信号量，成功则返回，否则休眠当前g，并把当前信号量放入等待队列；lifo=true，将等待者放入队列头部
- > cansemacquire:若addr所对应的值通过cas成功减1，则抢占成功，直接返回
- > 申请sudog
- > 循环：
- > root.nwait+1
- > 调用cansemacquire成功，则 root.nwait-1,返回
- > 失败则将sudog放入sema树上（lifo=true放入前面）
- > goparkunlock:休眠g
- > 唤醒之后，重新调用cansemacquire，成退出
- > 继续循环
- Unlock
- > 状态-锁定状态：1，设置新的状态
- > 新状态不为0，则调用unlockSlow
- unlockSlow:
- > 饥饿模式：调用sync_runtime_Semrelease，直接设置调度器，运行被唤醒的g
- > 正常模式：循环：无等待者，则返回；3个标记有一个设置，则返回；计数器减1，并设置唤醒标记，调用runtime_Semrelease唤醒某个g，并不直接运行
- sync_runtime_Semrelease:semrelease1 ： 
- semrelease1:从等待队列出队，调用goredy唤醒g并放入runnext，下一轮执行；handoff =true，则直接运行调度，使p开始执行runnext
-  > root.nwait == 0 则返回
-  > 等待者为0，则直接返回
-  > 对调用root.dequeue出队，获取sudog
-  > root.nwait-1
-  > handoff = true且cansemacquire成功，则sudog.ticket = 1
-  > goready唤醒sudog对应的g，将g设置为runable，放入p的runnext下一轮运行
-  > sudog.ticket = 1 则调用goyield，设置当前g为runable，放入当前p的队列，执行调度
### 读写锁
-  RLock：readerCoun+1；若readerCount<0,则调用sync_runtime_SemacquireMutex，在readerSem挂起当前读锁
-  RUnlock：rw.readerCoun-1；readerCount<0,则调用rUnlockSlow
-  rUnlockSlow：rw.readerWait-1；readerWait=0，则sync_runtime_Semrelease唤醒writerSem，唤醒写操作
-  Lock：写锁定
-  > 获取互斥锁
-  > rw.readerCount - rwmutexMaxReaders 变为负数；告知读操作有写操作在进行
-  > rw.readerCount的副本不为0，则设置rw.readerWait = rw.readerCount
-  > sync_runtime_SemacquireMutex挂起写操作在writerSem
-  Unlock: 写解锁
-  > rw.readerCount + rwmutexMaxReaders 转为正数；
-  > 唤醒所有的读操作sync_runtime_Semrelease
-  > 释放互斥锁

### 死锁检测
- checkdead：检查死锁的场景：mexit，templateThread，incidlelocked，sysmon，mput
- > 调度器上锁
- > 正在发生panic则退出
- > 计算正在运行的m，run= （sched.mnext - sched.nmfreed）- sched.nmidle - sched.nmidlelocked - sched.nmsys
- > linux下，run > 0 则返回，即正在运行的m大于0，则非死锁
- > 当前运行的m为0，遍历allg：
- > 忽略系统g
- > 检查g的状态：若是_Gwaiting，_Gpreempted表示，grunning++；
- > _Grunnable,_Grunning,_Gsyscall状态，则抛出异常，因为m为0，不可能有运行的g
- > 若无正在运行的g ，grunning = 0 ，则抛出异常表示正在退出进程-死锁
- > 此处，grunning！=0，
- > timeSleepUntil获取最近超时的timmer和对应的p，从midle列表获取m，绑定m和p，唤醒m，返回
- > 遍历allp，若_p_.timers 个数大于0，返回，无死锁
- > 设置g.m.throwing = -1 表示不导出所有栈
- > 抛出异常，表示-所有g在sleep-死锁
## 引用

- https://blog.csdn.net/sydhappy/article/details/115500346

- https://github.com/farmerjohngit/myblog/issues/6

- https://github.com/farmerjohngit/myblog/issues/8
