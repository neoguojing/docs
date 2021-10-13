# memstat

## 结构
```
type mstats struct {
	// General statistics.
	alloc       uint64 // bytes allocated and not yet freed
	total_alloc uint64 // bytes allocated (even if freed)
	sys         uint64 // bytes obtained from system (should be sum of xxx_sys below, no locking, approximate)
	nlookup     uint64 // number of pointer lookups (unused)
	nmalloc     uint64 // number of mallocs
	nfree       uint64 // number of frees
  
	heap_sys      sysMemStat // virtual address space obtained from system for GC'd heap
	heap_inuse    uint64     // bytes in mSpanInUse spans
	heap_released uint64     // bytes released to the os
	heap_objects uint64 // total number of allocated objects

	stacks_inuse uint64     // bytes in manually-managed stack spans; computed by updatememstats
	stacks_sys   sysMemStat // only counts newosproc0 stack in mstats; differs from MemStats.StackSys

	mspan_inuse  uint64 // mspan structures
	mspan_sys    sysMemStat
	mcache_inuse uint64 // mcache structures
	mcache_sys   sysMemStat
	buckhash_sys sysMemStat // profiling bucket hash table
  
	gcWorkBufInUse           uint64     // computed by updatememstats
	gcProgPtrScalarBitsInUse uint64     // computed by updatememstats
	gcMiscSys                sysMemStat // updated atomically or during STW
  
	other_sys sysMemStat // updated atomically or during STW

	next_gc uint64 //gc结束时的目标heap_live

	last_gc_unix    uint64 // last gc (in unix time)
	pause_total_ns  uint64
	pause_ns        [256]uint64 // circular buffer of recent gc pause lengths
	pause_end       [256]uint64 // circular buffer of recent gc end times (nanoseconds since 1970)
	numgc           uint32
	numforcedgc     uint32  // number of user-forced GCs
	gc_cpu_fraction float64 // fraction of CPU time used by GC
	enablegc        bool
	debuggc         bool

	by_size [_NumSizeClasses]struct {
		size    uint32
		nmalloc uint64
		nfree   uint64
	}

	last_gc_nanotime uint64 // last gc (monotonic time)
	tinyallocs       uint64 // number of tiny allocations that didn't cause actual allocation; not exported to go directly
	last_next_gc     uint64 // next_gc for the previous GC
	last_heap_inuse  uint64 // heap_inuse at mark termination of the previous GC

	triggerRatio float64 //触发标记的堆增长率

	gc_trigger uint64 //触发标记的堆大小，在end时计算

	heap_live uint64 //被gc认为的活跃的heap大小bytes

	// heap_scan is the number of bytes of "scannable" heap. This
	// is the live heap (as counted by heap_live), but omitting
	// no-scan objects and no-scan tails of objects.
	//
	// Whenever this is updated, call gcController.revise().
	//
	// Read and written atomically or with the world stopped.
	heap_scan uint64 //可扫描内存，不包含no-scan对象

	heap_marked uint64 //上一轮gc标记的数量

	// heapStats is a set of statistics
	heapStats consistentHeapStats
  
	gcPauseDist timeHistogram
}

```
