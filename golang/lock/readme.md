# Go 并发原语与 Linux 锁机制深度解析（图解增强版）

> 核心导读：Go 的锁并非单一机制，而是**硬件原子指令**、**Linux 内核 Futex** 与 **Go 调度器（GMP）** 的精密协作。本文用大量 ASCII 图例与示例代码，把「全局哈希表 + Treap」「Futex」「runtime.mutex」「用户态 sync.Mutex」四个核心机制从结构、原理到运行流程逐层拆开讲解。

---

## 第一部分 全局哈希表与 Treap（sema 机制）图解

### 1.1 为什么需要全局哈希表

Go 用户态所有需要同步的组件（`sync.Mutex.sema`、`sync.Cond`、`WaitGroup`、`Channel` 内部等）一旦要休眠，都会落到同一个通用设施上——**sema 信号量表**。

问题在于：全世界可能有成千上万个不同的锁地址，如果只用一把大锁把所有等待者保护起来，所有锁的睡眠/唤醒都会在这一把全局锁上串行化，立刻成为性能瓶颈。于是 Go 采用 **哈希分桶**：

- 用等待地址 `addr` 哈希到一个**固定大小 251**（素数，减少碰撞）的数组 `semtable[251]`；
- 每个桶独立加锁，互不干扰；
- 每个桶内部用一棵 **Treap（树堆）** 管理等待同一地址的 goroutine。

### 1.2 整体结构图（哈希表 -> semaRoot -> Treap -> 等待链）

```text
                    等待地址 addr（如某个 Mutex.sema 的地址）
                                  |
                                  v
        hash = uintptr(addr / 8) % 251        （除以8消除对齐低位, 再模素数）
                                  |
                                  v
  +------------------------------------------------------------------+
  |                       semtable [251]  (全局数组)                  |
  +------------------------------------------------------------------+
  |  [0]  semaRoot{lock, treap, nwait}   + 缓存行填充(pad)           |
  |  [1]  semaRoot{lock, treap, nwait}   + pad                       |
  |  ...                                                              |
  |  [k]  semaRoot{lock, treap, nwait}  <---- 本次命中的桶            |
  |  ...                                                              |
  | [250] semaRoot{lock, treap, nwait}   + pad                       |
  +------------------------------------------------------------------+
                                  |
                    命中 semaRoot[k]，先 lock.Lock() 保护
                                  |
                                  v
                     semaRoot.k.treap  (Treap 树堆根节点)
                                  |
          +-----------------------+-----------------------+
          |                       |                       |
       sudog(addr=A)          sudog(addr=B)           sudog(addr=C)   不同地址分枝
          |
     waitlink|  (相同 addr 的等待者用链表串起来, 减少树节点数)
          v
       sudog(addr=A) -> sudog(addr=A) -> sudog(addr=A)   同一锁的等待队列
          (head)                                        (tail)
```

### 1.3 semaRoot 与 sudog 结构体

```go
// 每个桶的根: 用一棵 treap 管理本桶内所有等待地址
type semaRoot struct {
    lock  mutex     // runtime 内部锁, 保护本桶
    treap *sudog    // treap 根, 按 addr 排序
    nwait uint32    // 本桶等待者总数(读可不加锁)
}

// 一个 sudog 代表"某个正在等待同步对象的 G"
type sudog struct {
    g    *g             // 绑定的 goroutine
    elem unsafe.Pointer // channel 传输的数据

    // Treap 树结构(按 addr 排序, c 是随机权重维持堆性质)
    parent *sudog
    left   *sudog
    right  *sudog
    c      int32        // 随机数, 维持 treap 的堆平衡

    // 同一 addr 的等待链表(FIFO, 公平)
    waitlink *sudog
    waittail *sudog
    nwait    int32

    ticket uint32       // handoff 公平移交标记
}
```

### 1.4 Treap 为什么用「树堆」而不是链表

Treap = **Tree（按 addr 排序的二叉查找树）+ Heap（按随机权重 c 维护的堆）**。

```text
          (c=98) addr=B          <- 树按 addr 有序, 堆按 c 最大在上
          /         \
   (c=70) addr=A    (c=60) addr=D
        \                \
     (c=40) addr=C     (c=20) addr=E
```

