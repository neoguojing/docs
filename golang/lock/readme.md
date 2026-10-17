# Go 并发原语与 Linux 锁机制深度解析（图解增强版）

> 核心导读：Go 的锁并非单一机制，而是**硬件原子指令**、**Linux 内核 Futex** 与 **Go 调度器（GMP）** 的精密协作。本文用大量 ASCII 图例与示例代码，把「全局哈希表 + Treap」「Futex」「runtime.mutex」「用户态 sync.Mutex」「读写锁 sync.RWMutex」「死锁检测机制」六个核心机制从结构、原理到运行流程逐层拆开讲解。

---

## 第一部分 全局哈希表与 Treap（sema 机制）图解

### 1.1 为什么需要全局哈希表

Go 用户态所有需要同步的组件（`sync.Mutex.sema`、`sync.Cond`、`WaitGroup`、`Channel` 内部等）一旦要休眠，都会落到同一个通用设施上——**sema 信号量表**。

问题在于：全世界可能有成千上万个不同的锁地址，如果只用一把大锁把所有等待者保护起来，所有锁的睡眠/唤醒都会在这一把全局锁上串行化，立刻成为性能瓶颈。于是 Go 采用 **哈希分桶**：

* 用等待地址 `addr` 哈希到一个**固定大小 251**（素数，减少碰撞）的数组 `semtable[251]`；
* 每个桶独立加锁，互不干扰；
* 每个桶内部用一棵 **Treap（树堆）** 管理等待同一地址的 goroutine。

### 1.2 整体结构图（哈希表 -> semaRoot -> Treap -> 等待链）

```text
                    等待地址 addr（如某个 Mutex.sema 的地址）
                                |
                                v
        hash = uintptr(addr / 8) % 251        （除以8消除对齐低位, 再模素数）
                                |
                                v
  +------------------------------------------------------------------+
  |                        semtable [251]  (全局数组)                |
  +------------------------------------------------------------------+
  |  [0]  semaRoot{lock, treap, nwait}   + 缓存行填充(pad)            |
  |  [1]  semaRoot{lock, treap, nwait}   + pad                       |
  |  ...                                                             |
  |  [k]  semaRoot{lock, treap, nwait}   <---- 本次命中的桶            |
  |  ...                                                             |
  | [250] semaRoot{lock, treap, nwait}   + pad                       |
  +------------------------------------------------------------------+
                                |
                    命中 semaRoot[k]，先 lock.Lock() 保护
                                |
                                v
                     semaRoot.k.treap  (Treap 树堆根节点)
                                |
          +---------------------+---------------------+
          |                     |                     |
     sudog(addr=A)         sudog(addr=B)         sudog(addr=C)   不同地址分枝
          |
     waitlink|  (相同 addr 的等待者用链表串起来, 减少树节点数)
          v
        sudog(addr=A) -> sudog(addr=A) -> sudog(addr=A)   同一锁的等待队列
          (head)                                     (tail)

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
    g    *g              // 绑定的 goroutine
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

    ticket uint32        // handoff 公平移交标记
}

```

### 1.4 Treap 为什么用「树堆」而不是链表

Treap = **Tree（按 addr 排序的二叉查找树）+ Heap（按随机权重 c 维护的堆）**。

```text
         (c=98) addr=B          <- 树按 addr 有序, 堆按 c 最大在上
         /         \
   (c=70) addr=A    (c=60) addr=D
        \               \
     (c=40) addr=C     (c=20) addr=E

```

* **为什么不用纯链表**：链表查找某地址要 O(N)；用 B-Tree 维护成本高；Treap 用随机优先级天然自平衡，**插入/删除/查找均为 O(log N)**，且实现简单。
* **相同 addr 折叠成链表**：若 100 个 G 等同一个锁，不必在树里建 100 个节点，只建 1 个树节点 + 一条 waitlink 链表，既省内存又保证同锁的 FIFO 公平。

### 1.5 P 的 sudogcache 对象池

```text
 加锁需要 sudog             唤醒后归还 sudog
        |                       ^
        v                       |
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
   +----------------------------------+
   |        futex_hash_bucket 数组 (内核) |
   +----------------------------------+
   |  bucket[i]: { spinlock, list_head }  <--- 命中    |
   +----------------------------------+
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
    if (atomic_exchange(&lock, 0) == 2) {        // 之前有等待者
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
// 状态(在 key 低 32bit):  0=空闲    1=锁定    2=锁定且有人 sleep

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
   futexsleep(&l.key低32位, 2, -1)    <---- M 在此真正挂起
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

### 4.1 结构体定义与内存状态布局

`sync.Mutex` 结构体本身非常轻量，仅包含两个字段：

```go
type Mutex struct {
    state int32  // 核心状态位：综合记录锁占用、唤醒标志、饥饿标志及等待者计数
    sema  uint32 // 信号量。注意：它自身不作为计数器，而是将其内存地址作为 Key，落入第一部分提到的全局 semtable 桶中进行哈希定位。
}

