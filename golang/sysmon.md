# sysmon
> 系统进程，运行在系统栈中;可以抢占p，长时间运行的g和执行netpoll
> start work 或进入系统调用前，会唤醒sysmon
- 1. 检查死锁checkdead，设置启动标记
- 2. 进入循环：
- > 休眠（start 10微秒，50次之后翻倍，大于10ms，则休眠10ms）
- > mDoFixup
- > 若stw或者所有p都停止了，则定时休眠sysmon在sysmonnote
- > cgo 处理
- > 若超过10ms未调用netpoll，则调用netpoll，将唤醒的g注入调度系统
- > 尝试唤醒scavenge
- > retake:抢占处于系统调用的p
- > gc时间是否到了，到了则注入g启动gc
- 
