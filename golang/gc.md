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


## 结构体
```
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

```
  
## 函数

- gcinit：设置mheap_.sweepdone = 1，初始化gc百分比和work参数：startSema，markDoneSema

