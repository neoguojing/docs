# 网络

## netpoll
> runtime/netpoll.go
> runtime/netpoll_epoll.go

### epoll
- 
### 常量

### 结构
- pollDesc
- pollCache
### 函数

- netpollinit
- netpollopen
- netpollBreak
- netpollIsPollDescriptor
- netpollinited：netpoll是否初始化
- netpoll：检查就绪的网络连接
- pollWork：判断是否有网络事件供当前p处理
- 
