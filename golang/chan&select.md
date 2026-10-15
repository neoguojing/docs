
# Go Channel & Select 原理

## 1. 核心模型

Channel 底层核心结构是 `hchan`：

```go
type hchan struct {
    qcount   uint          // 当前 buffer 中元素数量
    dataqsiz uint          // buffer 容量
    buf      unsafe.Pointer
    elemsize uint16
    closed   uint32

    elemtype *_type
    sendx    uint          // buffer 写位置
    recvx    uint          // buffer 读位置

    recvq    waitq         // 等待接收的 G
    sendq    waitq         // 等待发送的 G
    lock     mutex
}
```

可以把 Channel 理解成：

```text
                         hchan
                           │
              ┌────────────┼────────────┐
              ↓            ↓            ↓
            recvq         buf          sendq
              │            │             │
          waiting G    环形队列      waiting G
```

### 核心字段

| 字段         | 含义              |
| ---------- | --------------- |
| `qcount`   | buffer 当前元素数    |
| `dataqsiz` | buffer 容量       |
| `buf`      | 环形 buffer       |
| `sendx`    | 下一次写入位置         |
| `recvx`    | 下一次读取位置         |
| `recvq`    | 等待接收的 goroutine |
| `sendq`    | 等待发送的 goroutine |
| `closed`   | 是否关闭            |
| `lock`     | 保护 channel 状态   |

> `sendq/recvq` 中等待的是 `sudog`，而 `sudog` 关联具体的 goroutine `G`。

---

# 2. Channel 创建：makechan

```go
ch := make(chan int, 10)
```

`makechan` 根据 channel 类型和容量分配内存。

### 2.1 无缓冲

```go
make(chan int)
```

```text
hchan
 └── buf = nil
```

只需要分配 `hchan`。

### 2.2 有缓冲

```go
make(chan int, 10)
```

如果元素类型**不包含指针**：

```text
┌──────────────┬────────────────────┐
│    hchan     │      buffer        │
└──────────────┴────────────────────┘
```

一次性分配连续内存。

如果元素包含指针，则 `hchan` 和 buffer 分开分配，便于 GC 正确处理指针。

### 其他

- channel 元素大小不能超过 `65535`（约 64 KiB）。
- `hchan` 保存 channel 的运行时状态。
- `makechan` 本身不创建 goroutine。

---

# 3. Send / Receive：Channel 的核心流程

这是理解 channel 最重要的一部分。

## 3.1 Send

```text
                         send
                           │
                       c == nil?
                       /      \
                     Yes      No
                      │        │
                   gopark      ↓
                          closed?
                          /    \
                        Yes    No
                         │      │
                       panic    ↓
                           recvq 有等待者？
                            /        \
                          Yes        No
                           │          │
                       直接交接       ↓
                                  buf 有空间？
                                   /     \
                                 Yes     No
                                  │       │
                              写入 buf   加入 sendq
                                          ↓
                                       gopark
```

记忆：

> **send：recvq → buf → sendq**

具体过程：

1. `c == nil`：普通 send 永久阻塞。
2. channel 已关闭：`panic`。
3. `recvq` 有等待者：直接把数据交给 receiver。
4. buffer 未满：写入 buffer。
5. buffer 满：创建 `sudog`，加入 `sendq`，`gopark`。

---

## 3.2 Receive

```text
                         recv
                           │
                       c == nil?
                       /      \
                     Yes      No
                      │        │
                   gopark      ↓
                           closed && empty?
                           /          \
                         Yes          No
                          │            │
                     zero value       ↓
                                  sendq 有等待者？
                                   /       \
                                 Yes       No
                                  │         │
                              直接交接      ↓
                                       buf 有数据？
                                        /      \
                                      Yes      No
                                       │        │
                                   读取 buf    加入 recvq
                                                 ↓
                                              gopark
```

记忆：

> **recv：sendq → buf → recvq**

具体过程：

