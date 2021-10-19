# GC

问题：
清扫阶段，没有stw，不会影响结果吗？

## 总结
- 启动gc有三种方式：手动，定时和分配内存时达到阈值
- gc触发条件：1.heap_live > gc_trigger;2.gcpercent小于0不会触发定时清理，上次到现在的时间大于forcegcperiod；3.gcTrigger.n-work.cycles > 0
- 工作模式：生产者：g扫描置灰的对象写入栈/gcw任务缓冲，内存屏障产生对象放入wbbuf;若gcw缓冲满了则刷入全局缓冲；消费者：g从本地gcw缓冲获取对象，执行标记
- gc流程：
- > 准备阶段：启动gomaxprocs个后台扫描g（暂停放入gcBgMarkWorkerPool）；重置pageMark等
- > stw ：确保上一轮的清扫工作完成（循环sweepone），清理sudog，syncpool和deferpool,计算gc参数
- > mark：计算markroot的个数和job个数，标记tiny对象；开启写屏障（非白对象都会被标记为黑色）
- > start world：换新sysmon，调度m执行p；schedule唤醒gcBgMarkWorker，执行标记任务；mallocgc调用gcAssistAlloc辅助一定部分的标记任务
- > gcMarkDone:将所有p的写屏障缓存刷入gcw
- > stw： 关闭写屏障，唤醒所有辅助gc的g，计算gc的扫描结果
- > _GCmarktermination：在g0上标记当前g
- > _GCoff：唤醒sweep.g（后台sweep），唤醒work.sweepWaiters(在mark term 到 sweep切换时等待的g)
- > start world:释放栈和清理所有p的mcache
- > bgsweep循环清理对象
- stw：本质是设置所有的p的状态为stop；重启世界时：则优先从netpoll获取g
- 标记用法：
- > gcmarkBits/markBits : 在gc时灰色对象的对应bit被置为1
- > allocBits: 清扫完成会使用gcmarkBits赋值，并重新创建新的gcmarkBits
- > heapBits： 辅助扫描，确定对象是否包含指针
- > pageMarks： 页中存在被标记的span，用于回收页时判断
- 扫描流程：
- > 1.编译器为结构体填充_type对象，记录指针信息到ptrdata和gcdata字段
- > 2.mallocgc时调用heapBitsSetType，根据编译器信息填充heapBits
- > 3.scanstack扫描g的栈空间，获取对象，利用type.ptrdata, type.gcdata信息调用scanblock扫描
- > 4.scanobject扫描内存地址指向的object，利用heapBit判断指针是否存在，进行对象标记
- gc参数计算: 主要计算需要启用的mark协程个数等，gc目标是占用25%的cpu时间，最高不超过30%
- 如何获取清理对象：从mcentral unswept队列获取
- GOGC的用法：GOGC=off不需要gc,实际上GOGC=100000;默认值为100
- uintptr: 是一个整形存储指针，不会被gc扫描
- bgscavenge：定期回收一个物理页大小的内存给操作系统
## gc内存
- 结构体的地址对齐
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
### 辅助gc
> 辅助gc：主要由g.gcAssistBytes决定，若为正数，则g有帮助gc过，有富于，不需要辅助gc，负数：则需要辅助gc进行扫描工作
- gcFlushBgCredit：
- > work.assistQueue.q.empty 阻塞assist g列表为空，则设置gcController.bgScanCredit返回
- > 否则：若gp.gcAssistBytes >= 0，则清零gp.gcAssistBytes = 0，调用ready让g继续运行；否则，增加gp.gcAssistBytes ，放g回到work.assistQueue.q
- > gcController.bgScanCredit += gp.gcAssistBytes
- gcAssistAlloc: 在mallocgc调用，当g.gcAssistBytes<0时
- > 计算扫描工作量让gcAssistBytes变为正数,若工作量不够，可以从gccontroller偷一部分工作量
- > 系统栈调用：gcAssistAlloc1
- > 若 gp.param != nil，则调用gcMarkDone
- > 若g.gcAssistBytes<0依旧成立，g可抢占，调用Gosched()，恢复之后goto retry
- > gcParkAssist:将g放入assist 队列，并park，恢复之后 goto retry
- gcAssistAlloc1： 切换当前g状态为_Gwaiting，调用gcDrainN，完成之后切换状态为_Grunning
- gcDrainN：将灰色对象变黑，直到处理scanWork个单元的扫描工作，或者G被抢占
- > 循环直到gp被抢占，或者工作量达标：
- > 调用gcw.balance() tryGetFast tryGet wbBufFlush等确保获取到一个工作
- > 若工作依旧未空，则尝试markroot
- > scanobject扫描对象
- > 
- gcParkAssist: 将g放入work.assistQueue.q，调用goparkunlock暂停
- gcWakeAllAssists: 从work.assistQueue.q获取多个g，调用injectglist从runq中获取p，交给m执行
### stw
#### 相关参数
- sched.stopwait： stw时会设置为gomaxprocs，每停止一个p（_Pgcstop），则减1
- sched.gcwaiting : stw或者freezetheworld时，会设置为1
- sched.stopnote : stw时会在此信号量上sleep
```
//stw 前置条件
semacquire(&worldsema, 0)
m.preemptoff = "reason"
systemstack(stopTheWorldWithSema)

m.preemptoff = ""
systemstack(startTheWorldWithSema)
semrelease(&worldsema)
```
- stopTheWorldWithSema：需要获取worldsema，同时静止抢占，系统栈调用
- > 设置sched.stopwait = gomaxprocs sched.gcwaiting =1
- > preemptall尝试抢占所有g
- > 设置当前p为_Pgcstop，sched.stopwait--
- > 遍历allp，尝试设置所有_Psyscall的p为_Pgcstop
- > pidleget从sched.pide获取p，切换状态为_Pgcstop
- > sched.stopwait > 0 则循环，notetsleep休眠100um，重新尝试preemptall
- startTheWorldWithSema：启动world
- > 若netpollinited，调用netpoll获取glist，injectglist注入到调度系统
- > sched.gcwaiting = 0,唤醒sysmon
- > wakep():为p分配m
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

