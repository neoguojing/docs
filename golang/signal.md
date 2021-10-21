# 信号

## 总结：
- 
- signal_unix.go
- os_linux.go
- signal_amd64.go

对于 Linux来说，实际信号是软中断，许多重要的程序都需要处理信号。信号，为 Linux 提供了一种处理异步事件的方法

- 信号有65种
- SIGKILL和SIGSTOP 不能被屏蔽
- 信号编号小于等于31的信号都是不可靠信号，肯能会丢失
- 阻塞信号状态：暂时屏蔽该信号，用户可读写
- pending：信号未决状态，内核依据阻塞信号设置，用户只读
- sigprocmask()系统调用改变和检查自己的信号掩码的值：
```
数how的取值及含义如下：

SIG_BOLCK          ：新的进程信号掩码是其当前值和set指定信号集的并集；

SIG_UNBOCK       ：新的进程信号掩码是其当前值和~set信号集的交集，因此set指定的信号集将不被屏蔽；

SIG_SETBLOCK   ：直接将进程信号掩码设为set；
```
- 信号类型：
- > _SigNotify:信号通知
- > _SigKill: 不通知,signal.Notify
- > _SigThrow： signal.Notify
- > _SigPanic: 信号来之内核，
- > _SigDefault: 不需要监控
- > _SigUnblock: 非阻塞的信号
- > _SigIgn: 忽略信号

- _SIGPROF： sigprof
- _SIGURG：doSigPreempt
- _SigKill： dieFromSignal
## 结构体
```
//信号的执行体
type sigactiont struct {
	sa_handler  uintptr //信号处理函数，对应sigtramp
	sa_flags    uint64  //信号码
	sa_restorer uintptr  //信号处理返回地址，值为sigreturn
	sa_mask     uint64  // 信号掩码
}
```


## 函数
- initsig：进程启动时调用，遍历sigtable[64],调用setsig设置sigactiont结构体
- m.gsignal :处理信号的g
- > mcommoninit中调用malg分配一个32k栈大小的g，用于满足大部分架构，且栈不会扩容
- > 每个m都有这样一个g
- setsig
- signalM：调用tgkill发送信号到对应m
- sighandler： 信号处理函数
- sigtramp：汇编代码，会调用sigtrampgo，次为go信号的实际handler
- sigtrampgo：真正的runtime信号处理函数：
- > sigfwdgo是否需要转发给g处理，true，直接返回
- > cgo处理？？
- > 非cgo，setg保存g
- > 调整signal栈
- > 调用sighandler处理信号
- cgoSigtramp：cgo的信号处理函数，汇编代码
- sigreturn：汇编代码
- doSigPreempt:处理g的抢占信号
- sigfwdgo：signal是否要转发给go处理
- sigFetchG:通过栈地址获取g；g被保存在栈底
- setg：汇编代码，保存g，使得能够被needm使用


## 引用
- https://medium.com/a-journey-with-go/go-gsignal-master-of-signals-329f7ff39391
