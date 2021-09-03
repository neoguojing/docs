# GC

问题：
清扫阶段，没有stw，不会影响结果吗？

## 三色标记法

- 程序所创建的对象初始默认标记为白色
- gc开始：根节点能够遍历到的对象全部标记为灰色
- 遍历灰色对象，被引用的对象标记为灰色，本身标记为黑色，直到没有灰色对象
- 回收白色对象

## 并行标记

为提高效率，期望并行标记；
标记必须stw，影响效率；
如何在减少或者不stw的情况下，提高标记效率？

不做stw标记，在满足以下条件下，会丢失对象：
条件1: 一个白色对象被黑色对象引用**(白色被挂在黑色下)**
条件2: 灰色对象与它之间的可达关系的白色对象遭到破坏**(灰色同时丢了该白色)**

### 屏障机制

在不stw的情况下，启动标记

- 强三色不变式： 
  不允许黑色对象引用白色对象；
  插入屏障: 在gc时，只在堆内存分配起作用，栈不需要启用（会影响效率）；1.新建引用对象会被标记为灰色；2.改变引用对象，新被引用对象标记为灰色；
  缺点：
  结束时需要STW来重新扫描栈，标记栈上引用的白色对象的存活；
  
- 弱三色不变式： 
  所有被黑色对象引用的白色对象都处于灰色保护状态.1.黑色-》灰色-》白色 2. 黑色-》白色《=灰色 ，这两种情况下是合法的。
  删除屏障：被删除的对象，如果自身为灰色或者白色，那么被标记为灰色。
  缺点：回收精度低，GC开始时STW扫描堆栈来记录初始快照，这个过程会保护开始时刻的所有存活对象。

- 混合写屏障：1.8引入
  特点：避免了对栈re-scan的过程，极大的减少了STW的时间
  1.gc开始时，扫描栈，标记所有对象为黑色（解决重复扫描栈的问题）；栈上新建对象均标记为黑色（避免引入屏障）；
  2.gc开始后，堆上对象：新建对象标记为灰色；被删除对象标记为灰色；

## 屏障是如何实现的？

## 常量
- GOGC：环境变量，off：关闭gc，0：不断触发gc，对调试由帮助；默认100，当栈达到4MB是触发gc
- gcpercent：=GOGC
- heapminimum：触发gc的最小堆；4mb * uint64(gcpercent) / 100
- defaultHeapMinimum：4mb
- mheap->sweepgen：每轮gc结束后+2，和mspan.sweepgen共同决定是span是否清晰
- > sweepgen == h->sweepgen - 2: 需要被清晰
- > sweepgen == h->sweepgen - 1: 正在被清洗
- > sweepgen == h->sweepgen : 已经被清晰，准备使用
- > sweepgen == h->sweepgen + 1 ： gc开始时被缓存，需要被清洗
- > sweepgen == h->sweepgen +3 : 已经被清洗和缓存，且仍然被缓存
- gcBlackenEnabled: 当gcphase == _GCmark时，允许后台标记worker可以标黑对象
- forcegcperiod ：int64 = 2 * 60 * 1e9  //强制gc时间
- worldsema： 授权m尝试stw
- gcsema：授权m阻塞gc，直到并发gc完成
- gcCreditSlack:2000
- sweep sweepdata：清扫状态
## gcphase
- _GCoff  ：gc没有运行，后台在清扫，写屏障未开启
- _GCmark ：标记根信息和工作缓冲：分配的对象未黑色，启用写屏障
- _GCmarktermination ：标记结束阶段:分配黑色对象, p辅助gc, 写屏障开启

### gcmode
- gcBackgroundMode gcMode = iota ：并发gc和清理
- gcForceMode                    // stw和并发清理
- gcForceBlockMode               // stwgc和清理 ，用户强制模式