### gc参数和指标计算
```
type gcControllerState struct {
	scanWork int64 //扫描工作计数,通过累积gcw.scanWork获得
	bgScanCredit int64 //后台扫描增加，辅助扫描消费
	assistTime int64 //辅助gc的时间
	dedicatedMarkTime int64 //专注gc的时间
	fractionalMarkTime int64
	idleMarkTime int64
	markStartTime int64
	dedicatedMarkWorkersNeeded int64 //需要专注gc的g个数
	assistWorkPerByte uint64 //每个辅助g应该负责多少工作
	assistBytesPerWork uint64 // 1/assistWorkPerByte
	fractionalUtilizationGoal float64 //fractional mark worker需要花费的时间
	_ cpu.CacheLinePad
}
```
#### 常量和公式
- gcBackgroundUtilization： 后台标记的cpu利用率，0.25
- gcGoalUtilization：标记的目标cpu利用率
- dedicatedMarkWorkersNeeded = gomaxprocs * 0.25+0.5 （取整）
- totalUtilizationGoal = gomaxprocs * 0.25
- utilError = dedicatedMarkWorkersNeeded/totalUtilizationGoal -1  |utilError| > 0.3 则fractionalUtilizationGoal ！= 0
- fractionalUtilizationGoal = （totalUtilizationGoal - dedicatedMarkWorkersNeeded）/gomaxprocs
- gcpercent： GOGC 默认为100，off时为10000
```
gomaxprocs = 8
dedicatedMarkWorkersNeeded = 8 * 0.25 + 0.5 = 2
totalUtilizationGoal = 8 * 0.25 = 2
utilError = 0
fractionalUtilizationGoal = 0
```
#### 函数
- startCycle：gc启动时调用,next_gc=heap_live+1M计算dedicatedMarkWorkersNeeded和fractionalUtilizationGoal，调用revise
- revise：重新修订gc参数，memstats.heap_scan,memstats.heap_live, or memstats.next_gc变化时都有调用，主要计算assistWorkPerByte
- > 期望的扫描量 = heap_scan * 100 / （100 + gcpercent），默认为heap_scan * 100 /200 = heap_scan/2，为可扫描对象的一半
- > 若heap_live > next_gc 或 scanwork > 期望的扫描量,则next_gc * 1.1 期望的扫描量 = heap_scan
- > 计算剩余工作量量 = 期望的扫描量（heap_scan）- scanwork ,< 1000 则赋值1000
- > 堆剩余量 = next_gc - heap_live
- > assistWorkPerByte = 计算剩余工作量量/堆剩余量
- > assistBytesPerWork = 堆剩余量 / 计算剩余工作量量
- endCycle: gc结束时调用
- > 人为触发不计算
- > goalGrowthRatio(实际增长率) = （next_gc - heap_marked）/heap_marked
- > actualGrowthRatio = heap_live/heap_marked - 1
- > assistDuration = 当前时间 - markStartTime
- > 利用率 = 0.25 + assistTime/assistDuration * gomaxprocs
- > triggerError = goalGrowthRatio - memstats.triggerRatio - utilization/gcGoalUtilization*(actualGrowthRatio-memstats.triggerRatio)
- >triggerRatio =  memstats.triggerRatio + triggerGain*triggerError
- gcSetTriggerRatio 
- >  next_gc = max(heap_marked + heap_marked * gcpercent/100,heap_marked*(1+triggerRatio))
- >  0.6 * gcpercent / 100 <= triggerRatio =< 0.95 * gcpercent / 100
- > gc_trigger = heap_marked * (1+triggerRatio)
- > sweepPagesPerByte = (pagesInUse - pagesSwept) / (gc_trigger-heap_live)
### 标记 s.elemsize == sys.PtrSize 表示span存的是指针
#### markBits 操作mspan.gcmarkBits，设置/清除/移动/判断等
- greyobject： 获取markBits，已标记则返回，未标记则设置标记（对应位置设置为1）
- gcmarknewobject: 分配新对象时，直接设置对应的标记位
- sweep：处理special对象是未标记则设置标记
- wbBufFlush1: 未标记则设置标记
```
type markBits struct {
	bytep *uint8
	mask  uint8 //mask为index在该byte中的位置
	index uintptr
}
```
- markBitsForAddr：依据地址获取对应的markBits
- > spanOf: 根据地址获取对应的span：依赖heapArean.spans获取
- > mspan.objIndex: 根据地址p获取p地址在mspan内部的对象索引
- > markBitsForIndex 获取markBits
- allocBitsForIndex: 依据allocBitIndex在allocBits获取对应的byte和对应位置的mask，返回markBits
- markBitsForIndex：依据objIndex在gcmarkBits获取对应的byte和对应位置的mask，返回markBits
- markBitsForBase：返回mspan基地址对应的markBits

