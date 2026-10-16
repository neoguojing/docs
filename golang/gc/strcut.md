# Go 垃圾回收（GC）核心数据结构与交互关系

本文基于 Go 语言垃圾回收（GC）静态数据结构架构图，对 GC 相关的核心数据结构、任务池机制、写屏障缓存以及 Goroutine 调度/辅助标记进行了整理、修正与扩展。

---

## 一、 全局内存管理与状态控制结构

### 1. `mheap`
全局堆内存管理核心结构，控制 GC 清扫（Sweep）与内存分配轮次。
- **`sweepgen`**: 全局清扫代数（Sweep Generation）。每完成一次 GC 会递增。
- **`sweepdone`**: 清扫完成标志（初始化为 `1`，GC 开始时置 `0`）。
- **`arenas`**: 堆内存 `heapArena` 索引数组。
- **`sweepers`**: 当前活跃的清扫协程（`sweepgoroutine`）数量。

### 2. `heapArena`
管理一段连续的堆内存区域（如 64MB）。
- **`pageMarks`**: 位图（Bitmap），记录当前 Arena 内每个 Span 对应 Object 的标记状态（使用状态 / 是否已标记）。

### 3. `mcache`
每个 P（Processor）绑定的本地线程缓存。
- **`flushGen`**: 标识当前 `mcache` 中 Span 刷新的代数。若 `flushGen != mheap.sweepgen`，说明需要清扫并更新本地缓存。

### 4. `gcController`
GC 速率控制器，实现 **Pacer** 算法，动态调整 GC 触发时机与 GC Worker/Assist 的 CPU 资源配额。
- **`bgScanCredit`**: 后台扫描累积的 Credit（信用额度），用于抵扣 Mutator 触发 GC 辅助标记（Assist）的工作量。

### 5. `sweepdata`
清扫阶段全局控制状态。
- **`g`**: 当前执行后台清扫（`bgsweep`）的 Goroutine。
- **`parked`**: `bgsweep` 是否处于 休眠/Park 状态。
- **`started`**: 清扫是否已启动。
- **`nbgsweep`**: 后台清扫累计完成的 Span 数量。
- **`npausesweep`**: 清扫阶段暂停次数。
- **`centralIndex`**: 未清扫 SpanClass 遍历的起始位置索引。

---

## 二、 GC 标记任务池与缓存架构 (Work Buffers)

Go GC 的三色标记采用了**多级任务缓存**机制，防止全局锁竞争：
`g.gcWork (P本地)` <---> `work.full / work.empty (全局工作池)` <---> `workbuf (基本内存块)`

```
+------------------------------------------------------------------+
|                            work (全局池)                         |
|   full: [workbuf] -> [workbuf]    empty: [workbuf] -> [workbuf]   |
+------------------------------------------------------------------+
           ^ putfull / getfull              ^ putempty / getempty
           |                                |
+------------------------------------------------------------------+
|                          gcWork (Per-P)                          |
|             wbuf1 (主工作块)    wbuf2 (备用工作块)                   |
+------------------------------------------------------------------+
```

### 1. `work` (全局 GC 工作池)
- **`bytesMarked`**: 累计标记的对象字节数。
- **`initialHeapLive`**: GC 开始时的存活堆内存字节数。
- **`full`**: 存放**已填满标记指针**的 `workbuf` 链表（待被 Worker 处理）。
- **`empty`**: 存放**空闲可重用**的 `workbuf` 链表。
- **`markrootJobs` / `markrootNext`**: 根节点标记任务总数及下一个等待标记的 Root 节点索引。
- **`assistQueue`**: 因辅助标记被挂起的 Goroutine 等待队列。
- **`sweepWaiters`**: 等待清扫的 Goroutine 列表。

### 2. `gcWork` (Per-P 本地标记缓存)
每个 P 拥有一个 `gcWork` 实例，消除多核并发竞争：
- **`wbuf1` / `wbuf2`**: 指向两个 `workbuf`。`wbuf1` 为当前主写入块，`wbuf2` 为备用/次级块。
- **`bytesMarked`**: 当前 P 本地已标记的字节数。
- **`scanWork`**: 当前 P 完成的扫描工作量统计。
- **`flushedWork`**: 已刷入全局 `work` 的工作量。

### 3. `workbuf` (基本任务 Buffer 块)
- **`node`**: 用于挂载到全局链表（`work.full` 或 `work.empty`）的指针。
- **`nobj`**: 当前 `obj` 数组中等待扫描的对象指针数量。
- **`obj[2048 - sz]`**: 数组，存储待扫描的对象指针。

---

## 三、 写屏障缓存 (`wbBuf`)