- **为什么不用纯链表**：链表查找某地址要 O(N)；用 B-Tree 维护成本高；Treap 用随机优先级天然自平衡，**插入/删除/查找均为 O(log N)**，且实现简单。
- **相同 addr 折叠成链表**：若 100 个 G 等同一个锁，不必在树里建 100 个节点，只建 1 个树节点 + 一条 waitlink 链表，既省内存又保证同锁的 FIFO 公平。

### 1.5 P 的 sudogcache 对象池

```text
 加锁需要 sudog            唤醒后归还 sudog
        |                        ^
        v                        |
  p.sudogcache [128]  <---- 复用, 避免 malloc, 降低 GC 压力
        |
   (本地无空闲) -> runq.put(sudog) -> 全局 sudogcache -> 实在没有才 new
```

---

## 第二部分 Futex（Fast Userspace Mutex）示例与图解

### 2.1 核心哲学：无竞争不进内核

传统 System V 信号量每次 P/V 都进内核。Futex 的关键洞察：**绝大多数加解锁是无竞争的**，应纯用户态用一条原子指令解决；只有真正要阻塞时才进内核排队。

### 2.2 内核数据结构

```text
                  uaddr（用户态某个 uint32 地址）
                          |
                          v
        hash = jhash(uaddr) % FUTEX_HASH_SIZE   （内核哈希表）
                          |
                          v
   +--------------------------------------------------+
   |        futex_hash_bucket 数组 (内核)             |
   +--------------------------------------------------+
   |  bucket[i]: { spinlock, list_head }  <--- 命中    |
   +--------------------------------------------------+
                          |  bucket 内自旋锁保护
                          v
        futex_q (等待队列链表, 按 uaddr 组织)
             node(uaddr, task, ...) -> node -> node
```

### 2.3 用户态锁变量约定

```text
   uaddr 的值约定（Linux 经典 futex 语义）:
     0 = 锁空闲, 无人等待
     1 = 锁被占用, 无人排队
     2 = 锁被占用, 且有等待者在队列中（解锁时需通知内核）
```

### 2.4 futex_wait 流程（带示例）

```text
futex_wait(uaddr, val, timeout):
   1. 加 bucket 自旋锁
   2. if (*uaddr != val)  -> 立即返回 -EAGAIN    (值变了说明无需等, 防 TOCTOU)
   3. 进程状态置 TASK_INTERRUPTIBLE
   4. 把当前 task 挂到 bucket 的等待链表
   5. 释放自旋锁
   6. 设置超时定时器(可选)
   7. schedule() 真正挂起进程, 直到被唤醒/超时/信号
```

### 2.5 futex_wake 流程

```text
futex_wake(uaddr, cnt):
   1. 找到 uaddr 对应的 bucket
   2. 加 bucket 自旋锁
   3. 遍历链表, 找到匹配 uaddr 的节点
   4. wake_up_process() 唤醒, 最多 cnt 个
   5. 释放自旋锁
```

### 2.6 最小可运行示例（C 调用 futex）

```c
#include <linux/futex.h>
#include <sys/syscall.h>
#include <unistd.h>
#include <stdatomic.h>

static _Atomic int lock = 0;   // 这就是 futex 关注的 uaddr

// 加锁: 0->1 抢锁, 抢不到则进内核睡
void my_lock(void) {
    int expect = 0;
    if (!atomic_compare_exchange_strong(&lock, &expect, 1)) {
        // 抢锁失败: 期望 *uaddr 仍为 1, 睡在内核
        // FUTEX_WAIT 会原子地"检查 *uaddr==1 再睡眠"
        syscall(SYS_futex, &lock, FUTEX_WAIT, 1, NULL, NULL, 0);
        // 醒来后再抢一次(简化: 这里再 CAS 一轮)
        while (!atomic_compare_exchange_strong(&lock, &expect, 1))
            syscall(SYS_futex, &lock, FUTEX_WAIT, 1, NULL, NULL, 0);
    }
}

// 解锁: 1->0, 若可能有人睡则唤醒
void my_unlock(void) {
    if (atomic_exchange(&lock, 0) == 2) {       // 之前有等待者
        syscall(SYS_futex, &lock, FUTEX_WAKE, 1, NULL, NULL, 0);
    }
}
```

### 2.7 Go runtime 对 futex 的封装

