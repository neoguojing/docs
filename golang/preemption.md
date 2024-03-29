# preemption
- 抢占大多数时候意味这一次调度，调用schedule
## 总结
- 什么时候时非安全点：1.正在执行runtime代码的g，因为大部分runtime代码不能被抢占
- g何时被抢占：1.被阻塞syscall，mutex和chan 2.执行超过10ms（sysmon），在执行函数时检查栈标志，实行抢占；3.无函数执行但超过10ms，执行异步抢占
- 安全点的作用：在这个点，g的cpu使用率处于最低，gc可以知道g的栈的全部信息；这样runtime可以用最小的代价停止g的调度，并精确的扫描g的栈
- 安全点(safe point)分类：
- > 阻塞的安全点：g被停止调度，阻塞在同步原语或者系统调用中
- > 同步安全点：g在检查一个抢占请求；主要是通过设置gp.stackguard0==stackPreempt，在执行函数检查栈是触发morestack，在调用newstack是执行抢占调度
```
    //同步抢占触发
    gp.preemptStop = true
    gp.preempt = true
    gp.stackguard0 = stackPreempt
```
- > 异步安全点：实现主要是通过os的信号量等；异步安全点要求：能够安全的挂起和扫描栈，有充足的栈注入ayncPreempt，不会死锁等。
```
```
- 抢占的实现大部分在newstack函数实现
- m可被抢占的条件canPreemptM：mp.locks == 0，m为处理内存分配，preemptoff="",且m绑定p处于运行状态
- g可以异步抢占的条件：：g是否可以被异步抢占：g.preempt 和p.preempt为true，g是运行状态
- 异步抢占：1.sysmon发现g运行超过10ms，向m发送SIGURG信号；2.handler被执行，修改g的程序计数器，使g暂停，并指向下一轮调度。解决没有函数调用的循环无法被抢占的问题
- 强制抢占可以调用runtime.GoSched
- GODEBUG=asyncpreemptoff=1 ：关闭异步抢占

![signal](./golang-signal&prempt.drawio.png)

### 参数
- gp.preempt：true为可以抢占
- gp.stackguard0 = stackPreempt：每次g中的函数调用都会检查此标记，来检测抢占
- preemptMSupported： 异步抢占标记
- _p_.preempt：p应该执行调度，不管当前的g在执行什么
- mp.signalPending：抢占信号
- _g_.m.locks++: 禁止抢占
- _SIGURG： 带外数据，更高的优先级
- sched.safePointWait:等待到达安全点的p的数量
- sched.safePointFn: p到达安全点要执行的函数，一般在gc时候使用
- p.runSafePointFn:执行sched.safePointFn 在下一个安全点
- m.gsignal： 处理信号量
- asyncPreemptStack: asyncPreempt函数需要的栈空间，init时初始化
### 函数
- preemptall：遍历allp，调用preemptone，尽力停止所有g
- preemptone：
- > m和g为空，返回false
- > 设置g的抢占标记gp.preempt gp.stackguard0 = stackPreempt
- > 异步抢占：设置_p_.preempt，调用preemptM
- preemptM：设置m的抢占信号为1，调用系统调用tgkill，发送_SIGURG给m对应的线程
- acquirem：禁止抢占
- releasem：恢复抢占
- suspendG：將g挂起在安全的，markroot中調用，状态机，尝试_Gpreempted-> _Gwaiting| -> _Gwaiting|_Gscan
- resumeG : 对于stopped的g使用ready唤醒
- preemptPark ： 切换g状态为_Gscan|_Gpreempted，dropg解绑m和g，切换状态_Gpreempted，调用schedule
- gopreempt_m ： 切换状态为_Grunnable，解绑m和g，将g放入全局p，schedule
- asyncPreempt:保存用户寄存器，调用asyncPreempt2
- asyncPreempt2：设置g的asyncSafePoint=true，preemptStop=true，则preemptPark，否则gopreempt_m，设置asyncSafePoint=false
- doSigPreempt： 处理g的抢占信号
- > wantAsyncPreempt判断g是否需要抢占,isAsyncSafePoint是否在异步安全点，修改g的pc指向asyncPreempt
- > 重置m.signalPending
- wantAsyncPreempt：g是否可以被异步抢占：g.preempt 和p.preempt为true，g是运行状态
- isAsyncSafePoint：
- > m可以被抢占
- > g的栈能够存储asyncPreemptStack
- > 检查pc是否时一个不安全点
- retake： 每个sysmon循环都会调用,异步抢占超时的g和解除系统调用时m和p
- > 遍历所有p：
- > p处于_Prunning或_Psyscall
- > 更新sysmontick
- > 若上次p被调度的时间+10ms <= 当前时间，表示g超时了 调用preemptone设置抢占标记，向m发出信号
- > 若p处于_Psyscall，更新syscalltick或者设置p为_Pidle，调用handoffp从系统调用解锁p和m
## 引用
- https://medium.com/a-journey-with-go/go-asynchronous-preemption-b5194227371c
- https://medium.com/a-journey-with-go/go-goroutine-and-preemption-d6bc2aa2f4b7
