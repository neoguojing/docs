# Go GC 原理与 Runtime 源码整理（修正版）

> 本文基于原始笔记整理，**保留原文涉及的知识点、结构、函数、常量和代码片段**，同时修正明显的概念、公式、术语和流程错误，并将重复内容合并到对应章节。
>
> **版本说明**：Go GC 的 runtime 内部实现会随 Go 版本演进。本文重点描述当前 Go runtime 的通用架构；具体函数名、字段和实验性实现应以对应 Go 版本源码为准。当前 runtime 仍采用并发三色标记 + 混合写屏障 + 并发清扫的基本架构。官方源码明确说明：清扫可以在正常程序执行期间并发进行，而下一轮并发标记开始前必须确保必要的 sweep 工作已经完成。

---

# 1. GC 总览

## 1.1 GC 的目标

Go GC 的核心目标不是“把 GC CPU 固定限制在 25%”，而是通过 **GC pacing（GC 节奏控制器）**，让后台标记工作目标约占 `GOMAXPROCS` 可用 CPU 的 **25%**，并结合分配速度、扫描速度、堆大小和内存限制动态调整 GC 触发点以及 mutator assist。

可以把它理解为：

```text
                 应用程序分配速度
                       │
                       ▼
                 ┌─────────────┐
                 │ GC Pacer    │
                 │ gcController│
                 └──────┬──────┘
                        │
          ┌─────────────┼─────────────┐
          ▼             ▼             ▼
      触发时机       后台 worker     Assist 比例
          │             │             │
          └─────────────┼─────────────┘
                        ▼
                 在分配完成前完成标记
```

`gcBackgroundUtilization = 0.25` 是当前 runtime 中后台标记的目标 CPU 利用率。它不是“GC 总 CPU 使用率永远不超过 25%”的硬上限，因为 mutator assist、STW 阶段、idle worker 等都会产生额外 GC CPU 时间。

---

## 1.2 GC 触发方式

GC 主要有三种触发条件：

1. **Heap trigger**：存活堆大小达到 GC controller 计算出的 trigger。
2. **Time trigger**：距离上次 GC 超过 `forcegcperiod`，且 GC 没有被关闭。
3. **Cycle trigger**：用户显式要求执行某个 GC cycle，例如 `runtime.GC()`。

核心判断逻辑可以简化为：

```text
_gcOff
  │
  ├── heapLive >= heapTrigger ───────► GC
  │
  ├── now-lastGC > forcegcperiod ───► GC
  │
  └── requestedCycle > work.cycles ─► GC
```

当前 runtime 的 `gcTrigger.test()` 还要求 GC 当前处于 `_GCoff`，并且 GC 已启用；time trigger 在 `gcPercent < 0` 时不会触发。citeturn0search3

### 小堆与 4MB

原笔记中的“4MB 以下不会触发 GC”需要修正。

更准确地说：runtime 有 `heapMinimum`，用于避免极小堆因为频繁 GC 而产生过高固定开销。当前实现中默认的 `defaultHeapMinimum` 与 GOGC 有关，初始默认值对应 GOGC=100 时约为 4MB；它是**最小 heap goal 的保护机制**，不能简单理解成“4MB 以下绝对不会 GC”。GOGC=0 时则可以得到持续触发的效果，适合调试。citeturn3search0

---

# 2. 一个完整 GC Cycle 到底做什么？

Go GC 可以抽象成：

```text
                    GC Trigger
                        │
                        ▼
              ┌──────────────────┐
              │ 准备 / 初始化     │
              └────────┬─────────┘
                       │
                       ▼
              ┌──────────────────┐
              │ STW：Sweep Term  │
              │ 完成上一轮清扫    │
              │ clearpools       │
              │ 准备 root / worker│
              └────────┬─────────┘
                       │
                       ▼
              ┌──────────────────┐
              │ Start The World  │
              └────────┬─────────┘
                       │
                       ▼
       ┌────────────────────────────────┐
       │ Concurrent Mark                │
       │                                │
       │  Root Scan                     │
       │      ↓                         │
       │  Grey Object                   │
       │      ↓                         │
       │  scanobject                    │
       │      ↓                         │
       │  gcw / workbuf                 │
       │                                │
       │  + write barrier               │
       │  + mutator assist              │
       └────────────────┬───────────────┘
                        │
                        ▼
              ┌──────────────────┐
              │ STW：Mark Term    │
              │ Flush WB Buffer   │
              │ 重新确认 mark 完成 │
              └────────┬─────────┘
                       │
                       ▼
              ┌──────────────────┐
              │ _GCoff           │
              │ 关闭写屏障        │
              │ 开始 Sweep        │
              └────────┬─────────┘
                       │
                       ▼
              ┌──────────────────┐
              │ Concurrent Sweep │
              │ bgsweep          │
              │ + proportional    │
              │   / direct sweep  │
              └────────┬─────────┘
                       │
                       ▼
              ┌──────────────────┐
              │ Scavenge          │
              │ 向 OS 归还页       │
              └──────────────────┘
```

这里最容易混淆的一点就是：**Sweep 通常不是整个过程 STW。**

---

# 3. 为什么 Sweep 可以不 STW？——原问题

## 3.1 问题

> 清扫阶段，没有 STW，不会影响结果吗？

**不会。关键在于：Sweep 使用的是上一轮 Mark 已经确定好的存活信息，而不是重新判断对象是否存活。**

Mark 阶段结束后：

```text
mark bit = 1  → 本轮 GC 认为对象存活
mark bit = 0  → 本轮 GC 认为对象不可达，可以回收
```

Sweep 做的事情主要是：

```text
mspan
  │
  ├── object A → mark=1 → 保留
  ├── object B → mark=0 → 回收
  ├── object C → mark=1 → 保留
  └── object D → mark=0 → 回收
```

因此 Sweep 不需要再次遍历整个对象图。

官方 runtime 的 sweep 实现明确将 Sweep 分成两类：

- object reclaimer：回收 span 中未标记的对象槽位；
- span reclaimer：如果整个 span 没有存活对象，则可以直接回收整个 span。

这两类清扫都可以由后台 `sweepone` 并发推进，也可以在分配路径上由 `mcentral.cacheSpan` 等路径同步完成。citeturn1search0

---

## 3.2 为什么并发 Sweep 不会和用户分配冲突？

关键是 **span ownership + sweepgen + 分配前确保 span 已经 swept**。

可以简化为：

```text
        span
          │
          ▼
   ┌───────────────┐
   │ 未清扫 unswept │
   └───────┬───────┘
           │ sweepone / cacheSpan
           ▼
   ┌───────────────┐
   │ 正在清扫       │
   └───────┬───────┘
           │
           ▼
   ┌───────────────┐
   │ swept          │
   │ 可用于分配      │
   └───────────────┘
```

如果分配器需要一个 span，而这个 span 尚未完成 sweep，它不能直接拿来分配；它必须先确保该 span 已经 swept。

因此不是：

```text
GC sweep                  用户分配
    │                         │
    ├── 随便清扫 ─────────────►├── 随便使用
    │                         │
    └────────────── 数据竞争 ❌
```

而是：

```text
GC sweep                    allocator
   │                            │
   │ sweep span                 │
   │───────────────►            │
   │                            │
   │ sweepgen 更新              │
   │                            │
   └───────────────────────────►│
                                │
                         检查 span 是否 swept
                                │
                                ▼
                              使用
```

当前 runtime 中 `mspan.sweep` 在完成 mark bits → alloc bits 的切换、清理 mark bits 后，最终将 `s.sweepgen` 更新到当前 heap sweep generation；分配代码以 span 已经 swept 为前提使用它。citeturn1search0

---

## 3.3 那么什么时候必须 STW？

**下一轮 Mark 开始之前，必须保证上一轮需要完成的 Sweep 已经结束。**

当前 runtime 的 `gcStart` 会在第一次 STW 后调用 `finishsweep_m()`；该函数在 world stopped 状态下循环 `sweepone()`，确保没有剩余的 sweep 工作，然后才进入新的 concurrent mark。citeturn2view0turn1search0

因此可以记成：

```text
Sweep 可以并发
      ↓
下一轮 GC 即将开始
      ↓
STW
      ↓
finishsweep_m()
      ↓
确保上一轮 Sweep 完成
      ↓
开始新一轮 Mark
```

### 面试一句话

> **Go 的 Sweep 不需要全程 STW，因为存活信息已经在 Mark 阶段确定；Sweep 只根据 mark bits 回收对象/slot，并通过 sweepgen 和分配路径保证 span 在复用前已经完成清扫。新一轮 Mark 开始前，如果还有未完成的 Sweep，则会在 STW 阶段通过 `finishsweep_m()` 收尾。**

---

# 4. GC 工作模式：生产者 + 消费者

GC 标记本质上可以理解为一个生产者/消费者系统。

## 4.1 Producer

生产工作主要来自：

- root scanning；
- `scanobject` 发现的新对象；
- write barrier 产生的新标记工作；
- mutator assist；
- 各种 `gcw` / `workbuf`。

简化：

```text
Root / Object Scan
       │
       ▼
   grey object
       │
       ▼
   local gcw
       │
       ├── 满 ──► global work.full
       │
       └── 不满 ─► local workbuf

Write Barrier
       │
       ▼
     wbBuf
       │
       ▼
   wbBufFlush1
       │
       ▼
     gcw / global work
```

## 4.2 Consumer

后台 worker、mutator assist 等从工作队列中获取灰色对象，然后：

```text
gcw / workbuf
     │
     ▼
  grey object
     │
     ▼
 scanobject
     │
     ├── 找到 pointer
     │
     ▼
 greyobject
     │
     ▼
 加入 gcw
```

最终直到没有可处理的灰色对象。

---

# 5. 三色标记法

## 5.1 三种颜色

```text
白色：还没有被 GC 发现
灰色：已经发现，但对象内部的引用还没有完全扫描
黑色：对象已经扫描完成
```

GC 开始时：

```text
          Root
           │
           ▼
        ┌───────┐
        │ 灰色  │
        └───┬───┘
            │ scan
      ┌─────┴─────┐
      ▼           ▼
   ┌───────┐   ┌───────┐
   │ 灰色  │   │ 灰色  │
   └───────┘   └───────┘
      │
      ▼
   扫描完成
      │
      ▼
   ┌───────┐
   │ 黑色  │
   └───────┘
```