```

`state` 字段是一个 32 位的整数，被精密划分为四个独立功能区（位域）：

```text
 state 的 32bit 内存位域布局:
 +---------------------------------------------------+---+---+---+
 |             waiters (29 bit, 等待者计数)           | S | W | L |
 +---------------------------------------------------+---+---+---+
   ^                                                   ^   ^   ^
   | 高 29 位: 阻塞排队休眠的 Goroutine 数量            |   |   +-- 最低位 (Locked)：0=未锁, 1=已锁
   |                                                   |   +------ 次低位 (Woken)：0=无唤醒者, 1=已有被唤醒者在抢锁
   |                                                   +---------- 第 3 位 (Starving)：0=正常模式, 1=饥饿模式

```

---

### 4.2 两种运行模式：正常模式 vs 饥饿模式

| 维度 | 正常模式 (Normal Mode) 🌟 | 饥饿模式 (Starvation Mode) 🌙 |
| --- | --- | --- |
| **设计目标** | **追求最高吞吐量** | **解决尾延迟与「饥饿」问题** |
| **新来 G 的待遇** | 允许**自旋**并直接参与抢锁（允许插队） | **严禁插队**，直接排到等待队列尾部休眠 |
| **被唤醒 G 的待遇** | 与新来的、正在自旋的 G 共同竞争锁 | 锁直接交由队头被唤醒的 G (**Handoff**) |
| **进入条件** | 默认模式 | 当某个等待者单次等待锁的时间超过 **1ms** (`starvationThresholdNs`) 时触发 |
| **退出条件** | - | 当某次唤醒发现队列只剩最后一人，或总等待耗时已不足 1ms 时切回正常模式 |

---

### 4.3 加锁全景拆解：Lock 与 lockSlow

* **快路径 (Fast Path)**：`atomic.CompareAndSwapInt32(&m.state, 0, mutexLocked)` 成功直接返回。
* **慢路径 (`lockSlow`)**：
1. **自旋阶段 (`canSpin`)**：满足多核、`GOMAXPROCS > 1`、P 的本地队列为空时进行短暂自旋。
2. **计算新状态与 CAS 抢锁**：根据当前模式计算 `new` 状态并尝试 CAS 抢锁。
3. **休眠挂起 (`gopark`)**：获取 `sudog`，将其挂入对应哈希桶的 Treap 树中，调用 `goparkunlock` 让出 M。



---

### 4.4 解锁全景拆解：Unlock 与 unlockSlow

* **快路径 (Fast Path)**：`atomic.AddInt32(&m.state, -mutexLocked)` 后若无等待者直接返回。
* **慢路径 (`unlockSlow`)**：
* **饥饿模式**：触发 **Handoff** 直接交个队头 G，并调用 `goyield()` 让出 CPU。
* **正常模式**：减小 `waiters` 计数器，置 `woken` 标志，调用 `Semrelease` 唤醒某一个等待 G。



---

### 4.5 一次完整的「竞争 -> 挂起 -> 唤醒」时序图

```text
  G1 (持锁方)            semaRoot 内存及全局桶             G2 (抢锁方)
     |                                                      |
     |--- 1. Lock 成功 (state = 001)                        |
     |--- 2. 进入临界区执行业务代码                          |
     |                                                      |--- 1. Lock 尝试 CAS 失败
     |                                                      |--- 2. 自旋重试失败
     |                                                      |--- 3. 挂入 semtable 的 Treap
     |                                                      |--- 4. gopark 挂起 (G_waiting)
     |                                                    [ M 线程转去调度其他 G ]
     |
     |--- 3. 执行 Unlock()
     |--- 4. unlockSlow 发现有等待者
     |--- 5. Semrelease 唤醒队列中的 G2的sudog
     |--- 6. goready(G2) -> G2 状态变更为 _Grunnable
     |                                                      |
     |                                                      |--- 5. G2 被唤醒，重试抢锁成功
     |                                                      |--- 6. 顺利进入临界区执行！

```

---

## 第五部分 用户态读写锁 sync.RWMutex 深度解析

### 5.1 核心设计定位与结构体

```go
type RWMutex struct {
    w           Mutex  // 复用 sync.Mutex，保证写写互斥
    writerSem   uint32 // 写者等待读者退出的信号量
    readerSem   uint32 // 读者等待写者退出的信号量
    readerCount int32  // 核心状态：活跃读者数，负数代表写者存在
    readerWait  int32  // 写者到来时需要等待的旧读者剩余数
}