#### heapBits 用于快速判定span中是否有指针
- bitPointer = 1
- bitScan = 16
- heapBitsShift = 1
- wordsPerBitmapByte : 4 表示1个bitmap可以表示4个word
- bitScanAll： 1111 0000
- bitPointerAll： 0000 1111
```
type heapBits struct {
	//存储heapArean的bitmap的值，其中8 表示一个指针8字节，4表示每个bitmap可以表示4个指针的状态；
	//
	bitp  *uint8 //ha.bitmap[(addr/(8*4))%(2^21)]  
	shift uint32  // (addr / 8) & 3 :取值为0-7
	arena uint32 // Index of heap arena containing bitp
	last  *uint8 // Last byte arena's bitmap &ha.bitmap[len(ha.bitmap)-1]
}

使用方法：
hbits := heapBitsForAddr(base)//根据基地址计算heapBits
n := s.elemsize  //对象的大小
for i := uintptr(0); i < n; i += sys.PtrSize { //以指针大小为步长遍历
	if hbits.isPointer() {}
	hbits = hbits.next()
}


```
- heapArena.pageMarks: 哪些span上有已标记的对象
- heapArena.bitmap：存储arena里word的指针或者标量
- heapBitsForAddr：计算地址对应的heapBits
- > 从mheap_.arenas找到对应的heapArena
- >  heapBits.bitp = ha.bitmap[(addr/(sys.PtrSize*4))%heapArenaBitmapBytes]
- > h.shift = uint32((addr / sys.PtrSize) & 3)
- > h.arena = uint32(arena)
-> h.last = &ha.bitmap[len(ha.bitmap)-1]
- heapBits.initSpan:
- > 计算span可以保存指针的个数nw
- > 循环：nw>0：
- > 调用heapBits.forwardOrBoundary,下一个heapBits的位置和当前可以保存的指针个数
- > 根据指针数计算需要的bitmap的个数 n/4=nbyte
- > 若是指针则设置bitp指向的数组的所有bit为1
- > 不是指针调用memclrNoHeapPointers，清理bit，全0
- heapBit.forwardOrBoundary:调用forward，返回heapBits的位置和当前可以保存的指针个数
- heapBit.forward: 根据span存储的指针数，计算需要的字节数，并把heapBits的bitp向后移动n个字节
- heapBit.next:若heapBit当前描述地址p，则next描述p+8byte
- > 若shift<3,则shift+1
- > 否则若h.bitp != h.last，则bitp向后移动1个byte
- > 否则调用nextArena
- heapBit.nextArena:arena+1，计算下一个arenaidx找到对应的heaparean
- heapBit.isPointer: bitp左移shift位 与上1 ，非0则是指针，否则非指针
- heapBitsSetType： 从编译器type生成heapBits??
#### gcbits bitmap