当不存在灰色对象时：

```text
白色 → 不可达 → 回收
黑色 → 可达   → 保留
```

---

# 6. 并发 Mark 为什么需要写屏障？

如果整个 Mark 都 STW，用户程序不会修改对象图，因此三色标记很容易保证。

但是 Go 希望：

```text
GC Mark                    Mutator
   │                          │
   │ 扫描                     │ 修改对象引用
   │                          │
   └──────── 并发 ────────────┘
```

于是可能出现漏标。

## 6.1 漏标条件

经典的漏标场景可以抽象为：

1. **黑色对象引用白色对象**；
2. 灰色对象到白色对象的可达关系又被破坏。

例如：

```text
GC 已经扫描：

A(黑)
 │
 ▼
B(灰)
 │
 ▼
C(白)
```

Mutator 同时执行：

```text
B ──X──► C
A ─────────► C
```

如果没有屏障，GC 可能认为 C 不再可达，从而错误回收 C。

---

# 7. 写屏障

## 7.1 强三色不变式

要求：

> **黑色对象不能直接引用白色对象。**

也就是：

```text
黑 ──X──► 白
```

因此需要在引用发生变化时，把相关对象重新变成灰色。

### 插入屏障

核心思想：

```text
old ──► new
          │
          ▼
        shade(new)
```

新建立的引用对象被标记为灰色，使其不会被漏掉。

原笔记中的“只在堆内存分配起作用”需要修正：**写屏障针对的是可能改变堆中指针字段的写操作，不等同于“堆内存分配”。**

另外，纯插入屏障的经典实现会遇到栈重新扫描问题，因此 Go 最终采用了混合写屏障。

---

## 7.2 弱三色不变式

弱三色不变式允许：

```text
黑 → 白
```

但要求白色对象仍然受到灰色对象的保护，例如：

```text
黑 → 灰 → 白
```

或者：

```text
黑 → 白 ← 灰
```

### 删除屏障

当删除一个引用时，如果原对象尚未被安全处理，就将相关对象标记为灰色。

这样可以保证删除引用不会导致对象突然丢失。

缺点是需要在 GC 开始时建立足够的初始快照/栈处理，并且精度和实现复杂度存在权衡。

---

# 8. Go 的混合写屏障

Go 使用 **Hybrid Write Barrier**，将：

- Yuasa-style deletion barrier；
- Dijkstra-style insertion barrier

结合起来。

官方 runtime 的核心伪代码可以概括为：

```text
writePointer(slot, ptr):
    shade(*slot)          // deletion barrier
    if current stack is grey:
        shade(ptr)        // insertion barrier
    *slot = ptr
```

也就是：

```text
                 写指针
                   │
          ┌────────┴────────┐
          ▼                 ▼
       旧引用             新引用
          │                 │
          ▼                 ▼
        shade             stack灰？
                            │
                       yes ─┴─► shade
```

官方 runtime 当前源码明确描述 Go 使用这种混合屏障。citeturn0search4

### 主要优势

> **避免传统插入屏障在 GC 结束时对所有栈进行重新扫描，从而显著降低 STW 的 mark termination 成本。**

---

# 9. Go GC 的三个阶段

## `_GCoff`

- GC 没有运行；
- 后台可能正在 Sweep；
- 写屏障关闭；
- 正常分配继续运行。

## `_GCmark`

- 正在 Mark root 和 heap；
- 写屏障开启；
- 新分配对象直接按黑色处理；
- 后台 mark worker、mutator assist 共同推进 Mark。

## `_GCmarktermination`

- Mark 即将结束；
- 需要 STW；
- flush 各 P 的写屏障缓冲；
- 确认没有剩余 Mark 工作；
- 关闭写屏障并进入 Sweep。

官方 runtime 当前定义也是这三个阶段，并明确指出 `_GCmarktermination` 中写屏障仍然开启。citeturn0search3

```text
_GCoff
  │
  │ trigger
  ▼
_GCmark
  │
  │ mark done
  ▼
_GCmarktermination
  │
  │ sweep start
  ▼
_GCoff
```

---

# 10. GC 主流程

## 10.1 准备阶段

`gcStart()` 大致做：

1. 判断当前 G/M 是否适合启动 GC；
2. 必要时推进剩余 Sweep；
3. 获取 `startSema`，避免多个 G 同时启动 GC；
4. 重新检查 trigger；
5. 获取 `gcsema`；
6. 获取 `worldsema`；
7. 初始化每个 P 的状态；
8. 启动 `gcBgMarkStartWorkers()`；
9. `gcResetMarkState()`；
10. STW；
11. `finishsweep_m()`；
12. `clearpools()`；
13. `gcController.startCycle()`；
14. 设置 GC phase；
15. Start The World，进入并发 Mark。

当前 runtime 源码也明确显示：`gcStart` 会先进入 STW，完成 `finishsweep_m()` 后再让 assists/workers 开始新 cycle。citeturn2view0

---

## 10.2 STW 的作用

STW 的本质是：

> **停止用户 G 的执行，使所有 P 进入 GC 安全状态，从而安全地完成必须全局一致的 runtime 操作。**

当前 runtime 的 `stopTheWorldWithSema` 会：

```text
sched.stopwait = GOMAXPROCS
sched.gcwaiting = 1
        │
        ▼
   preemptall()
        │
        ▼
停止各 P
        │
        ▼
_Pgcstop / idle / syscall 状态处理
        │
        ▼
sched.stopwait == 0
        │
        ▼
World Stopped
```

启动 World 时：

```text
World Stopped
     │
     ▼
处理 netpoll 中可运行 G
     │
     ▼
sched.gcwaiting = 0
     │
     ▼
wakep()
     │
     ▼
World Running
```

原笔记中“g 都在全局队列上等待”过于绝对。STW 时 G 的状态可能不同：运行中的 G 会被抢占，阻塞中的 G 可能继续保持等待状态，系统调用中的 P 也有专门处理逻辑；重新启动世界时，runtime 会重新注入可运行 G。

---

# 11. Mark 阶段

## 11.1 Mark Worker

进程启动时 runtime 会准备后台 Mark Worker；真正进入 Mark 阶段后，scheduler 根据需要选择 worker。

主要有：

```text
gcMarkWorkerDedicatedMode
    专用 worker，尽量持续运行

 gcMarkWorkerFractionalMode
    运行一部分 CPU 时间

 gcMarkWorkerIdleMode
    只在系统空闲时帮助 GC
```

此外还有：

```text
gcMarkWorkerNotWorker
    当前 G 不是 Mark Worker
```

官方 runtime 当前实现明确将并发 Mark 分成 mutator assists 和三类 P 上的 mark workers。citeturn0search3

---

## 11.2 gcDrain

`gcDrain` 是 Mark Worker 消费 Mark 工作的核心逻辑之一。

可以简化为：

```text
                 gcDrain
                    │
          ┌─────────┴─────────┐
          ▼                   ▼
       markroot            gcw/workbuf
          │                   │
          ▼                   ▼
      root对象             灰色对象
          │                   │
          └─────────┬─────────┘
                    ▼
               scanobject
                    │
                    ▼
              新的灰色对象
                    │
                    └──────► gcw
```

它会根据不同 `gcDrainFlags` 控制：

- `gcDrainUntilPreempt`：执行直到被抢占；
- `gcDrainIdle`：空闲时执行 Mark；
- `gcDrainFractional`：执行 fractional worker 的 Mark；
- `gcDrainFlushBgCredit`：把 worker 的工作量计入后台扫描 credit。

---

# 12. Root 有哪些？

GC 必须从 Root 开始遍历对象图。

原笔记涉及的主要 root 类型包括：

- `data`：全局已初始化数据；
- `bss`：全局未初始化数据；
- `mspan` / span roots：与特殊对象、finalizer 等相关；
- goroutine stacks：所有需要扫描的 G 栈；
- mcache 相关 root / cache；
- finalizer / cleanup 等 runtime 管理的对象。

`moduledata` 记录了 data/bss 等全局区域的信息。

注意：`allg` 本身不是“对象 root”，更准确地说，runtime 通过 goroutine 列表找到需要扫描的 G，再扫描其栈及相关上下文。

当前源码的 `gcMarkRootPrepare` 会统计 data、BSS、span、stack 等 root 工作；同时也存在 finalizer 和 cleanup 等 root 类型。citeturn0search11

---

# 13. markroot

`markroot` 根据 root job 编号处理不同类型的 root。

可以理解为：

```text
markroot
   │
   ├── mcache / cache
   │
   ├── data
   │
   ├── bss
   │
   ├── span roots
   │
   └── goroutine stacks
             │
             ▼
         scanstack
```

原笔记中的顺序“mcache → data → bss → mspan → stack”可以作为理解上的 job 分类顺序，但不要把它理解成 GC 语义上的固定“扫描优先级”。

---

# 14. scanblock / scanstack / scanobject

## 14.1 scanblock

`scanblock` 根据编译器提供的 pointer bitmap 扫描一个内存区域。

核心逻辑：

```text
内存块
 │
 ▼
ptr bitmap
 │
 ├── bit=0 → scalar，不扫描
 │
 └── bit=1 → pointer
                │
                ▼
            找到对象
                │
                ▼
            greyobject
```

---

## 14.2 scanstack

扫描 G 的栈时，需要识别：

- 本地变量；
- 函数参数；
- 返回值相关的栈槽；
- defer；
- panic 相关指针；
- 栈上的对象。

因此原笔记中的“scanframeworker：本地变量、函数参数和返回值”可以修正为：

> **`scanframeworker` 等栈扫描逻辑依据栈帧元数据扫描栈帧中的指针槽，包括参数、局部变量以及编译器/runtime 描述的其他指针信息。**

并不是简单地“把整个栈当作一个字节数组扫描”。

---

## 14.3 scanobject

`scanobject(b)` 从对象 `b` 开始扫描：

```text
object b
  │
  ▼
heapBits
  │
  ├── 非 pointer → 跳过
  │
  └── pointer
        │
        ▼
     findObject
        │
        ▼
     greyobject
        │
        ▼
      gcw.put
```

