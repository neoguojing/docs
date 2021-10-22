# 信号
主要处理异步通知，同时实现了异步抢占
## 总结：
- 进程启动时为所有信号注册handler，initsig
- 通过signal.Notiy或runtime.signalM 发送信号量到m
- os通知m执行handler处理

## 参数
- signal_unix.go
- os_linux.go
- signal_amd64.go

对于 Linux来说，实际信号是软中断，许多重要的程序都需要处理信号。信号，为 Linux 提供了一种处理异步事件的方法

- 信号有65种
- SIGKILL和SIGSTOP 不能被屏蔽
- 信号编号小于等于31的信号都是不可靠信号，肯能会丢失
- 阻塞信号状态：暂时屏蔽该信号，用户可读写
- pending：信号未决状态，内核依据阻塞信号设置，用户只读
- rt_sigaction： 系统调用，为信号设置处理和返回函数等。
- raise： 系统调用，触发一个信号
- sigprocmask()系统调用改变和检查自己的信号掩码的值
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

type sigctxt struct {
	info *siginfo
	ctxt unsafe.Pointer
}

type siginfo struct {
	si_signo int32
	si_errno int32
	si_code  int32
	// below here is a union; si_addr is the only field we use
	si_addr uint64
}

<!-- out going队列 -->
var sig struct {
	note       note
	mask       [(_NSIG + 31) / 32]uint32
	wanted     [(_NSIG + 31) / 32]uint32
	ignored    [(_NSIG + 31) / 32]uint32
	recv       [(_NSIG + 31) / 32]uint32
	state      uint32
	delivering uint32
	inuse      bool
}

```


## 函数
- initsig：进程启动时调用，遍历sigtable[64],调用setsig设置sigactiont结构体，调用系统调用rt_sigaction设置信号处理函数；
- m.gsignal :处理信号的g
- > mcommoninit中调用malg分配一个32k栈大小的g，用于满足大部分架构，且栈不会扩容
- > 每个m都有这样一个g
- setsig：填充sigactiont，调用sigaction设置信号回调等
- sigaction：调用sysSigaction
- signalM：调用tgkill发送信号到对应m
- sighandler： 信号处理函数最终的信号处理函数
- > _SIGPROF信号交给sigprof
- > sigPreempt 调用doSigPreempt
- > _SI_USER|_SigNotify ：调用sigsend发送信号
- _SigKill：dieFromSignal处理
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
- sigFetchG:调用getg；其他通过栈地址获取g；g被保存在栈底
- setg：汇编代码，保存g，使得能够被needm使用；保存g到tls？
- sigsend：将信号加入sig.mask,并设置sig.state,通知recv接受;
- signal_recv

## 引用
- https://medium.com/a-journey-with-go/go-gsignal-master-of-signals-329f7ff39391
