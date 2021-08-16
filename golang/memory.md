# memory

## 概念
- small 分配器处理小于32kb的70个大小的内存分类
- 页可以被分裂为一个固定大小的许多对象的集合
- GOOS: 操作系统- darwin, freebsd, linux等
- GOARCH: 架构-386, amd64, arm, s390x 等
- noscan: 无需gc扫描的对象：nil或者无指针的对象，_type == nil || _type.ptrdata == 0
- Tiny allocator:小对象分配器，处理小于16字节的无指针类型分配；分配多个对象到一个内存块，只有在所有对象都失效的情况下，才会释放该内存块；主要处理小字符串和逃逸的独立变量，减少12%的内存分配和20%的堆大小；是一个堆指针，不受gc管理，在mcache的releaseAll清除

## 分配器

- fixalloc: 用于分配非堆对象,这些对象是分配器本身要使用的内存
- mheap: golang的堆, 以page(8kb)为单位管理内存 
- mspan: 堆中一系列在使用的内存
- mcentral: 固定大小的mspan集合
- mcache: 有空闲的mspan线程本地缓存
- mstats: 统计内存分配

## 内存回收
- MADV_FREE：标记过的内存，内核会等到内存紧张时才会释放。在释放之前，这块内存依然可以复用；速度更快，但是RSS内存显示无法立即下降；更积极的回收策略
- MADV_DONTNEED： 标记过的内存如果再次使用，会触发缺页中断；内存下降较快。1.16默认使用

## 各架构内存bit数
- 符号扩展: 有符号数短数扩展到长数,扩展部分的值为之前数的最高位
- zero扩展: 无符号短数扩展为长数
- amd64 : 48必填地址位,有符号扩展到64; 支持57bit,当前仅linux 支持
- arm64 : 48bit地址
- ppc64, mips64, and s390x 支持64bit寻址


## 参数
- FixedStack: linux下为2kb
- 可缓存的空闲栈大小为:2kb\4kb\8kb\16kb,大于16kb的栈则直接分配

## 函数
- mallocinit: 在调度器初始化是调用,初始化堆外内存分配器和锁,以及arenaHint
- arenaHint: 告诉系统如何扩容mheap
- mallocgc : 分配可gc的内存,small内存从per-p mcache分配,大内存从heap分配
- newobject: new关键字的实现,调用mallocgc
- newarray: 数组分配器,调用mallocgc
- sysAlloc: 分配大内存,调用mmap
- persistentalloc: sysAlloc的包装,被mallocgc等调用,实际分配内存

## mallocgc
- _GCmarktermination阶段不允许分配内存

### nextFreeFast mspan快速重用空闲对象
- allocCache 与 count trailing zero 算法解读？？？
- allocBits的结构？？？