对于大型对象，runtime 可以将扫描工作拆分成多个小块（oblet），以便于调度和并行处理；不要把原笔记中的“对象大于 128K 拆分”当成所有版本都固定不变的规则，具体阈值属于 runtime 实现细节。

---

# 15. 编译器如何告诉 GC“哪里是指针”？

这是 GC 很重要的一部分。

## 15.1 类型信息

编译器生成 `_type` 信息，其中包含与 GC 扫描相关的数据，例如：

```text
_type
 ├── ptrdata
 └── gcdata
       │
       ▼
  pointer bitmap
```

`ptrdata` 表示对象前多少字节需要考虑指针扫描；`gcdata` 提供更细粒度的 pointer bitmap。

---

## 15.2 mallocgc

分配对象时，runtime 根据类型信息设置对象对应的 heap bitmap：

```text
compiler type
     │
     ▼
heapBitsSetType
     │
     ▼
heapArena.bitmap
```

---

## 15.3 scanstack

栈扫描使用类型/栈元数据：

```text
G stack
  │
  ▼
stack frame metadata
  │
  ▼
ptrdata + gcdata
  │
  ▼
scanblock
```

---

# 16. heapBits

## 16.1 作用

`heapBits` 用于快速判断 heap 中一个 word 是否包含 pointer。

原笔记中的核心概念可以保留：

- `bitPointer = 1`：表示 pointer；
- `bitScan = 16`：与 bitmap 编码/扫描有关；
- `heapBitsShift = 1`；
- `wordsPerBitmapByte = 4`：一个 bitmap byte 描述 4 个 pointer-sized word；
- `bitPointerAll = 00001111`；
- `bitScanAll = 11110000`。

具体 bitmap 编码属于 runtime 内部实现细节，面试时重点理解：

> **heapBits 的核心作用就是告诉 GC：一个对象的哪些 machine word 是 pointer，需要继续追踪。**

---

## 16.2 heapBits 结构

```go
type heapBits struct {
    bitp  *uint8
    shift uint32
    arena uint32
    last  *uint8
}
```

理解：

- `bitp`：当前 bitmap byte；
- `shift`：当前 word 在 bitmap byte 中的位置；
- `arena`：所在 heap arena；
- `last`：当前 arena bitmap 的末尾位置。

---

## 16.3 使用方式

```go
hbits := heapBitsForAddr(base)
n := s.elemsize

for i := uintptr(0); i < n; i += sys.PtrSize {
    if hbits.isPointer() {
        // 这是 pointer，需要继续扫描
    }
    hbits = hbits.next()
}
```

核心思想：**以 pointer size 为单位遍历对象，根据 bitmap 判断当前 word 是否为 pointer。**

---

## 16.4 heapBitsForAddr

基本过程：

```text
对象地址
   │
   ▼
找到 heapArena
   │
   ▼
计算 bitmap byte
   │
   ▼
计算 shift
   │
   ▼
heapBits
```

原笔记涉及的计算关系：

```text
bitp = arena.bitmap[(addr / (PtrSize * 4)) % bitmapBytes]
shift = (addr / PtrSize) & 3
```

具体实现可能随 Go 版本变化，但核心映射关系不变。

---

# 17. gcbits / markBits / allocBits

这是最容易混淆的一组概念。

## 17.1 markBits

`mspan.gcmarkBits`：记录本轮 GC 的 Mark 结果。

```text
bit = 1 → 本轮已标记 / 存活
bit = 0 → 本轮未标记
```

注意：**“灰色”不是由 mark bit 单独表示的。**

更准确的模型是：

```text
对象是否已经被发现/标记
        │
        ├── mark bit
        │
        └── 是否仍有待扫描工作
                 │
                 └── gcw/workbuf
```

所以原笔记中的：

> “gcmarkBits/markBits：在 GC 时灰色对象的对应 bit 被置为 1”

应修正为：

> **mark bit 表示对象已经被 GC 标记；对象处于灰色还是黑色，还取决于它是否已经完成扫描。灰色对象通常表现为“已经标记但仍存在待扫描工作”。**

---

## 17.2 allocBits

`mspan.allocBits`：记录当前哪些 object slot 已经被分配。

Sweep 完成后：

```text
allocBits = gcmarkBits
```

然后重新创建清零的 `gcmarkBits`，供下一轮 GC 使用。

也就是：

```text
本轮 Mark
   │
   ▼
gcmarkBits
   │
   │ Sweep
   ▼
allocBits
   │
   └── 作为下一轮分配状态

同时

gcmarkBits → 重新清零
```

当前 `mspan.sweep` 代码也明确执行 `s.allocBits = s.gcmarkBits`，随后分配新的 `gcmarkBits`。citeturn1search0

---

# 18. markBits 关键操作

```go
type markBits struct {
    bytep *uint8
    mask  uint8
    index uintptr
}
```

核心函数：

### `markBitsForAddr`

根据地址找到：

```text
address
  ↓
spanOf
  ↓
mspan
  ↓
objIndex
  ↓
markBits
```

### `markBitsForIndex`

根据 object index 找到 `gcmarkBits` 中对应 byte 和 mask。

### `allocBitsForIndex`

根据 object index 找到 `allocBits` 对应 bit。

### `markBitsForBase`

获取 span 基地址对应的 mark bit。

---

# 19. greyobject

`greyobject` 是标记对象的核心入口之一。

简化流程：

```text
object address
     │
     ▼
find span / objIndex
     │
     ▼
检查 mark bit
     │
 ┌───┴────┐
 │        │
已标记    未标记
 │        │
返回      设置 mark bit
          │
          ▼
      pageMarks
          │
          ▼
      gcw.put(object)
```

需要注意：

- 如果对象已经标记，直接返回；
- 设置 mark bit；
- 更新 page mark；
- 如果是 `noscan` 对象，则不需要继续扫描内部指针；
- 否则放入 GC work buffer，等待 `scanobject`。

---

# 20. pageMarks

`heapArena.pageMarks` 用于记录哪些 heap page/span 上存在本轮 GC 已标记对象。

用途主要是帮助后续 span/page 回收判断：

```text
page
 │
 ├── 没有 marked object
 │       ↓
 │     更容易整页回收
 │
 └── 有 marked object
         ↓
       保留
```

原笔记中的：

> `greyObject/gcmarknewobject/wbBufFlush1` 中设置对应 bit 为 1

可以作为 runtime 内部实现的理解，但不同 Go 版本具体调用链可能发生变化。

---

# 21. gcbitsArena

GC bitmap 本身也需要内存管理。

```go
type gcBits uint8

type gcBitsArena struct {
    free uintptr
    next *gcBitsArena
    bits [65520]gcBits
}
```

全局：

```go
var gcBitsArenas struct {
    lock     mutex
    free     *gcBitsArena
    next     *gcBitsArena
    current  *gcBitsArena
    previous *gcBitsArena
}
```

核心概念：

- `gcBitsChunkBytes = 65536`；
- `gcBitsHeaderBytes = 16`；
- 每个 bit 对应一个 span object slot；
- `gcBitsArenas` 管理 bitmap arena 的复用和分配。

### `newMarkBits(nelems)`

根据对象数量计算 bitmap 所需字节数：

```text
bytes = ceil(nelems / 64) * 8
```

等价写法：

```text
((nelems + 63) / 64) * 8
```

然后通过 `gcBitsArena.tryAlloc` 获取空间。

### `tryAlloc`

通过原子加操作推进 `free`：

```text
current arena
     │
     ├── 有空间 → 原子分配
     │
     └── 无空间 → 尝试下一个 arena
                     │
                     └── 都失败 → newArenaMayUnlock
```

`newArenaMayUnlock` 会分配新的 `gcBitsArena`，并在必要时清零可复用区域。

---

# 22. GC Work Buffer：gcw

`gcWork` 是每个 P 上的重要 Mark 工作状态。

```go
type gcWork struct {
    wbuf1, wbuf2 *workbuf
    bytesMarked  uint64
    scanWork     int64
    flushedWork  bool
}
```

可以理解为：

```text
             P
             │
          local gcw
         /         \
      wbuf1       wbuf2
       │
       ▼
   灰色对象列表
       │
       ▼
     scanobject
```

当本地 work 不够时，会通过 `balance()` 等逻辑从全局 work 获取；本地 work 太多时，也可能把部分工作迁移到全局队列。

---

# 23. work 全局结构

原笔记中的核心结构：

```go
var work struct {
    full  lfstack
    empty lfstack

    wbufSpans struct {
        lock mutex
        free mSpanList
        busy mSpanList
    }

    bytesMarked uint64
    markrootNext uint32
    markrootJobs uint32

    nproc uint32
    tstart int64
    nwait uint32

    nFlushCacheRoots int
    nDataRoots, nBSSRoots, nSpanRoots, nStackRoots int

    startSema uint32
    markDoneSema uint32

    bgMarkReady note
    bgMarkDone uint32

    mode gcMode
    userForced bool
    totaltime int64

    assistQueue struct {
        lock mutex
        q gQueue
    }

    sweepWaiters struct {
        lock mutex
        list gList
    }

    cycles uint32
    stwprocs, maxprocs int32
    tSweepTerm, tMark, tMarkTerm, tEnd int64
    pauseNS int64
    pauseStart int64

    heap0, heap1, heap2, heapGoal uint64
}
```

### 关键理解

- `full`：可消费的 work buffer；
- `empty`：可复用的空 work buffer；
- `markrootNext / markrootJobs`：root scanning 工作；
- `nproc`：参与 GC worker 的数量；
- `nwait`：等待 Mark 工作完成的 worker 数量；
- `assistQueue`：因没有足够 credit/work 而阻塞的 assist G；
- `sweepWaiters`：等待进入 sweep 阶段的 G；
- `cycles`：完成的 GC cycle 数。

---

# 24. gcController：GC Pacer

`gcController` 的作用：

> **决定什么时候触发 GC、需要多少后台 Mark worker，以及 mutator assist 应该承担多少扫描工作。**

核心状态：

```go
type gcControllerState struct {
    scanWork                    int64
    bgScanCredit                int64
    assistTime                  int64
    dedicatedMarkTime           int64
    fractionalMarkTime          int64
    idleMarkTime                int64
    markStartTime               int64
    dedicatedMarkWorkersNeeded  int64
    assistWorkPerByte           uint64 // 概念字段，具体版本类型可能不同
    assistBytesPerWork          uint64
    fractionalUtilizationGoal   float64
}
```