```go
// runtime/os_linux.go 简化
func futexsleep(addr *uint32, val uint32, ns int64) {
    if *addr != val { return }          // 值已变, 不睡
    // ns<0: 无限睡眠; 否则带超时
    // -> 系统调用 SYS_futex, FUTEX_WAIT
}
func futexwakeup(addr *uint32, cnt uint32) {
    // -> 系统调用 SYS_futex, FUTEX_WAKE, 最多唤醒 cnt 个
}
```

---

## 第三部分 运行时内部锁 runtime.mutex 示例与图解

### 3.1 定位：调度器自用，挂起的是 M 不是 G

```text
  用途: 保护 scheduler 内部结构(allp, runq, gFree, gwq, timers...)
  阻塞单位: OS 线程 M 直接 futexsleep (因为此时往往 G 正在 park, 不能依赖 G 调度)
  复杂度: 远比 sync.Mutex 简单, 无饥饿模式, 无 sema 树
```

### 3.2 数据结构与状态

```go
// GOEXPERIMENT=staticlockranking 关闭时
type mutex struct {
    lockRankStruct   // 默认空结构体
    key uintptr      // 一个 32bit 标志当作 futex 的 uaddr
}
// 状态(在 key 低 32bit):  0=空闲   1=锁定   2=锁定且有人 sleep
```

### 3.3 lock2 加锁流程图

```text
  lock2(lk, skip)
        |
        v
   m.locks++ ; 关闭抢占 (aslock)
        |
        v
   xchg(&l.key, 1)  -----> 返回旧值 old
        |
   old==0 ? --是----> 抢到锁, m.locks--, 返回
        |否(锁被别人占)
        v
   自旋 procyield(30 * pause) 约 4 次, 每轮再 xchg 抢一次
        |抢到
        v  返回
        |没抢到
   osyield()  (sched_yield 主动让出 CPU, 给持锁者机会跑完)
        |
   xchg 再抢一次
        |抢到 返回
        |没抢到
   xchg(&l.key, 2)  (把状态标为"有人要睡")
        |旧值若为 1(之前无人睡) 说明需要登记 sleep
   futexsleep(&l.key低32位, 2, -1)   <---- M 在此真正挂起
        ^
        | 被 futexwakeup 唤醒后继续循环, 重新 xchg 抢锁
        +-----------------------------------+
```

### 3.4 unlock 解锁流程图

```text
  unlock(lk)
        |
   xchg(&l.key, 0)  -> 旧值 old
        |
   old==2 ?  --是--> futexwakeup(&l.key低32位, +1)  唤醒一个等待 M
        |否
        v
   m.locks-- ; 恢复抢占
```

### 3.5 runtime.mutex vs sync.Mutex 对比

```text
        +----------------+-------------------+-----------------------+
        |    维度        | runtime.mutex     | sync.Mutex            |
        +----------------+-------------------+-----------------------+
        | 服务对象       | 调度器内部数据    | 用户业务数据          |
        | 阻塞单位       | OS 线程 M 睡眠    | goroutine G 挂起      |
        | 自旋           | 有(短)            | 有(canSpin 条件多)    |
        | 饥饿模式       | 无                | 有(>1ms 触发)         |
        | 等待组织       | 直接 futex 队列   | semaRoot + sudog treap|
        | 抢占影响       | m.locks++ 禁抢占  | 不涉及 m.locks        |
        +----------------+-------------------+-----------------------+
```

---

## 第四部分 用户态锁 sync.Mutex 设计结构、运行机制与图解

### 4.1 数据结构与位域布局

```go
type Mutex struct {
    state int32
    sema  uint32   // 仅作为 semaRoot 哈希定位的"地址钥匙"
}

const (
    mutexLocked   = 1 << iota // 0b001  已锁定
    mutexWoken                // 0b010  已有 G 被唤醒(防重复唤醒)
    mutexStarving             // 0b100  饥饿模式
    mutexWaiterShift = iota   // 3      waiters 左移 3 位存放
)
const starvationThresholdNs = 1e6 // 1ms
```

```text
 state 的 32bit 布局:
 +---------------------------------------------------+---+---+---+
 |              waiters (29 bit, 等待者计数)          | W | K | L |
 +---------------------------------------------------+---+---+---+
   ^                                                ^   ^   ^
   | 高29位: 排在队列里休眠的 G 数量                 |   |   最低位 locked 锁被持有
   |                                                |   次低位 woken 已有唤醒者在抢
   |                                                第3位 starving 进入饥饿模式
```