1. `c == nil`：普通 receive 永久阻塞。
2. channel 已关闭且 buffer 为空：立即返回零值。
3. `sendq` 有等待者：直接从 sender 获取数据。
4. buffer 有数据：从 buffer 读取。
5. buffer 为空：加入 `recvq`，`gopark`。

---

## 3.3 直接交接 vs Buffer

### 无缓冲

```text
sender G
   │
   │ data
   ↓
receiver G
```

sender 和 receiver 直接完成数据交接，不经过 buffer。

### 有缓冲

```text
sender G → buffer → receiver G
```

buffer 可以降低生产者和消费者必须同时到达的要求。

因此：

> **buffer 的作用主要是解耦生产速度和消费速度，而不是消除阻塞。**

---

# 4. Sudog、gopark、goready

## 4.1 sudog

`sudog` 可以理解为：

> **G 与 channel 等同步对象之间的等待关系。**

```text
G
│
└── sudog
      │
      ├── g
      ├── elem
      ├── c
      └── ...
```

channel：

```text
sendq / recvq
       │
       ↓
     sudog
       │
       ↓
       G
```

当 G 因 channel 阻塞时：

```text
acquireSudog()
      ↓
加入 sendq / recvq
      ↓
gopark()
```

`acquireSudog` 通常优先从：

```text
P.sudogcache
    ↓
sched.sudogcache
    ↓
new sudog
```

获取，减少内存分配。

---

## 4.2 gopark / goready

```text
G running
    │
    ↓
 gopark
    │
    ↓
G waiting
```

其他 goroutine 进行匹配：

```text
send / recv / close
       ↓
   goready(G)
       ↓
G runnable
```

注意：

> `goready` 是让 G 进入 runnable 状态，并不是立即在当前线程执行。

---

# 5. Close Channel

核心函数：

```text
closechan
```

流程：

```text
                     close(ch)
                         │
                    closed = 1
                         │
             ┌───────────┴───────────┐
             ↓                       ↓
           recvq                    sendq
             │                       │
        唤醒所有 G               唤醒所有 G
             │                       │
             └───────────┬───────────┘
                         ↓
                      glist
                         ↓
                    goready()
```

### 关闭后的行为

| 操作                    | 结果              |
| --------------------- | --------------- |
| `close(nil)`          | panic           |
| `close(closed)`       | panic           |
| send → closed channel | panic           |
| recv → closed + 有数据   | 正常读取            |
| recv → closed + 空     | 零值 + `ok=false` |

例如：

```go
v, ok := <-ch
```

当 channel 已关闭且没有数据：

```text
v  = zero value
ok = false
```

所以：

```go
for v := range ch {
    ...
}
```

本质上就是持续 receive，直到：

```text
closed && buffer empty
```

---

# 6. Select：同时等待多个 Channel

典型代码：

```go
select {
case v := <-ch1:
    ...
case ch2 <- x:
    ...
default:
    ...
}
```

每个 case 对应一个 `scase`：

```go
type scase struct {
    c    *hchan
    elem unsafe.Pointer
}
```

核心函数：

```text
selectgo
```

---

## 6.1 Select 的两个顺序

selectgo 会维护两个重要顺序：

```text
pollorder
    ↓
随机顺序
    ↓
决定 case 检查顺序


lockorder
    ↓
按 hchan 地址排序
    ↓
决定加锁顺序
```

记忆：

> **pollorder 管选择，lockorder 管锁。**

### 为什么 pollorder 随机？

如果：

```text
h1 ready
ch2 readyc
```

不能固定永远：

```text
ch1 → ch2
```

否则可能长期偏向第一个 case。

因此 runtime 会随机生成 `pollorder`，多个 case 同时 ready 时随机选择。

---

### 为什么 lockorder 排序？

假设两个 G：

```text
G1: lock(ch1) → lock(ch2)
G2: lock(ch2) → lock(ch1)
```

可能形成死锁。

因此 runtime 按：

```text
hchan 地址从小到大
```

形成统一的 `lockorder`。