当前 runtime 的实现还包含 `heapLive`、`heapScan`、`heapMarked`、`heapScanWork`、`stackScanWork`、`globalsScanWork`、`memoryLimit` 等状态；因此原笔记中的结构体应理解为“核心字段摘要”，而不是完整结构体定义。citeturn2view1

---

# 25. GC CPU 利用率与 Worker 数量

核心常量：

```text
gcBackgroundUtilization = 0.25
gcGoalUtilization        = 0.25
```

因此：

```text
totalUtilizationGoal = GOMAXPROCS × 0.25
```

专用 worker 的数量近似：

```text
dedicatedMarkWorkersNeeded
    ≈ GOMAXPROCS × 0.25
```

实现中会通过取整方式得到最接近目标利用率的 worker 数量。

### 示例

```text
GOMAXPROCS = 8

8 × 0.25 = 2

因此大致需要 2 个 dedicated mark worker
```

如果目标不能由整数 worker 精确表达，就使用 fractional worker 补足。

例如：

```text
GOMAXPROCS = 5

5 × 0.25 = 1.25

1 个 dedicated worker
+ 0.05 fractional utilization
```

官方 runtime 对 fractional worker 的定义也是为了补齐无法用整数 dedicated worker 精确表示的 CPU 利用率。citeturn0search3turn2view1

---

# 26. GC 参数计算

## 26.1 startCycle

`gcController.startCycle()` 在新 GC cycle 开始时：

- 清零本轮 scan work；
- 清零 assist time；
- 清零 dedicated/fractional/idle mark time；
- 设置 `markStartTime`；
- 记录本轮初始 `heapLive`；
- 计算 dedicated worker 数；
- 计算 fractional utilization；
- 调用 `revise()`。

当前 runtime 源码明确将这些工作集中在 `startCycle()` 中。citeturn2view1

---

## 26.2 revise

`revise()` 根据：

- `heapScan`；
- `heapLive`；
- 已经完成的 scan work；
- GOGC；
- heap goal；
- memory limit；

重新计算 assist ratio。

核心思想不是简单的“扫描一半 heap”，而是：

```text
需要完成的 scan work
        │
        ▼
预计 GC 可提供的后台 CPU
        │
        ▼
计算 GC runway
        │
        ▼
计算剩余 heap distance
        │
        ▼
assistWorkPerByte
```

因此原笔记中：

> `期望的扫描量 = heap_scan * 100 / (100 + gcpercent)`

可以作为某些历史版本/简化模型的理解，但**不应当当成当前 runtime 的完整公式**。当前 pacer 已经同时考虑 `heapGoal`、扫描工作、分配速度、CPU 时间以及 `GOMEMLIMIT`。citeturn3search0

---

# 27. GOGC

## 27.1 基本含义

默认：

```text
GOGC = 100
```

直观理解：如果上一轮 GC 后存活对象约为 `L`，那么在没有其他限制时，下一轮 GC 的目标堆大小大致围绕：

```text
L × (1 + GOGC/100)
```

例如：

```text
L = 100 MB
GOGC = 100

heap goal ≈ 200 MB
```

官方文档也是这样解释 GOGC=100 的。citeturn0search2turn0search5

注意：这是**理解公式**，不是当前 runtime 所有 trigger/goal 计算的完整实现。

---

## 27.2 GOGC=0

```text
GOGC = 0
```

意味着极其激进的 GC，目标接近“尽快收集”，常用于调试 GC 行为。

当前 runtime 源码说明，GOGC=0 会使 `heapMinimum` 变为 0，从而即使堆很小也可能持续触发 GC。citeturn3search0

---

## 27.3 GOGC=off

```text
GOGC=off
```

表示关闭自动 GC。

原笔记中的：

> “GOGC=off 实际上 GOGC=100000”

需要修正：**不能简单理解为 `100000`；runtime 内部会使用负值表示 GC disabled，并在必要的强制 GC/内存限制场景下有专门处理。** 官方文档明确规定 `GOGC=off` disables GC。citeturn0search7turn0search10

另外，在当前 Go 中还存在 `GOMEMLIMIT`，所以“关闭 GOGC”并不等价于“runtime 完全不关心内存压力”。citeturn0search7

---

# 28. GOMEMLIMIT

原文没有展开，但在理解当前 GC pacing 时需要补充。

`GOMEMLIMIT` 是 Go runtime 的软内存限制，作用于 runtime 管理的大部分内存，包括 Go heap 等；它会影响 GC heap goal。

因此当前 GC 的关系更准确地表示为：

```text
             GOGC
               │
               ▼
        normal heap goal
               │
               ├──────────┐
               │          │
               ▼          ▼
          GOMEMLIMIT   sweep distance
               │          │
               └────┬─────┘
                    ▼
                heap goal
                    │
                    ▼
                 trigger
```

官方 runtime 当前 `gcController.heapGoalInternal()` 会比较 GOGC 派生的 heap goal 与 memory-limit 派生的 goal，并选择更小者。citeturn3search0

---

# 29. 辅助 GC：Mutator Assist

## 29.1 为什么需要 Assist？

如果应用程序分配速度太快：

```text
应用分配速度
      │
      ▼
超过 GC 后台 Mark 速度
      │
      ▼
GC 来不及完成 Mark
```

这时不能无限制地让堆继续增长，因此让**正在分配内存的 G 帮助 GC**。

这就是 mutator assist。

---

## 29.2 gcAssistBytes

每个 G 有与 assist 相关的工作债务。

可以粗略理解为：

```text
gcAssistBytes > 0
    → 有 credit
    → 暂时不需要 assist

 gcAssistBytes < 0
    → 有 GC debt
    → 分配时需要帮助 GC
```

因此原笔记：

> “若为正数，则 g 有帮助 gc 过，有富于；负数则需要辅助 gc”

可以保留，但术语更准确地说是 **GC assist credit/debt**。

---

## 29.3 gcAssistAlloc

`mallocgc()` 分配内存时可能触发：

```text
mallocgc
   │
   ▼
g.gcAssistBytes < 0 ?
   │
  yes
   ▼
gcAssistAlloc
   │
   ├── 计算需要完成的 scan work
   │
   ├── 尝试从后台 GC credit 获取工作
   │
   └── gcDrainN
```

如果仍然没有完成债务：

```text
Gosched / park
      │
      ▼
assistQueue
      │
      ▼
等待其他 GC worker 产生足够 work/credit
```

---

## 29.4 gcDrainN

`gcDrainN`：

> **让当前 G 执行一定数量的 GC scan work。**

大致：

```text
获取 gc work
     │
     ▼
scanobject
     │
     ▼
产生新 work
     │
     ▼
继续直到：
  ① 工作量达标
  ② G 被抢占
```

原笔记中的 `gcw.balance()`、`tryGetFast()`、`tryGet()`、`wbBufFlush` 等都属于“获取更多 GC 工作”的实现细节。

---

# 30. Background Mark Worker

## `gcBgMarkStartWorkers`

启动/准备 Mark worker。

原笔记中的：

> “启动 GOMAXPROCS 个 gcBgMarkWorker 的 G 放入 pool 中”

应理解为：runtime 会准备对应的 worker G，并在真正需要 Mark 时通过 scheduler 使用；**不是说 GOMAXPROCS 个 worker 会一直同时运行。**

---

## `gcBgMarkWorker`

主要循环：

```text
worker
  │
  ▼
等待调度
  │
  ▼
获取 worker mode
  │
  ├── Dedicated
  ├── Fractional
  └── Idle
  │
  ▼
gcDrain
  │
  ▼
完成/被抢占
  │
  ▼
重新进入 worker pool
```

当前 runtime 源码明确存在 `gcBgMarkWorker`，并根据 `gcMarkWorkerMode` 调用不同的 drain 逻辑。citeturn0search3

---

# 31. gcMarkDone

Mark 完成不能简单理解成“所有 worker 都返回了”。

需要同时满足：

```text
所有 Mark worker 都完成
        AND
没有新的 GC work
        AND
写屏障产生的 work 已经处理
```

典型判断：

```text
work.nwait == work.nproc
&&
!gcMarkWorkAvailable(p)
```

之后进入 Mark Termination。

---

# 32. Mark Termination

这是 GC 中非常重要的一个 STW 阶段。

```text
Concurrent Mark
      │
      ▼
所有 worker 接近完成
      │
      ▼
gcMarkDone
      │
      ▼
STW
      │
      ▼
flush 所有 P 的 wb buffer
      │
      ▼
确认没有新 work
      │
      ▼
关闭写屏障
      │
      ▼
进入 Sweep
```

`gcMarkDone` 会获取必要的同步量，并通过 `forEachP` 刷新每个 P 的 write barrier buffer；如果仍发现 GC work，则可能重新启动 world 继续 Mark，再次检查完成条件。

---

# 33. gcMarkTermination

主要工作：

1. 设置 `_GCmarktermination`；
2. 在系统栈执行最终 Mark 处理；
3. 清理 P 上的 write barrier buffer / gcw；
4. 设置 `_GCoff`；
5. 关闭 write barrier；
6. 准备 Sweep；
7. 唤醒等待 Sweep 的 G；
8. Start The World；
9. 清理 work buffer、空闲栈和 mcache 等。

原笔记中的：

> `_GCoff → 唤醒 sweep.g → 唤醒 sweepWaiters`

这个整体理解是正确的，但要注意：**进入 `_GCoff` 表示 Mark 结束并关闭写屏障，随后 Sweep 可以并发进行。**

---

# 34. Sweep

## 34.1 Sweep 的目标

Mark 得到：

```text
mark bit = 1 → live
mark bit = 0 → dead
```

Sweep：

```text
遍历 span
   │
   ▼
检查 alloc/mark 信息
   │
   ├── live → 保留
   │
   └── dead → 回收 slot
```

当前 runtime 对 sweep 的定义就是：释放 Mark 阶段没有标记的对象，并清理 mark bits，为下一轮 GC 准备。citeturn1search0

---

## 34.2 bgsweep

`bgsweep` 是后台 Sweep G。

大致：

