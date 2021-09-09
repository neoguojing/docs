# 网络
## 概述
- 调度/GC/sysmon等流程，会调用netpoll函数，遍历就绪的事件，并把事件中存储的pd的rg/wg的状态设置为pdReady
- FD的Read和Write函数会调用pd.wait函数，尝试扭转pd状态为pdWait，并调用gopark挂起当前go程，同时尝试赋值pd状态为当前go程，成功则直接恢复g的调度，否则，等待下一次调度
- FD的Read/Write函数首先调用系统调用读或者写fd，若fd暂时没有数据，会调用pd.wait异步等待新的事件的到来；
- gopark是实现异步的关键
## tips
```
 var old uintptr = 0
 a:=unsafe.Pointer(old)
 
 a 为nil
```

## netpoll
> runtime/netpoll.go
> runtime/netpoll_epoll.go
> internal/poll/fd_poll_runtime.go
### epoll

- EAGAIN：read操作时触发，提示现在暂时没有数据，请稍后再试
#### 原理
- 内核高速cache区：内核启动时开辟，已经建立于物理内存映射，建立了slab层
- socket以红黑树的形式保存
- 就绪socket列表
- mmap实现消息copy
- LT：epoll_wait在检查fd上是否有未处理的事件，有的话重新放回就绪列表，下次还会通知
- ET：只有新的中断到了才会返回相应事件；
#### 函数
- int epoll_create ( int size ) ：size可不填；创建一个file节点作为fd，创建红黑树和就绪列表；
- int epoll_create1 ( int flag ) ：
- > 当flag = EPOLL_CLOEXEC，创建的epfd会设置FD_CLOEXEC;FD_CLOEXEC:对于socket，父进程fork子进程时，会关闭相应的socket，防止子进程拥有相关端口的使用权；对于文件防止子进程拥有父进程文件；
- int epoll_ctl ( int epfd , int op , int fd , struct epoll_event * event )：添加fd到红黑树，并注册回调函数给内存中断；
```
EPOLLIN:表示关联的fd可以进行读操作了。
EPOLLOUT:表示关联的fd可以进行写操作了。
EPOLLRDHUP(since Linux 2.6.17):表示套接字关闭了连接，或者关闭了正写一半的连接。
EPOLLPRI:表示关联的fd有紧急优先事件可以进行读操作了。
EPOLLERR:表示关联的fd发生了错误，epoll_wait会一直等待这个事件，所以一般没必要设置这个属性。
EPOLLHUP:表示关联的fd挂起了，epoll_wait会一直等待这个事件，所以一般没必要设置这个属性。
EPOLLET:设置关联的fd为ET的工作方式，epoll的默认工作方式是LT。
EPOLLONESHOT (since Linux 2.6.2):设置关联的fd为one-shot的工作方式。表示只监听一次事件，如果要再次监听，需要把socket放入到epoll队列中。

typedef union epoll_data {
  void        * ptr ;
  int          fd ;
  uint32_t     u32 ;
  uint64_t     u64 ;
} epoll_data_t ;
 
struct epoll_event {
  uint32_t     events ;      /* Epoll events */
  epoll_data_t data ;        /* User data variable */
} ;
```
- int epoll_wait(int epfd, struct epoll_event *events, int maxevents, int timeout)：
- > 返回：0表示超时，n表示活跃的事件数量
- > 网络中断道来，copy网卡数据到内核，调用fd注册的回调函数，将相应的fd放入就绪列表；
- > 该函数检查就绪列表，有事件则返回，并把事件copy到用户态，清空就绪列表；无事件则超时后返回；
- int epoll_pwait(int epfd, struct epoll_event *events, int maxevents, int timeout,  const sigset_t *sigmask)： 允许捕获一个信号量
### 常量
- pdReady：1，io就绪通知挂起；
- pdWait：2，一个g准备在该信号量上挂起等待
- pollBlockSize：大小为4k
- netpollWaiters： 等待pdWait的g的数量
### 结构
- pollDesc ：大小为232字节，不在gc内存上分配，使用persistentalloc分配
- pollCache：pollDesc的缓存，
### 函数

