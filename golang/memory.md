# memory
## 总结
- Tiny allocator:小对象分配器，处理小于16字节的无指针类型分配；分配多个对象到一个内存块，只有在所有对象都失效的情况下，才会释放该内存块；主要处理小字符串和逃逸的独立变量，减少12%的内存分配和20%的堆大小；是一个堆指针，不受gc管理，在mcache的releaseAll清除
- tiny内存由spanClass为5的span承载
- noscan: 无需gc扫描的对象：nil或者无指针的对象，_type == nil || _type.ptrdata == 0
- spanClass：[1,67] * 2 分为noscan和scan，对应8B-32k的大小；
- mallocgc做了什么？
- > 1.写屏障若开启，则当前g的gcAssistBytes小于0，则需要辅助gc
- > 2.无指针且小于16B，则使用mcache分配tiny类型的内存
- > 3.大于32k则使用allocLarge直接分配需要数量的page
- > 4.对于16B到32k的内存，从mcache分配
- > 5.对于包含指针的类型，初始化heapBits
- > 6.gc中，则需要标记对象
- > 
- 如何在span中快速找到未分配的内存？
- > 
## 概念
- small 分配器处理小于32kb的70个大小的内存分类
- 页可以被分裂为一个固定大小的许多对象的集合
- GOOS: 操作系统- darwin, freebsd, linux等
- GOARCH: 架构-386, amd64, arm, s390x 等
- Huge pages：为了减轻页表管理的难度，对于GB内存大页为2MB ，对于TB内存大页为1GB，预分配
- THP：透明大页，redhad6引入，抽象层，使开发人员更方便管理大页，是动态分配的
- scav：scav中保存的是空闲并且已经垃圾回收的span。

## mmap：
- PROT_NONE ： 页不可访问
- MAP_SHARED ： 与其它所有映射这个对象的进程共享映射空间。对共享区的写入，相当于输出到文件。直到msync()或者munmap()被调用，文件实际上不会被更新。
- MAP_FIXED ： 如果由start和len参数指定的内存区重叠于现存的映射空间，重叠部分将会被丢弃。如果指定的起始地址不可用，操作将会失败。并且起始地址必须落在页的边界上。
- MAP_PRIVATE ： 建立一个写入时拷贝的私有映射。内存区域的写入不会影响到原文件
- MAP_ANONYMOUS/MAP_ANON ： 匿名映射，映射区不与任何文件关联

## 分配器

- fixalloc: 用于分配非堆对象,这些对象是分配器本身要使用的内存，没有足够的空间，调用persistentalloc，分配16kb的空间；会记录当前空闲空间的起始地址，剩余的空闲空间和在使用的空闲空间；优先从空闲列表上获取

## 主要数据结构
- mheap: golang的堆, 以page(8kb)为单位管理内存 
- mspan: 堆中一系列在使用的内存
- mcentral: 固定大小的mspan集合
- mcache: 有空闲的mspan线程本地缓存
- mstats: 统计内存分配

## mallocinit 内存初始化
- mheap_.init() ：各种分配器初始化，central的spanclass初始化，以及pageAlloc初始化
- mcache0 = allocmcache() ： 使用cachealloc，本质是一种固定分配器fixalloc，分配mcache对象大小的固定内存
- mheap_.arenaHint 初始化，构建128个arenaHint，用于堆分配，告诉堆的增长方向；堆的起始地址为0x00c0.易于区分、适应gcc编译器，能够不连续

## 全局变量和函数
- physHugePageSize: 从/sys/kernel/mm/transparent_hugepage/hpage_pmd_size获取，用于匿名内存映射和tmpfs/shmem等，常用大小2m
- physPageSize： 物理页大小 4k
- pageSize：8192=8k
- pallocChunkPages：512
- pallocChunkBytes： 4MB
- pallocChunksL1Shift: 13
- pallocChunksL2Bits： 13
- pageCachePages：64
- mheap_ ： 全局堆空间
- mcache0 ：全局缓存
- PtrSize： 8字节，64bit，系统指针大小
- arenaBaseOffset/minOffAddr :  0xffff800000000000  1<<47 对应arena map 的0地址
- maxOffAddr/maxSearchAddr : 0x7FFFFFFFFFFF
- globalAlloc: 全局persistentAlloc，带锁
- persistentChunkSize: 256k
- persistentChunks： 全局chuck内存基地址
- heapArenaBytes： 1<<26 = 64MB
- logHeapArenaBytes： 26
- pagesPerArena：8192=heapArenaBytes/pageSize  每个heapArena拥有的页数
- heapArenaBitmapBytes：2^21
- arenaL1Bits ： 1
- arenaL2Bits： 48
## madvise 内存的分配方式或者说是分配的细节方式
- MADV_FREE：标记过的内存，内核会等到内存紧张时才会释放。在释放之前，这块内存依然可以复用；速度更快，但是RSS内存显示无法立即下降；更积极的回收策略
- MADV_DONTNEED： 标记过的内存如果再次使用，会触发缺页中断；内存下降较快。1.16默认使用
- _MADV_HUGEPAGE ： 与khugepaged沟通，将对应内存块提升为大页映射