### gcMarkWorkerMode
- gcMarkWorkerNotWorker //下一次调度不启动worker
- gcMarkWorkerDedicatedMode //worker应该不被抢占的运行
- gcMarkWorkerFractionalMode //参与标记任务但可被抢占和调度
- gcMarkWorkerIdleMode //仅在空闲时参与标记任务

### gcDrainFlags
- gcDrainUntilPreempt： 执行直到 g.preempt
- gcDrainIdle ：空闲时执行标记
- gcDrainFractional： 自我抢占直到pollFractionalWorkerExit返回true
- gcDrainFlushBgCredit：添加worker的负债到gcController.bgScanCredit
## 概念

### 何时触发gc
- 进程启动后启动forcegchelper协程，由sysmon定时触发gc
- 主动调用GC函数
- mallocgc调用时触发：堆内存达到阈值

### gc的root有哪些
> data和bss由linker生成，由moduledata结构体记录
- mcache
- data 
- bss
- mheap.markArenas：代表mspan喜喜
- allg:代表栈信息
- finblock：由销毁机制的对象

### 内存屏障相关

- wbBufEntries：256
- wbBufEntryPointers： 每个写屏障写入buf的指针数量
- shade：查找对象findObject，将对象greyobject

#### 结构体
```
type wbBuf struct {
	next uintptr
	end uintptr
	buf [512]uintptr
}
var writeBarrier struct {
	enabled bool    // compiler emits a check of this before calling write barrier
	pad     [3]byte // compiler uses 32-bit load for "enabled" field
	needed  bool    // whether we need a write barrier for current GC phase
	cgo     bool    // whether we need a write barrier for a cgo check
	alignme uint64  // guarantee alignment so that compiler can use a 32 or 64-bit load
}

```

#### 函数
- wbBufFlush1: 将写屏障buf同步到gc work 队列
- setGCPhase：设置gcphase，设置写屏障的状态

### 标记
- markBits
- heapBits
- pageMarks

## 结构体
```
type gcTriggerKind int  //触发gc的类型
const (
	gcTriggerHeap gcTriggerKind = iota //堆大小达到了阈值
	gcTriggerTime //定时触发
	gcTriggerCycle //用户强制触发
)

var work struct {
	full  lfstack          // lock-free list of full blocks workbuf
	empty lfstack          // lock-free list of empty blocks workbuf
	pad0  cpu.CacheLinePad // prevents false-sharing between full/empty and nproc/nwait
	wbufSpans struct {
		lock mutex
		// free is a list of spans dedicated to workbufs
		free mSpanList
		// busy is a list of all spans containing workbufs 
		busy mSpanList
	}

	// 本轮循环中被标记为黑色的对象，包括直接变黑的对象
	bytesMarked uint64
	markrootNext uint32 // next markroot job
	markrootJobs uint32 // number of markroot jobs
	nproc  uint32
	tstart int64
	nwait  uint32

	// Number of roots of various root types. Set by gcMarkRootPrepare.
	nFlushCacheRoots                               int
	nDataRoots, nBSSRoots, nSpanRoots, nStackRoots int
	// startSema protects the transition from "off" to mark or
	// mark termination.
	startSema uint32
	// markDoneSema protects transitions from mark to mark termination.
	markDoneSema uint32

	bgMarkReady note   // 后台标记go已经启动
	bgMarkDone  uint32 // cas to 1 when at a background mark completion point
	// mode is the concurrency mode of the current GC cycle.
	mode gcMode
	// userForced indicates the current GC cycle was forced by an
	// explicit user call.
	userForced bool
	// totaltime is the CPU nanoseconds spent in GC since the
	// program started if debug.gctrace > 0.
	totaltime int64
	// assistQueue is a queue of assists that are blocked because
	// there was neither enough credit to steal or enough work to
	// do.
	assistQueue struct {
		lock mutex
		q    gQueue
	}
	// sweepWaiters is a list of blocked goroutines to wake when
	// we transition from mark termination to sweep.
	sweepWaiters struct {
		lock mutex
		list gList
	}

	// cycles is the number of completed GC cycles
	cycles uint32
	stwprocs, maxprocs                 int32
	tSweepTerm, tMark, tMarkTerm, tEnd int64 // nanotime() of phase start
	pauseNS    int64 // total STW time this cycle
	pauseStart int64 // nanotime() of last STW
	// debug.gctrace heap sizes for this cycle.
	heap0, heap1, heap2, heapGoal uint64
}

<!-- 决定何时触发并发gc，需要启用多少个work进行辅助和后台标记 -->
var gcController gcControllerState

type gcControllerState struct {
	
	scanWork int64 //本轮scan work个数
	bgScanCredit int64 //扫描负债计数器，后台扫描增加负债，辅助扫描减少负债
	assistTime int64 //辅助扫描时间
	dedicatedMarkTime int64 //专门扫描worker花费的时间
	fractionalMarkTime int64 //分型扫描花费时间
	idleMarkTime int64 //空闲mark时间
	markStartTime int64
	dedicatedMarkWorkersNeeded int64 //专注扫描worker个数
	assistWorkPerByte uint64 //辅助worker百分比
	assistBytesPerWork uint64 // 1/assistWorkPerByte.
	fractionalUtilizationGoal float64
}

<!-- 生产者消费者模式 -->
type gcWork struct {
	wbuf1, wbuf2 *workbuf //1 指向push和pop的buf，2指向待丢弃的
	bytesMarked uint64
	scanWork int64
	flushedWork bool
}
// State of background sweep.
type sweepdata struct {
	lock    mutex
	g       *g
	parked  bool
	started bool
	nbgsweep    uint32
	npausesweep uint32
	centralIndex sweepClass
}


```
  
