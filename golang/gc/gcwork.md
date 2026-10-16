下面把这张图从“**函数流水账**”重新整理成“**Go GC 一次完整生命周期 + 触发路径 + 并发标记 + Mark Assist + 终止阶段**”的结构。重点保留图中的关键函数和判断条件，方便面试记忆。

# Go GC 完整流程

## 1. 一句话理解

> **Go GC = 触发 GC → Mark 准备 → 并发标记 → Mark Assist → 标记结束 → Mark Termination → 清扫/Scavenge → 等待下一轮。**

核心目标：

```text
发现不可达对象
      ↓
标记仍然存活的对象
      ↓
清理未标记对象
      ↓
回收空间，并将部分物理内存归还 OS
```

---

# 2. 从 STAR 角度理解 GC

| 阶段 | 内容 |
|---|---|
| **S - Situation** | Go 程序不断分配对象，堆持续增长 |
| **T - Task** | 找出仍然存活的对象，回收不可达对象，同时尽量减少 STW |
| **A - Action** | GC 通过写屏障 + 并发 Mark + Mark Assist + Sweep 完成回收 |
| **R - Result** | 不可达对象被回收，堆空间重新可分配，部分空闲页进一步归还 OS |

因此 Go GC 的核心不是“停止程序然后扫描整个堆”，而是：

> **尽量并发执行，把 GC 工作分摊到后台 GC Worker、Mutator 和必要的短暂 STW 阶段。**

---

# 3. GC 从哪里开始？

图中有 **3 条主要触发路径**：

```text
                 GC 触发
                   │
       ┌───────────┼────────────┐
       ↓           ↓            ↓
     mallocgc     sysmon      runtime/GC
       │           │            │
       │           │         gcWaitOnMark
       │           │
       ↓           ↓
   分配压力       后台监控
       │           │
       └──────┬────┘
              ↓
            gcStart
```

### ① mallocgc 触发

对象分配时检查：

```text
heap 增长 / 分配压力
        ↓
达到 GC 条件
        ↓
触发 GC
```

这是最重要的业务路径。

---

### ② sysmon 触发

`sysmon` 是 Go runtime 的后台监控线程，可以发现 GC 相关条件并推动 GC。

---

### ③ runtime.GC()

用户主动调用：

```go
runtime.GC()
```

进入等待/启动 GC 的路径。

---

# 4. GC 初始化：gcinit

程序启动：

```text
runtime.main
    ↓
gcinit
    ↓
初始化 GC 参数 / 状态
    ↓
后台 GC 可以工作
```

同时 runtime 会建立：

```text
bgscavenge
sweep
```

等后台工作机制。

可以简单理解：

> `gcinit` 是 GC 系统初始化，不代表一次 GC cycle 的正式开始。

---

# 5. 一次 GC Cycle 的核心状态机

真正重要的是：

```text
gcStart
  │
  ├── mark phase prepare
  │
  ├── 开启写屏障
  │
  ├── 启动 GC Mark Workers
  │
  ├── 并发 Mark
  │      ├── GC Worker
  │      ├── Mutator Assist
  │      └── 写屏障
  │
  ├── gcMarkDone
  │
  ├── Mark Termination
  │
  ├── 关闭写屏障
  │
  └── Sweep
```

可以把整个过程记成：

> **Start → Prepare → Mark → Assist → Done → Termination → Sweep**

---

# 6. gcStart：开始一轮 GC

图中的核心流程：

```text
gcStart
  ↓
startSema
  ↓
worldsema
  ↓
trigger.test
  ↓
gcMarkStartWorkers
```

### 关键任务

`gcStart` 主要完成：

1. 获取 GC 启动所需同步资源
2. 根据 GC trigger 判断是否真的需要 GC
3. 准备 Mark 阶段
4. 启动 GC Worker
5. 进入 Mark Phase

其中：

```text
gcMarkStartWorkers
        ↓
创建/唤醒后台 GC Worker
```

---

# 7. Mark 阶段：GC 到底标记什么？

这是整个 GC 最核心的问题。

### 根对象 → 可达对象

```text
Root
 │
 ├── goroutine stack
 ├── global variables
 ├── runtime roots
 └── registers / other roots
          │
          ↓
       对象 A
          │
          ↓
       对象 B
          │
          ↓
       对象 C
```

GC 从 Root 开始遍历对象引用关系。

如果：

```text
Root → A → B → C
```

那么：

```text
A、B、C = reachable = 存活
```

而：

```text
D    ← 没有任何 Root 能到达
```

则：

```text
D = unreachable = 垃圾
```

**注意：**

> GC 并不是判断“对象有没有被使用”，而是判断对象是否仍然能够从 GC Roots 到达。

---

# 8. Mark 是怎么实现的？

图中出现：

```text
gcBgMarkWorker
        ↓
gcDrain
```

这是 Mark 的核心。

可以抽象成：