## 各架构内存bit数
- 符号扩展: 在amd64架构中，只用48bit地址位，且多余的16bit应该和48bit的最高bit相同（第47bit）；导致高地址是负数，低地址是正数
- zero扩展: 无符号短数扩展为长数
- amd64 : 47必填地址位,有符号扩展到64（4级页表转换）; 支持57bit,当前仅linux 支持（5级页表）
- arm64 : 48bit地址
- ppc64, mips64, and s390x 支持64bit寻址
- 
![64bit架构虚拟地址空间示意](https://pic1.zhimg.com/v2-dc953d14b0591cc5c8ee5f754dcdf158_r.jpg?source=1940ef5c)


## 内存状态
- None ： 未保存的，未map的
- Reserved： runtime保有，不能访问
- Prepared： 不会归还的内存，访问结果未定义，可能出错或返回0
- Ready： 可被安全访问

## 函数
- mallocinit: 在调度器初始化是调用,初始化堆外内存分配器和锁,以及arenaHint
- arenaHint: 告诉系统如何扩容mheap，用fixAlloc分配
- mallocgc : 分配可gc的内存,small内存从per-p mcache分配,大内存从heap分配
- newobject: new关键字的实现,调用mallocgc
- newarray: 数组分配器,调用mallocgc
- sysAlloc: 分配大内存,调用 mmap(nil, n, _PROT_READ|_PROT_WRITE, _MAP_ANON|_MAP_PRIVATE, -1, 0)
- sysReserve：  mmap(v, n, _PROT_NONE, _MAP_ANON|_MAP_PRIVATE, -1, 0)
- sysUsed ：madvise(addr unsafe.Pointer, n uintptr, flags int32) int32
- persistentalloc: sysAlloc的包装,被mallocgc等调用,实际分配内存；由全局和per-p两种
```
persistentalloc流程：
1.分配内存大于65535，直接调用sysAlloc
2. acquirem上锁，防止抢占
3. 优先从per-p获取persistentAlloc，否则使用全局persistentAlloc
4. 若偏移off+size大于256k，或这base基地址位nil，则sysAlloc分配256k的内存，地址赋给base，更新全局persistentChunks ？？？
5 否则，base+off，off+size
6.返回base+off的地址
```

## mspan
s.elemsize == sys.PtrSize
则该span分配的都是指针
```
type mspan struct {
	next *mspan     // next span in list, or nil if none
	prev *mspan     // previous span in list, or nil if none
	list *mSpanList // For debugging. TODO: Remove.
	startAddr uintptr // address of first byte of span aka s.base()
	npages    uintptr // number of pages in span
	manualFreeList gclinkptr // list of free objects in mSpanManual spans
	// If freeindex == nelem, this span has no free objects.
	freeindex uintptr
	nelems uintptr // number of object in the span.
	allocCache uint64 //缓存部分allocBits，最低bit对应freeindex指向的elem或object，0表示已分配，是allocBits取反
	allocBits  *gcBits //指针，指向bitmap；每个bit对应一个elem，，超出的部分忽略，1表示已分配；
	gcmarkBits *gcBits

	// sweep generation:
	// if sweepgen == h->sweepgen - 2, the span needs sweeping
	// if sweepgen == h->sweepgen - 1, the span is currently being swept
	// if sweepgen == h->sweepgen, the span is swept and ready to use
	// if sweepgen == h->sweepgen + 1, the span was cached before sweep began and is still cached, and needs sweeping
	// if sweepgen == h->sweepgen + 3, the span was swept and then cached and is still cached
	// h->sweepgen is incremented by 2 after every GC
	sweepgen    uint32
	allocCount  uint16        // number of allocated objects
	spanclass   spanClass     // size class and noscan (uint8)
	state       mSpanStateBox // mSpanInUse etc; accessed atomically (get/set methods)
	needzero    uint8         // needs to be zeroed before allocation
	elemsize    uintptr       // computed from sizeclass or from npages
	limit       uintptr       // end of data in span
	speciallock mutex         // guards specials list
	specials    *special      // linked list of special records sorted by offset.
}
```
### 状态
- mSpanDead不可用
- mSpanInUse gc heap使用的span
- mSpanManual ：供栈分配使用

### 类型
- spanAllocHeap：span用于堆空间
- spanAllocStack： span用于栈空间
- spanAllocPtrScalarBits： gc bitmap
- spanAllocWorkBuf： work buf空间
### 初始化
> 一般由mheap.allocSpan初始化
- 设置startAddr：起始地址；设置npages：占用的page大小；设置state为dead
- allocNeedsZero 判断是否需要置零
- 非堆空间span：更新 limit，设置状态为mSpanManual
- 堆空间：若spanclass==0，则所有空间为一个elem，nelems=1，elemsize=所有分配的空间；否则，elemsize为spanclass对应的大小，nelems=分配空间/elemsize，更新div相关参数
- freeindex=0 allocCache全部置1表示空闲 ，构建nelems的gcmarkBits和allocBits（全部为0），状态置mSpanInUse
- mspan.scavenge()把span释放还给OS
### 函数
- nextFreeIndex：
- > 若freeindex == snelems 则返回freeindex
- > 若Ctz64(allocCache)==64:则没有空闲elem，(freeindex + 64) &^ (64 - 1)偏移到下一个allocCache（freeindex保证为64的整数倍），若freeindex>=nelems ,则freeindex=nelems，返回freeindex;否则，freeindex / 8找到字节索引，调用refillAllocCache，设置新的allocCache，计算偏移
- > 调整freeindex+1，allocCache>>1

- refillAllocCache(whichbyte): 从 allocBits的第whichbyte开始取64bit，取反赋予allocCache
- sweep
## mcache per-p

- func (c *mcache) nextFree(spc spanClass) (v gclinkptr, s *mspan, shouldhelpgc bool):
- > 调用mheap.nextFreeIndex计算下一个freeIndex
- > 若freeIndex == s.nelems，表明当前mspan没有空闲，调用refill获取新的span，并获取span的freeindex，由于该操作是重分配，需要参与gc帮助标记
- > 若freeIndex<mspan.nelems,则返回新的对象起始地址和对应的span

- func (c *mcache) refill(spc spanClass) ：填充mcache.alloc[spanClass]:
- > 归还当前mcache.alloc[spc]到mcentral，调用函数mcentral.uncacheSpan(s)
- > 调用mcentral.cacheSpan()，获取一个span，若没有空闲内存，则需要mcentral.grow分配新的span
- > 为mcache.alloc[spc]赋新值
- releaseAll:释放缓存的mspan
- > 遍历alloc：
- > span不为空，统计分配的obj：s.nelems-s.allocCount，则调用mcentral.uncacheSpan回收
- > gcController.revise()更新统计值
- stackcache_clear 回收stackcache：遍历 c.stackcache[order].list，调用stackpoolfree
## mcentral
- grow：
- > mheap_.alloc 分配新的span
- > heapBit.initSpant初始化span对应的bitmap
- uncacheSpan
- cacheSpan ：给mcache分配一个mspan
- > deductSweepCredit执行清扫
- > 从partial 已清扫列表获取一个span，存在，则跳转到havespan
- > 从partial 未清扫列表获取span，调用mspan.sweep清扫该span，跳转到havespan
- > 从full，未清扫列表获取span，调用mspan.sweep清扫该span，调用s.nextFreeIndex()获取freeindex，若freeIndex != s.nelems，跳转到havespan；否认将清扫完成的span挂到full，已清扫
- > 调用mcentral.grow，分配一个新的span
- > 根据新的span，计算freeindex和allocCache
## mheap
- freeManual：mspan.needzero = 1 调用freeSpanLocked
- freeSpanLocked:
- > 若状态为mSpanInUse，则mheap.pagesInUse -= mspan.npages，计算arean和page所应等
- > 更新统计值
- > 调用pageAlloc.free,释放page
- > 设置span为dead
- > 调用freeMSpanLocked
- freeMSpanLocked：
- > 尝试将span放入pp.mspancache.buf，做缓存
- > 调用fixalloc.free,将inuse减去span的大小，将空闲地插入list头部
- grow： 添加npage to mheap
- alloc：
- allocSpan：分配一个mspan，有npage大小，必须在系统栈调用
```
1. p不为空，且分配页数小于16的情况下，从p.pageCache分配，若pageCache为空，则调用mheap的pageAlloc分配内存；然后尝试从pageCache分配mspan
2. 若成功跳转到HaveSpan，初始化span相关参数，
3. 否则调用pageAlloc.alloc分配，失败调用mheap.grow,成功则重新调用pageAlloc.alloc分配，否则返回nil
4。调用mheap.allocMSpanLocked,分配span，转到第2步
```
- allocMSpanLocked：
- allocManual：必须在系统栈调用，手动分配npage内存，底层调用allocSpan，无需spanclass
- allocNeedsZero: 判断是否需要置零
- > 
### arena管理
```
type heapArena struct {
	bitmap [2^21]byte  //2bit设置代表一个word，本arean区域的word的指针或标量的bitmap
	spans [8192]*mspan  //page到span的映射
	pageInUse [1024]uint8 //mSpanInUse的span，只有span对应的第一页被映射；allocSpan中设置标识，freeSpanLocked中清空标识
	pageMarks [1024]uint8 //有被标记object的span，用首页page的标记代表mspan，greyobject/gcmarknewobject/wbBufFlush1中使用
	pageSpecials [1024]uint8
	checkmarks *checkmarksMap
	zeroedBase uintptr //??
}
```
- arenaIndex：地址空间按64MB分割的索引 （地址-0xffff800000000000）/64
- arenaIdx.l1 = 0
- arenaIdx.l2 = arenaIndex & (2^48-1) 即arenaIndex本身
- setSpans：将新分配的span，和构成span的page做映射放入mheap.arenas，为1：n的关系，

## 页管理
### pageAlloc 页分配器 位于mheap
- minOffAddr ： 0xffff800000000000
- maxOffAddr ： 2^48-1 + 0xffff800000000000 = 0x7FFFFFFFFFFF
```
type pallocSum uint64

type pageAlloc struct {
  summary [5][]pallocSum   //[5]{[2^14]pallocSum,[2^17]pallocSum,[2^20]pallocSum,[2^23]pallocSum,[2^26]pallocSum}，从0-4依次代表基数树的每一层
  chunks [8192]*[8192]pallocData //pallocData 用pageBits代表4MB的内存空间，即2^22, chunks总共可以表示2^22 * 2^13 * 2^13 = 2^48
  searchAddr offAddr //maxSearchAddr : 0x7FFFFFFFFFFF
  start, end chunkIdx
  inUse addrRanges
  mheapLock *mutex
}

var levelShift = [5]uint{34,31,28,25,22}

type pallocData struct {
	pallocBits
	scavenged pageBits
}

type pageBits [8]uint64   //每个bit代表一页（8k），则uint64代表：512k，pageBits总共可以代表：512*8=4MB即一个chuck大小

```
- init初始化: 
- > inuse:分配16个addrRange（16byte，2个指针）
- > pageAlloc初始化：summary，5层数组，每个level依次分配2^14,2^(17),2^(20),2^(23),2^(26） 乘以uint64的空间，使用sysReserve分配
- allocToCache分配缓冲区pageCache：
- > 使用searchAddr计算ci
- > 若p.summary[len(p.summary)-1][ci] != 0，则从searchAddr开始寻找可用的page；
- > 否则，p.find借助基数树查询可用searchAddr，分配内存
- > 更新chunks和summary，以及searchAddr
- chunk： 大小为4MB，管理512个page，每个page 8192字节（8k）
- chunkIndex(): (searchAddr-0xffff800000000000)/4MB ，内存地址转换为chunkIdx
- chunkBase(): chunkIdx*4MB+0xffff800000000000 ,chunkIdx转换为内存地址
- chunkPageIndex(): searchAddr%4MB/8k，4MB中page的索引 ,又叫searchIndex
- chunkIdx.l1(): i/8192
- chunkIdx.l2() ：i & 8191
- chunkOf:返回chunkIdx对应的pageBits
- pallocBits.find1: i*64 + uint(sys.TrailingZeros64(^x)) 通过TZ，计算尾部的0的个数，以此判断空闲页的偏移
- alloc:分配内存
- free：释放npage内存内存到page堆
- > 若释放的地址小于searchAddr，则p.searchAddr = b
- > 若npages ==1，则找到对应的chuckIndex，调用pageBits.free1,设置page对应的bit为0
- > 若为多页，计算chuck起始和结束索引和page的起始和结束索引，若在一个chuck内（4MB）则调用pageBits.free，clearRange清空所有bit，否则需要清理多个chuck
- > 调用update,更新元数据
- update
- summarize: 
- find: 在基数树上从searchAddr开始查找一个连续的npage内存块
- > 遍历level=5次：
- > 
### pageCache 页缓存 位于p
```
type pageCache struct {
	base  uintptr // base address of the chunk
	cache uint64  // 64-bit bitmap representing free pages (1 means free)
	scav  uint64  // 64-bit bitmap representing scavenged pages (1 means scavenged)
}
```
- cache：64bit，每个bit管理一页内存（8k），则总共可以管理512k；最后一个bit指向base开始的第一页？，

## mallocgc
- _GCmarktermination阶段不允许分配内存
### nextFreeFast mspan快速重用空闲对象
- Ctz64计算allocCache开通为0的bit数量（从左到右）theBit，表示空闲内存的偏移；0表示已分配；每个bit表示一个elem
- 若theBit<64,从freeindex开始加上偏移量，即找到空闲对象的位置索引，s.freeindex + uintptr(theBit)=result
- result<nelems: 更新freeindex=result+1 allocCache>>theBit + 1 allocCount++ ,返回result*s.elemsize + s.base()，一个虚拟地址

## 引用
https://blog.csdn.net/qq_52212721/article/details/117604905
