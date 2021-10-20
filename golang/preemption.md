# preemption
- 抢占大多数时候意味这一次调度，调用schedule
## 总结
- g被抢占的条件：1.g在安全点 
- 安全点的作用：在这个点，g的cpu使用率处于最低，gc可以直到g的栈的全部信息；这样runtime可以用最小的代价停止g的调度，并精确的扫描g的栈
- 安全点(safe point)分类：
- > 阻塞的安全点：g被停止调度，阻塞在同步原语或者系统调用中
- > 同步安全点：g在检查一个抢占请求；主要是通过设置gp.stackguard0==stackPreempt，在执行函数检查栈是触发morestack，在调用newstack是执行抢占调度
- > 异步安全点：用户code控制；实现主要是通过os的信号量等；
- 抢占的实现大部分在newstack函数实现
- m可被抢占的条件canPreemptM：mp.locks == 0，m为处理内存分配，preemptoff="",且m绑定p处于运行状态
- g可以异步抢占的条件：：g是否可以被异步抢占：g.preempt 和p.preempt为true，g是运行状态
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
- > wantAsyncPreempt判断g是否需要抢占,isAsyncSafePoint是否在异步安全点，注冊asyncPreempt函數和返回點函數
- > 重置m.signalPending
- wantAsyncPreempt：g是否可以被异步抢占：g.preempt 和p.preempt为true，g是运行状态
- isAsyncSafePoint： ？？？

## 引用
- https://medium.com/a-journey-with-go/go-asynchronous-preemption-b5194227371c
- https://medium.com/a-journey-with-go/go-goroutine-and-preemption-d6bc2aa2f4b7