const rwmutexMaxReaders = 1 << 30

```

### 5.2 readerCount 的正负魔法

* `>= 0`：当前纯读者模式，数值即为活跃读者数。
* `< 0`：有写者在等待或持锁，真实读者数 = `readerCount + rwmutexMaxReaders`。

### 5.3 RLock / RUnlock 读锁运行流程详解

#### 1. RLock（加读锁流程）

```text
  RLock()
     |
     v
  atomic.AddInt32(&rw.readerCount, 1)  -> 将活跃读者计数 +1
     |
     v
  新计算出的 readerCount 值是否小于 0 ?
   ├── 是 (说明此时有写者正在等待或已经持有了写锁)
   │     └── 触发阻塞：sync_runtime_SemacquireMutex(&rw.readerSem, false, 0, 0)
   │         (当前 reader 挂起排队，等待写者释放锁后来唤醒它)
   └── 否 (无写者竞争，纯并发读环境)
         └── 直接加锁成功！安全进入读临界区。

```

#### 2. RUnlock（释放读锁流程）

```text
  RUnlock()
     |
     v
  r := atomic.AddInt32(&rw.readerCount, -1) -> 将活跃读者计数 -1
     |
     v
  r 是否小于 0 ? (小于 0 意味着当前环境存在写者，且刚才释放的可能是「旧读者」或部分读者)
   ├── 是
   │     └── atomic.AddInt32(&rw.readerWait, -1) 是否等于 0 ?
   │           ├── 是 (说明你是被当前写者阻塞的最后一个旧读者，终于把所有快照读者清空了)
   │           │     └── 唤醒写者：sync_runtime_Semrelease(&rw.writerSem, false, 1)
   │           └── 否 (还有其他旧读者没走完)
   │                 └── 直接返回，静静等待
   └── 否 (纯读者环境下释放读锁)
         └── 直接返回，无需做任何额外通知

```

### 5.4 Write Lock / Unlock 写锁运行流程与防饥饿机制简述

* **Lock**：先抢内部 `w` Mutex 确保写写互斥，再将 `readerCount` 减去 $2^{30}$ 形成「拦截网」（阻止新读者）；若发现有旧读者（`readerWait != 0`），则在 `writerSem`上休眠。
* **Unlock**：将 `readerCount` 加上 $2^{30}$ 恢复正数，循环唤醒所有因为写者而被阻塞的读者（通过 `readerSem`），最后释放内部 `w` Mutex 允许下一个写者竞争。

---

## 第六部分 Go 运行时的死锁检测机制 (`checkdead`)

### 6.1 触发场景与核心判定原则

当 Go 运行时无法继续调度任何 Goroutine 时，就会触发死锁检测函数 `checkdead`。它通常在特定生命周期或底层管理线程退出/挂起时触发：

* **触发场景**：`mexit`（线程退出）、`templateThread`、`incidlelocked`、`sysmon`（系统监控）、`mput` 等。

### 6.2 `checkdead` 核心执行流程解析

```text
                     checkdead() 被触发
                             |
                   1. 加调度器锁 sched.lock
                   2. 若正在发生 panic (panicking > 0) -> 直接退出
                             |
                             v
         计算当前活跃运行的 M 数量 (run):
         run = (sched.mnext - sched.nmfreed) - sched.nmidle - sched.nmidlelocked - sched.nmsys
                             |
            run > 0 ? (Linux 下有活跃运行的 M 线程)
             ├── 是 --> 非死锁，释放锁并返回
             └── 否 --> M 数量为 0，开始全面盘点所有 Goroutine (遍历 allg)
                             |
                             v
           遍历全局所有的 Goroutine (allg):
           ├── 忽略系统级内部 G
           ├── 检查 G 的状态：
           │     ├── 若为 _Gwaiting 或 _Gpreempted (说明在等待或被抢占) -> grunning++
           │     └── 若为 _Grunnable、_Grunning、_Gsyscall (活跃态) 
           │           └── 抛出异常：M 数量为 0 但存在活跃状态的 G (状态不一致)
                             |
         所有 G 检查完毕后，判断 grunning 是否为 0？
          ├── 若 grunning == 0 (没有任何等待或存活的 G)
          │     └── 抛出致命错误：no goroutines running - deadlock! (进程退出)
          └── 若 grunning != 0 (存在正在睡眠等待的 G)
                |
                v
       检查定时器与可唤醒源：
       1. 调用 timeSleepUntil() 获取最近超时的 timer 及其绑定的 P
       2. 从 midle 列表获取空闲 M，绑定 M 和 P，并尝试唤醒它 -> 若能被唤醒则非死锁，返回
       3. 遍历 allp：若任何一个 _p_.timers 的定时器个数大于 0 -> 有唤醒源，返回非死锁
                |
                v
       若所有检查均未通过（所有的 G 都在 sleep 且没有任何定时器/事件源可唤醒）：
       - 设置 g.m.throwing = -1 （禁止导出冗余栈信息）
       - 抛出致命死锁异常：all goroutines are asleep - deadlock!

