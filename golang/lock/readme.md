# Go 锁机制：原理、实现与面试指南

注：以下源码细节以 Go 1.22 前后版本为参考，不同小版本实现可能有微调。

## 1. 总览

Go 的锁体系可以分成三层：

1. **硬件原子原语**：`lock` 前缀、`xchg`、`cmpxchg`、`PAUSE`、内存屏障。
2. **内核阻塞原语**：Linux 下主要依赖 `futex`，Go 封装为 `futexsleep`、`futexwakeup`。
3. **运行时同步原语**：`runtime` 内部 `mutex`、用户态 `sync.Mutex`、`sync.RWMutex`、`sema` 信号量。

核心原则：

- **多核才适合自旋**：单核自旋没有意义，还会占用调度资源。
- **锁不可拷贝**：`sync.Mutex`、`sync.RWMutex` 一旦使用后拷贝，会导致状态不一致。
- **结构体中的锁应使用指针**：避免值拷贝后锁状态分裂。
- **自旋锁不可重入**：同一 goroutine 重复加锁会死锁。
- **用户态 Mutex 不是自旋锁**：它是“CAS 快路径 + 自旋 + 信号量休眠”的混合锁。

---

## 2. 底层原子与系统原语

### 2.1 原子操作

| 原语 | 作用 | 说明 |
| ------ |------ |------ |
| `lock` 前缀 | 保证指令原子性 | 现代 CPU 通常锁缓存行，不是简单锁总线 |
| `xchg` | 交换并返回旧值 | 天然具有原子性，常用于抢锁 |
| `cmpxchg` | CAS | 旧值匹配则写入新值，并返回旧值 |
| `PAUSE` | 提示 CPU 处于自旋等待 | 降低功耗、缓解内存序冲突 |
| `barrier()` | 编译器屏障 | 防止寄存器缓存旧值，强制从内存重读 |
| `osyield` | 主动让出 CPU | Linux 下对应 `sched_yield` |

### 2.2 futex

Go 的 `runtime` 内部锁依赖 Linux `futex`。

```text
用户空间 uaddr
     │
     ▼
futex_wait: 若 *uaddr == val，则进入内核休眠
futex_wake: 唤醒等待在该 uaddr 上的线程
```

`futex_wait` 大致流程：

1. 加自旋锁保护 hash bucket。
2. 检查 `*uaddr == val`。
3. 不等则立即返回。
4. 相等则将进程加入等待队列。
5. 挂起进程，等待唤醒或超时。

`futex_wake` 大致流程：

1. 找到 `uaddr` 对应的 hash bucket。
2. 加锁保护。
3. 找到等待节点。
4. 唤醒对应进程。
5. 释放锁。

Go 封装为：

```go
func futexsleep(addr *uint32, val uint32, ns int64)
func futexwakeup(addr *uint32, cnt uint32)
```

---

## 3. 运行时内部锁 `runtime.mutex`

`runtime.mutex` 是 Go 调度器内部使用的锁，不是用户直接使用的 `sync.Mutex`。

```go
type mutex struct {
    lockRankStruct
    key uintptr
}
```

### 3.1 加锁流程

```text
m.locks++
   │
   ▼
xchg 抢锁
   │
   ├─ 成功 → 返回
   │
   └─ 失败 → 自旋（约 4 次 + PAUSE）
                │
                ├─ 成功 → 返回
                │
                └─ 失败 → osyield 让出 CPU
                            │
                            └─ 仍失败 → futexsleep 休眠
```

### 3.2 解锁流程

```text
xchg 将锁置为未锁定
   │
   ▼
若之前状态为休眠 → futexwakeup 唤醒等待线程
   │
   ▼
m.locks--
```

### 3.3 静态锁排序

Go 支持 `GOEXPERIMENT=staticlockranking`，用于检测锁顺序问题，默认关闭。开启后会记录锁获取顺序，帮助发现潜在死锁。

---

## 4. 用户锁 `sync.Mutex`

### 4.1 数据结构

```go
type Mutex struct {
    state int32
    sema  uint32
}
```

`state` 位域：

```text
| 29 bits waiters | 1 bit starving | 1 bit woken | 1 bit locked |
```

常量：

```go
mutexLocked        = 1 << iota // 0b0001
mutexWoken                     // 0b0010
mutexStarving                  // 0b0100
mutexWaiterShift = iota        // 3
starvationThresholdNs = 1e6    // 1ms
```

### 4.2 两种模式

| 模式 | 特点 | 适用 |
| ------ |------ |------ |
| 正常模式 | 唤醒者需与新来者竞争，吞吐高 | 大多数场景 |
| 饥饿模式 | 锁直接交给队列头部等待者，公平性高 | 等待超过 1ms 时进入 |

### 4.3 Lock 流程

```text
CAS(state, 0, locked)
   │
   ├─ 成功 → 快速加锁
   │
   └─ 失败 → lockSlow()
               │
               ├─ 可自旋 → PAUSE 自旋，尝试设置 woken
               │
               ├─ 非饥饿 → 尝试 CAS 抢锁
               │
               ├─ 饥饿 → 不抢锁，waiters++
               │
               ├─ 等待超 1ms → 设置 starving
               │
               └─ 抢不到 → SemacquireMutex 休眠
```