```text
sellock
   ↓
正序加锁

selunlock
   ↓
反序解锁
```

---

# 7. Selectgo 完整流程

这是 select 最值得记的一张图：

```text
                         selectgo
                            │
              ┌─────────────┴─────────────┐
              ↓                           ↓
         pollorder                    lockorder
          随机生成                    地址排序
              │                           │
              └─────────────┬─────────────┘
                            ↓
                          加锁
                            ↓
                    遍历 pollorder
                            ↓
                     是否有 ready case?
                       /             \
                     Yes             No
                      │                │
                      ↓            有 default?
                  执行 case        /        \
                      │          Yes        No
                      │           │          │
                      │           ↓          ↓
                      │         解锁       创建 sudog
                      │           │          ↓
                      │           │       加入各队列
                      │           │          ↓
                      │           │        gopark
                      │           │          ↓
                      │           │        被唤醒
                      │           │          ↓
                      │           │       找 winner
                      │           │          ↓
                      │           │       清理其他
                      │           │        sudog
                      └───────────┴──────────┐
                                             ↓
                                            解锁
                                             ↓
                                           return
```

---

# 8. Select 阻塞与唤醒

假设：

```go
select {
case <-ch1:
case <-ch2:
case ch3 <- x:
}
```

所有 case 都没有 ready，也没有 `default`。

runtime 会为每个 case 创建一个 `sudog`：

```text
                     G
                     │
                  waiting
                     │
        ┌────────────┼────────────┐
        ↓            ↓            ↓
     sudog1       sudog2       sudog3
        │            │            │
       ch1          ch2          ch3
      recvq        recvq        sendq
```

然后：

```text
gopark(G)
```

### 假设 ch2 先 ready

```text
ch2
 ↓
sudog2
 ↓
sudog2.g
 ↓
goready(G)
```

唤醒后：

```text
gp.param
    ↓
sudog2
    ↓
确定 case 2 获胜
```

然后必须：

```text
sudog1 → 从 ch1 队列删除
sudog3 → 从 ch3 队列删除
```

否则这些残留 sudog 后续可能错误地再次唤醒同一个 G。

最终：

```text
找到 winner
    ↓
清理其他 sudog
    ↓
解锁
    ↓
返回 case index
```

---

# 9. Select 中的特殊情况

## nil channel

```go
select {
case <-nil:
case <-ch:
}
```

nil channel：

```text
永远不 ready
```

因此可以把 nil channel 当作：

> **动态禁用某个 select case 的手段。**

---

## default

```go
select {
case v := <-ch:
    use(v)
default:
    return
}
```

所有 case 都没有 ready：

```text
default
   ↓
立即返回
```

所以：

> **default 使 select 具有非阻塞语义。**

---

## closed channel

```text
case <-closedCh
```

立即 ready。

如果：

```go
v, ok := <-closedCh
```

则：

```text
ok == false
```

而：

```go
case closedCh <- x
```

会 panic。

---

## 重复 channel

合法：

```go
select {
case <-ch:
case <-ch:
case <-other:
}
```

如果 `ch` 和 `other` 都 ready，那么 `ch` 有两个候选 case，`other` 一个，因此从 case 数量上可以形成约：

```text
P(ch)     ≈ 2/3
P(other) ≈ 1/3
```

这可以作为一种简单的概率权重技巧，但不应该把 select 当作精确随机数工具。

---

# 10. Channel 与 Select 的统一理解

最终可以把整个 runtime 机制压缩成下面这张图：

```text
                    Channel
                       │
          ┌────────────┼────────────┐
          ↓            ↓            ↓
        recvq         buf          sendq
          │            │            │
          ↓            ↓            ↓
       receiver       data        sender
          │                           │
          └───────────┬───────────────┘
                      ↓
                 send / recv
                      │
               无法立即完成
                      ↓
                    sudog
                      ↓
                  gopark(G)
                      │
             其他 G / close
                      ↓
                  goready(G)
```

select 只是把：

```text
一个 channel 的等待
```