```text
bgsweep
   │
   ▼
gopark 等待
   │
   ▼
被唤醒
   │
   ▼
sweepone()
   │
   ├── 有工作 → 继续
   └── 没工作 → park
```

当前 runtime 会批量执行 `sweepone`，并根据是否存在其他繁忙 worker 决定何时让出 CPU。citeturn1search0

---

# 35. sweepone

`sweepone()`：

1. 获取 sweep ownership；
2. 从 `mcentral` 的 unswept 集合寻找 span；
3. 检查 span 状态；
4. 使用 sweep generation 获取 sweep ownership；
5. 调用 `mspan.sweep()`；
6. 更新 swept page 等统计；
7. 返回本次回收的页数。

```text
mcentral
  │
  ├── fullUnswept
  └── partialUnswept
          │
          ▼
    nextSpanForSweep
          │
          ▼
      sweepone
          │
          ▼
     mspan.sweep
```

---

# 36. mspan.sweep

核心步骤：

```text
mspan
 │
 ▼
处理 special / finalizer
 │
 ▼
根据 mark bits 判断 dead/live
 │
 ▼
计算 allocCount
 │
 ▼
freeindex = 0
 │
 ▼
allocBits = gcmarkBits
 │
 ▼
重新创建清零 gcmarkBits
 │
 ▼
设置 sweepgen
 │
 ├── 全部释放 → freeSpan
 │
 ├── 全部占用 → fullSwept
 │
 └── 部分占用 → partialSwept
```

当前 runtime 源码明确：

```text
s.allocBits = s.gcmarkBits
s.gcmarkBits = newMarkBits(...)
```

并在最终使 span 可重新用于分配前更新 `s.sweepgen`。citeturn1search0

---

# 37. sweepgen

`sweepgen` 用于判断 span 在本轮 sweep 中处于什么状态。

原笔记中的状态表可以整理为：

| span.sweepgen 相对 heap.sweepgen | 含义 |
|---|---|
| `heap.sweepgen - 2` | 尚未清扫 |
| `heap.sweepgen - 1` | 正在清扫 / 已取得 sweep ownership |
| `heap.sweepgen` | 已清扫，可用于分配 |
| `heap.sweepgen + 1` | 特定缓存/代际状态，具体含义依 runtime 版本实现 |
| `heap.sweepgen + 3` | 特定缓存/已清扫缓存状态，不能简单理解成普通 span 生命周期状态 |

最重要的只记前三个：

```text
-2 → unswept
-1 → sweeping
 0 → swept
```

当前源码的 `sweepLocker.tryAcquire` 也明确从 `sweepGen-2` CAS 到 `sweepGen-1`，而 sweep 完成后设置到当前 `sweepgen`。citeturn1search0

---

# 38. 如何获取需要清扫的对象？

原笔记：

> 从 mcentral unswept 队列获取。

正确且完整的说法是：

```text
mheap.central[i].mcentral
       │
       ├── fullUnswept
       │
       └── partialUnswept
                │
                ▼
       nextSpanForSweep()
                │
                ▼
             sweepone()
```

如果 allocator 自己在 `cacheSpan` 时遇到尚未清扫的 span，也可能直接推进 Sweep，即所谓 **direct/proportional sweep**。官方源码明确说明 object reclaimer 可以由后台 `sweepone` 或 `mcentral.cacheSpan` 同步驱动。citeturn1search0

---

# 39. 清扫与分配：deductSweepCredit

为了避免后台 Sweep 来不及完成，分配路径也可能承担一部分 Sweep 工作。

可以理解为：

```text
allocator
   │
   ▼
cacheSpan()
   │
   ▼
发现 sweep work
   │
   ▼
deductSweepCredit
   │
   ▼
sweepone
   │
   ▼
拿到可用 span
```

因此 Go 的 Sweep 并不是：

> “只有一个后台 sweeper 慢慢扫。”

而是：

> **后台 Sweep + 分配路径上的 proportional/direct sweep 共同保证清扫进度。**

---

# 40. finishsweep_m

这是回答“为什么下一轮 Mark 不会受到上一轮 Sweep 干扰”的关键函数。

```go
func finishsweep_m() {
    for sweepone() != ^uintptr(0) {
    }
}
```

它运行时要求 World 已经 stopped，用来确保所有必须完成的 Sweep work 已经 drained。当前 runtime 源码明确写着：**sweeping 必须在 concurrent scan 开始前完成**；如果强制 GC 在 Sweep 中途发生，则这个函数尤其重要。citeturn1search0turn2view0

---

# 41. clearpools

`clearpools()` 主要处理 runtime cache：

- `sync.Pool`；
- `sched.sudogcache`；
- `sched.deferpool` 等。

这些缓存如果不在适当阶段处理，部分内存可能无法及时回收。

当前 `gcStart` 在新一轮并发扫描前也会调用 `clearpools()`。citeturn2view0

---

# 42. scavenge

**Sweep 和 Scavenge 不是一回事。**

### Sweep

解决：

> 哪些对象/slot 已经死亡？

### Scavenge

解决：

> 已经没有用的 heap pages，哪些可以真正归还给 OS？

所以：

```text
GC Mark
  │
  ▼
确定 dead objects
  │
  ▼
Sweep
  │
  ▼
span/page 出现可释放空间
  │
  ▼
Scavenge
  │
  ▼
sysUnused / 归还 OS
```

---

# 43. bgscavenge

`bgscavenge` 是后台 scavenger。

大致：

```text
bgscavenge
    │
    ▼
park / timer
    │
    ▼
被 sysmon 或 runtime 唤醒
    │
    ▼
scavengeOne / scavengeRangeLocked
    │
    ▼
归还 page
    │
    ▼
依据 EWMA 决定下一次休眠时间
```

原笔记中的：

- `scavengeGoal`：希望归还 OS 的内存目标；
- `heapRetained`：保留堆 / RSS 相关估算；
- `scavengeEWMA`：控制后台 scavenging 节奏的指数移动平均量；
- `pageAlloc.scav`；
- `pageAlloc.scavengeRangeLocked`；
- `pageAlloc.scavengeOne`；

均属于这一层。

---

# 44. semacquire / worldsema / gcsema

## semacquire

用于获取 runtime semaphore。

简化：

```text
semacquire(addr)
      │
      ▼
cansemacquire
      │
   ┌──┴──┐
   │     │
 success fail
   │     │
 return  │
         ▼
      sudog
         │
         ▼
    semaRoot queue
         │
         ▼
       gopark
```

`cansemacquire` 通过 CAS 尝试完成信号量获取；失败后创建/获取 `sudog` 并进入等待队列。

---

## semaRoot

原笔记中的描述可以保留：runtime 使用 `semaRoot` 管理 semaphore 等待者，内部采用基于 treap 的结构组织等待队列。

---

## worldsema

`worldsema`：控制谁可以执行 STW / Start The World 的关键同步。

```text
获取 worldsema
     │
     ▼
Stop The World
     │
     ▼
GC 全局操作
     │
     ▼
Start The World
     │
     ▼
释放 worldsema
```

## gcsema

用于串行化 GC cycle，避免多个 G 同时启动并发 GC。

---

# 45. STW 相关状态

核心字段：

```text
sched.stopwait
    STW 时需要停止的 P 数量

sched.gcwaiting
    表示 GC / freeze-the-world 等待世界停止

sched.stopnote
    等待 STW 完成的同步点
```

STW 前：

```go
semacquire(&worldsema, 0)
m.preemptoff = "reason"
systemstack(stopTheWorldWithSema)
```

恢复：

```go
m.preemptoff = ""
systemstack(startTheWorldWithSema)
semrelease(&worldsema)
```

---

# 46. forEachP

`forEachP` 是 runtime 在需要让所有 P 执行同一个安全点函数时使用的重要机制。

可以理解为：

```text
forEachP(fn)
    │
    ├── 设置 safePointFn
    ├── 要求所有 P 到安全点
    ├── preemptall()
    ├── idle P 直接执行
    ├── syscall P 做 handoff
    └── 当前 P 执行 fn
```

它特别适合 GC 中需要“所有 P 都处理一次”的操作，例如刷新 write barrier buffer。

---

# 47. injectglist

`injectglist`：把一批 G 注入调度系统。

```text
glist
  │
  ▼
状态 → _Grunnable
  │
  ▼
全局 runq
  │
  ▼
startm / P local runq
```

Start The World 后，runtime 也会处理 netpoll 中已经就绪的 G，再注入调度系统。原笔记中“重启世界时优先从 netpoll 获取 G”的理解可以保留，但更准确地说是 **Start The World 会检查 netpoll，并将可运行 G 注入调度系统**。

---

# 48. gcMarkRootPrepare

作用：提前统计本轮 root Mark 工作量。

主要包括：

```text
mcache roots
 data roots
 bss roots
 span roots
 stack roots
```

然后计算：

```text
markrootJobs
```

供 `markroot()` 分批消费。

---

# 49. gcMarkTinyAllocs

Tiny allocation 是 Go allocator 对小对象的一种优化。

GC 开始时，需要处理 P 上 tiny allocator 中尚未正常进入普通对象 Mark 流程的对象。

因此会遍历各 P 的 tiny allocation 状态，并通过：

```text
findObject
    ↓
greyobject
```

确保这些对象在本轮 Mark 中得到正确处理。

---

# 50. gcMarkWorkAvailable

用于判断当前是否仍有 Mark 工作。

核心来源包括：

```text
p.gcw != empty
       OR
work.full != empty
       OR
markrootNext < markrootJobs
```

因此：

```text
有 root job
   OR
有 local gcw
   OR
有 global work
      ↓
仍然有 Mark work
```

---

# 51. gcmarknewobject

新对象在 GC Mark 阶段分配时，需要保证不会因为“出生在 Mark 中途”而成为漏标对象。

Go 的实现会让新分配对象直接处于黑色语义，避免它必须重新进入完整的对象图扫描流程。

```text
mallocgc
   │
   ▼
gcmarknewobject
   │
   ├── mark bit
   ├── page mark
   └── 对象可安全进入当前 GC 生命周期
```

---

# 52. uintptr

`uintptr` 是一种无符号整数类型，足以保存一个指针值的整数表示。

**重要：`uintptr` 不是 GC 可追踪的指针。**

例如：