### 4.2 两种模式（正常 / 饥饿）对比

```text
 +-------------------+----------------------------------+---------------------------+
 |                   |            正常模式               |        饥饿模式            |
 +-------------------+----------------------------------+---------------------------+
 | 新来的 G          | 可自旋, 可 CAS 抢锁(允许插队)    | 禁止抢, 直接排到队尾休眠   |
 | 唤醒的 G          | 与新来的 G 重新竞争              | 直接拿到锁(handoff 移交)   |
 | 设计目标          | 高吞吐(谁热谁拿)                 | 公平, 解决尾延迟           |
 | 进入条件          | 默认                             | 某 G 等待 > 1ms            |
 | 退出条件          | -                                | 队列只剩最后一个 或 等待<1ms|
 +-------------------+----------------------------------+---------------------------+
```

### 4.3 模式切换状态图

```text
            (默认)
        +-------------+
        |  正常模式   | <---------------------------------+
        +------+------+                                  |
               |  某等待者等待时长 > 1ms                  |
               |  CAS 设置 mutexStarving                  | 最后一个等待者拿到锁
               v                                          | (waiters==1) 或 等待<1ms
        +-------------+                                  |
        |  饥饿模式   | ---------------------------------+
        +-------------+
        Unlock 时: handoff=true 直接把锁移交队头 G
```

### 4.4 Lock 完整流程图（快路径 + lockSlow）

```text
                 Lock()
                   |
        CAS(state, 0, mutexLocked) ?
          / 是(快路径)        \ 否
        返回                 lockSlow()
                                |
                +---------------v----------------+
                |  自旋循环 for {                 |
                |   canSpin()?                    |
                |     是: 自旋(设 woken, doSpin)   |
                |     CAS(state,new,new+locked)?  |
                |   计算 new:                     |
                |     非饥饿 -> new |= locked      |
                |     (locked|starving) -> waiters+1|
                |     超时且未饥饿 -> new|=starving |
                |     若已 woken -> 清 new.woken    |
                |   CAS(state,old,new) 成功:      |
                |     旧未 locked/starving -> 抢锁成功 break|
                |     否则 -> 要休眠               |
                |  }                              |
                +---------------|----------------+
                                | 抢不到锁
                                v
        sync_runtime_SemacquireMutex(addr, lifo, tryOnBusyPoll)
                                |
                                v
                semacquire1:
                  cansemacquire()? CAS sema-1 成功 -> 直接返回
                  否则:
                    取 sudog (来自 p.sudogcache)
                    root = semtable[hash(addr)]
                    root.lock.Lock()
                    root.nwait++
                    cansemacquire()? 成功 -> nwait--, 返回
                    失败 -> 把 sudog 挂进 treap (lifo 头插)
                    goparkunlock()  <---- 当前 G 置 _Gwaiting, 让出 M
                    (被唤醒后再 cansemacquire)
                  醒来后计算等待时长, 决定是否维持饥饿
```

### 4.5 Unlock 完整流程图

```text
                 Unlock()
                   |
        new = state - mutexLocked
        CAS(state, old, new)  失败则重试
                   |
        new == 0 ?  --是--> 无锁无等待者, 直接返回
                   |否
                   v
        unlockSlow(new)
          |
   new & mutexStarving ?
     / 是                                \ 否(正常模式)
   handoff: Semrelease(addr, 1, true)     for {
   直接把锁交给队头 G,                      new & (locked|woken|starving) !=0 ?
   并 goyield 让出 CPU 给该 G 跑            有等待者? 无 -> return
                                            waiters-- , 设置 woken
                                            Semrelease(addr, 1, false)
                                            唤醒一个 G 重新竞争
                                          }
```

### 4.6 semrelease1（唤醒）流程图

```text
  semrelease1(addr, handoff):
     root = semaRoot(hash(addr))
     root.lock.Lock()
     root.nwait == 0 ? -> return  (无等待者)
     sudog = root.dequeue(s, addr)   (从 treap/链表摘出一个等待者)
     root.nwait--
     if handoff && cansemacquire() { sudog.ticket = 1 }  (公平移交凭证)
     gp := sudog.g
     goready(gp)        (G 置 _Grunnable, 放入 p.runnext)
     root.lock.Unlock()
     if sudog.ticket==1 { goyield() }  (让出 CPU, 确保被移交者优先跑)
```