```text
Root
 ↓
扫描对象
 ↓
发现 pointer
 ↓
找到新对象
 ↓
标记对象
 ↓
加入 work queue
 ↓
继续扫描
```

即：

```text
work queue
    ↓
取对象
    ↓
扫描对象中的 pointer
    ↓
发现新对象
    ↓
mark
    ↓
加入 work queue
    ↓
继续
```

`gcDrain` 可以理解成：

> **不断从 GC work queue 中取待扫描对象，并把对象图继续向外扩展。**

---

# 9. 对象“被标记”到底意味着什么？

Go GC 使用对象对应的 **GC bitmap / mark bits** 等元数据记录标记状态。

简化理解：

```text
heap object

┌───────────────┐
│ object data   │
│               │
└───────────────┘
       ↑
       │
   GC metadata
       │
   mark bit = 1
```

其中：

```text
mark bit = 0
    ↓
当前 GC cycle 尚未标记

mark bit = 1
    ↓
当前 GC cycle 发现该对象可达
```

因此 Mark 阶段本质就是：

> **遍历对象图 + 设置对象的存活标记。**

---

# 10. 为什么需要写屏障？

因为 Mark 是**并发执行**的。

GC 在扫描：

```text
A → B
```

与此同时，Mutator（业务 goroutine）可能修改：

```text
A → C
```

甚至：

```text
A → B
```

变成：

```text
A → C
```

如果 GC 恰好在这个过程中扫描，就可能漏掉对象。

所以 Go 使用：

> **Write Barrier（写屏障）**

保证：

```text
Mutator 修改引用
        ↓
写屏障记录相关对象
        ↓
GC 不遗漏需要继续扫描的对象
```

因此可以记住：

```text
并发 Mark
   +
Write Barrier
   =
保证并发情况下 Mark 正确性
```

---

# 11. GC Worker：后台帮 GC 干活

图中：

```text
gcBgMarkStartWorkers
        ↓
gcBgMarkWorker
        ↓
gcDrain
```

GC Worker 的职责就是：

> **在后台不断执行 Mark 工作。**

大致：

```text
GC Worker
    ↓
获取 mark work
    ↓
gcDrain
    ↓
扫描对象
    ↓
标记新对象
    ↓
继续 drain
```

这样业务 goroutine 不需要承担全部 GC 工作。

---

# 12. Mark Assist：业务 goroutine 也可能帮忙

如果：

```text
业务分配速度
      >
GC 后台标记速度
```

那么 GC 就可能跟不上。

于是 Go 会让正在分配内存的 Mutator 帮忙：

```text
mallocgc
   ↓
需要分配
   ↓
GC mark 工作不足
   ↓
Mark Assist
   ↓
当前 goroutine 执行 gcDrain
   ↓
帮助 GC 标记
   ↓
继续分配
```

图右侧正是这条路径：

```text
mallocgc
   ↓
gcAssistAlloc
   ↓
gcParkAssist / assist queue
   ↓
gcAssistAlloc
```

### 核心思想

> **谁分配得多，谁可能承担更多 GC 工作。**

这就是 Go GC 中非常重要的：

**Mutator Assist**

---

# 13. 为什么有 Mark Assist？

可以从 GC pacing 理解：

```text
程序分配速度 ↑
        ↓
GC 工作量也必须 ↑
        ↓
后台 Worker 不够
        ↓
让 Mutator 分担
```

最终目标：

```text
分配速度
    ≈
GC 完成速度
```

避免：

```text
heap 不断增长
    ↓
GC 永远追不上
    ↓
内存压力越来越大
```

---

# 14. gcDrain：整个 Mark 阶段的核心动作

可以把 `gcDrain` 单独记下来：

```text
gcDrain
  │
  ├── 从 work queue 获取对象
  │
  ├── 扫描对象
  │
  ├── 找到 pointer
  │
  ├── 标记目标对象
  │
  ├── 新对象加入 work queue
  │
  └── 重复
```

所以面试时：

> **问“Go GC Mark 阶段具体干什么？”**

可以直接回答：

> 从 GC Roots 出发遍历对象图，通过 GC bitmap 等元数据记录对象存活状态，并通过 `gcDrain` 消费和产生 mark work；后台 GC Worker 和 Mutator Assist 都可以执行这部分工作。

---

# 15. Mark 什么时候结束？

当：

```text
GC Worker
 +
Mutator Assist
 +
Write Barrier
```

最终使得：

```text
待扫描对象 = 0
```

并且满足并发 Mark 终止条件后：

```text
gcMarkDone
```

进入：

```text
Mark Termination
```

---

# 16. Mark Termination：为什么还需要 STW？

并发 Mark 虽然大部分工作可以和业务并行，但最终必须建立一个**一致的 GC 状态**。

因此进入：

```text
gcMarkTermination
```

执行最终的 GC 状态收尾。

核心思想：