```go
p := new(Object)
u := uintptr(unsafe.Pointer(p))
```

GC 不会因为 `u` 保存了地址，就认为 `p` 指向的对象仍然存活。

因此：

```text
*Object pointer
     │
     ▼
GC 可以追踪

uintptr
     │
     ▼
GC 不扫描
```

如果需要跨越某些操作确保对象存活，应正确使用 Go 的 `runtime.KeepAlive` 等机制，而不能把 `uintptr` 当成 GC pointer。

---

# 53. GC 关键函数总表

| 函数 | 作用 |
|---|---|
| `gcStart` | 启动 GC cycle |
| `gcTrigger.test` | 判断是否满足 GC trigger |
| `gcBgMarkStartWorkers` | 准备后台 Mark worker |
| `gcBgMarkWorker` | 后台 Mark worker 主循环 |
| `gcDrain` | 消费 Mark work |
| `gcDrainN` | 消费指定量的 Mark work，主要用于 assist |
| `markroot` | 扫描 GC roots |
| `scanblock` | 根据 bitmap 扫描内存块 |
| `scanstack` | 扫描 G stack |
| `scanobject` | 扫描对象内部 pointer |
| `findObject` | 根据地址定位 span/object |
| `greyobject` | 标记对象并产生扫描 work |
| `gcMarkDone` | 判断并推动 Mark → termination |
| `gcMarkTermination` | STW 完成最终 Mark 并关闭 WB |
| `gcmarknewobject` | Mark 阶段处理新分配对象 |
| `gcMarkRootPrepare` | 准备 root job |
| `gcMarkTinyAllocs` | 处理 tiny allocation |
| `clearpools` | 清理 runtime pools |
| `finishsweep_m` | 在新 Mark 前确保 Sweep 完成 |
| `sweepone` | 清扫一个 span |
| `mspan.sweep` | 真正回收 span 中 dead object |
| `bgsweep` | 后台 Sweep G |
| `deductSweepCredit` | 分配路径承担 Sweep 工作 |
| `bgscavenge` | 后台向 OS 归还 page |
| `scavengeOne` | 回收一部分 page |
| `forEachP` | 让所有 P 到安全点执行函数 |
| `injectglist` | 将 G 注入 scheduler |

---

# 54. GC 常量 / 参数速记

| 名称 | 含义 |
|---|---|
| `GOGC` | GC heap growth target，默认 100 |
| `gcPercent` | runtime 内部保存的 GOGC |
| `defaultHeapMinimum` | 默认最小 heap goal，默认配置下约 4MB |
| `heapMinimum` | 小堆 GC 的最小 heap 尺度，受 GOGC 影响 |
| `gcBackgroundUtilization` | 后台 Mark CPU 利用率目标，0.25 |
| `gcGoalUtilization` | Mark 总体目标利用率，当前等于 0.25 |
| `gcCreditSlack` | 本地 scan credit 更新全局状态的 slack，2000 |
| `forcegcperiod` | time trigger 的周期，当前 runtime 为 2 分钟量级的常量 |
| `worldsema` | STW / world transition 的同步量 |
| `gcsema` | GC cycle 串行化 |
| `gcBlackenEnabled` | 是否允许后台 worker / assist 进行 blackening |
| `sweepgen` | span sweep generation |
| `pageMarks` | page 上是否存在本轮 marked object 的信息 |
| `heapBits` | 判断 heap word 是否为 pointer |
| `gcmarkBits` | 本轮 GC 的 Mark bit |
| `allocBits` | object slot allocation bit |

---

# 55. gcmode

```go
type gcMode int

const (
    gcBackgroundMode gcMode = iota
    gcForceMode
    gcForceBlockMode
)
```

含义：

```text
gcBackgroundMode
    并发 Mark + 并发 Sweep

gcForceMode
    STW GC + 并发 Sweep

gcForceBlockMode
    STW GC + STW Sweep
```

后两种主要用于 runtime 调试/强制模式，不是普通应用 GC 的默认模式。citeturn2view0

---

# 56. gcMarkWorkerMode

```go
gcMarkWorkerNotWorker

gcMarkWorkerDedicatedMode
gcMarkWorkerFractionalMode
gcMarkWorkerIdleMode
```

记忆：

```text
Dedicated  → 专职干活
Fractional → 补 CPU 百分比
Idle       → 空闲时干活
NotWorker  → 普通 G
```

---

# 57. gcDrainFlags

```text
gcDrainUntilPreempt
    → 一直执行到被抢占

gcDrainIdle
    → 只利用空闲时间执行

gcDrainFractional
    → 按 fractional utilization 执行

gcDrainFlushBgCredit
    → 将 worker work 计入后台 credit
```

---

# 58. 一次完整 GC 的面试版流程

```text
① Trigger
   │
   ├─ heap trigger
   ├─ time trigger
   └─ cycle trigger
   │
   ▼
② gcStart
   │
   ▼
③ STW
   │
   ├─ finishsweep_m
   ├─ clearpools
   ├─ reset mark state
   ├─ prepare roots
   └─ calculate GC pacing
   │
   ▼
④ Start World
   │
   ▼
⑤ Concurrent Mark
   │
   ├─ markroot
   ├─ scanobject
   ├─ background worker
   ├─ mutator assist
   └─ write barrier
   │
   ▼
⑥ gcMarkDone
   │
   ▼
⑦ STW Mark Termination
   │
   ├─ flush write barrier buffer
   ├─ confirm no work
   └─ close write barrier
   │
   ▼
⑧ _GCoff
   │
   ▼
⑨ Concurrent Sweep
   │
   ├─ bgsweep
   ├─ sweepone
   ├─ direct/proportional sweep
   └─ mspan.sweep
   │
   ▼
⑩ Scavenge
   │
   └─ 将合适的 page 归还 OS
```

---

# 59. GC 中最容易混淆的三个层次

## 第一层：对象是否存活？

```text
三色 Mark
   ↓
mark bits
```

回答：

> **这个对象本轮 GC 是否可达？**

---

## 第二层：对象 slot 是否可再次分配？

```text
Sweep
   ↓
allocBits
   ↓
span swept
```

回答：

> **这个内存槽位是否已经完成清扫，可以重新分配？**

---

## 第三层：物理 page 是否归还 OS？

```text
Scavenge
   ↓
sysUnused
   ↓
OS
```

回答：

> **这部分虚拟内存是否可以进一步归还操作系统？**

所以一定要区分：

```text
Mark      = 判断活不活
Sweep     = 回收对象/slot
Scavenge  = 向 OS 归还 page
```

---

# 60. GC 工作关系图

```text
                         ┌──────────────┐
                         │  Application  │
                         └──────┬───────┘
                                │ mallocgc
                                ▼
                      ┌──────────────────┐
                      │  Allocation      │
                      └───────┬──────────┘
                              │
                    heap trigger reached?
                              │
                              ▼
                      ┌──────────────────┐
                      │     gcStart      │
                      └───────┬──────────┘
                              │
                              ▼
                         ┌─────────┐
                         │   STW   │
                         └────┬────┘
                              │
                  finishsweep_m / roots
                              │
                              ▼
                         ┌─────────┐
                         │  Mark   │◄──────────┐
                         └────┬────┘           │
                              │                │
                ┌─────────────┼────────────┐   │
                │             │            │   │
                ▼             ▼            ▼   │
             worker        assist      barrier ─┘
                │             │            │
                └─────────────┴────────────┘
                              │
                              ▼
                     ┌────────────────┐
                     │ Mark Termination│
                     └───────┬────────┘
                             │ STW
                             ▼
                       ┌───────────┐
                       │  Sweep    │
                       └─────┬─────┘
                             │
                    ┌────────┴────────┐
                    ▼                 ▼
               bgsweep           direct sweep
                    │                 │
                    └────────┬────────┘
                             ▼
                        free spans
                             │
                             ▼
                        Scavenger
                             │
                             ▼
                            OS
```

---

# 61. 原笔记中的“扫描流程”统一理解

原笔记的四步可以整理为：

### 1. 编译器生成类型元数据

```text
Go struct/type
     ↓
_type
     ↓
ptrdata + gcdata
```

### 2. 对象分配时建立 heap metadata

```text
mallocgc
   ↓
heapBitsSetType
   ↓
heapBits
```

### 3. 扫描 root

```text
scanstack / scanblock
       ↓
找到 root pointer
       ↓
findObject
       ↓
greyobject
```

### 4. 扫描 heap object

```text
gcw
 ↓
scanobject
 ↓
heapBits
 ↓
找到 pointer
 ↓
greyobject
 ↓
gcw
```

因此可以记成一句：

> **编译器告诉 GC“哪里可能是指针”，heapBits 帮 GC 快速判断 heap word，Mark worker 通过 gcw 不断消费和产生扫描任务，最终遍历整个可达对象图。**

---

# 62. GC 参数和指标：如何理解

原笔记中的主要指标：

```text
scanWork
    本轮实际完成的扫描工作

bgScanCredit
    后台 worker 产生、assist 消费的扫描 credit

assistTime
    mutator assist 花费的时间

dedicatedMarkTime
    dedicated worker 花费的时间

fractionalMarkTime
    fractional worker 花费的时间

idleMarkTime
    idle worker 花费的时间

markStartTime
    本轮 Mark 开始时间

dedicatedMarkWorkersNeeded
    需要的 dedicated worker 数量

assistWorkPerByte
    每分配一个 byte 应承担多少 scan work

assistBytesPerWork
    scan work 与分配 byte 的反向换算关系

fractionalUtilizationGoal
    fractional worker 的 CPU 时间目标
```

当前 runtime 中这些字段共同服务于 pacer，而不是孤立使用。citeturn2view1

---

# 63. endCycle

`endCycle()` 在 GC cycle 结束时根据实际运行结果更新下一轮 pacing。

主要考虑：

```text
本轮：

heapLive
heapMarked
heapScanWork
assistTime
mark duration
allocation growth

       ↓

下一轮：
trigger ratio
heap goal
runway
assist ratio
```

原笔记中的：

```text
goalGrowthRatio
actualGrowthRatio
assistDuration
triggerError
triggerRatio
```

可以保留为理解 pacer 的历史/核心数学模型，但当前 runtime 已增加 memory-limit 和更复杂的 heap-goal / trigger 约束，因此不要把这些公式当成源码的完整等式。