### 4.7 一次完整「竞争-睡眠-唤醒」时序图

```text
  G1 (持锁)            内核/semaRoot              G2 (抢锁)
     |  Lock 成功(state=locked)                      |
     |  ---- 临界区 ----                              |
     |                                        Lock CAS 失败
     |                                        自旋 noSpin? 不满足
     |                                        Semacquire 取 sudog
     |                                        hash->root, 入 treap
     |                                        nwait++ (=1)
     |                                        gopark  --> G2 进入 _Gwaiting
     |                                     [M 去跑别的 G, 不阻塞线程]
     |  Unlock()
     |   new=state-locked, waiters=1 !=0
     |   unlockSlow: 正常模式, waiters--, 置 woken
     |   Semrelease -> dequeue(G2的sudog)
     |   goready(G2) -> G2 入 runnext
     |                              G2 醒来 -> cansemacquire 成功
     |                              G2 拿到锁, 继续临界区
```

---

## 附：读写锁 RWMutex 结构回顾（与上述机制协同）

```go
type RWMutex struct {
    w           Mutex   // 写写互斥, 复用上面的 sync.Mutex 全套机制
    writerSem   uint32  // 写者等读者退出
    readerSem   uint32  // 读者等写者退出
    readerCount int32   // 正=读者数; 减 2^30 后变负=有写者
    readerWait  int32   // 写者到来时活跃读者的快照(防写者饿死)
}
```

```text
 ReaderCount 视角:
   +N   正常, N 个读者
   Lock 写者: readerCount -= 2^30  => 变负
   新读者见负数 -> 阻塞 readerSem   (写者能"插队"挡住新读者)
   旧读者逐个 RUnlock -> readerWait--, ==0 时唤醒 writerSem  (只等快照里的旧读者 => 不饿死)
   写者 Unlock -> readerCount += 2^30 转正 -> 唤醒全部 readerSem
```

---

## 错误修正汇总

| 原笔记说法 | 修正 | 依据 |
|---|---|---|
| 饥饿阈值 10ms | **1ms** | `starvationThresholdNs = 1e6` 纳秒 |
| lock 前缀锁总线 | 现代 x86 锁**缓存行** | P6 后 Cache Locking 协议 |
| 单核不自旋只写 GOMAXPROCS<1 | 应为 **GOMAXPROCS <= 1**，且含「已持锁不自旋」「P 本地队列非空停止自旋」等 | `sync_runtime_canSpin` |
| 自旋锁可递归 | **不可重入** | 同 G 重复 Lock 即死锁 |
| 结构体锁值拷贝 | **必须用指针** | 拷贝复制 state/sema 致状态分裂 |
| 写锁会饿死 | **不会** | readerWait 是快照, 新读者被负 readerCount 拦截 |
| 死锁检测万能 | 仅检测「全 G 睡眠且无唤醒源/定时器」 | `checkdead` |

---

## 面试高频问答

1. **sema 为什么要哈希分桶 + Treap？** 避免全局单锁瓶颈，分桶降竞争；Treap 对同地址等待者提供 O(log N) 增删查，并把相同 addr 折叠为 FIFO 链表保证公平。
2. **Futex 的「快」在哪？** 无竞争时纯用户态一条 CAS 完成，不进内核；只有真阻塞才进内核排队，用 uaddr 值做一致性校验防 TOCTOU。
3. **runtime.mutex 和 sync.Mutex 阻塞单位区别？** runtime 版睡 M（线程），用户版 park 的是 G（协程），M 继续跑别的 G。
4. **饥饿模式为什么是 1ms？** 太短频繁切换抖动，太长尾延迟大；1ms 兼顾吞吐与公平。
5. **state 里 woken 位的作用？** 已有唤醒者在抢锁时置位，防止 Unlock 重复唤醒导致惊群。
6. **sudog 从哪来？** 优先 P 的 sudogcache 本地池，无则全局，再无可 new，归还时复用，减少 GC。
7. **handoff=true 与 goyield 配合做什么？** 把锁直接移交队头 G 并让出 CPU，确保被移交者优先执行，保证公平。
