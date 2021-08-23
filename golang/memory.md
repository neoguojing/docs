# memory

## 概念
- small 分配器处理小于32kb的70个大小的内存分类
- 页可以被分裂为一个固定大小的许多对象的集合
- GOOS: 操作系统- darwin, freebsd, linux等
- GOARCH: 架构-386, amd64, arm, s390x 等
- noscan: 无需gc扫描的对象：nil或者无指针的对象，_type == nil || _type.ptrdata == 0
- Tiny allocator:小对象分配器，处理小于16字节的无指针类型分配；分配多个对象到一个内存块，只有在所有对象都失效的情况下，才会释放该内存块；主要处理小字符串和逃逸的独立变量，减少12%的内存分配和20%的堆大小；是一个堆指针，不受gc管理，在mcache的releaseAll清除
- Huge pages：为了减轻页表管理的难度，对于GB内存大页为2MB ，对于TB内存大页为1GB，预分配
- THP：透明大页，redhad6引入，抽象层，使开发人员更方便管理大页，是动态分配的

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
- 
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
### 状态
- mSpanDead不可用
- mSpanInUse gc heap使用的span
- mSpanManual ：供栈分配使用

## mheap
- grow： 添加npage to mheap
- allocSpan：分配一个mspan，有npage大小，必须在系统栈调用
```
1. p不为空，且分配页数小于16的情况下，从p.pageCache分配，若pageCache为空，则调用mheap的pageAlloc分配内存；然后尝试从pageCache分配mspan
2. 若成功跳转到HaveSpan，初始化span相关参数，
3. 否则调用pageAlloc.alloc分配，失败调用mheap.grow,成功则重新调用pageAlloc.alloc分配，否则返回nil
4。调用mheap.allocMSpanLocked,分配span，转到第2步
```
- allocMSpanLocked：
- allocManual：必须在系统栈调用，手动分配npage内存，底层调用allocSpan，无需spanclass
## 页管理
### pageAlloc 页分配器 位于mheap
```
type pageAlloc struct {
  summary [5][]pallocSum   //[5]{[2^14]pallocSum,[2^17]pallocSum,[2^20]pallocSum,[2^23]pallocSum,[2^26]pallocSum}
  chunks [8192]*[8192]pallocData //pallocData 用bitmap代表4MB的内存空空，即2^22, chunks总共可以表示2^22 * 2^13 * 2^13 = 2^48
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
- > 
- >
- chunk： 大小为4MB，管理512个page，每个page 8192字节（8k）
- chunkIndex(): (searchAddr-0xffff800000000000)/4MB ，内存地址转换为chunkIdx
- chunkBase(): chunkIdx*4MB+0xffff800000000000 ,chunkIdx转换为内存地址
- chunkPageIndex(): searchAddr%4MB/8k，4MB中page的索引 ,又叫searchIndex
- chunkIdx.l1(): i/8192
- chunkIdx.l2() ：i & 8191
- pallocBits.find1: i*64 + uint(sys.TrailingZeros64(^x)) 通过TZ，计算尾部的0的个数，以此判断空闲也的偏移
- alloc:分配内存
  
### pageCache 页缓存 位于p
```
type pageCache struct {
	base  uintptr // base address of the chunk
	cache uint64  // 64-bit bitmap representing free pages (1 means free)
	scav  uint64  // 64-bit bitmap representing scavenged pages (1 means scavenged)
}
```
- cache：64bit，每个bit管理一页内存（8k），则总共可以管理512k；最后一个bit指向base开始的第一页？，
 
## gcbits bitmap
- newMarkBits
- newAllocBits

## mallocgc
- _GCmarktermination阶段不允许分配内存
### nextFreeFast mspan快速重用空闲对象
- allocCache 与 count trailing zero 算法解读？？？
- allocBits的结构？？？

## 引用
https://blog.csdn.net/qq_52212721/article/details/117604905