##### 概念
- Xadduintptr： 先将两个数交换，然后相加的和付给第一个变量
- *gcBits :指向8字节对齐的一串字节
- 每个bit对应span的一个elem，多余的bit不做处理
- mspan.allocBits: 在allocSpan中调用newAllocBits分配，在sweep中用gcmarkBits赋值
- mspan.gcmarkBits:在allocSpan中调用newMarkBits分配，在sweep中重新分配
##### 全局变量和函数
- gcBitsChunkBytes： 65536
- gcBitsHeaderBytes : 16
- gcBitsArenas: 全局bitmap入口
- newArenaMayUnlock： 初始化或分配新的gcBitsArena：gcBitsArenas.free==nil,使用sysAlloc分配一个新的；否则，gcBitsArenas.free.next调用memclrNoHeapPointers将65536字节清零；
```
type gcBits uint8

var gcBitsArenas struct {
	lock     mutex
	free     *gcBitsArena //可被重用的
	next     *gcBitsArena // 下一个gc循环可用的
	current  *gcBitsArena //当前在用的
	previous *gcBitsArena //上一个gc使用的
}
type gcBitsArena struct {
	free uintptr // free is the index into bits of the next free byte; read/write atomically
	next *gcBitsArena
	bits [65520]gcBits
}
```
- func newMarkBits(nelems uintptr) *gcBits：新bit全部置零
- > 计算nelems需要的字节数：((nelems + 63) / 64) * 8
- > 调用：gcBitsArena.tryAlloc :xadd指令，使得gcBitsArena.free = gcBitsArena.free+要分配的字节，返回gcBitsArena.bits[free-分配的字节]；字节满足需求则返回
- > tryAlloc 分配失败，重试依次
- > 重试失败，newArenaMayUnlock分配一个新gcBitsArena为fresh
- > 由于上一步无锁，另外一个线程可能更新了列表，tryAlloc重新试一次，成功则将fresh挂到全局列表
- > 上一步失败，则fresh.tryAlloc,失败则抛出异常
- > fresh挂到列表上
- newAllocBits：调用newMarkBits
#### pageMarks
- pageIndexOf: 通过指针获取对象所在的arena page索引和mask
- greyObject/gcmarknewobject/wbBufFlush1中设置对应的bit为1
- reclaimChunk回收page时利用pageMark甄别空闲页

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
- findObject：找到p指针对应的mspan，span的起始地址和p指针所代表的对象在mspan中的objIndex：
- > 找到对应span，检查指针合法性
- > 找到span起始地址和对象在span中的索引
- greyobject：依据objindex获取 s.gcmarkBits，判断是否标记，未标记则标记，同时标记 arena.pageMarks，noscan对象直接返回，否则gcw.put将对象放入工作队列wbuf1.obj
- scanobject：扫描b指向的对象为起点的对象，
- > 获取heapBits，span和elem大小
- > 对象大于128k，拆分为oblet？？
- > 以b为起始地址，遍历步长为8byte，遍历obj，通过heapArean.bit的判断是都为指针，找到指针？？，调用findObject和greyobject标记
- pollWork：判断idle worker是否需要处理网络事件
- pollFractionalWorkerExit：  判断frac标注是否可以自我抢占a
- mcache清理：在acquirep调用时，调用prepareForSweep
- gcenable()：被main函数调用，执行bgsweep和bgscavenge
- gcTrigger.test:测试是否需要gc：gcphase必须等于_GCoff
- > gcTriggerHeap： 存活的堆内存大于阈值
- notetsleepg:休眠g
- forEachP:作为一个混合屏障,强制所有的p都执行同一个函数
- > 设置sched.safePointWait和sched.safePointFn
- > 设置所有p的p.runSafePointF，使其在下一个安全点执行sched.safePointFn
- > preemptall() 挂起所有p
- > 遍历所有处于pidle列表的p，执行sched.safePointFn
- > 在当前p执行sched.safePointFn
- > 遍历allp，将_Psyscall的p切换到_Pidle，调用handoffp，将系统调用和lock的p从m解绑
- > sched.safePointWait > 0:notetsleep休眠100um，preemptall尝试抢占
- > sched.safePointFn = nil
- injectglist：将glist注入到调度系统
- > 切换所有g状态为_Grunnable
- > 若当前p为nil，globrunqputbatch则将g放入sched.runq,调用startm调度g，返回
- > 否则，globrunqputbatch则将g放入sched.runq,调用startm调度g，多余的g调用runqputbatch放入当前p
### gcController
- gcController.findRunnableGCWorker: 返回一个针对p的标记worker/g ，前置条件gcBlackenEnabled != 0.
- > 从全局gcBgMarkWorkerPool获取一个node
- > 设置p的gcMarkWorkerMode
- > 设置node.gp的状态为_Grunnable，返回
### mark
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
- > 索引落在cache区域，调用flushmcache清理缓存
- > 落在data bss，调用markrootBlock标记
- > 索引=0，则scanblock，遍历allfin
- > 索引=1，markrootFreeGStacks销毁死亡的g栈空间
- > 索引落在span区间：markrootSpans
- > 否则执行g栈空间销毁，系统栈运行：根据索引从allgs找到g，若是自我扫描，则需要设置g状态为_Gwaiting；suspendG阻塞g，返回状态，状态为dead，则退出；否则调用scanstack，完成之后resumeG，自我扫描需要切换g为_Grunning
- flushmcache:清理allp[i]的内容,调用mcache.releaseAll和stackcache_clear清理mcache的堆和栈
- markrootBlock：计算shard，调用scanblock
- scanblock：遍历所有指针，检查bitmap，若对应位置为1，则调用greyobject或者放入栈工作缓冲putPtr；使用显式的指针bitmap，如typ.gcdata
- > 遍历所有的指针：
- > 遍历bitmap，bit==0继续
- > 否则遍历bitmap（8次）：
- > 对应位置为1,则表示灰色对象，找到对应指针，若是堆对象则调用greyobject，否则调用putPtr放入栈工作缓冲
- > bitmap左移一位
- > 用于扫描非堆roots：，否则调用findObject找到对象的起始地址，mspan和span的索引，调用greyobject置灰对象；若时栈对象，则放入栈扫描buf
- markrootFreeGStacks：释放空闲栈：遍历sched.gFree.stack，调用stackfree释放g的栈，将g放入sched.gFree.noStack
- markrootSpans：标记specialobject：遍历bitmap，找到对饮span，使用scanobject扫描可达对象，scanblock扫描root本身
- scanstack ： 只扫描_Grunnable, _Gsyscall, _Gwaiting的g
- > 判读是否可以收缩栈，可以则调用shrinkstack,否则设置gp.preemptShrink = true，在newstack中收缩
- > scanblock扫描gp.sched.ctxt
- > scanframeworker:扫描栈帧
- > 扫描defer
- > 将g.panic指针放入栈扫描队列stackScanState中
- > 将栈对象建立二叉搜索树
- > 遍历二叉搜索树：
- > 从stackScanState中getPtr，调用state.findObject找到包含指针的栈对象
- >  obj.typ == nil，则表示已经扫描，遍历下一个
- > 否则设置typ为nil，调用scanblock,已栈地址,typ.ptrdata(指针个数)，t.gcdata(指针掩码)等为参数
- > 遍历所有stackObjectBuf，将buf放入work的empty的
- scanframeworker：扫描栈帧：包括本地变量，函数参数和返回：？？？
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
- gcResetMarkState: 系统栈调用，设置标记的优先级：并发或者stw，重置所有g的栈扫描状态heapBits判断是否包含指针
- gcBgMarkPrepare： 设置work参数
- gcMarkRootPrepare： 统计data，bss，mspan和栈的信息作为根对象的个数
- gcMarkTinyAllocs: 遍历allp，p.mcache.tiny,调用findObject greyobject标记对象
- gcMarkWorkAvailable：mark是否可用：p.gcw不为空或者work.full不为空或者work.markrootNext < work.markrootJobs，则返回true
- gcMark:
- > 遍历allp: 
- > p.wbBuf清空
- > p.gcw.dispose将多余工作迁移到全局
- > 统计heap_scan
- gcmarknewobject：标记一个新分配的对象为黑色，由mallocgc设置
- > 标记对象：gcmarkBits
- > 标记页面：heapArena.pageMarks
- > 更新gcw的统计值
### sweep
- numSweepClasses：272
- 
#### 结构体
- sweep sweepdata：清扫全局遍历
```
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
#### 函数
- bgsweep：执行清理任务，main函数中启动
- > sweep.g = getg() sweep.parked = true
- > 调用goparkunlock暂停当前g
- > 进入循环
- > 执行sweepone，若未完全清扫，执行Gosched，在新的循环中持续清扫
- > freeSomeWbufs 释放一些 work.wbufSpans到heap,返回true，则调用Gosched，在新的循环中持续释放
- > sweep.parked = true goparkunlock
- bgscavenge:
- sweepone：清扫一个未清理的span和返回page到heap
- > 进入循环
- >  mheap_.nextSpanForSweep() 找到一个待清扫的span，找不到则mheap_.sweepdone=1,退出
- > mspan.state!=mSpanInUse continue，找到则s.sweepgen-=1 退出循环
- > 调用mspan.sweep 执行清理
- > mheap_.reclaimCredit=mspan.npages
- > 系统栈调用：mheap_.pages.scavengeStartGen()
- > readyForScavenger
- freeSomeWbufs：释放work.wbufSpans.free相关spab
- > 系统栈 遍历work.wbufSpans.free.first64次，
- > work.wbufSpans.free.remove 从列表移除mpan
- > mheap_.freeManual 释放span
- > 若work.wbufSpans.free不为空，则返回true
- mspan.sweep:执行清理 
- > mheap_.pagesSwept+=mspan.npages
- > 处理s.specials ？？？
- > 若s.freeindex < s.nelems ： 处理僵死对象，检查分配了但是空闲的对象
- > 设置allocCount 为被标记的对象（被标记的对象不能清理）
- > 设置freeindex 为0
- > 设置allocBits=gcmarkBits
- > 重置gcmarkBits
- > 重置refillAllocCache
- > 设置sweepgen=heap.sweepgen
- > 若被标记的对象（countAlloc）个数为0，表示所有对象均需要释放，则释放freeSpan该span，返回true
- > 若被标记的对象等于nelems，表示span满了，放入mcentral.fullSwept,否则放入mcentral.partialSwept
- > 若为大对象且有空闲，则直接释放freeSpan，返回true
- > 若大对象无空闲，则放入mcentral.fullSwept
- mheap.nextSpanForSweep:下一个需要被清扫的span
- > 遍历mheap.central[spc].mcentral的full和partial的unswept集合，找到一个span
- > 更新sweep.centralIndex,作为下一个查找的起点
- gcSweep
- > mheap_.sweepgen += 2,重置mheap的sweep参数
- > 设置sweep.parked=false，调用ready(sweep.g, 0, true)，设置后台清扫g未可运行
- finishsweep_m：确保所有span被清除
- > 循环调用sweepone，直到清除完成
- > 遍历 mheap_.central ，清空mheap_.central[i].mcentral，未清理部分
- > wakeScavenger
- > nextMarkBitArenaEpoch:调整gcBitsArenas
- clearpools：
- > 清空sync.Pools
- > 清空sched.sudogcache
- > 清空sched.deferpool
### gcw
- balance： 迁移部分work到全局队列

### scavenge 回收清扫的span
- scavengeGoal: 需要归还给os的内存
- heapRetained ：rss内存估算
- scavengeEWMA：0.01 辅助scav应该占用的单核时间
- mheap.cavengeAll
- pageAlloc.scav
- pageAlloc.scavengeRangeLocked: 调用sysUnused释放page
- pageAlloc.scavengeOne：最小回收4k，最大由参数决定
- > 找到合适大小的page，调用scavengeRangeLocked，释放该页
- pageAlloc.scavenge：
- > 调用scavengeOne，知道释放足够byte
- bgscavenge:后台scav；被sysmon或者finishsweep_m阶段唤醒
- > 启动设置park参数，设置定时器，调用gopark,暂停
- > 被唤醒之后，进入死循环：
- > 系统栈调用：pageAlloc.scavenge回收一个物理页，更新scav统计信息
- > 若未释放，则挂起
- > 依据scavengeEWMA计算sleeptime，并调用scavengeSleep休眠
- > 计算新的scavengeEWMA
## 引用

- https://blog.csdn.net/qq_33339479/article/details/108491796
- https://blog.csdn.net/qiya2007/article/details/107291497
- https://www.jianshu.com/p/ebd8b012572e
