# 网络

## netpoll
> runtime/netpoll.go
> runtime/netpoll_epoll.go

### epoll

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

### 结构
- pollDesc
- pollCache
### 函数

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
- netpollinited：netpoll是否初始化
- netpoll：检查就绪的网络连接,delay<0 永久阻塞，==0不阻塞，大于0等待ns，返回一个就绪g列表runnable
- > 
- > 
- pollWork：判断是否有网络事件供当前p处理
- 

## 引用
- https://blog.csdn.net/justmeloo/article/details/40184039
- https://blog.csdn.net/lixungogogo/article/details/52226479