```

---

## 错误修正汇总

| 原笔记说法 | 修正 | 依据 |
| --- | --- | --- |
| 饥饿阈值 10ms | **1ms** | `starvationThresholdNs = 1e6` 纳秒 |
| lock 前缀锁总线 | 现代 x86 锁**缓存行** | P6 后 Cache Locking 协议 |
| 单核不自旋只写 GOMAXPROCS<1 | 应为 **GOMAXPROCS <= 1**，且含「已持锁不自旋」「P 本地队列非空停止自旋」等 | `sync_runtime_canSpin` |
| 自旋锁可递归 | **不可重入** | 同 G 重复 Lock 即死锁 |
| 结构体锁值拷贝 | **必须用指针** | 拷贝复制 state/sema 致状态分裂 |
| 写锁会饿死 | **不会** | readerWait 是快照, 新读者被负 readerCount 拦截 |
| 死锁检测万能 | 仅检测「全 G 睡眠且无唤醒源/定时器」 | `checkdead` |

---

## 面试高频问答汇总

1. **sema 为什么要哈希分桶 + Treap？**
避免全局单锁瓶颈，通过 251 个桶降竞争；Treap 对同地址等待者提供 $O(\log N)$ 的高效增删查，并把相同 `addr` 折叠为 FIFO 链表，兼顾性能与公平。
2. **Futex 的「快」在哪？**
无竞争时纯用户态一条原子 CAS 指令完成，不陷入内核；只有真阻塞时才系统调用进内核排队，并用 `uaddr` 值做一致性校验防 `TOCTOU`。
3. **runtime.mutex 和 sync.Mutex 阻塞单位有何区别？**
`runtime.mutex` 保护调度器底层，挂起的是 OS 线程 M；`sync.Mutex` 保护用户业务，挂起的是 Goroutine G，M 可以转去调度其他 G。
4. **sync.Mutex 的饥饿模式为什么阈值设定为 1ms？**
设定 1ms 是在吞吐量与公平性之间做权衡：太短会导致模式频繁切换引起 CPU 抖动，太长会导致高并发下的尾延迟（Tail Latency）无法忍受。
5. **sync.Mutex 中 state 里的 woken 位起什么作用？**
已有被唤醒的 G 在自旋抢锁时会置位 `woken`，持锁者在 Unlock 时如果看到 `woken` 为真，就不会去主动唤醒休眠队列中的 G，从而避免重复唤醒造成「惊群效应」。
6. **sudog 从哪里来？去哪里回？**
优先从当前 P 的本地 `sudogcache` 对象池获取，没有则从全局申请，用完后归还，从而避免频繁内存分配，降低 GC 压力。
7. **sync.RWMutex 如何防止写者饥饿（Writer Starvation）？**
当写者发起加锁时，会将 `readerCount` 减去 $2^{30}$ 变为负数，形成一个「拦截网」，让后续所有新来的读锁直接排队休眠；同时通过 `readerWait` 记录写者到来瞬间的旧读者快照，只等这些旧读者走完就立刻获得写锁，防止被源源不断的读请求淹没。
8. **为什么不能拷贝赋值 sync.Mutex 或 sync.RWMutex？**
锁的内部状态（`state`、`sema` 等）在加锁后可能记录了等待者计数、唤醒标志或指向特定内存地址的钥匙。拷贝会导致新旧实例状态分裂、sema 关联错乱，进而引发死锁或不可预知的内存并发安全问题。
9. **Go 运行时的 `checkdead` 是如何判定死锁的？**
当所有 OS 线程 M 停止运行时（`run == 0`），Go 会检查所有 Goroutine（`allg`）的状态。如果所有存活的 G 均处于等待状态（`_Gwaiting` 或 `_Gpreempted`），且系统中没有任何未完成的定时器（Timers）或其他唤醒源，运行时便会判定所有 G 都在睡眠且无法被唤醒，从而触发 `all goroutines are asleep - deadlock!` 致命异常。

---