扩展成：

```text
多个 channel 的等待
```

即：

```text
             select G
          /      |      \
       sudog1  sudog2  sudog3
         │       │       │
        ch1     ch2     ch3
```

**这是理解整个 Channel + Select 源码最核心的模型。**

---

# 11. 面试问题按主题聚合

## A. Channel 基础

1. 无缓冲 channel 和有缓冲 channel 有什么区别？
2. channel 底层 `hchan` 有哪些核心字段？
3. `sendx` 和 `recvx` 的作用是什么？
4. 为什么 channel 使用环形 buffer？
5. channel buffer 为什么不是越大越好？
6. channel 元素为什么存在大小限制？

---

## B. Send / Receive

1. `ch <- x` 底层执行流程是什么？
2. send 为什么优先检查 `recvq`？
3. receive 为什么需要检查 `sendq`？
4. 无缓冲 channel 如何完成数据传递？
5. buffer 满时 send 会发生什么？
6. buffer 空时 receive 会发生什么？
7. `typedmemmove`、`typedmemclr` 大致作用是什么？
8. channel 阻塞时阻塞的是 G 还是 OS Thread？

**最重要的两句话：**

```text
send：recvq → buf → sendq
recv：sendq → buf → recvq
```

---

## C. Close

1. close channel 的底层流程？
2. close 后还能 receive 吗？
3. close 后为什么 send 会 panic？
4. 为什么 close 时需要同时处理 `recvq` 和 `sendq`？
5. `close(nil)` 和 `close(closed)` 为什么 panic？
6. `v, ok := <-ch` 中 `ok=false` 代表什么？

---

## D. Sudog / Scheduler

1. sudog 是什么？
2. G 为什么不能直接挂到 channel 队列？
3. `gopark` 和 `goready` 分别做什么？
4. acquireSudog 为什么有 cache？
5. channel 如何唤醒阻塞的 goroutine？

核心模型：

```text
G ↔ sudog ↔ channel
```

---

## E. Select

1. selectgo 的整体流程？
2. `pollorder` 和 `lockorder` 有什么区别？
3. 为什么 pollorder 要随机？
4. 为什么 lockorder 按 hchan 地址排序？
5. 为什么加锁正序、解锁反序？
6. select 没有 ready case 时怎么办？
7. default 为什么可以实现非阻塞 select？
8. select 阻塞时，一个 G 如何等待多个 channel？
9. select 唤醒后如何知道哪个 case 获胜？
10. 为什么唤醒后需要清理其他 sudog？
11. select 中可以出现重复 channel 吗？
12. nil channel 在 select 中有什么作用？

---

# 12. 面试最终速记版

如果只剩 **2 分钟复习**，记下面这张：

```text
Channel
│
├── hchan
│   ├── buf
│   ├── sendq
│   ├── recvq
│   ├── sendx / recvx
│   └── lock
│
├── send
│   └── recvq → buf → sendq
│
├── recv
│   └── sendq → buf → recvq
│
├── close
│   └── closed=1 → 唤醒 recvq + sendq
│
└── 阻塞
    └── sudog → gopark → goready

Select
│
├── pollorder
│   └── 随机 → 决定检查/选择顺序
│
├── lockorder
│   └── hchan地址排序 → 防止锁死
│
├── ready
│   └── 直接执行 case
│
├── default
│   └── 无 ready → 立即返回
│
└── 都不 ready
    └── 多个 sudog
        ↓
      挂到多个 channel
        ↓
      gopark
        ↓
      某 channel ready
        ↓
      goready
        ↓
      找 winner
        ↓
      清理其他 sudog
```

**真正需要掌握的不是 50 多个 runtime 细节，而是 5 条主线：**

> **① `hchan` 管状态；② `sendq/recvq` 管等待者；③ `sudog` 把 G 和 channel 关联起来；④ `gopark/goready` 实现阻塞与唤醒；⑤ `selectgo` 用 `pollorder + lockorder + sudog` 同时等待多个 channel。**