自旋条件 `sync_runtime_canSpin` 大致包括：

- 自旋次数未超过上限；
- `GOMAXPROCS > 1`；
- 当前 P 没有过多可运行 goroutine；
- 当前 goroutine 尚未持有锁。

### 4.4 Unlock 流程

```text
state -= mutexLocked
   │
   ├─ 没有等待者 → 返回
   │
   ├─ 饥饿模式 → handoff=true，直接交给队列头部等待者
   │
   └─ 正常模式 → 唤醒一个等待者，由它继续竞争
```

### 4.5 状态流转图

```text
          新 goroutine 抢锁成功
                 ▲
                 │
        ┌────────┴────────┐
        │   正常模式       │
        │  新来者可自旋    │
        └────────┬────────┘
                 │ 等待 > 1ms
                 ▼
        ┌────────────────┐
        │   饥饿模式      │
        │  锁直接移交队首 │
        └────────┬────────┘
                 │ 队尾等待者获取锁
                 │ 或等待时间 < 1ms
                 ▼
           回到正常模式
```

---

## 5. 信号量 `sema` 与等待队列

`sync.Mutex` 的 `sema` 字段并不是传统计数信号量，而是用于定位等待队列的地址。

```go
type semaRoot struct {
    lock  mutex
    treap *sudog
    nwait uint32
}
```

全局表：

```go
var semtable [251]struct {
    root semaRoot
    pad  [CacheLineSize - unsafe.Sizeof(semaRoot{})]byte
}
```

通过 `sema` 地址哈希到 `semtable`，找到对应 `semaRoot`。每个地址对应一棵 treap，相同地址的等待者通过 `sudog.waitlink` 组成链表。

```text
Mutex.sema
    │
    ▼
semtable[hash]
    │
    ▼
semaRoot.treap
    │
    ├─ sudog(addr=A) → sudog → sudog
    ├─ sudog(addr=B) → sudog
    └─ sudog(addr=C) → sudog → sudog → sudog
```

`runtime_SemacquireMutex` 会：

1. 尝试 `cansemacquire` 快速抢占；
2. 失败则申请 `sudog`；
3. 加入等待树；
4. `goparkunlock` 休眠当前 goroutine；
5. 被唤醒后再次检查是否获得锁。

---

## 6. 读写锁 `sync.RWMutex`

### 6.1 数据结构

```go
type RWMutex struct {
    w           Mutex
    writerSem   uint32
    readerSem   uint32
    readerCount int32
    readerWait  int32
}
```

其中：

```go
const rwmutexMaxReaders = 1 << 30
```

### 6.2 readerCount 的含义

`readerCount` 同时承担两个职责：

- 正数：表示当前活跃读锁数量；
- 负数：表示有写锁正在等待或持有。

写锁通过：

```go
atomic.AddInt32(&rw.readerCount, -rwmutexMaxReaders)
```

将 `readerCount` 变为负数，从而阻止新的读锁进入。

### 6.3 RLock / RUnlock

```text
RLock:
  readerCount++
     │
     ├─ >= 0 → 读锁获取成功
     │
     └─ < 0  → 有写锁等待，阻塞在 readerSem

RUnlock:
  readerCount--
     │
     ├─ >= 0 → 无写锁等待，直接返回
     │
     └─ < 0  → 调用 rUnlockSlow
                 │
                 ▼
               readerWait--
                 │
                 ├─ != 0 → 仍有读者，不唤醒写锁
                 │
                 └─ == 0 → 最后一个读者离开，唤醒 writerSem
```

### 6.4 Lock / Unlock

```text
写加锁 Lock:
  w.Lock()
     │
     ▼
  readerCount -= rwmutexMaxReaders
     │
     ▼
  记录当前读者数到 readerWait
     │
     ├─ 无读者 → 直接获得写锁
     │
     └─ 有读者 → 阻塞在 writerSem，等待 readerWait == 0

写解锁 Unlock:
  readerCount += rwmutexMaxReaders
     │
     ▼
  唤醒所有等待的读者 readerSem
     │
     ▼
  w.Unlock()
```

### 6.5 写锁为什么不会饿死？

写锁不会因持续读请求而饿死，因为：

1. 写锁获取时，会把 `readerCount` 减去 `rwmutexMaxReaders`；
2. 此后新来的读锁发现 `readerCount < 0`，会直接阻塞；
3. `readerWait` 是写锁到来时活跃读者的快照；
4. 只有这些旧读者释放后，写锁才会被唤醒。

所以写锁只等待“已经拿到读锁的读者”，不会无限等待新读者。

### 6.6 读写锁关系图

