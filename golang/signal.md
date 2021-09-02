# 信号

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

## 函数
