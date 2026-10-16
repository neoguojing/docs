对，你这个要求是对的。上一版的问题不是“内容少”，而是**把 Runtime 源码按函数调用顺序罗列出来了**，读者会知道“调用了什么”，但不知道“为什么这样设计、数据是什么、对象怎么从白变黑、bit 到底记录什么”。

你这份文档更适合改成下面这种**“问题 → 核心概念 → 数据结构 → 算法 → 源码实现 → 完整流程 → 例子 → 面试重点”**的结构。

另外我核对了当前 Go runtime 源码：Go GC 当前仍然是**并发、精确、非分代、非移动的 mark-sweep GC，并使用 write barrier**；官方 `mgc.go` 对整个 GC cycle 的描述也明确把 Sweep Termination、Mark、Mark Termination、Sweep 分开。[GitHub](https://github.com/golang/go/blob/master/src/runtime/mgc.go?utm_source=chatgpt.com)

---

# Go GC 原理与 Runtime 源码结构化学习文档

## 1. 先建立一张总图：Go GC 到底在解决什么问题？

GC 的核心问题只有一句话：

> **找到所有仍然“可达”的对象，保留下来；不可达对象最终变成可重新分配的空间。**

Go 使用的是 **Mark-Sweep（标记-清扫）**：

```text
                 Heap
                  │
                  ▼
        ┌──────────────────┐
        │ 找到 GC Roots     │
        └────────┬─────────┘
                 │
                 ▼
        ┌──────────────────┐
        │ 从 Root 遍历对象  │
        │      Graph       │
        └────────┬─────────┘
                 │
          可达对象全部标记
                 │
                 ▼
        ┌──────────────────┐
        │      Mark        │
        │  标记存活对象     │
        └────────┬─────────┘
                 │
                 ▼
        ┌──────────────────┐
        │      Sweep       │
        │  回收未标记对象   │
        └────────┬─────────┘
                 │
                 ▼
           空间重新利用
```

官方 GC Guide 也是以 **root → scanning → reachability → mark → sweep** 作为基本模型。[Go语言](https://go.dev/doc/gc-guide?utm_source=chatgpt.com)

---

# 2. 第一个核心问题：哪些对象需要标记？

这是理解 Go GC 最重要的问题。

不要一上来就看 `gcw`、`markBits`、`heapBits`。

先问：

> **GC 从哪里开始？**

答案是：

## 2.1 GC Root

Root 不是普通堆对象，而是 GC 已知的“入口”。

可以理解成：

```text
             GC Roots
                │
      ┌─────────┼─────────┐
      │         │         │
     Stack     Data      BSS
      │
      │
    Global
      │
      ▼
    Heap
```

你原文中的：

```text
mcache
data
bss
mheap.markArenas
allg
finblock
```

实际上是在描述 Runtime 需要处理的不同 Root/特殊 Root。

当前 `mgcmark.go` 还明确包含 finalizer、free G stack、cleanup 等固定 Root 类型。[Go语言](https://go.dev/src/runtime/mgcmark.go?utm_source=chatgpt.com)

---

# 3. 第二个核心问题：GC 到底要标记什么？

答案不是：

> “把所有对象都标记。”

而是：

> **从 Root 出发，沿着指针能够访问到的对象，全部标记为存活。**

例如：

```text
Root
 │
 ▼
 A
 ├────► B
 │      │
 │      └────► D
 │
 └────► C

 E ─────► F
```

假设 Root 只能访问：

```text
Root → A → B → C → D
```

那么：

```text
A B C D   = 存活
E F       = 不可达
```

因此：

```text
A B C D → mark
E F     → 不 mark
```

下一阶段 Sweep：

```text
E F → 回收
```

所以：

> **Mark 的本质不是“标记所有对象”，而是“遍历可达对象图”。**

---

# 4. 第三个核心问题：对象为什么能够被正确扫描？

这里是 Go GC 很关键的地方。

GC 必须知道：

> 一个对象里面，哪些字段是指针？

例如：

```go
type User struct {
    Name string
    Age  int
    Next *User
}
```

内存可能类似：

```text
User object

+------------------+
| Name             | 不是直接指针
+------------------+
| Age              | 非指针
+------------------+
| Next             | ← 指针
+------------------+
```

GC 不能简单地：

```text
每 8 bytes 都当成指针
```

否则：

```text
Age = 0x12345678
```

可能被误认为是一个地址。

因此 Go GC 是：

> **Precise GC（精确 GC）**

---

# 5. 编译器告诉 GC：哪些位置是指针

这是整个扫描体系的第一层。

编译器生成类型信息：

```text
runtime._type
    │
    ├── ptrdata
    │
    └── gcdata
```

其中：

```text
ptrdata
    ↓
对象前多少字节可能包含指针

gcdata
    ↓
bitmap / pointer mask
    ↓
具体哪些 word 是指针
```

所以：

```text
Go 源代码
   ↓
编译器分析类型
   ↓
_type
   ↓
ptrdata + gcdata
   ↓
GC 知道对象哪些位置是指针
```

这就是你原文：

> 编译器为结构体填充 `_type` 对象，记录指针信息到 `ptrdata` 和 `gcdata`

这一部分真正应该放在**“GC 如何知道哪里有指针”**这个章节，而不是放在函数流水账里面。

---

# 6. heapBits 和 gcdata 不要混在一起

这是原文最容易让人混淆的地方之一。

实际上可以这样理解：

```text
              “哪里存在指针？”
                     │
          ┌──────────┴──────────┐
          │                     │
       类型信息              Heap metadata
          │                     │
       gcdata                heapBits
          │                     │
   描述类型中的指针布局       描述堆对象布局
```

### `gcdata`

更偏向：

> **类型层面的 pointer bitmap**

例如：

```text
对象有 4 个 word

word:
  0   1   2   3

gcdata:
  1   0   1   0

表示：

word 0 → pointer
word 1 → scalar
word 2 → pointer
word 3 → scalar
```

---

### `heapBits`

更偏向：

> **堆中实际对象布局的 pointer bitmap**

也就是：

```text
Heap
│
├── object A
├── object B
├── object C
│
└── bitmap
      ↓
告诉 GC：
这个 word 是 pointer
那个 word 不是 pointer
```

因此：

> **`gcmarkBits` 解决“对象活不活”；`heapBits` 解决“对象里面哪里是指针”。**

这是非常重要的一组区分。

---

# 7. 第四个核心问题：对象如何从“白”变成“黑”？

现在才进入三色标记。

## 7.1 三种颜色

```text
白色
 ↓
还没有发现
 ↓
可能是垃圾


灰色
 ↓
已经发现
 ↓
但是内部指针还没有全部扫描


黑色
 ↓
已经发现
 ↓
内部指针也已经扫描完成
```

例如：

```text
Root
 │
 ▼
 A
 │
 ▼
 B
```

开始：

```text
A = 白
B = 白
```

扫描 Root：

```text
A = 灰
```

然后扫描 A：

```text
A = 黑
B = 灰
```

然后扫描 B：

```text
B = 黑
```

最终：

```text
A = 黑
B = 黑
```

---

# 8. 但是 Runtime 里真的存储“三种颜色”吗？

**不是简单地存储 2 bit：**

```text
00 = white
01 = grey
10 = black
```

这是学习三色标记时非常容易产生的误解。

Go runtime 核心上主要通过 **mark bits + work queue / scanning state** 表达标记和扫描进度。

也就是说：

```text
mark bit = 1
```

主要表达：

> **这个对象已经被标记过。**

而不是简单等价：

```text
mark bit = 黑色
```

因为一个对象：

```text
刚被发现
```

和：

```text
已经完成扫描
```

在算法语义上不同。

所以应该这样记：

```text
        对象状态

       ┌─────────────┐
       │ 未发现       │
       │ white       │
       └──────┬──────┘
              │ greyobject
              ▼
       ┌─────────────┐
       │ 已标记       │
       │ gray        │
       └──────┬──────┘
              │ scanobject
              ▼
       ┌─────────────┐
       │ 已扫描       │
       │ black       │
       └─────────────┘
```

**mark bit 主要负责“是否已经标记”；work queue 负责“还有没有待扫描工作”。**

---

# 9. `gcmarkBits` 到底是什么？

这是整个 GC 最重要的数据结构之一。

`mspan` 中有：

```go
type mspan struct {
    ...
    allocBits
    gcmarkBits
    ...
}
```

可以简化理解成：

```text
mspan

objects:

obj0 obj1 obj2 obj3 obj4 obj5
 │    │    │    │    │    │
 ▼    ▼    ▼    ▼    ▼    ▼

gcmarkBits:

 0    1    1    0    1    0
```

含义：

```text
obj0 → 未标记
obj1 → 已标记
obj2 → 已标记
obj3 → 未标记
obj4 → 已标记
obj5 → 未标记
```

所以：

> **每一个 bit 对应 span 中的一个 object。**

当前 Go runtime 的实现仍然明确使用 `gcmarkBits`；`markBitsForIndex` 根据对象在 span 中的 `objIndex` 找到对应 bit。[Go语言](https://go.dev/src/runtime/mgcmark_nogreenteagc.go?utm_source=chatgpt.com)

---

# 10. `markBits` 结构体到底做什么？

原文：

```go
type markBits struct {
    bytep *uint8
    mask  uint8
    index uintptr
}
```

不要把它理解成：

> “保存一个对象的 mark 状态。”

更准确地说：

> **它是一个操作 bitmap 中某一个 bit 的游标/描述符。**

例如：

```text
bitmap

byte0
┌────────────────┐
│ 1 0 0 1 0 1 0 0│
└────────────────┘
  ↑
  mask
```

`bytep`：

```text
指向这个 byte
```

`mask`：

```text
指出当前 object 对应哪个 bit
```

例如：

```text
bytep → 10100100
           ↑
          mask
```

于是：

```go
*bytep & mask
```

就可以判断：

```text
bit 是否已经设置
```

而：

```go
*bytep |= mask
```

就可以：

```text
设置 mark bit
```

---

# 11. `greyobject()` 到底干了什么？

这是理解 Mark 的关键函数。

可以把它简化成：

```text
对象地址 p
   │
   ▼
findObject()
   │
   ▼
找到：
mspan
objIndex
   │
   ▼
markBitsForIndex()
   │
   ▼
找到 gcmarkBits 中对应 bit
   │
   ▼
bit == 1 ?
 ┌──────┴──────┐
 是            否
 │             │
返回       设置 bit = 1
               │
               ▼
          放入 gcw
               │
               ▼
          等待 scanobject
```

所以 `greyobject()` 的核心不是：

> “把对象颜色设置成 gray”

而是：

> **标记对象 + 把对象加入待扫描工作队列。**

这两件事要同时理解。

---

# 12. 一个完整例子：A → B → C

假设：

```text
Root
 │
 ▼
 A
 │
 ▼
 B
 │
 ▼
 C
```

初始：

```text
gcmarkBits

A = 0
B = 0
C = 0
```

---

## 第一步：扫描 Root

发现 A：

```text
greyobject(A)
```

于是：

```text
A.mark = 1
```

同时：

```text
gcw.push(A)
```

现在：

```text
A = 已标记
B = 未标记
C = 未标记

gcw:
[A]
```

---

## 第二步：scanobject(A)

GC 从 `gcw` 获取 A：

```text
gcw.pop()
     ↓
     A
```

扫描 A：

```text
A
│
└── B
```

发现 B：

```text
greyobject(B)
```

于是：

```text
B.mark = 1

gcw:
[B]
```

A 扫描完成：

```text
A = black
B = gray
C = white
```

---

## 第三步：scanobject(B)

发现：

```text
B → C
```

于是：

```text
C.mark = 1
gcw.push(C)
```

最终：

```text
A = black
B = black
C = gray
```

---

## 第四步

继续：

```text
scanobject(C)
```

如果 C 没有指针：

```text
gcw = empty
```

那么：

```text
A B C = black
```

Mark 完成。

---

# 13. 所以 Mark 阶段实际上是两个动作

这是你原文需要重新组织的核心。

```text
             Mark
              │
       ┌──────┴──────┐
       │             │
       ▼             ▼
    Mark Object   Scan Object
       │             │
       │             ▼
       │        找到内部 pointer
       │             │
       │             ▼
       └──────► greyobject()
                     │
                     ▼
                  gcw
                     │
                     ▼
                scanobject
```

可以记成：

> **发现对象 → 标记 → 入队 → 扫描 → 发现新对象 → 再标记 → 再入队。**

这才是 `gcDrain + greyobject + scanobject + gcw` 之间的真正关系。

---

# 14. `gcw` 是什么？

`gcw`：

```go
type gcWork struct {
    wbuf1 *workbuf
    wbuf2 *workbuf
    bytesMarked uint64
    scanWork int64
    flushedWork bool
}
```

它本质上就是：

> **Mark 阶段的“待处理对象队列 + 统计信息”。**

可以画成：

```text
                 GC Worker
                    │
                    ▼
                 gcDrain
                    │
                    ▼
              gcw.tryGet()
                    │
                    ▼
              ┌─────────────┐
              │  Work Queue │
              └──────┬──────┘
                     │
                     ▼
               scanobject()
                     │
             ┌───────┴───────┐
             │               │
         找到新对象       没有新对象
             │
             ▼
        greyobject()
             │
             ▼
          gcw.put()
```

---

# 15. 为什么还要有 `work.full` / `work.empty`？

因为 GC Worker 不应该每次：

```text
加锁 → 全局队列
```

否则多个 GC worker 会竞争。

因此使用：

```text
P 本地 gcw
       │
       ▼
    workbuf
       │
       │ 满
       ▼
 work.full
```

大致结构：

```text
GC Worker
   │
   ▼
P.local gcw
   │
   ├── push/pop
   │
   ▼
workbuf
   │
   └── 满了
         ↓
     global full
```

所以：

> **本地 work buffer 降低并发竞争，全局 work queue 用于 worker 之间交换任务。**

---

# 16. `gcDrain()` 到底是什么？

`gcDrain()` 是：

> **一个 GC worker 真正“干活”的地方。**

核心循环可以理解成：

```text
gcDrain
   │
   ▼
获取 root work
   │
   ▼
获取 gcw 中对象
   │
   ▼
scanobject
   │
   ▼
发现新对象
   │
   ▼
greyobject
   │
   ▼
继续获取 work
   │
   └──────────────┐
                  │
                  ▼
          work 是否为空？
             │       │
            否       是
             │       │
             └───────┘
                 │
                 ▼
             Mark Done
```

所以你原来的：

> `gcDrain：执行根和对象扫描`

太粗了。

应该记：

> **`gcDrain` = Mark worker 的工作循环；负责不断从 work queue / root jobs 获取工作，并通过 `scanobject` 扫描对象，通过 `greyobject` 发现新对象。**

---

# 17. Root 是怎么扫描的？

Root 本身不是只有一个对象。

Runtime 会把 Root 拆成很多 job。

例如：

```text
markroot
   │
   ├── data
   ├── bss
   ├── mspan
   ├── stack
   ├── finalizer
   └── 其他 runtime root
```

因此：

```text
gcMarkRootPrepare()
        │
        ▼
统计 root 数量
        │
        ▼
生成 markroot jobs
        │
        ▼
gcDrain()
        │
        ▼
markroot(job)
```

---

# 18. Stack 为什么特别重要？

因为：

```go
func foo() {
    p := &Object{}
}
```

`p` 可能就在 Goroutine 的栈上。

因此：

```text
Stack
 │
 ├── local variable
 ├── function argument
 ├── return value
 └── defer / panic 等 runtime 信息
```

都可能包含指向 Heap 的指针。

所以 GC 必须扫描：

```text
G stack
   ↓
stack frame
   ↓
local / argument / return
   ↓
pointer
   ↓
heap object
```

---

# 19. `scanstack()` 做什么？

核心逻辑：

```text
scanstack(g)
    │
    ▼
找到 g 的 stack
    │
    ▼
扫描 stack frame
    │
    ├── local variable
    ├── argument
    ├── return information
    ├── defer
    └── panic
          │
          ▼
      找到 pointer
          │
          ▼
      heap object?
          │
          ▼
      greyobject()
```

因此：

> **Stack 是 Root 的来源之一，scanstack 是 Root → Heap 的重要入口。**

---

# 20. 为什么需要 Write Barrier？

现在进入 Go GC 最难的一部分。

假设：

```text
GC 正在扫描：

A → B
```

此时：

```text
A 已经黑
B 还白
```

程序突然执行：

```go
A.ptr = C
```

其中 C 是白对象。

那么可能出现：

```text
A(black)
 │
 └──► C(white)
```

如果 GC 已经认为 A 扫描完了：

```text
A 不会再次扫描
```

那么 C 就可能被误认为垃圾。

这就是并发 GC 的核心问题。

---

# 21. 三色不变式解决什么问题？

核心目标：

> **避免一个已经扫描完成的黑对象，突然指向一个 GC 完全不知道的白对象。**

于是需要 barrier。

---

# 22. 插入屏障

插入：

```text
A → C
```

如果 C 是白色：

```text
C → 灰色
```

于是：

```text
A black
 │
 └── C gray
```

GC 后面会扫描 C。

但是 Go 的实际实现不是简单地依赖经典 Dijkstra 插入屏障，因为 Go 还需要处理栈扫描等问题。

---

# 23. 删除屏障

另一种危险：

```text
A(gray) → B(white)
```

程序把：

```text
A → B
```

删除。

如果这是 B 最后一个引用：

```text
B
```

可能变成不可达。

但 GC 可能还没有扫描 A。

于是需要保护 B。

删除屏障可以：

```text
删除引用时：

B white → gray
```

这样：

```text
B 不会丢失
```

---

# 24. Go 为什么采用 Hybrid Write Barrier？

这是必须重点记忆的。

Go 当前 runtime 使用 write barrier，并且采用混合屏障思路。官方 runtime 对 GC 的描述明确指出 GC 是 concurrent mark-and-sweep，并使用 write barrier。[GitHub](https://github.com/golang/go/blob/master/src/runtime/mgc.go?utm_source=chatgpt.com)

核心思想：

```text
GC 开始
   │
   ▼
扫描栈
   │
   ▼
栈上的对象直接视为 black
   │
   ▼
后续运行期间
   │
   ├── 新分配对象
   │
   └── 指针修改
        │
        ▼
      write barrier
```

这样避免了传统方案中：

```text
Mark 完成
    ↓
再次 STW
    ↓
重新扫描所有栈
```

这就是：

> **Hybrid Write Barrier 的一个核心价值：减少 Mark Termination 时重新扫描栈带来的 STW。**

---

# 25. 新分配对象为什么直接处理？

GC Mark 阶段程序仍然可以：

```go
x := new(Object)
```

这个对象是新产生的。

如果把它当成普通白对象：

```text
new object
    ↓
white
```

就可能产生很多并发处理问题。

因此 Mark 阶段的新分配对象需要被特殊处理。

你原文：

```text
gcmarknewobject
```

就是这里。

可以理解成：

```text
mallocgc()
    │
    ▼
创建对象
    │
    ▼
GC 正在 Mark？
    │
    ▼
gcmarknewobject()
    │
    ▼
设置 mark bit
```

---

# 26. 这时可以把几个 Bitmap 放在一起理解

这是原文最应该结构化的地方。

| Bitmap | 解决什么问题 |
|---|---|
| `gcmarkBits` | 对象是否已经被本轮 GC 标记 |
| `allocBits` | 对象是否当前已分配 |
| `heapBits` | 对象内部哪些 word 是 pointer |
| `pageMarks` | 哪些 page/span 存在被标记对象 |

可以画成：

```text
                    Heap Metadata
                         │
       ┌─────────────────┼─────────────────┐
       │                 │                 │
       ▼                 ▼                 ▼
  gcmarkBits          allocBits         heapBits
       │                 │                 │
       │                 │                 └─ 哪些位置是 pointer
       │                 │
       │                 └────────────────── 对象是否 allocated
       │
       └──────────────────────────────────── 对象是否 marked


                    pageMarks
                        │
                        ▼
              哪些 page/span 有 marked object
```

这张图应该成为整篇文档的核心。

---

# 27. `allocBits` 和 `gcmarkBits` 为什么有两个？

假设一个 span：

```text
obj0 obj1 obj2 obj3

allocBits:
  1    1    1    0

gcmarkBits:
  1    0    1    0
```

说明：

```text
obj0：已分配 + 存活
obj1：已分配 + 不存活
obj2：已分配 + 存活
obj3：未分配
```

Sweep：

```text
allocBits = gcmarkBits
```

于是：

```text
新的 allocBits:

1 0 1 0
```

obj1 就变成：

```text
free
```

这就是 Mark → Sweep 的真正连接。

当前 runtime 的 `mspan.sweep` 明确执行：

```text
gcmarkBits → allocBits
```

然后重新创建清零的 `gcmarkBits`，供下一轮 GC 使用。[Go语言](https://go.dev/src/runtime/mgcsweep.go?utm_source=chatgpt.com)

---

# 28. 所以 Sweep 到底做什么？

不要记：

> “Sweep = 删除对象。”

更准确：

> **Sweep 遍历 span，根据 `gcmarkBits` 判断哪些已分配对象在本轮 GC 中没有存活标记，然后把这些对象重新变成可分配空间。**

例如：

```text
Mark 完成

gcmarkBits:
1 0 1 0 0 1

allocBits:
1 1 1 1 1 1
```

Sweep：

```text
1 → 保留
0 → 回收
1 → 保留
0 → 回收
0 → 回收
1 → 保留
```

得到：

```text
allocBits:
1 0 1 0 0 1
```

---

# 29. 重点问题：Sweep 为什么可以没有 STW？

这是你原文开头提出的问题。

答案：

> **因为 Go 把“标记”和“清扫”严格分离，而且 Sweep 不再需要修改程序正在读取的对象引用关系。**

Mark 阶段：

```text
GC 还在判断：

这个对象活不活？
```

如果此时直接回收：

```text
程序可能还持有 pointer
```

所以不能随便回收。

但是 Mark 完成之后：

```text
所有存活对象已经确定
```

此时：

```text
white object
```

就可以被认为是：

```text
dead
```

于是 Sweep 可以并发进行。

官方 runtime 明确说明：Sweep 是 concurrent 的，既可以由后台 sweeper 逐个 span 清扫，也可以在 goroutine 需要新 span 时按需进行；而且为了防止并发访问未清扫 span，分配路径会确保取得的 span 已经 sweep。[GitHub](https://github.com/golang/go/blob/master/src/runtime/mgc.go?utm_source=chatgpt.com)

---

# 30. 为什么不会出现“程序正在访问一个被 Sweep 的对象”？

关键不是：

> “Sweep 很安全，所以不需要 STW。”

而是：

> **GC 的正确性保证已经在 Mark 阶段完成。**

例如：

```text
A → B
```

如果 B 仍然可达：

```text
Root → A → B
```

那么：

```text
B.mark = 1
```

Sweep 不会回收 B。

只有：

```text
B.mark = 0
```

才会回收。

所以：

```text
Mark
 ↓
确定 live/dead
 ↓
Sweep
 ↓
只回收 dead
```

这是核心逻辑。

---

# 31. 但 Sweep 又为什么还需要 STW？

这里非常容易混淆：

> **Sweep 本身可以并发 ≠ GC 从来不需要为了 Sweep 做 STW。**

新一轮 GC 开始时，如果上一轮：

```text
还有 span 没 sweep
```

Runtime 会先处理这些 unswept spans。

当前 runtime 的 GC 高层流程明确有：

```text
1. Sweep Termination
   ↓
   STW
   ↓
   finishsweep_m()
```

然后才进入：

```text
Mark
```

也就是说：

```text
上一轮 Sweep 没完成
       │
       ▼
新一轮 GC 开始
       │
       ▼
STW
       │
       ▼
finishsweep_m()
       │
       ▼
保证没有 unswept span
       │
       ▼
开始新的 Mark
```

这是为了保证新的 GC 周期不会和旧的 Sweep 状态发生冲突。[GitHub](https://github.com/golang/go/blob/master/src/runtime/mgc.go?utm_source=chatgpt.com)

---

# 32. Sweep 的核心数据结构：`sweepgen`

`mheap` 有：

```go
sweepgen
```

`mspan` 也有：

```go
sweepgen
```

它实际上是：

> **用 generation 判断一个 span 是否已经完成当前轮 Sweep。**

核心关系：

```text
s.sweepgen == h.sweepgen - 2
        ↓
    needs sweep

s.sweepgen == h.sweepgen - 1
        ↓
    sweeping

s.sweepgen == h.sweepgen
        ↓
    swept / ready
```

当前 `mheap.go` 源码就是这样定义的。[GitHub](https://github.com/golang/go/blob/master/src/runtime/mheap.go?utm_source=chatgpt.com)

所以：

```text
mheap.sweepgen
      │
      │ 对比
      ▼
mspan.sweepgen
      │
      ▼
判断 span 是否可以使用
```

---

# 33. `mspan.sweep()` 的真正逻辑

可以把它压缩成：

```text
mspan.sweep()
      │
      ▼
遍历对象
      │
      ▼
检查 gcmarkBits
      │
      ├─────────────┐
      │             │
     1              0
      │             │
      ▼             ▼
    保留            释放
      │             │
      └──────┬──────┘
             ▼
      统计 nalloc
             │
             ▼
     allocBits = gcmarkBits
             │
             ▼
     gcmarkBits = new zero bitmap
             │
             ▼
       更新 sweepgen
```

当前 runtime 源码正是将 `gcmarkBits` 交换为 `allocBits`，再创建新的空 `gcmarkBits`。[Go语言](https://go.dev/src/runtime/mgcsweep.go?utm_source=chatgpt.com)

---

# 34. Sweep 为什么还要和 `mcentral` / `mcache` 联系起来？

Go 分配并不是：

```text
malloc → OS
```

而是：

```text
G
 │
 ▼
P
 │
 ▼
mcache
 │
 ▼
mcentral
 │
 ▼
mheap
```

所以当：

```text
mcache 没有可用 span
```

需要从：

```text
mcentral
```

拿 span。

但是：

> **拿到的 span 必须是已经 sweep 的。**

所以：

```text
mcache
  │
  ▼
cacheSpan()
  │
  ▼
mcentral
  │
  ▼
unswept span?
  │
  ▼
sweep
  │
  ▼
拿给 mcache
```

这就是你原文 `deductSweepCredit`、`cacheSpan`、`mcentral unswept` 应该放的位置。

---

# 35. GC 的完整流程应该这样记

现在把前面的知识串起来：

```text
                GC Trigger
                    │
                    ▼
              gcStart()
                    │
                    ▼
       ┌────────────────────────┐
       │ Sweep Termination      │
       │ STW                    │
       │ finishsweep_m()        │
       └───────────┬────────────┘
                   │
                   ▼
          Prepare Mark
                   │
       ┌───────────┼───────────┐
       │           │           │
       ▼           ▼           ▼
   reset mark   root jobs   write barrier
       │
       └───────────┬───────────┘
                   │
                   ▼
              Start World
                   │
                   ▼
              Concurrent Mark
                   │
          ┌────────┼─────────┐
          │        │         │
          ▼        ▼         ▼
       Mark Root  gcDrain  Assist
          │        │         │
          └────────┼─────────┘
                   │
                   ▼
              scanobject()
                   │
                   ▼
              greyobject()
                   │
                   ▼
                  gcw
                   │
                   └───────►继续扫描
                              │
                              ▼
                         Work exhausted
                              │
                              ▼
                         gcMarkDone
                              │
                              ▼
                            STW
                              │
                              ▼
                      Flush write buffer
                              │
                              ▼
                     Mark Termination
                              │
                              ▼
                         GCoff
                              │
                              ▼
                       Disable barrier
                              │
                              ▼
                         Start World
                              │
                              ▼
                      Concurrent Sweep
                       │             │
                       │             │
                 bgsweep()       allocation
                       │             │
                       └──────┬──────┘
                              ▼
                         mspan.sweep
                              │
                              ▼
                     allocBits ← markBits
                              │
                              ▼
                     reclaim dead objects
```

这张图比原来的函数列表更重要。

---

# 36. GC Worker、Assist、Mutator 三者是什么关系？

这是另外一个非常重要的结构。

```text
                     GC Mark Work
                          │
          ┌───────────────┼────────────────┐
          │               │                │
          ▼               ▼                ▼
    Dedicated Worker  Fractional Worker  Assist
          │               │                │
          │               │                │
      专门 GC 工作      碎片时间 GC       用户 G 帮忙
```

### Dedicated Worker

```text
GC 专用 G
```

尽量持续执行：

```text
gcDrain()
```

### Fractional Worker

利用：

```text
CPU 剩余碎片
```

执行 GC。

### Assist

用户 goroutine 分配内存时：

```text
mallocgc()
   │
   ▼
发现 gcAssistBytes < 0
   │
   ▼
gcAssistAlloc()
   │
   ▼
gcDrainN()
   │
   ▼
用户 G 帮 GC 做 Mark
```

所以：

> **GC 不只是后台 GC worker 在干活，Mutator 本身也可能被迫帮 GC。**

---

# 37. 为什么需要 GC Assist？

假设：

```text
G1：疯狂分配
G2：普通业务
G3：普通业务
```

如果只有后台 GC：

```text
G1 分配得太快
      ↓
Heap 快速增长
      ↓
GC 来不及 Mark
```

于是 Go 会让：

```text
G1
```

承担一部分 Mark 工作。

可以理解为：

```text
你制造垃圾越快
        ↓
你需要承担更多 GC 工作
```

这就是：

> **Mutator Assist**

---

# 38. `gcAssistBytes` 可以怎么理解？

它本质上是：

> **这个 G 对 GC 的“工作债务/信用”。**

简化理解：

```text
gcAssistBytes > 0
        ↓
有信用
        ↓
暂时不用 Assist

gcAssistBytes < 0
        ↓
有债务
        ↓
mallocgc()
        ↓
gcAssistAlloc()
        ↓
帮 GC Mark
```

因此：

```text
分配内存
   ↓
增加 GC debt
   ↓
必要时 Assist
   ↓
完成一定扫描工作
   ↓
偿还 debt
```

---

# 39. GC Pacer 应该放在哪里？

你原文把大量公式直接堆在一起，实际上应该先回答：

> **Pacer 是干什么的？**

答案：

> **Pacer 决定“什么时候 GC、需要多少 GC 工作、多少 CPU 应该用于 GC、Mutator 需要承担多少 Assist”。**

所以：

```text
                GC Pacer
                   │
       ┌───────────┼────────────┐
       │           │            │
       ▼           ▼            ▼
   什么时候开始   要做多少工作   谁来做
                              │
                    ┌─────────┼─────────┐
                    ▼         ▼         ▼
                  worker    fractional  assist
```

这比单独背：

```text
assistWorkPerByte
fractionalUtilizationGoal
dedicatedMarkWorkersNeeded
```

更容易理解。

---

# 40. `heap_live` 和 `next_gc` 是什么关系？

核心可以先记：

```text
heap_live
    ↓
当前活跃堆大小

next_gc
    ↓
预计下一次 GC 的目标堆大小
```

例如：

```text
当前：

heap_live = 100 MB

GOGC = 100
```

简单直觉：

```text
允许堆继续增长约一个“基准堆”的量
```

于是下一轮 GC 大致围绕：

```text
~200 MB
```

这个量级发生。

但要注意：

> **当前 Go 的实际 GC trigger/pacer 逻辑比“heap_live × 2”复杂，不能把 GOGC=100 简化成永远精确 2×。**

因此原文：

> “heap增长一倍”

应该作为**直觉模型**，而不是源码精确公式。

---

# 41. GC CPU 25% 到底是什么意思？

原文：

```text
gc的目标：cpu不超过25%
```

这个说法太绝对。

更准确：

> Go GC Pacer 的目标之一是让后台 Mark 工作大约占用 `GOMAXPROCS` 可用 CPU 的 25% 左右。

例如：

```text
GOMAXPROCS = 8

total utilization goal
= 8 × 25%
= 2 个 CPU
```

所以可以理解成：

```text
8 CPU

业务：
██████████████

GC：
████
```

而不是：

```text
整个程序 CPU 使用率绝对不能超过 25%
```

---

# 42. `dedicatedMarkWorkersNeeded`

例如：

```text
GOMAXPROCS = 8

目标：
8 × 25% = 2
```

所以大致需要：

```text
2 个 dedicated mark worker
```

剩余不足的部分：

```text
fractional worker
```

补充。

因此：

```text
             Mark CPU Goal
                  │
          ┌───────┴───────┐
          ▼               ▼
     Dedicated         Fractional
        2 CPU             补余量
```

---

# 43. GC Trigger 有哪几种？

你的原文这里基本正确，但应该结构化。

```text
gcTriggerKind

    ├── gcTriggerHeap
    │      ↓
    │   heap 达到目标
    │
    ├── gcTriggerTime
    │      ↓
    │   长时间没有 GC
    │
    └── gcTriggerCycle
           ↓
        用户主动触发
```

对应：

```text
mallocgc()
     │
     ▼
gcTrigger.test()
     │
     ├── Heap
     ├── Time
     └── Cycle
```

---

# 44. `forcegcperiod`

定时 GC：

```text
forcegcperiod
```

用于防止一种情况：

```text
程序几乎不分配
      ↓
Heap trigger 永远达不到
      ↓
长时间不 GC
```

因此还有：

```text
时间 trigger
```

它保证即使分配很少，也不会无限期不进行 GC。

---

# 45. `GOGC` 应该怎么理解？

最简单：

```text
GOGC
 ↓
控制 GC 相对堆增长目标
```

默认：

```text
GOGC = 100
```

意味着允许堆相对于上一轮存活堆有比较明显的增长，然后触发下一轮 GC。

重要的是：

```text
GOGC 越大
    ↓
GC 更晚
    ↓
堆更大
    ↓
GC CPU 压力通常降低

GOGC 越小
    ↓
GC 更早
    ↓
堆更小
    ↓
GC 更频繁
```

此外，当前 Go 还有 **memory limit** 机制，因此实际 pacer 不应该只用 GOGC 一个参数解释。官方 GC Guide 也将 GC CPU、内存占用和 memory limit 作为整体成本模型讨论。[Go语言](https://go.dev/doc/gc-guide?utm_source=chatgpt.com)

---

# 46. `uintptr` 为什么不能当普通指针？

这是 GC 很经典的面试问题。

```go
var p uintptr
```

即使：

```text
p
```

里面存的是一个真实地址：

```text
0xc000123456
```

GC 也不会把它当成：

```text
pointer
```

原因：

```text
uintptr
   ↓
integer
   ↓
不是 GC pointer
```

所以：

```text
object A
   │
   └── uintptr(address of B)
```

GC 不会因为这个 `uintptr` 找到 B。

因此：

> **需要 GC 追踪的地址必须保持为真正的 Go pointer，而不是 `uintptr`。**

---

# 47. Sweep 与 Scavenge 必须分开

原文这里也容易混。

## Sweep

解决：

> **对象级别的垃圾回收。**

```text
span
 ↓
objects
 ↓
根据 markBits
 ↓
释放 dead objects
```

---

## Scavenge

解决：

> **物理页是否可以归还给 OS。**

所以：

```text
Sweep
 ↓
释放对象
 ↓
形成空闲 page/span
 ↓
Scavenge
 ↓
把不用的物理页归还 OS
```

例如：

```text
Heap

+----+----+----+----+
| A  | B  | C  | D  |
+----+----+----+----+

Sweep：

+----+----+---------+
| A  | C  |  free   |
+----+----+---------+

Scavenge：

free page
   ↓
sysUnused
   ↓
OS
```

所以：

> **Sweep ≠ 归还 OS。**

---

# 48. 最后把整个 GC 用“6 个问题”记住

如果你准备面试，我建议整篇文档最终收敛成这 6 个问题。

---

## Q1：哪些对象需要标记？

```text
GC Roots
  ↓
data / bss / stack / runtime roots
  ↓
沿 pointer 遍历
  ↓
所有可达 Heap Object
```

---

## Q2：怎么知道对象里面哪个字段是指针？

```text
编译器
  ↓
_type
  ↓
ptrdata + gcdata
  ↓
pointer bitmap
```

Heap 中进一步通过：

```text
heapBits
```

辅助确定实际布局。

---

## Q3：怎么记录对象是否存活？

```text
mspan
  ↓
gcmarkBits
  ↓
1 bit / object
```

例如：

```text
objects:     A B C D
mark bits:   1 0 1 0
             │ │ │ │
             │ │ │ └─ dead
             │ │ └─── live
             │ └───── dead
             └─────── live
```

---

## Q4：怎么完成整个对象图遍历？

```text
Root
 ↓
markroot
 ↓
greyobject
 ↓
gcw
 ↓
gcDrain
 ↓
scanobject
 ↓
发现新 pointer
 ↓
greyobject
 ↓
gcw
 ↓
……
```

这就是 Mark。

---

## Q5：并发 Mark 时程序修改指针怎么办？

```text
Mutator
   │
   ▼
修改 pointer
   │
   ▼
Write Barrier
   │
   ▼
防止 white object 被遗漏
```

Go 使用 Hybrid Write Barrier 来降低重新扫描栈的成本。

---

## Q6：Mark 完成以后怎么回收？

```text
gcmarkBits
     │
     ▼
   Sweep
     │
     ▼
未标记对象 → free
     │
     ▼
allocBits = gcmarkBits
     │
     ▼
重新生成空 gcmarkBits
```

然后：

```text
free page
   ↓
Scavenge
   ↓
OS
```

---

# 49. 最重要的一张“源码对应关系图”

最后建议你以后看 Go GC 源码时，始终按照这张图定位：

```text
                    Go GC
                      │
       ┌──────────────┼──────────────┐
       │              │              │
       ▼              ▼              ▼
     Trigger          Mark          Sweep
       │              │              │
       │              │              │
   gcTrigger.test   gcDrain       sweepone
   gcStart          markroot      mspan.sweep
                    scanobject
                    greyobject
                    gcw
                      │
                      │
             ┌────────┴────────┐
             ▼                 ▼
          “对象活吗？”       “哪里是指针？”
             │                 │
             ▼                 ▼
        gcmarkBits          heapBits
             │                 │
             │                 │
             ▼                 ▼
          Sweep              Scan
             │
             ▼
        allocBits
             │
             ▼
         可重新分配
```

再加上并发控制：

```text
                  GC Mark
                     │
          ┌──────────┼──────────┐
          │          │          │
          ▼          ▼          ▼
        Worker     Assist    Write Barrier
          │          │          │
          └──────────┼──────────┘
                     ▼
                   gcw
                     │
                     ▼
                scanobject
```

**这样你再去看 `mgc.go / mgcmark.go / mgcsweep.go`，就不是“函数流水账”，而是在验证这张架构图。**

另外，你原文里涉及的 `gcmarkBits → allocBits`、`sweepgen`、unswept span、后台 Sweep 与分配路径 Sweep 等细节，我建议保留，因为这些才是从“三色标记法”进入 **Go Runtime 实现**的关键桥梁。当前 runtime 源码也明确保留了这些机制。[GitHub](https://github.com/golang/go/blob/master/src/runtime/mheap.go?utm_source=chatgpt.com)

如果按这个思路继续整理，**你这份 719 行原始笔记不应该再按“函数 1、函数 2、函数 3”展开，而应该重构成：**

**① GC 总体架构 → ② Root/对象 → ③ Pointer bitmap → ④ 三色标记 → ⑤ Mark Bits → ⑥ Work Queue → ⑦ 并发 Mark → ⑧ Write Barrier → ⑨ GC Assist/Pacer → ⑩ Sweep → ⑪ Scavenge → ⑫ 完整源码调用链 → ⑬ 面试重点。**

这样才比较适合你现在这种**“既要理解原理，又要能落到 Go runtime 源码”**的学习方式。
