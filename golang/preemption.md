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
### 参数
- gp.preempt：true为可以抢占
- gp.stackguard0 = stackPreempt：每次g中的函数调用都会检查此标记，来检测抢占
- preemptMSupported： 异步抢占标记
- _p_.preempt：p应该执行调度，不管当前的g在执行什么
- mp.signalPending：抢占信号
- _g_.m.locks++: 禁止抢占
- _SIGURG： 带外数据，更高的优先级
### 函数
- preemptall：遍历allp，调用preemptone，尽力停止所有g
- preemptone：
- > m和g为空，返回false
- > 设置g的抢占标记gp.preempt gp.stackguard0 = stackPreempt
- > 异步抢占：设置_p_.preempt，调用preemptM
- preemptM：设置m的抢占信号为1，调用系统调用tgkill，发送_SIGURG给m对应的线程
- acquirem：禁止抢占
- releasem：恢复抢占
- suspendG
- resumeG
- preemptPark
- gopreempt_m
