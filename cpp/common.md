# 通用知识

## 信号处理
- SIGPIPE   tcp通道单向关闭，协议栈返回rst，linux产生   signal(SIGPIPE, SIG_IGN); 屏蔽方法
- SIGINT    ctrl+c信号
- SIGSEGV   是当一个进程执行了一个无效的内存引用，或发生段错误时发送给它的信号
- SIG_IGN   忽略
- SIG_DFL   默认信号处理程序
- SIGABRT   多次free导致的SIGABRT、abort、assert