```text
              RWMutex
          ┌──────────────┐
          │  w Mutex     │  保护写-写互斥
          │  readerCount │  记录读者数 / 写者标记
          │  readerWait  │  写锁等待的读者快照
          │  writerSem   │  写锁等待读者完成
          │  readerSem   │  读锁等待写锁完成
          └──────────────┘

读者 ──► readerCount++ ──► <0 ? ──► 阻塞 readerSem
写者 ──► readerCount-2^30 ──► 负数阻止新读者
读者退出 ──► readerWait-- ──► ==0 ? ──► 唤醒 writerSem
写者退出 ──► readerCount+2^30 ──► 唤醒所有 readerSem
```

---

## 7. 死锁检测

### 7.1 触发时机

Go 的死锁检测主要在调度器状态变化时触发，例如：

- M 退出；
- M 变为空闲；
- `nmidlelocked` 变化；
- `sysmon` 监控循环。

### 7.2 判断逻辑

核心公式大致为：

```text
run = mcount - nmidle - nmidlelocked - nmsys
```

- 若 `run > 0`，说明仍有 M 在运行，通常不认为死锁；
- 若 `run == 0`，再检查所有 goroutine；
- 若所有用户 goroutine 都处于 `_Gwaiting`、`_Gpreempted` 等阻塞状态；
- 且没有待触发的定时器；
- 则抛出：

```text
fatal error: all goroutines are asleep - deadlock!
```

### 7.3 局限性

Go 的死锁检测并不完善：

- 只要还有一个 M 能继续运行，就不会报死锁；
- 有定时器、网络轮询、信号处理、cgo 线程时，可能掩盖死锁；
- 它检测的是“所有 goroutine 都睡着且无人能唤醒”，不是通用循环等待检测。

---

## 8. 常见错误修正

| 原笔记说法 | 修正 |
| ------ |------ |
| 饥饿阈值 10ms | 应为 **1ms**，即 `starvationThresholdNs = 1e6` 纳秒 |
| lock 前缀锁总线 | 现代 x86 通常锁缓存行，不一定是全局总线 |
| 单核不自旋只写 GOMAXPROCS<1 | 应包含 `GOMAXPROCS <= 1` 和“当前持锁 goroutine 不自旋”等条件 |
| 自旋锁可递归 | 自旋锁和 `sync.Mutex` 都不可重入 |
| 结构体锁随便拷贝 | 锁使用后禁止值拷贝，应传指针 |
| 写锁会饿死 | `readerWait` 是快照，新读者会被阻塞，写锁不会饿死 |
| 死锁检测万能 | 只能检测“全部 goroutine 睡眠且无唤醒源”的场景 |

---

## 9. 锁类型对比

| 类型 | 是否阻塞 | 是否自旋 | 适用场景 |
| ------ |------ |------ |------ |
| `sync.Mutex` | 是 | 短暂自旋 | 通用互斥 |
| 自旋锁 | 否 | 持续自旋 | 临界区极短、多核 |
| `RWMutex` | 是 | 内部复用 Mutex | 读多写少 |
| `atomic` | 否 | 无 | 计数器、标志位 |
| `sync.Once` | 是 | 否 | 单次初始化 |

---

## 10. 面试高频问题

### 1. `sync.Mutex` 的 state 字段如何编码？

低 3 位分别是 `locked`、`woken`、`starving`，高 29 位记录等待者数量。

### 2. 正常模式和饥饿模式有什么区别？

正常模式偏向吞吐，唤醒者需要和新来者竞争；饥饿模式偏向公平，锁直接交给队列头部等待者。

### 3. 饥饿模式何时进入和退出？

等待超过 1ms 进入饥饿模式；当队列只剩最后一个等待者，或等待时间不足 1ms 时退出。

### 4. 为什么 Mutex 不能拷贝？

拷贝会复制 `state` 和 `sema`，导致两个实例共享或分裂锁状态，引发死锁或数据竞争。

### 5. `sync.RWMutex` 如何避免写锁饥饿？

写锁通过 `readerWait` 快照只等待当前活跃读者，新读者会被 `readerCount < 0` 阻塞。

### 6. `readerCount` 为什么能同时表示读者数和写者状态？

正数表示读者数量；写锁到来时减去 `1<<30`，使其变为负数，从而阻止新读者进入。

### 7. Go 的 sema 和 Linux futex 有什么关系？

Go 的 `sema` 提供 sleep/wakeup 原语，Linux 下底层依赖 `futex` 实现。

### 8. 自旋一定比休眠好吗？

不一定。临界区极短、竞争不激烈时自旋更好；临界区长或竞争激烈时，自旋会浪费 CPU，应休眠。

### 9. Go 死锁检测一定能发现死锁吗？

不能。只要有 goroutine 或 M 仍可运行，或者存在定时器、网络轮询、信号处理，就可能不触发死锁报错。

### 10. `sync.Mutex` 是可重入锁吗？

不是。同一 goroutine 重复 `Lock` 会导致死锁。

---

## 11. 实践建议

- 优先使用 `sync.Mutex`，不要过早优化成自旋锁。
- 读多写少再考虑 `RWMutex`。
- 简单计数器优先用 `atomic`。
- 锁粒度尽量小，避免在持锁时执行 IO、网络调用。
- 使用 `go test -race` 检测数据竞争。
- 结构体中持有锁时，尽量通过指针传递，避免值拷贝。