---

# 64. 为什么 GC 不直接 STW 做完？

如果完全 STW：

```text
Application
    │
    X
    │
   STW
    │
    ├── scan root
    ├── scan heap
    ├── sweep
    └── restart
```

优点：简单。

缺点：

```text
heap 越大
   ↓
scan 越久
   ↓
STW 越长
   ↓
应用延迟越高
```

Go 的核心选择是：

```text
只有必须全局一致的阶段 STW
                │
                ├── Sweep termination
                └── Mark termination

其余工作尽量并发
                │
                ├── Mark
                ├── Sweep
                └── Scavenge
```

这就是 Go GC 低延迟设计的核心。

---

# 65. 最终速记版

## 65.1 三句话

> **Mark：从 Root 出发做三色标记，确定对象是否存活。**

> **Sweep：根据 Mark bits 回收死亡对象和 slot，不需要重新遍历对象图，通常并发执行。**

> **Scavenge：把已经没有用途的 page 尽可能归还给 OS。**

---

## 65.2 两个 STW

```text
GC Start
   │
   ▼
STW #1
   │
   └── finish previous sweep
       + prepare GC
   │
   ▼
Concurrent Mark
   │
   ▼
STW #2
   │
   └── Mark Termination
       + flush WB
       + confirm Mark done
   │
   ▼
Concurrent Sweep
```

---

## 65.3 三个核心 Metadata

```text
heapBits
    → 哪里是 pointer？

gcmarkBits
    → 哪些对象被本轮 GC 标记？

allocBits
    → 哪些 object slot 当前已分配？
```

---

## 65.4 三种 GC 工作

```text
Background Worker
    → GC 自己的 G 帮助 Mark

Mutator Assist
    → 应用 G 分配时顺便帮助 Mark

Write Barrier
    → 应用修改引用时防止漏标
```

---

## 65.5 Sweep 为什么不用全程 STW？

```text
Mark 已经确定：

live  → mark=1
死    → mark=0

Sweep 只需要依据这个结果回收

并通过：

sweepgen
+ span ownership
+ allocation-time sweeping

保证 span 在复用前已经清扫。

如果下一轮 Mark 开始时仍有 Sweep 未完成：

STW → finishsweep_m → 完成 Sweep → Mark
```

**这就是“Sweep 可以并发，但不能让未完成 Sweep 影响下一轮 Mark”的关键。**

---

# 66. 原文涉及的引用

- CSDN：<https://blog.csdn.net/qq_33339479/article/details/108491796>
- CSDN：<https://blog.csdn.net/qiya2007/article/details/107291497>
- 简书：<https://www.jianshu.com/p/ebd8b012572e>

建议实际学习 runtime 时，优先以对应 Go 版本的官方源码为准，因为 GC 的内部字段、pacer 公式、bitmap 实现和实验性优化会随版本变化。

---

# 67. 一张图记住 Go GC

```text
                         Go GC
                           │
          ┌────────────────┼────────────────┐
          │                │                │
          ▼                ▼                ▼
       Trigger          Mark             Sweep
          │                │                │
    ┌─────┼─────┐     三色标记        mark=0 → free
    │     │     │        │            mark=1 → keep
  Heap   Time  Cycle      │                │
    │     │     │         │             sweepgen
    └─────┴─────┘         │                │
          │               │                │
          ▼               ▼                ▼
        gcStart       Write Barrier      mspan.sweep
          │               │                │
          ▼               │                ▼
        STW #1            │             allocBits
          │               │                │
          ▼               │                ▼
  finishsweep_m           │             free span
          │               │                │
          ▼               │                ▼
   Concurrent Mark ◄──────┘            Scavenge
          │                                 │
          ▼                                 ▼
       STW #2                              OS
          │
          ▼
        _GCoff
          │
          ▼
    Concurrent Sweep
```

> **最终记忆：GC = Trigger → STW准备 → Concurrent Mark → STW终止 → Concurrent Sweep → Scavenge。**

---

# 68. 原笔记函数与字段逐项补全

本节把原笔记中容易被前面章节合并掉的函数、字段重新列出，确保学习时可以直接和 runtime 源码对照。

## 68.1 GC 初始化与触发

### `gcinit`

初始化 GC controller、GC percent、work 同步量等；同时初始化 heap sweep 状态。原笔记中的 `mheap_.sweepdone = 1` 是初始化阶段的 sweep 状态设置。

### `forcegchelper`

后台定时 GC helper：

```text
forcegchelper
    │
    ▼
等待 / park
    │
    ▼
达到 forcegcperiod
    │
    ▼
gcStart(gcTriggerTime)
```

### `notetsleepg`

让当前 G 在 note 上休眠，直到被唤醒或超时；GC helper 等 runtime 后台 G 会使用类似机制。

### `gcTrigger`

```go
type gcTrigger struct {
    kind gcTriggerKind
    now  int64
    n    uint32
}
```

三种 kind：

```text
gcTriggerHeap
    heapLive >= trigger

gcTriggerTime
    now - lastGC > forcegcperiod

gcTriggerCycle
    requested cycle 尚未完成
```

当前源码的 `gcTrigger.test()` 正是按照这三个条件判断。citeturn2view0

---

## 68.2 `gcStart` 逐步对应

原笔记中的流程整理如下：

```text
gcStart
 │
 ├─ acquirem / 判断是否允许启动
 ├─ 推进必要的 sweep
 ├─ 获取 work.startSema
 ├─ 再次 trigger.test
 ├─ 获取 gcsema
 ├─ 获取 worldsema
 ├─ 检查 allp 的 mcache flush 状态
 ├─ gcBgMarkStartWorkers
 ├─ gcResetMarkState
 ├─ stopTheWorldWithSema
 ├─ finishsweep_m
 ├─ clearpools
 ├─ work.cycles++
 ├─ gcController.startCycle
 ├─ 设置 GC phase
 └─ startTheWorldWithSema
```

这里有一个重要修正：

> **“启动 worker”和“worker 开始执行 Mark”不是同一件事。**

`gcBgMarkStartWorkers()` 负责准备 worker；真正执行 Mark 是调度器在 `_GCmark` 阶段选择 worker 后调用 `gcBgMarkWorker`。

---

## 68.3 `gcResetMarkState`

作用：

- 重置本轮 Mark 状态；
- 初始化/重置 P 的 `gcw`；
- 重置相关工作统计；
- 准备新的 mark cycle；
- 配合 stack scan state 等状态重置。

可以理解为：

```text
上一轮 Mark 状态
      │
      ▼
 gcResetMarkState
      │
      ▼
新一轮 Mark 的空工作状态
```

---

## 68.4 `gcBgMarkPrepare`

准备后台 Mark 所需要的 `work` 状态，例如：

- `markrootNext`；
- `markrootJobs`；
- worker 计数；
- work buffer 相关状态。

---

## 68.5 `gcMarkRootPrepare`

统计 root 数量：

```text
mcache roots
     │
data roots
     │
bss roots
     │
span roots
     │
stack roots
     ▼
markrootJobs
```

注意：这里的 job 是为了**分批并行处理 root**，不是说每一个 root 都对应一个独立 goroutine。

---

## 68.6 `gcMarkTinyAllocs`

处理 tiny allocator 中需要纳入当前 GC 标记范围的对象：

```text
allp
 │
 ▼
p.mcache.tiny
 │
 ▼
findObject
 │
 ▼
greyobject
```

---

# 69. Mark Worker 函数逐项补全

## `findRunnableGCWorker`

scheduler 需要运行 GC worker 时调用。

核心逻辑：

```text
gcBlackenEnabled == 0
       │
       └── 不启动

否则
       │
       ▼
从 gcBgMarkWorkerPool 获取 worker
       │
       ▼
设置 p.gcMarkWorkerMode
       │
       ▼
worker G → _Grunnable
```

---

## `gcBgMarkWorker`

核心循环：

```text
park
 │
 ▼
进入 worker pool
 │
 ▼
被 scheduler 选中
 │
 ▼
根据 mode 执行：
 │
 ├─ Dedicated   → gcDrain
 ├─ Fractional  → gcDrain + fractional
 └─ Idle        → gcDrain + idle
 │
 ▼
统计 worker 时间
 │
 ▼
重新 park
```

当 worker 判断 Mark 工作已经结束，会参与 `gcMarkDone`。

---

## `gcDrain`

执行两类工作：

```text
① markroot
② scanobject
```

获取 work 的顺序可以粗略理解为：

```text
local gcw
   ↓
balance / tryGetFast
   ↓
tryGet global work
   ↓
wbBufFlush
   ↓
继续获取
```

当扫描工作累计到一定程度：

```text
gcw.scanWork
      │
      ▼
gcFlushBgCredit
      │
      ▼
gcController.bgScanCredit
```

---

## `gcFlushBgCredit`

作用：

> 将当前 worker 已经完成的扫描工作转换成全局 background scan credit，供 mutator assist 消费。

```text
background worker
      │
      ▼
scan work
      │
      ▼
gcFlushBgCredit
      │
      ▼
bgScanCredit
      │
      ▼
mutator assist 消费
```

---

# 70. Assist 函数逐项补全

## `gcAssistAlloc`

在 `mallocgc()` 路径发现：

```text
g.gcAssistBytes < 0
```

就执行 assist。

它会：

1. 计算需要完成的 scan work；
2. 尝试从 `bgScanCredit` 中获取可用 credit；
3. 不够时自己执行 `gcDrainN`；
4. 完成后减少 debt；
5. 仍不足时可能让 G park。

---

## `gcAssistAlloc1`

主要是实际执行 assist 的内部路径，切换到适合执行 GC 工作的状态/栈环境，然后调用：

```text
gcDrainN
```

完成后恢复 G 的正常运行状态。

---

## `gcParkAssist`

如果没有足够的 work/credit：

```text
gcParkAssist
     │
     ▼
加入 work.assistQueue.q
     │
     ▼
goparkunlock
     │
     ▼
等待唤醒
```

---

## `gcWakeAllAssists`

当 GC 已经产生足够工作或 Mark 状态发生变化时，runtime 可以唤醒阻塞的 assist G：

```text
assistQueue
    │
    ▼
取出多个 G
    │
    ▼
injectglist
    │
    ▼
重新进入调度
```