#### 通用接口
- poll_runtime_pollServerInit: 调用netpollinit，并设置netpollInited=1
- poll_runtime_isPollServerDescriptor：判断fd是否被netpoll使用
- poll_runtime_pollOpen：
- > pollcache.alloc()分配一个pd
- > 初始化pd，调用netpollopen
- poll_runtime_pollClose：调用netpollclose，释放pd
- poll_runtime_pollReset：设置rg和wg为0
- poll_runtime_pollWait：轮询rg/wg的状态，直到状态为pdReady，表示io就绪
- > 循环调用netpollblock，若状态为pdReady，则返回0，表示有网络事件到达，否则继续调用；期间检查错误，若出现其他错误，则返回错误码
- poll_runtime_pollWaitCanceled//只在window使用
- poll_runtime_pollSetDeadline：
- > 设置rd/wd
- > 设置rt/wt，调用resettimer，设置计时器
- > rd/wd < 0 表示设置时间超时，调用netpollunblock，返回rg/wg
- > 若rg/wg不为nil，调用netpollgoready
- poll_runtime_pollUnblock
- netpollinited：netpoll是否初始化
- func netpollgoready(gp *g, traceskip int)：netpollWaiters-1 调用goready是g变为runable
- netpollready：当epoll事件就绪时，尝试设置rg/wg为pdReady，可能的情况下返回pd绑定的g
- > 调用netpollunblock(pd, xx, true)，返回rg/wg，非nil，则放入glist
- netpollblock：返回true表示IO ready，false表示超时或者关闭，参数waitio在epoll下永远为false；只对windows的完成io有用
- > rg/wg == pdReady，设置rg/wg返回true，
- >  循环直到rg/wg == 0,则设置值为pdWait
- > 若pd的错误为0 ，则调用gopark暂停g的同时调用netpollblockcommit，尝试设置rg/wg的值由pdwait变为g，成功则马上恢复g的执行，否则等待调度
- netpollunblock：轮询，rg/wg为pdready返回nil，否则尝试切换状态为pdready，并返回rg/wg的值，可能为pdwait或者g
- > rg/wg状态为pdReady，返回nil
- > ioready为false && rg/wg未就绪，则返回nil
- > cas不断循环尝试设置rg/wg为pdReady，成功则返回rg/wg的原始值：若原始值为pdWait，则返nil，若原始值为g，则返回g的指针
- pollWork：判断是否有网络事件供当前p处理
- netpollReadDeadline: 为timer使用的回调函数，调用netpolldeadlineimpl
- netpollWriteDeadline：为timer使用的回调函数，调用netpolldeadlineimpl
- netpolldeadlineimpl：调用netpollunblock，返回rg/wg，rg/wg！= nil，调用netpollgoready，恢复g调用
#### linux 接口
- netpollinit：
- > epollcreate1(_EPOLL_CLOEXEC) 创建close on exec的fd
- > nonblockingPipe()：创建非阻塞的管道，返回读netpollBreakRd和写netpollBreakWr fd
- > 注册_EPOLLIN事件，和自定义的事件netpollBreakRd（上一步非阻塞管道的读fd）
- > epollctl(epfd, _EPOLL_CTL_ADD, r, &ev)
- netpollopen(fd uintptr, pd *pollDesc): 
- > 注册读写和关闭事件，触发模式未ET，自定义数据为传入的pd
- > epollctl(epfd, _EPOLL_CTL_ADD, int32(fd), &ev): 注册传入的fd和ev
- netpollBreak：中断epollwait
- > 抢占netpollWakeSig
- > 向netpollBreakWr写入一个字节，循环写，直到写成功
- netpollIsPollDescriptor
- netpoll：检查就绪的网络连接,delay<0 永久阻塞，==0不阻塞，大于0等待ns，返回一个就绪g列表runnable
- > 调用epollwait
- > 若返回值< 0，若delay > 0 ,返回空glist，否则重新调用epollwait
- > 否则，遍历所有返回的事件：
- > 若ev.data 等于&netpollBreakRd，若deley！=0，调用read读取netpollBreakRd管道，修改netpollWakeSig=0，继续遍历
- > 根据事件类型确定时读模式还是写模式
- > 从ev.data获取pollDesc，设置pd.everr
- > 调用netpollready(&toRun, pd, mode)

- 

## 引用
- https://blog.csdn.net/justmeloo/article/details/40184039
- https://blog.csdn.net/lixungogogo/article/details/52226479