## 函数
- semacquire(addr): 获取信号量
- > cansemacquire原子的设置addr的值为0.若不为0，则返回false
- > 原子操作失败，获取sudog和semaRoot，初始化sudog
- > 进入循环：
- > semaRoot加锁，nwait加1，重新尝试cansemacquire，成功则nwait-1
- > 失败，调用semaRoot.queue,将当前地址加入等待队列，goparkunlock挂起g
- cansemacquire： 自旋的将addr的值减一，使用CAS
- semaRoot.queue 将sudog加入semaroot等待队列；semaRoot时一个treap树，随机平衡二叉搜索树
- > 初始化sudog，elem保存sema地址
- > 遍历semaRoot.treap,若sema地址已经存在，将sudog加入treap列表末尾，否则插入treap树的末尾
- gcinit：设置mheap_.sweepdone = 1，初始化gc百分比和work参数：startSema，markDoneSemabgscavenge：：:
- forcegchelper： 后台定时gc
- > 无限循环，forcegc加锁，设置idle标志，goparkunlock暂停当前g，gcStart开始gc
- gcStart：开始gc
- > 禁止抢占
- > 当前g为g0或者禁止抢占的情况下，直接返回
- > 开启抢占
- > 调用sweepone，清扫未处理的span
- > 获取work.startSema 状态,并重新trigger.test()，非true，则返回
- > 设置work.userForced = trigger.kind == gcTriggerCycle
- > 抢占gcsema和worldsema 标志
- > 遍历allp，检查p.mcache.flushGen 是否等于 mheap_.sweepgen，否则抛出异常
- > gcBgMarkStartWorkers 启动mark协程
- > 系统栈调用：gcResetMarkState
- > 设置work参数，系统栈调用stopTheWorldWithSema
- > 系统栈调用：finishsweep_m，完成清扫工作
- > clearpools： 清理缓存:subdog等
- > gcController.startCycle()：计算本轮gc的参数
- > work.heapGoal：设置
- > 若不是并发标记，schedEnableUser停止所有用户go的调度
- > 设置gcphase，gcBgMarkPrepare，gcMarkRootPrepare，gcMarkTinyAllocs，设置gcBlackenEnabled
- > 禁止抢占
- > startTheWorldWithSema:
- > 释放worldsema，开启抢占
- > 非并发模式下，调用Gosched
- > 释放startSema
- bgsweep
- bgscavenge:
- gcBgMarkStartWorkers: 启动mark worker协程，暂时不运行，直到mark阶段
- > 启动gomaxprocs个gcBgMarkWorker，调用notetsleepg休眠当前g在bgMarkReady
- gcBgMarkWorker：执行mark
- > 获取当期g，设置gp.m.preemptoff = "GC worker init"
- > 创建gcBgMarkWorkerNode，并设置初始值，notewakeup唤醒worker在bgMarkReady,此时worker可以在调度时被findRunnableGCWorker使用
- > 进入循环：
- > gopark停止该worker，并将worker放入gcBgMarkWorkerPool
- > node.m.set(acquirem()) 禁止抢占
- > 进入系统栈执行：
- > 若pp.gcMarkWorkerMode=gcMarkWorkerDedicatedMode，执行gcDrain，flag为gcDrainUntilPreempt|gcDrainFlushBgCredit，若gp.preempt为true，则将当前p内的所有g放入全局p，重新执行gcDrain
- > 若pp.gcMarkWorkerMode=gcMarkWorkerFractionalMode，执行gcDrain加一个gcDrainFractional标记
- > 若pp.gcMarkWorkerMode=gcMarkWorkerIdleMode,执行gcDrain加入gcDrainIdle
- > 切换g状态为_Grunning，退出系统栈
- > 切换pp.gcMarkWorkerMode = gcMarkWorkerNotWorker
- > incnwait == work.nproc && !gcMarkWorkAvailable(nil) 判断mark结束，则开启抢占，执行gcMarkDone
- gcDrain：执行根和对象扫描
- > 计算抢占和flushBgCredit等标记
- > 循环执行markroot,标记根对象，直到被抢占或stw
- > 循环执行scanobject，若全局工作队列为空，则迁移gcwork部分工作到全局gcw.balance()，tryGetFast获取work，否则tryGet获取，做不到wbBufFlush将写屏障缓存移到工作队列，gcw.tryGet()
- > 计算债务，若gcw.scanWork>gcCreditSlack,gcFlushBgCredit将债务转移到全局账户
- markroot： 标记root对象，对象编号按照mcache，data，bss，mspan，stack的顺序连续编号
- 索引落在cache区域，调用flushmcache清理缓存
- 落在data bss，调用markrootBlock标记
- 索引=0，则scanblock，遍历allfin
- 索引=1，markrootFreeGStacks销毁死亡的g栈空间
- 索引落在span区间：markrootSpans
- 否则执行g栈空间销毁，系统栈运行：根据索引从allgs找到g，若是自我扫描，则需要设置g状态为_Gwaiting；suspendG阻塞g，返回状态，状态为dead，则退出；否则调用scanstack，完成之后resumeG，自我扫描需要切换g为_Grunning
- flushmcache:清理allp[i]的内容,调用mcache.releaseAll和stackcache_clear清理mcache的堆和栈
- markrootBlock：计算shard，调用scanblock
- scanblock：用于扫描非堆roots：遍历bitmap，bit==0继续，否则调用findObject找到对象的起始地址，mspan和span的索引，调用greyobject置灰对象；若时栈对象，则放入栈扫描buf
- markrootFreeGStacks：释放空闲栈：遍历sched.gFree.stack，调用stackfree释放g的栈，将g放入sched.gFree.noStack
- markrootSpans：标记specialobject：遍历bitmap，找到对饮span，使用scanobject扫描可达对象，scanblock扫描root本身
- scanstack ： 只扫描_Grunnable, _Gsyscall, _Gwaiting的g，？？？？
- > 判读是否可以收缩栈，可以则调用shrinkstack,否则设置gp.preemptShrink = true，下一轮收缩
- > scanblock扫描gp.sched.ctxt
- > scanframeworker:扫描栈帧
- > 扫描defer，panic
- scanframeworker：扫描栈帧：包括本地变量，函数参数和返回：？？？
- findObject：找到p指针对应的堆对象：
- > 找到对应span，检查指针合法性
- > 找到span起始地址和对象在span中的索引
- greyobject：依据objindex获取 s.gcmarkBits，判断是否标记，未标记则标记，同时标记 arena.pageMarks，noscan对象直接返回，否则gcw.put将对象放入工作队列wbuf1.obj
- scanobject：扫描b指向的对象为起点的对象，
- > 获取heapBits，span和elem大小
- > 对象大于128k，拆分为oblet？？
- > 以b为起始地址，遍历步长为8byte，遍历obj，通过heapArean.bit的判断是都为指针，找到指针？？，调用findObject和greyobject标记
- gcMarkDone：mart to  mark termination 满足work.nwait == work.nproc && !gcMarkWorkAvailable(p)，才可以转换
- > 获取markDoneSema互斥量
- > 获取worldsema，用于stw
- > 系统栈调用forEachP，wbBufFlush1刷每个p的写屏障缓存到gcw，将gcw的wbuf返给work的buf
- > 调用stopTheWorldWithSema，
- > 进入系统栈：遍历所有p调用wbBufFlush1，若gcw不为空，则调用startTheWorldWithSema重启workd，跳到top重新执行
- > gcWakeAllAssists ：唤醒所有阻塞的assist g，
- > 释放：markDoneSema
- >  gcController.endCycle()：计算下一轮gc的ratio
- > gcMarkTermination
- gcMarkTermination：
- > 设置gcphase为_GCmarktermination，暂停当前g
- > 系统栈g0执行gcMark，遍历p，清空p.wbBuf和gcw.wbuf1
- > 系统栈执行：设置gcphase为_GCoff，关闭写屏障，调用gcSweep
- > 设置当前g为_Grunning，开启抢占，计算统计值
- > injectglist将work.sweepWaiters.list清扫g注入调度系统
- > 系统栈startTheWorldWithSema
- > prepareFreeWorkbufs:將work.wbufSpans.busy转移到free
- > freeStackSpans清理空闲栈
- > 系统栈forEachP清理mcache
- > 释放worldsema ，gcsema开启抢占
- gcFlushBgCredit
- gcResetMarkState: 系统栈调用，设置标记的优先级：并发或者stw，重置所有g的栈扫描状态heapBits判断是否包含指针
- stopTheWorldWithSema：
- startTheWorldWithSema
- gcBgMarkPrepare
- gcMarkRootPrepare： 统计data，bss，mspan和栈的信息作为根对象的个数
- gcMarkTinyAllocs
- gcMarkWorkAvailable：判断mark是否结束
- pollWork：判断idle worker是否需要处理网络事件
- pollFractionalWorkerExit：  判断frac标注是否可以自我抢占
- startTheWorldWithSema
- mcache清理：在acquirep调用时，调用prepareForSweep
- gcController.findRunnableGCWorker: 返回一个针对p的标记worker/g
- gcenable()：被main函数调用，执行bgsweep和bgscavenge
- gcTrigger.test:测试是否需要gc：gcphase必须等于_GCoff
- > gcTriggerHeap： 存活的堆内存大于阈值
- sweepone：清扫未处理的span
- notetsleepg:休眠g
- forEachP:
- injectglist：将glist注入到调度系统
- > 切换所有g状态为_Grunnable
- > 若当前p为nil，globrunqputbatch则将g放入sched.runq,调用startm调度g，返回
- > 否则，globrunqputbatch则将g放入sched.runq,调用startm调度g，多余的g调用runqputbatch放入当前p
- > 
### gcw
- balance： 迁移部分work到全局队列
## 引用

- https://blog.csdn.net/qq_33339479/article/details/108491796