---

# 71. Root 扫描函数逐项补全

## `markrootBlock`

把 root 区域划分成 block/shard，然后调用 `scanblock()` 扫描。

```text
root block
   │
   ▼
scanblock
```

---

## `markrootFreeGStacks`

处理已经死亡、无需继续保留的 G stack：

```text
sched.gFree.stack
       │
       ▼
stackfree
       │
       ▼
sched.gFree.noStack
```

---

## `markrootSpans`

处理 span root / special object 等：

```text
span bitmap
    │
    ▼
找到相关 span
    │
    ├── scanobject
    └── scanblock
```

原笔记中的“标记 special object”可以理解为与 finalizer/special 等 runtime 元数据相关的 root 扫描。

---

## `flushmcache`

遍历 P：

```text
allp
 │
 ▼
p.mcache.releaseAll
 │
 ▼
stackcache_clear
```

作用是把 P 上与 allocator/cache 相关的状态整理到 GC 可以正确观察和统计的状态。

---

## `scanframeworker`

用于扫描 stack frame 元数据描述的栈帧指针。

重点记：

```text
stack frame
   │
   ├── 参数
   ├── 局部变量
   ├── 返回相关栈槽
   └── 其他编译器描述的 pointer
          │
          ▼
       scanblock
```

---

# 72. gcMark / Mark 收尾逐项补全

## `gcMark`

可以看成 Mark 阶段的统一收尾/统计入口之一：

```text
遍历 allp
   │
   ├── flush / 清理 p.wbBuf
   ├── p.gcw.dispose
   └── 统计 heap scan work
```

原笔记中的：

> “遍历 allp：p.wbBuf 清空；p.gcw.dispose 将多余工作迁移到全局；统计 heap_scan”

可以作为源码阅读时的重点。

---

## `gcMarkTermination`

收尾过程：

```text
_GCmarktermination
      │
      ▼
清理 P 上 WB buffer / gcw
      │
      ▼
_GCoff
      │
      ▼
关闭写屏障
      │
      ▼
唤醒 sweep waiters
      │
      ▼
Start The World
```

---

## `prepareFreeWorkbufs`

把已经不再使用的 work buffer span 从 busy 状态移动到 free 状态，为后续复用/释放做准备。

```text
work.wbufSpans.busy
          │
          ▼
prepareFreeWorkbufs
          │
          ▼
work.wbufSpans.free
```

---

## `freeStackSpans`

释放已经不需要的 goroutine stack span。

注意：

> “释放 G 的栈”不等于“释放所有正在运行 G 的栈”。这里主要处理 runtime 判断已经空闲、可回收的 stack span。

---

# 73. Sweep 函数逐项补全

## `sweepdata`

```go
type sweepdata struct {
    lock       mutex
    g          *g
    parked     bool
    started    bool
    nbgsweep   uint32
    npausesweep uint32
    centralIndex sweepClass
}
```

核心用途：

- 管理后台 sweeper G；
- 记录 parked 状态；
- 管理 sweep class 的当前位置；
- 协调后台 Sweep。

当前版本字段可能有所变化，应以对应版本源码为准。

---

## `numSweepClasses`

原笔记：

```text
numSweepClasses = 272
```

这是某一 runtime 版本下由 span class 数量计算得到的值，**不要当成跨 Go 版本永远固定的 ABI 常量**。

核心关系是：

```text
numSweepClasses
    = numSpanClasses × 2
```

其中 ×2 是为了分别处理 full / partial sweep class。

当前 runtime 源码也是这么定义的。citeturn1search0

---

## `mheap.nextSpanForSweep`

从：

```text
mheap.central[spc].mcentral
```

中查找：

```text
fullUnswept
partialUnswept
```

找到后 pop 出来并交给调用者。

如果所有 class 都没有未清扫 span，则标记 sweep work 已 drained。

---

## `mspan.sweep`

需要重点记住的动作：

```text
① 处理 specials
② 找到未标记 object
③ 计算 nalloc / nfreed
④ 重置 freeindex
⑤ allocBits = gcmarkBits
⑥ new gcmarkBits
⑦ refill alloc cache
⑧ sweepgen = heap.sweepgen
⑨ 放回 fullSwept / partialSwept
⑩ 如果整个 span 空，则 freeSpan
```

当前源码明确要求进入 `mspan.sweep` 时不能被普通抢占打断，并通过 sweep generation 保证 span 处于正确的 sweep ownership。citeturn1search0

---

## `freeSomeWbufs`

释放 GC work buffer 使用的 span：

```text
work.wbufSpans.free
      │
      ▼
取部分 span
      │
      ▼
mheap.freeManual
```

它与普通对象 Sweep 不同，主要负责 runtime 自己的 GC work buffer 内存。

---

## `deductSweepCredit`

分配时根据剩余 sweep 工作量计算当前分配操作需要承担多少 Sweep。

```text
allocation
    │
    ▼
需要 span
    │
    ▼
剩余 sweep work
    │
    ▼
deductSweepCredit
    │
    ▼
sweepone
```

这也是为什么 Sweep 能保持渐进式完成，而不需要一次性 STW 扫完整个 heap。

---

## `gcSweep`

新一轮 Sweep 开始时负责更新 sweep generation，并准备后台 sweeper。

原笔记中的：

```text
mheap_.sweepgen += 2
```

是理解 sweep generation 的关键动作；但具体配套状态机应结合对应 Go 版本 `mgcsweep.go` 阅读。

---

# 74. Scavenge 函数逐项补全

## `mheap_.scavengeAll`

从 heap 的 page allocator 角度执行较大范围的 scavenging。

## `pageAlloc.scav`

保存 page allocator 的 scavenging 状态。

## `pageAlloc.scavengeRangeLocked`

在持有相应锁的情况下，把指定 page range 标记为可归还给 OS，并调用底层 `sysUnused` 等机制。

## `pageAlloc.scavengeOne`

尝试找到适合大小的一段 page 并执行一次 scavenging。

## `pageAlloc.scavenge`

不断调用 `scavengeOne`，直到：

```text
已经归还足够 byte
        OR
没有合适 page
```

## `bgscavenge`

后台循环：

```text
park
 │
 ▼
被唤醒
 │
 ▼
scavenge
 │
 ▼
更新统计
 │
 ▼
EWMA 计算 sleep time
 │
 ▼
再次 park
```

原笔记中的 `scavengeEWMA = 0.01` 可以理解为后台 scavenging 节奏控制参数；具体含义和数值以对应 Go 版本 runtime 源码为准。

---

# 75. 最终知识地图

```text
                         ┌─────────────────┐
                         │      GC         │
                         └────────┬────────┘
                                  │
             ┌────────────────────┼────────────────────┐
             │                    │                    │
             ▼                    ▼                    ▼
          Trigger              Mark                  Sweep
             │                    │                    │
     ┌───────┼───────┐            │            ┌───────┴───────┐
     │       │       │            │            │               │
   Heap     Time   Cycle          │         Background      Direct
     │       │       │            │          sweep          sweep
     └───────┴───────┘            │            │               │
             │                     │            └───────┬───────┘
             ▼                     │                    │
          gcStart                  │                    ▼
             │                     │                mspan.sweep
             ▼                     │                    │
           STW #1                  │                    ▼
             │                     │               allocBits
             ├─ finishsweep_m      │                    │
             ├─ clearpools         │                    ▼
             ├─ reset state        │               swept span
             └─ prepare roots      │                    │
             │                     │                    ▼
             ▼                     │                Scavenge
       Concurrent Mark             │                    │
             │                     │                    ▼
       ┌─────┼────────┐            │                   OS
       │     │        │            │
       ▼     ▼        ▼            │
     Root  Worker   Assist         │
       │     │        │            │
       └─────┼────────┘            │
             │                     │
        Write Barrier ─────────────┘
             │
             ▼
       gcMarkDone
             │
             ▼
           STW #2
             │
      flush WB buffer
             │
      confirm no work
             │
       close WB
             │
             ▼
           _GCoff
             │
             ▼
       Concurrent Sweep
```

---

# 76. 面试重点清单

如果只准备 Go GC 面试，优先掌握下面 15 个问题：

1. Go GC 为什么是三色标记？
2. 为什么并发 Mark 需要 Write Barrier？
3. Go 为什么使用 Hybrid Write Barrier？
4. Go GC 有几个 STW？分别干什么？
5. Sweep 为什么可以并发？
6. Sweep 如何保证不会和 allocator 冲突？
7. `sweepgen` 是干什么的？
8. `markBits`、`allocBits`、`heapBits` 分别干什么？
9. `gcw/workbuf` 是干什么的？
10. Background Mark Worker 和 Mutator Assist 有什么区别？
11. `gcAssistBytes` 为什么会出现正负？
12. `gcController` / pacer 为什么存在？
13. GOGC=100 到底是什么意思？
14. `uintptr` 为什么不能被 GC 当作 pointer？
15. Sweep、Scavenge、GC 三者是什么关系？

其中第 5、6、7 个问题就是原笔记开头“**清扫阶段没有 STW，会不会影响结果？**”的核心。

---

# 77. 最重要的一条主线

最终不要把 Go GC 记成一堆 runtime 函数，而应该建立下面这条因果链：

```text
对象被分配
   ↓
编译器/运行时记录“哪里有指针”
   ↓
GC Trigger
   ↓
STW 准备
   ↓
Root Mark
   ↓
并发扫描对象图
   ↓
Write Barrier 防止并发修改造成漏标
   ↓
Background Worker + Mutator Assist 推进 Mark
   ↓
STW Mark Termination
   ↓
得到最终 mark bits
   ↓
关闭写屏障
   ↓
并发 Sweep
   ↓
根据 mark bits 回收 dead objects
   ↓
allocBits 接管本轮结果
   ↓
sweepgen 标记 span 已完成 Sweep
   ↓
后续分配只能使用已 swept 的 span
   ↓
Scavenge
   ↓
部分物理页归还 OS
```

> **一句话总结：Mark 决定“谁活着”，Sweep 决定“哪些内存槽可以再次使用”，Scavenge 决定“哪些 page 可以还给 OS”；Sweep 不需要全程 STW，但新一轮 Mark 开始前必须把必要的上一轮 Sweep 收尾。**
