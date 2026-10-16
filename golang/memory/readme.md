下面我把你的笔记按 **“内存层次 → 数据结构 → 分配路径 → GC 配合 → 页管理 → OS → 逃逸分析”** 重新组织，并修正几个关键错误。重点不是流水账，而是建立一张能够串起来的 **Go Runtime Memory Allocator 心智模型**。

> 下面以当前 Go runtime 源码为基准；部分内部结构会随 Go 版本变化，因此像 `mcentral` 的字段名、`mallocgc` 的具体 fast path 不应死记。当前源码仍保持“小对象 ≤32KB、8KB page、68 个 size class、per-P mcache”的核心模型。[Go](https://go.dev/src/runtime/malloc.go?utm_source=chatgpt.com)

# Go Runtime Memory — 结构化总结

## 1. 一张图先建立整体模型

```text
                         Go Program
                             │
                       new / make / escape
                             │
                             ▼
                         mallocgc()
                             │
              ┌──────────────┼──────────────┐
              │              │              │
          Tiny <16B      Small ≤32KB      Large >32KB
          noscan only        │              │
              │              │              │
              ▼              ▼              ▼
          mcache.tiny    mcache.alloc   mheap.alloc
                             │              │
                             ▼              ▼
                         mcentral        pageAlloc
                             │              │
                             ▼              ▼
                           mspan          pages
                             │              │
                             └──────┬───────┘
                                    ▼
                                  mheap
                                    │
                         ┌──────────┴──────────┐
                         ▼                     ▼
                    heapArena              OS memory
                         │                     │
                  heap bitmap             mmap / madvise
                  page → span
```

**核心思想：**

> `mallocgc` 决定“我要什么类型、什么大小的对象”；  
> `mcache` 尽量无锁地满足小对象；  
> `mcentral` 管理同一 size class 的 span；  
> `mheap/pageAlloc` 管理更底层的 page/span；  
> `heapArena` 保存 GC 和地址映射所需要的元数据。

Go runtime 官方源码也是按照 `fixalloc → mheap → mspan → mcentral → mcache` 这一层次描述 allocator 的。[Go](https://go.dev/src/runtime/malloc.go?utm_source=chatgpt.com)

---

# 2. 最重要的几个单位

| 概念 | 大小/含义 | 作用 |
|---|---:|---|
| OS physical page | 通常 4KB | CPU/MMU/OS 的物理页概念 |
| Go page | **8KB** | Go runtime allocator 的基本 page |
| chunk | **4MB** | `pageAlloc` 管理的 page bitmap 块 |
| heapArena | **64MB** | Go heap arena 元数据管理范围 |
| Tiny | **<16B** | 极小的无指针对象 |
| Small | **≤32KB** | size class 分配 |
| Large | **>32KB** | 直接从 heap 获取 span |

当前 Go 源码明确规定：

```text
pageSize      = 8KB
maxSmallSize  = 32KB
TinySize      = 16B
NumSizeClasses = 68
```

因此你的：

```text
page = 8KB
chunk = 4MB
arena = 64MB
```

这个核心数字组合是正确的。[Go](https://go.dev/src/internal/runtime/gc/sizeclasses.go?utm_source=chatgpt.com)

---

# 3. 内存分配器的核心层次

## 3.1 mcache：最快的一级缓存

```text
                P
                │
             mcache
       ┌────────┼─────────┐
       ▼        ▼         ▼
     tiny    alloc[]   stackcache
               │
               │
          68 × 2 spanClass
               │
               ▼
             mspan
```

`mcache` 是 **per-P** 的，不需要锁，因此是小对象分配的最快路径。当前源码明确将其定义为 per-P small-object cache。[Go](https://go.dev/src/runtime/mcache.go?utm_source=chatgpt.com)

主要内容：

```text
mcache
├── tiny
├── tinyoffset
├── alloc[spanClass]
├── stackcache
└── allocation statistics
```

注意：

> **mcache 不是直接缓存对象，而主要缓存 mspan。**

例如：

```text
P0
 │
 └── mcache
      │
      ├── class 16B noscan → mspan A
      ├── class 32B scan   → mspan B
      ├── class 64B noscan → mspan C
      └── ...
```

---

# 4. mspan：真正承载对象的内存块

`mspan` 是 allocator 中非常关键的概念：

> **一段连续 Go pages，专门用于某一种 size class 的对象分配。**

例如：

```text
mspan
startAddr = 0x10000000
npages    = 1
sizeclass = 5
elemsize  = 48B
```

那么：

```text
8KB / 48B ≈ 170 个对象
```

因此：

```text
mspan
┌────┬────┬────┬────┬────┬────┬─────┐
│48B │48B │48B │48B │... │48B │tail │
└────┴────┴────┴────┴────┴────┴─────┘
  0    1    2    3         169
```

当前 size class 表中 class 5 就是 48B、8KB span、170 objects。[Go](https://go.dev/src/internal/runtime/gc/sizeclasses.go?utm_source=chatgpt.com)

---

# 5. spanClass：size class + noscan

这是你原文中一个需要修正的地方。

你写：

> `spanClass：[1,67] * 2`

更准确地说：

```text
NumSizeClasses = 68

spanClass = sizeclass << 1 | noscan
```

所以理论范围：

```text
sizeclass: 0 ~ 67
spanClass : 0 ~ 135
```

其中：

```text
spanClass bit0
    0 → scan
    1 → noscan
```

例如：

```text
sizeclass = 5   // 48B

scan:
5 << 1 | 0 = 10

noscan:
5 << 1 | 1 = 11
```

因此：

```text
                 spanClass
                     │
              ┌──────┴──────┐
              ▼             ▼
           sizeclass      noscan
              │             │
           对象大小       是否需要GC扫描
```

当前源码也是：

```go
makeSpanClass(sizeclass, noscan)
```

并通过低 bit 区分 `noscan`。[Go](https://go.dev/src/runtime/mheap.go?h=new\&utm_source=chatgpt.com)

---

# 6. Tiny Allocator

## 6.1 为什么需要 Tiny？

大量对象非常小，例如：

```go
x := "a"
y := struct{ x byte }{}
```

如果每一个都单独分配一个 heap object，会产生大量 allocator/GC 管理开销。

所以 Go 对：

```text
< 16B
+
noscan
```

的对象使用 Tiny allocator。

当前源码明确说明 Tiny block 为 **16B**，目标主要是 small strings 和 standalone escaping variables。官方源码给出的 benchmark 中，分配次数约减少 12%，heap size 约减少 20%。[Go](https://go.dev/src/runtime/malloc.go?utm_source=chatgpt.com)

---

## 6.2 Tiny 的核心模型

不是：

```text
一个 tiny object → 一个独立内存块
```

而是：

```text
                 16B Tiny Block
          ┌─────────────────────┐
          │ A │ B │ C │ unused  │
          └─────────────────────┘
```

例如：

```text
A = 5B
B = 3B
C = 4B
```

可能组合成：

```text
16B block
┌─────┬───┬────┬────┐
│ A   │ B │ C  │    │
│ 5B  │3B │ 4B │4B  │
└─────┴───┴────┴────┘
```

只有整个 Tiny block 中的内容都不可达后，GC 才能回收对应 block。

因此 Tiny allocator 的关键收益是：

> **多个极小对象共享一个 allocation slot。**

---

## 6.3 Tiny 与 mcache

Tiny 本身属于：

```text
mcache.tiny
```

但它背后的 16B block 最终仍来自：

```text
mcache
   ↓
tinySpanClass
   ↓
mspan
```

当前源码中：

```go
tinySpanClass = tinySizeClass<<1 | 1
```

也就是 **16B + noscan** 对应的 span class。[Go](https://go.dev/src/runtime/mheap.go?h=new\&utm_source=chatgpt.com)

特别注意：

> `mcache` 本身是 non-GC memory，但 `mcache.tiny` 是一个 heap pointer，所以 runtime 在 `releaseAll` 等阶段需要特殊处理并清理它。[Go](https://go.dev/src/runtime/mcache.go?utm_source=chatgpt.com)

---

# 7. noscan：GC 是否扫描对象

判断核心：

```text
typ == nil || typ.ptrdata == 0
```

可以理解为：

```text
noscan
  ↓
对象内部没有需要 GC 跟踪的 pointer
  ↓
GC 不需要扫描对象内容
```

例如：

```go
int
float64
[100]byte
struct {
    a int
    b float64
}
```

属于典型 noscan。

而：

```go
struct {
    p *Node
}
```

属于 scan。

---

# 8. mallocgc：最重要的总入口

可以把 `mallocgc` 记成：

```text
                 mallocgc(size, typ)
                         │
                         ▼
                 判断对象属性
                         │
             ┌───────────┼───────────┐
             ▼           ▼           ▼
          tiny         small        large
         <16B          ≤32KB        >32KB
         noscan
             │           │           │
             ▼           ▼           ▼
          mcache      mcache       mheap
                         │           │
                         ▼           ▼
                      mspan       pageAlloc
```

当前源码对外的核心分类仍是：

```text
small objects ≤ 32KB
large objects > 32KB
``` :chatgpt-content-reference{index="9"}


---

## 8.1 mallocgc 核心流程

简化为：

```text
mallocgc(size, typ, needzero)
        │
        ├── size == 0 ?
        │       └── zerobase
        │
        ├── 判断 typ 是否含 pointer
        │
        ├── Tiny?
        │      │
        │      └── <16B + noscan
        │
        ├── Small?
        │      │
        │      └── ≤32KB
        │            ↓
        │          sizeclass
        │            ↓
        │          mcache
        │
        └── Large?
               │
               └── >32KB
                     ↓
                  mallocgcLarge
                     ↓
                   mheap
```

当前实现还有一些针对 no-scan allocation、malloc header、实验特性的 fast path，因此不要把源码强行记成单一固定分支。[Go](https://go.dev/src/runtime/malloc.go?utm_source=chatgpt.com)

---

# 9. Small Object：真正的快速路径

例如：

```go
p := new(MyStruct) // 48B
```

假设：

```text
48B
↓
sizeclass = 5
↓
spanClass = 10/11
↓
P.mcache.alloc[spanClass]
↓
mspan
```

---

## 9.1 如何找到空闲对象？

这是 `mspan` 中非常重要的一套机制：

```text
allocBits
```

每个 bit 对应一个 object slot。

例如：

```text
allocBits:

1 1 0 1 0 0 1 1
│ │ │ │ │ │ │ │
│ │ │ │ │ │ │ └─ 已分配
│ │ │ │ │ │ └─── 已分配
│ │ │ │ │ └───── 空闲
│ │ │ │ └─────── 空闲
│ │ │ └───────── 已分配
│ └───────────── 已分配
└─────────────── 已分配
```

通常：

```text
1 = allocated
0 = free
```

---

## 9.2 allocCache：为什么更快？

`mspan` 不会每次都直接扫描整个 `allocBits`。

它缓存：

```text
allocCache
```

并且：

```text
allocCache = ^allocBits
```

对应当前 `freeindex` 附近的部分。

因此：

```text
allocCache
      │
      ▼
  Ctz64()
      │
      ▼
找到第一个空闲 slot
```

当前 runtime 明确采用这种设计：`allocCache` 保存 `allocBits` 的补码，并使最低 bit 对应 `freeindex`，因此可以直接使用 `ctz` 找空闲 slot。[Go](https://go.dev/src/runtime/mheap.go?h=new\&utm_source=chatgpt.com)

---

# 10. nextFreeFast

核心可以记成：

```text
allocCache
      │
      ▼
Ctz64(allocCache)
      │
      ▼
theBit
      │
      ▼
freeindex + theBit
      │
      ▼
object index
      │
      ▼
base + index × elemsize
```

例如：

```text
freeindex = 10

allocCache:
111011001011...
      ↑
    第一个 0

theBit = 3

objectIndex = 10 + 3 = 13
```

最终：

```text
objAddr = span.base() + 13 × span.elemsize
```

然后：

```text
freeindex++
allocCache >>= ...
allocCount++
```

**面试记忆点：**

> `freeindex` 定位搜索起点，`allocCache` 提供局部 bitmap，`Ctz64` 快速找到第一个空闲 slot。

---

# 11. mcache 满了怎么办？

```text
mcache
  │
  │ 当前 mspan 没有 free slot
  ▼
mcache.nextFree()
  │
  ▼
mcache.refill()
  │
  ▼
mcentral.cacheSpan()
  │
  ├── partial swept
  ├── partial unswept → sweep
  ├── full unswept → sweep
  └── 没有可用 span
          │
          ▼
      mcentral.grow()
          │
          ▼
      mheap.allocSpan()
```

所以：

```text
mcache
   ↓
mcentral
   ↓
mheap
   ↓
pageAlloc
   ↓
OS
```

这就是 Go allocator 最重要的一条调用链。

---

# 12. mcentral：同一种规格的 Span 管理中心

可以把 `mcentral` 理解成：

> **某一个 size class 的 mspan 仓库。**

例如：

```text
mcentral[48B]

        ┌─────────────────────┐
        │      mcentral       │
        ├─────────────────────┤
        │ partial swept       │
        │ partial unswept     │
        │ full swept          │
        │ full unswept        │
        └─────────────────────┘
                 │
                 ▼
              mspan
```

注意一个容易误解的地方：

> **mcentral 并不直接维护每个 object 的 free list；真正的对象空闲信息在 mspan 的 bitmap 中。**

当前源码明确说明 mcentral 管理的是不同状态的 mspan 集合。[Go](https://go.dev/src/runtime/mcentral.go?utm_source=chatgpt.com)

---

# 13. mcentral.cacheSpan()

核心目标：

```text
给 mcache 找一个“有空闲对象”的 mspan
```

优先考虑：

```text
1. partial + swept
2. partial + unswept → sweep
3. full + unswept → sweep
4. 都没有 → grow
```

为什么存在 swept / unswept？

因为 GC 后：

```text
GC mark
   ↓
对象存活情况确定
   ↓
sweep
   ↓
回收死亡对象对应 slot
   ↓
mspan 出现 free slot
```

所以 allocator 可以：

> **一边分配，一边推动 sweep。**

---

# 14. mheap：更底层的堆管理器

可以把：

```text
mcache / mcentral
```

理解成：

> **对象级管理**

而：

```text
mheap / pageAlloc
```

是：

> **page/span 级管理**

核心职责：

```text
mheap
├── span 管理
├── page 管理
├── heapArena
├── mcentral
├── mspan 元数据
└── OS memory
```

官方 runtime 也是以 8KB page 为粒度管理 heap。[Go](https://go.dev/src/runtime/malloc.go?utm_source=chatgpt.com)

---

# 15. mheap.allocSpan()

目标：

```text
我要 n 个连续 page
```

简化流程：

```text
allocSpan(npage)
      │
      ▼
P.pageCache ?
      │
   ┌──┴──┐
   │     │
  有    无
   │     │
   │     ▼
   │   pageAlloc.alloc
   │     │
   └─────┤
         ▼
       得到 pages
         │
         ▼
     allocMSpanLocked
         │
         ▼
       mspan
         │
         ▼
      初始化
```

其中 `pageCache` 是 **per-P page cache**，可以让部分 page allocation 避免全局锁。当前源码明确说明它用于 per-P 无锁 page allocation。[Go](https://go.dev/src/runtime/mpagecache.go?utm_source=chatgpt.com)

---

# 16. pageCache

当前：

```go
type pageCache struct {
    base  uintptr
    cache uint64
    scav  uint64
}
```

```text
cache
64 bit
 │
 ├── bit 0 → 一个 8KB page
 ├── bit 1 → 一个 8KB page
 ├── ...
 └── bit 63 → 一个 8KB page

64 × 8KB = 512KB
```

所以你原文：

> pageCache 可缓冲 64 页

是正确的。

但要注意：

> `pageCache` 的 `cache` 是 **64 页 = 512KB**，而不是 4MB；4MB 是 `pageAlloc chunk` 的大小。[Go](https://go.dev/src/runtime/mpagecache.go?utm_source=chatgpt.com)

---

# 17. pageAlloc：管理 Go pages

这是比 `mspan` 更底层的 page allocator。

```text
pageAlloc
│
├── chunks
│     ├── chunk 0 → 4MB
│     ├── chunk 1 → 4MB
│     └── ...
│
├── pallocBits
│
├── scavenged
│
└── summary radix tree
```

核心：

```text
4MB chunk
=
512 × 8KB page
```

每个 chunk：

```text
pageBits [8]uint64

8 × 64 = 512 bits
       ↓
512 pages
       ↓
512 × 8KB = 4MB
```

官方源码也明确说明 page allocator 以 Go `pageSize` 管理映射页，并使用 bitmap + radix tree 高效搜索连续页。[Go](https://go.dev/src/runtime/mpagealloc.go?utm_source=chatgpt.com)

---

# 18. pageAlloc 为什么需要 radix tree？

假设：

```text
我要连续 100 个 page
```

不能每次：

```text
从头扫描 bitmap
```

否则：

```text
O(address space)
```

所以 Go 使用：

```text
bitmap
   +
radix tree summary
```

结构：

```text
                Summary
                   │
          ┌────────┴────────┐
          ▼                 ▼
       level 0            level 1
          │
          ▼
       level 2
          │
          ▼
       level 3
          │
          ▼
       level 4
          │
          ▼
       chunk/page bitmap
```

高层 summary 表示：

> **下面这一大片地址范围是否存在满足条件的空闲 page。**

于是寻找连续 page 时可以快速跳过大量“不可能有空间”的区域。

---

# 19. heapArena：地址 → Span / GC metadata

`heapArena` 可以理解为：

> **Go heap 地址空间对应的元数据块。**

当前 64-bit Unix 平台下，一个：

```text
heapArena = 64MB
```

对应：

```text
64MB / 8KB
= 8192 pages
```

所以：

```text
heapArena
┌─────────────────────────────┐
│ bitmap                      │
│                             │
│ spans[8192]                 │
│                             │
│ pageInUse                   │
│ pageMarks                   │
│ pageSpecials                │
└─────────────────────────────┘
```

---

# 20. heapArena 最重要的 3 类信息

### ① bitmap

记录对象内部：

```text
哪个 word 是 pointer？
哪个 word 是 scalar？
```

用于 GC 扫描。

---

### ② spans

```text
page index
     ↓
mspan*
```

例如：

```text
page 100 ───────┐
page 101 ───────┼──→ mspan A
page 102 ───────┘

page 103 ───────┐
page 104 ───────┼──→ mspan B
page 105 ───────┘
```

因此通过对象地址可以快速：

```text
address
  ↓
arena
  ↓
page index
  ↓
mspan
```

---

### ③ pageMarks

记录：

> 这个 page 对应的 span 是否有被 GC mark 的对象。

GC 新对象标黑时，会更新对应 arena 的 `pageMarks`。[Go](https://go.dev/src/runtime/mgcmark.go?utm_source=chatgpt.com)

---

# 21. mspan 中最重要的 Bitmap

一个 mspan 主要涉及三类状态：

```text
                mspan
                  │
        ┌─────────┼─────────┐
        ▼         ▼         ▼
     allocBits gcmarkBits  heapBits
        │         │           │
        │         │           └─ 对象内部 pointer 信息
        │         └───────────── GC mark
        └─────────────────────── 是否已经分配
```

可以记成：

```text
allocBits
    ↓
“这个 slot 有没有被分配？”

gcmarkBits
    ↓
“这个 object 在本轮 GC 是否存活/已标记？”

heapBits
    ↓
“这个 object 内部哪些位置是 pointer？”
```

---

# 22. GC 与 allocator 如何连接？

这是整套知识中最重要的部分。

```text
mallocgc
   │
   ▼
得到 object
   │
   ├── needzero
   │
   ├── heapBits 初始化
   │
   └── GC 正在 mark？
          │
          ▼
      gcmarknewobject
          │
          ▼
      mark object black
```

当前 runtime 在 GC mark 阶段分配新对象时，会调用 `gcmarknewobject`，直接将新对象标黑，并更新 span/page 的 mark 信息。[Go](https://go.dev/src/runtime/mgcmark.go?utm_source=chatgpt.com)

---

# 23. 为什么新对象需要“分配即黑”？

假设：

```text
GC 正在 Mark
```

此时程序：

```text
p = new(Object)
```

如果：

```text
new Object = white
```

那么 GC 可能认为：

```text
它还没被扫描
```

产生错误。

所以 GC mark 阶段：

```text
新对象
  ↓
allocate black
```

即：

```text
新对象直接认为“已标记”
```

当前源码正是这么处理的。[Go](https://go.dev/src/runtime/malloc.go?utm_source=chatgpt.com)

---

# 24. Write Barrier 与 mallocgc

这里你原来的表述需要调整。

不能简单记成：

> “write barrier 开启 → gcAssistBytes < 0 → 辅助 GC”

更准确是：

```text
mallocgc
   │
   ├── GC assist accounting
   │
   ├── 当前 G 是否需要 assist？
   │
   └── 必要时执行 gcAssistAlloc
```

也就是说：

> **GC assist 是分配者帮助 GC 完成标记工作的机制；它和 write barrier 有关联，但不能把两者等价。**

面试最好记：

```text
分配越多
   ↓
GC assist credit 消耗
   ↓
credit 不足
   ↓
当前 goroutine 帮助 GC mark
```

---

# 25. GC 回收与 mspan

一个 object 死亡后，并不是：

```text
object死亡
   ↓
立即把物理内存还给OS
```

而是：

```text
GC Mark
   ↓
确定 dead object
   ↓
Sweep
   ↓
mspan 中对应 slot 变成 free
   ↓
以后再次分配
```

因此：

```text
GC 回收
    ≠
立即 OS 回收
```

这是理解 Go 内存 RSS 的关键。

---

# 26. Span 回收 vs OS 回收

分两个层次：

```text
object dead
    ↓
slot free
    ↓
mspan reusable
    ↓
span/page 不再需要
    ↓
pageAlloc free
    ↓
scavenging
    ↓
madvise
    ↓
OS
```

所以：

> **Go GC 主要解决“对象是否可达”；allocator/scavenger 决定“这些物理/虚拟页什么时候归还 OS”。**

---

# 27. scav：Scavenging

你原文：

> scav 中保存的是空闲并且已经垃圾回收的 span

这个说法过于粗略。

更准确：

> **scavenging 针对已经不再被 Go allocator 使用的内存页，把物理 backing 尽可能归还/释放给 OS，同时保留虚拟地址空间，使 runtime 后续仍可重新使用。**

可以记：

```text
GC
 ↓
对象死亡

allocator
 ↓
page 不再使用

scavenger
 ↓
释放物理 backing

virtual address
 ↓
可以继续保留
```

---

# 28. madvise

`madvise` 本质：

> 应用告诉内核：这段映射未来希望怎样使用。

常见：

### MADV_DONTNEED

```text
这块内存暂时不用
       ↓
OS 可以回收 backing pages
       ↓
以后再次访问可能产生 page fault
```

特点：

```text
RSS下降更积极
```

---

### MADV_FREE

```text
这块内存暂时不用
       ↓
内核在内存紧张时再回收
       ↓
如果重新使用，可以直接复用
```

特点：

```text
回收更惰性
RSS不一定立即下降
```

---

### MADV_HUGEPAGE

提示内核：

```text
这段区域适合使用 huge page
```

---

### MADV_NOHUGEPAGE

告诉内核：

```text
不要将该区域合并为 huge page
```

---

# 29. Huge Page 与 THP

不要混淆：

```text
Huge Page
    ↓
一种大页机制

THP
    ↓
Linux Transparent Huge Pages
    ↓
内核自动管理大页
```

典型：

```text
普通 page
4KB

Huge Page
2MB / 1GB

THP
动态地尝试把普通页合并为大页
```

所以：

> Huge Page 是概念/机制；THP 是 Linux 对透明大页的一套自动管理机制。

---

# 30. mmap：Go 向 OS 要内存

最常见的两个路径：

```text
sysReserve
    ↓
mmap(... PROT_NONE ...)
```

用于：

> **预留虚拟地址空间**

而：

```text
sysAlloc
    ↓
mmap(... PROT_READ | PROT_WRITE ...)
```

用于：

> **实际建立可访问的匿名映射**

---

## mmap flags

| 参数 | 含义 |
|---|---|
| `PROT_NONE` | 不允许访问 |
| `MAP_PRIVATE` | 私有映射，写时复制语义 |
| `MAP_SHARED` | 共享映射 |
| `MAP_ANONYMOUS` | 不关联文件 |
| `MAP_FIXED` | 要求映射到指定地址，使用需非常谨慎 |

特别注意：

> `MAP_PRIVATE` 并不是“Go 私有内存”的意思，而是 mmap 的映射语义。

---

# 31. fixalloc

`fixalloc` 和 Go heap object allocator 是两套不同层次的东西。

```text
fixalloc
    ↓
固定大小对象
    ↓
runtime 自己使用
```

例如：

```text
mcache
mspan
runtime metadata
```

它本质是：

```text
固定大小
+
free list
+
chunk
```

当前源码中 `fixalloc` 是固定大小对象的 free-list allocator，并在没有空闲对象时通过 `persistentalloc` 获取新的 chunk。[Go](https://go.dev/src/runtime/mfixalloc.go?utm_source=chatgpt.com)

---

# 32. persistentalloc

可以记：

```text
persistentalloc
        │
        ├── 小分配
        │      ↓
        │   从 persistent chunk 切
        │
        └── 大分配
               ↓
             sysAlloc
```

特点：

> **persistentalloc 分配的内存不会被普通 GC 回收。**

Go runtime 文档也明确说明 persistentalloc 用于合并多个小的 runtime 内部分配，但这类内存没有普通 free 路径。[Go](https://go.dev/src/runtime/HACKING?utm_source=chatgpt.com)

---

# 33. 逃逸分析

这是 allocator 之前的一个重要环节：

```text
Go compiler
    │
    ▼
Escape Analysis
    │
    ├── 不逃逸
    │      ↓
    │    stack
    │
    └── 逃逸
           ↓
         heap
           ↓
        mallocgc
```

所以：

> **不是 `new()` 就一定 heap。**

编译器会判断对象生命周期是否可能超过当前栈帧。

---

# 34. 常见逃逸情况

### ① 返回局部变量地址

```go
func f() *int {
    x := 10
    return &x
}
```

```text
x
 ↓
函数结束后仍被使用
 ↓
heap
```

---

### ② 闭包捕获

```go
func f() func() int {
    x := 10
    return func() int {
        return x
    }
}
```

```text
x
 ↓
被闭包引用
 ↓
生命周期超过当前栈
 ↓
escape
```

---

### ③ 变量引用关系导致逃逸

核心不是简单记：

> “loop 外变量在 loop 内赋值就逃逸”

而是：

> **编译器通过对象之间的引用关系和生命周期判断是否需要将对象提升到 heap。**

你笔记中的“有向图 + 权重”可以作为理解模型，但不要把它当成当前 Go compiler 的源码级唯一实现公式。

简化理解：

```text
x ──引用──> y

&x  → 可能产生向外的引用
*x  → 改变引用分析中的间接层级

如果引用关系最终表明：
对象生命周期超过当前栈帧
        ↓
escape
```

---

# 35. 你最应该记住的完整分配链

这是整份文档最值得背的一张图：

```text
                 Go object allocation
                         │
                     mallocgc
                         │
          ┌──────────────┼──────────────┐
          │              │              │
       <16B            ≤32KB          >32KB
       noscan           small          large
          │              │              │
          ▼              ▼              ▼
      tiny block      sizeclass       mheap
          │              │              │
          ▼              ▼              ▼
       mcache         mcache         pageAlloc
                         │              │
                         ▼              ▼
                     mcentral       free pages
                         │              │
                         ▼              │
                       mspan ◄──────────┘
                         │
                  ┌──────┴──────┐
                  ▼             ▼
              allocBits     gcmarkBits
                  │             │
             slot是否使用     GC是否存活
```

---

# 36. 再向下追一层：Page

```text
mspan
  │
  ├── startAddr
  ├── npages
  ├── elemsize
  ├── nelems
  ├── freeindex
  ├── allocCache
  ├── allocBits
  ├── gcmarkBits
  ├── spanclass
  └── sweepgen
```

而：

```text
mspan
  ↓
由 mheap 管理
  ↓
占用若干 8KB pages
```

---

# 37. 再向下：chunk

```text
4MB chunk
┌──────────────────────────────────────────────┐
│ page0 │ page1 │ ... │ page510 │ page511     │
└──────────────────────────────────────────────┘
     8KB     8KB              ...       8KB
```

因此：

```text
1 chunk
= 512 pages
= 512 × 8KB
= 4MB
```

`pageAlloc` 使用 bitmap 管理这些 page，并通过 radix-tree summary 加速连续 page 查找。[Go](https://go.dev/src/runtime/mpagealloc.go?utm_source=chatgpt.com)

---

# 38. 再向上：heapArena

```text
heapArena = 64MB

┌─────────────────────────────────────┐
│ chunk │ chunk │ ... │ chunk         │
│  4MB  │  4MB  │     │  4MB          │
└─────────────────────────────────────┘

64MB / 4MB = 16 chunks
```

而：

```text
64MB / 8KB = 8192 pages
```

所以你可以形成一个非常好的层次记忆：

```text
8KB
 ↓
page

4MB
 ↓
chunk
 ↓
512 pages

64MB
 ↓
heapArena
 ↓
8192 pages
 ↓
16 chunks
```

---

# 39. 内存地址与 arena

64-bit Unix 平台典型模型：

```text
virtual address
       │
       ▼
arenaIndex(address)
       │
       ▼
heapArena
       │
       ├── bitmap
       ├── spans[]
       ├── pageInUse
       └── pageMarks
```

amd64 需要特别注意：

```text
硬件地址位限制
     ↓
当前 Linux 用户态通常使用 48-bit 地址范围
     ↓
Go runtime 对 arena index 做地址偏移处理
```

当前 Go 源码也特别说明 amd64 的地址空间和 `arenaBaseOffset` 的关系；Linux 当前支持 57-bit 地址能力，但 runtime 通常不会让普通 `mmap` 自行选择到 `1<<47` 以上。[Go](https://go.dev/src/runtime/malloc.go?utm_source=chatgpt.com)

所以你原文：

> “amd64 只用 48bit 地址位”

应该改成：

> **经典 amd64 地址转换使用 48-bit canonical virtual address；新硬件支持 57-bit，但实际可用地址空间还取决于 OS/页表配置。**

---

# 40. mallocinit

启动阶段：

```text
runtime 初始化
      │
      ▼
mallocinit
      │
      ├── mheap 初始化
      ├── mcentral 初始化
      ├── pageAlloc 初始化
      ├── arena metadata 初始化
      ├── mcache0
      └── arenaHint
```

`arenaHint` 的作用可以简单记成：

> **给 heap arena 扩张提供地址提示/方向信息。**

不要把它理解为“真正已经分配的 heap”。

---

# 41. 几个容易混淆的概念

| 概念 | 作用 |
|---|---|
| `mcache` | per-P，小对象快速分配 |
| `mcentral` | 某 size class 的 mspan 集合 |
| `mheap` | 全局 heap/span/page 管理 |
| `mspan` | 一段连续 page，承载同规格对象 |
| `pageAlloc` | 管理 8KB page |
| `pageCache` | per-P 的 page 快速缓存 |
| `heapArena` | heap 地址对应的元数据 |
| `allocBits` | object slot 是否已分配 |
| `gcmarkBits` | object 是否被 GC mark |
| `heapBits` | object 内部 pointer 布局 |
| `fixalloc` | runtime 固定大小对象 |
| `persistentalloc` | runtime 持久内存 |
| `scavenger` | 将不再使用的物理 backing 归还 OS |

---

# 42. 你的原文中需要重点修正的地方

### ① `spanClass`

原：

```text
[1,67] × 2
```

改：

```text
NumSizeClasses = 68
sizeclass = 0~67
spanClass = sizeclass<<1 | noscan
理论范围 0~135
```

其中 `0` sizeclass 是保留/特殊意义，实际普通小对象 size class 从 8B 开始。[Go](https://go.dev/src/internal/runtime/gc/sizeclasses.go?utm_source=chatgpt.com)

---

### ② Tiny spanClass

你写：

```text
tiny 内存由 spanClass 5 承载
```

这是**错误的**。

当前：

```text
TinySizeClass = 2
TinySize = 16B

tinySpanClass
= 2 << 1 | 1
= 5
```

所以：

> **spanClass 5 是正确的，但它对应的是 sizeclass 2 + noscan，也就是 16B tiny class。**

不要说成“spanClass 5 的 span 就是 Tiny allocator”，更准确是：

```text
tinySpanClass = 5
``` :chatgpt-content-reference{index="25"}


---

### ③ `allocCache`

原文：

> 0表示已分配

容易混淆。

应该记：

```text
allocBits:
    1 = allocated
    0 = free

allocCache:
    ≈ ^allocBits
    1 = free
    0 = allocated
```

因此：

```text
Ctz64(allocCache)
```

是在找：

> **第一个 free slot。** [Go](https://go.dev/src/runtime/mheap.go?h=new\&utm_source=chatgpt.com)


---

### ④ `mcentral`

不要记成：

> mcentral 保存 free object

更准确：

```text
mcentral
   ↓
保存 mspan 集合

mspan
   ↓
通过 allocBits 管理 object slots
``` :chatgpt-content-reference{index="27"}


---

### ⑤ `mcache`

不要说：

> mcache 是线程本地缓存

Go runtime 更准确是：

> **per-P cache**

因为 Go 的调度模型是：

```text
G → M → P
       │
       └── mcache
``` :chatgpt-content-reference{index="28"}


---

### ⑥ `mspan.scavenge()`

不要理解成：

```text
每个 mspan 自己直接负责把内存还给 OS
```

应该分层理解：

```text
mspan / mheap
      ↓
page 不再使用
      ↓
scavenger
      ↓
madvise
      ↓
OS
```

---

### ⑦ `persistentalloc`

不是：

> mallocgc 等调用它来实际分配普通 heap object

更准确：

```text
普通 Go heap object
    ↓
mallocgc
    ↓
mcache/mcentral/mheap

runtime 自己的非 GC 内存
    ↓
fixalloc / persistentalloc
    ↓
sysAlloc
```

`persistentalloc` 是 runtime 内部持久分配路径，而不是普通 Go object 的主要分配路径。[Go](https://go.dev/src/runtime/HACKING?utm_source=chatgpt.com)

---

# 43. 最终面试速记版

如果面试官问：

> **Go 内存分配器是怎么工作的？**

建议直接按下面回答：

```text
Go allocator 采用分层缓存设计。

1. 编译器先做 escape analysis：
   不逃逸 → stack；
   逃逸 → heap → mallocgc。

2. mallocgc 根据对象大小和是否含 pointer 分类：
   <16B 且 noscan → Tiny allocator；
   ≤32KB → Small allocator；
   >32KB → Large allocator。

3. Small allocation：
   mallocgc
      → 当前 P 的 mcache
      → 对应 spanClass 的 mspan
      → allocCache/allocBits 找 free slot。

4. mcache 没有空间：
   → mcentral.cacheSpan()
   → 获取可用 mspan；
   → 没有则 mcentral.grow()
   → mheap.allocSpan()
   → pageAlloc 分配连续 page。

5. mspan 是对象承载单位：
   一个 mspan 由若干 8KB page 组成，
   一个 span 对应一个 size class，
   allocBits 管理对象是否分配，
   gcmarkBits 管理 GC 标记，
   heapBits 描述对象内部 pointer 布局。

6. pageAlloc 管理 8KB page：
   4MB 一个 chunk，
   bitmap + radix-tree summary 查找连续 page。

7. GC 后：
   dead object → sweep → mspan slot 重新可用；
   不再需要的 page → scavenger → madvise → OS。

因此整个分配链是：

escape analysis
    ↓
mallocgc
    ↓
mcache
    ↓
mcentral
    ↓
mheap
    ↓
pageAlloc
    ↓
OS
```

**最核心的 6 个关键词：**

```text
mcache   → 快
mcentral → 同规格 span 仓库
mspan    → 对象承载
mheap    → 全局 heap/span 管理
pageAlloc→ page 管理
heapArena→ 地址/GC 元数据
``` :chatgpt-content-reference{index="30"}