```text
Concurrent Mark
       ↓
发现 work 已基本耗尽
       ↓
STW / Mark Termination
       ↓
完成最终一致性处理
       ↓
关闭 Mark 状态
       ↓
进入 Sweep
```

这里不要简单理解成：

> “Go GC 会 STW 很久。”

更准确的是：

> **Go GC 把绝大多数 Mark 工作放到并发阶段，Mark Termination 只承担最终收尾的 STW 工作。**

---

# 17. Sweep：清理没有被 Mark 的对象

Mark 阶段得到：

```text
marked object   = 存活
unmarked object = 垃圾
```

因此 Sweep：

```text
Mark
 │
 ├── marked
 │      ↓
 │    保留
 │
 └── unmarked
        ↓
      回收
```

可以理解为：

```text
Mark：找出谁活着
Sweep：回收谁死了
```

这是最重要的一句话。

---

# 18. Sweep 与后续内存回收

图中后半部分出现：

```text
gcMarkDone
     ↓
gcSweep
     ↓
mheap_pages.scavengeStartGen
     ↓
readyForScavenger
```

这里需要区分两个概念：

### Sweep

主要是：

> **回收 Go heap 中已经不可达对象所占用的 span / slot，使空间能够重新用于对象分配。**

### Scavenge

进一步处理：

> **将满足条件的空闲物理页归还给 OS。**

所以：

```text
Object
 ↓
Sweep
 ↓
Heap 空间重新可用
 ↓
Scavenge
 ↓
部分物理内存归还 OS
```

**不要把 Sweep 和 Scavenge 当成一件事。**

---

# 19. 整张图压缩成一条主线

如果面试要求你画 Go GC：

```text
                ┌───────────────┐
                │   GC Trigger  │
                └───────┬───────┘
                        ↓
                     gcStart
                        ↓
              ┌──────────────────┐
              │   Mark Prepare   │
              │ 开启写屏障/准备状态 │
              └────────┬─────────┘
                       ↓
              gcMarkStartWorkers
                       ↓
          ┌──────── Concurrent Mark ────────┐
          │                                 │
          │  GC Worker → gcDrain            │
          │       ↓                         │
          │  扫描对象 → 设置 mark           │
          │       ↓                         │
          │  加入 work queue                │
          │                                 │
          │  Mutator → Mark Assist          │
          │                                 │
          │  Write Barrier → 保证正确性     │
          └────────────────┬────────────────┘
                           ↓
                      gcMarkDone
                           ↓
                  Mark Termination
                           ↓
                   关闭写屏障
                           ↓
                        Sweep
                           ↓
                  回收不可达对象
                           ↓
                     Scavenge
                           ↓
                  部分内存归还 OS
                           ↓
                       下一轮 GC
```

---

# 20. 图中的几个核心模块应该怎么记

| 模块 | 一句话 |
|---|---|
| `gcStart` | **启动一轮 GC** |
| `gcMarkStartWorkers` | **启动后台 Mark Worker** |
| `gcBgMarkWorker` | **后台执行 Mark** |
| `gcDrain` | **真正消费 Mark Work、扫描对象** |
| `gcAssistAlloc` | **Mutator 因分配压力参与 Mark** |
| `gcMarkDone` | **Mark 阶段结束判断/收尾** |
| `gcMarkTermination` | **Mark 最终终止阶段，完成一致性收尾** |
| `gcSweep` | **回收未 Mark 的对象空间** |
| `scavenge` | **进一步把空闲页归还 OS** |
| `Write Barrier` | **保证并发 Mark 不漏对象** |

---

# 21. 最终记忆模型

不要背图里的几十个函数，抓住 **4 层**：

```text
① 什么时候开始？
   Trigger
   ├─ mallocgc
   ├─ sysmon
   └─ runtime.GC

② 怎么找活对象？
   Concurrent Mark
   ├─ GC Worker
   ├─ gcDrain
   ├─ Write Barrier
   └─ Mark Assist

③ 什么时候结束？
   gcMarkDone
       ↓
   Mark Termination

④ 怎么回收？
   Sweep
       ↓
   Scavenge
       ↓
   必要时归还 OS
```

### 面试版一句话

> **Go GC 采用并发三色标记-清扫思想：当分配等条件触发 GC 后，通过 `gcStart` 准备 Mark 阶段并启动 GC Worker；并发 Mark 过程中，Worker 和 Mutator Assist 通过 `gcDrain` 遍历对象图、设置存活标记，Write Barrier 保证并发修改下的正确性；Mark 完成后进入 Mark Termination 做最终收尾，然后 Sweep 回收不可达对象，Scavenge 则进一步将空闲物理页归还 OS。**

这张图最值得掌握的其实就是 **“Trigger → Mark → Assist/Barrier → Mark Done → Termination → Sweep → Scavenge”** 这条主线，图中的大量 `sema / park / wake / sched` 更多属于 **GC 与 Go 调度器之间的实现细节**，不应该和 GC 核心算法混在一起记。