Go 使用混合写屏障（Hybrid Write Barrier）。为了避免每次触发屏障都直接操作 `gcWork` 或加锁，引入了 `wbBuf`（写屏障缓冲）：

- **`status`**: 写屏障状态。当处于 `gcphase == _GCmark` 时开启。
- **`buf[512]`**: 环形/线性指针缓冲区，暂存写屏障捕获的指针。
- **`next` / `end`**: 当前指针写入位置与缓冲区末尾指针。
- **快速路径 (`wbBuf.putfast`)**: 仅需将指针存入 `buf` 数组并移动 `next` 指向。
- **慢速路径 (`wbBuf.flush`)**: 当 `buf` 填满（`next == end`）时，调用 `wbBuf.flush` 将缓冲区中的指针一次性刷新放入 `gcWork` (`gcw.putBatch`)。

---

## 四、 栈扫描缓存 (`stackScanState`)

GC 在扫描 Goroutine 栈指针时，会将栈上找到的指针暂存：

- **`buf` / `freeBuf`**: 挂载指向 `stackWorkBuf` 的链表。
- **`head` / `tail`**: 任务队列头尾指针。
- **`stackWorkBuf`**: 存放栈扫描待处理指针的结构（与 `workbuf` 类似，`obj[2048-sz]`）。
- **`stackObjectBuf`**: 存放栈上对象（如逃逸分析未逃逸但包含指针的变量）的缓冲区。

---

## 五、 GC 协作角色与 Worker 线程调度

Go 通过 **后台 Worker (Background Mark Worker)** 与 **Mutator Assist (用户协程辅助标记)** 共同完成三色标记。

```
              ┌───────────────────────────┐
              │  gcBgMarkWorkerPool (执行池)│
              └─────────────┬─────────────┘
                            │ findRunnableGCWorker
                            v
               ┌─────────────────────────┐
               │   gcBgMarkWorkerNode    │
               │   - node                │
               │   - *g                  │
               │   - m                   │
               └────────────┬────────────┘
                            │ 启动后台 Goroutine
                            v
               ┌─────────────────────────┐
               │ g (Background Worker)   │
               │ startpc = gcBgMarkWorker│
               └─────────────────────────┘
```

### 1. `gcBgMarkWorkerNode` 与 `gcBgMarkWorkerPool`
- **`gcBgMarkWorkerPool`**: 维护待运行 GC Worker 的空闲池。
- **`gcBgMarkWorkerNode`**: 包装分配给 GC 标记任务的 `g`、`m` 和节点信息。
- 通过调度器函数 `findRunnableGCWorker` 从池中取出并唤醒 Worker 进行标记工作。

### 2. 后台 Worker Goroutine (`g.startpc == gcBgMarkWorker`)
- **`gcMarkWorkerMode`**: 运行模式，分为：
  - `gcMarkWorkerDedicatedMode` (专有模式：不被打断，全力扫描)
  - `gcMarkWorkerFractionalMode` (比例模式：占用指定 CPU 比例)
  - `gcMarkWorkerIdleMode` (空闲模式：利用空闲 CPU 执行)

### 3. 用户协程辅助标记 (Mutator Assist)
当用户 Goroutine 分配堆内存过快，导致标记速度跟不上分配速度时，调度器会强制该 Goroutine 协助执行标记：
- **触发条件**: `g.gcAssistBytes < 0`（即该 G 的内存分配配额耗尽）。
- **`preempt` / `stackguard0`**: 触发抢占机制，强制 Goroutine 进入 `gcAssistAlloc` 帮助扫描对象，扫描完成后增加 `gcAssistBytes` 信用额度。
- **`gcscandone`**: 标记 Goroutine 的栈扫描是否完成。

---

## 六、 关键调用路径与数据流转

1. **写屏障触发**:
   - 用户代码执行指针赋值 $\to$ `wbBuf.putfast` 写入本地 `wbBuf` $\to$ 满了之后调用 `wbBuf.flush` $\to$ `gcw.putBatch` 汇入 `P.gcWork`。
2. **任务缓存同步**:
   - `gcWork.wbuf1` 满 $\to$ `putfull` 刷入全局 `work.full`。
   - `gcWork.wbuf1` 空 $\to$ `tryGet` / `getempty` 从全局 `work.empty` 或 `work.full` 申请新的 `workbuf`。
3. **后台 Worker 执行**:
   - `findRunnableGCWorker` 获取任务 $\to$ 执行 `gcBgMarkWorker` $\to$ 从 `gcWork` 弹出一级/二级对象进行三色标记。
4. **用户协助标记 (Assist)**:
   - 内存分配逻辑检测到 `gcAssistBytes < 0` $\to$ 挂起当前 Goroutine 或强制协助标记，将扫描到的对象存入 `gcWork` $\to$ 归还信用额度后恢复运行。
