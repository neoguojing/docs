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
- _GCoff  ：gc没有运行，后台在清扫，写屏障未开启
- _GCmark ：标记根信息和工作缓冲：分配的对象未黑色，启用写屏障
- _GCmarktermination ：标记结束阶段:分配黑色对象, p辅助gc, 写屏障开启
- forcegcperiod ：int64 = 2 * 60 * 1e9  //强制gc时间
- gcBackgroundMode gcMode = iota ：并发gc和清理
- gcForceMode                    // stw和并发清理
- gcForceBlockMode               // stwgc和清理 ，用户强制模式
- worldsema： 授权m尝试stw
- gcsema：授权m阻塞gc，直到并发gc完成
## 概念

### 何时触发gc
- 进程启动后启动forcegchelper协程，由sysmon定时触发gc
- 主动调用GC函数
- mallocgc调用时触发：堆内存达到阈值
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

	bgMarkReady note   // signal background mark worker has started
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
```
  
## 函数
- semacquire(addr): 获取信号量
- > cansemacquire原子的设置addr的值为0.若不为0，则返回false
- > 原子操作失败，
- cansemacquire： 自旋的将addr的值减一，使用CAS
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
- gcResetMarkState: 系统栈调用，设置标记的优先级：并发或者stw，重置所有g的栈扫描状态
- stopTheWorldWithSema：
- gcBgMarkPrepare
- gcMarkRootPrepare
- gcMarkTinyAllocs
- startTheWorldWithSema
- mcache清理：在acquirep调用时，调用prepareForSweep
- gcController.findRunnableGCWorker: 返回一个针对p的标记worker/g
- gcenable()：被main函数调用，执行bgsweep和bgscavenge
- gcTrigger.test:测试是否需要gc：gcphase必须等于_GCoff
- > gcTriggerHeap： 存活的堆内存大于阈值
- sweepone：清扫未处理的span
## 引用

- https://blog.csdn.net/qq_33339479/article/details/108491796
