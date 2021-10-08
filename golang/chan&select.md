# chan和select

## 总结
- nil chan写入/读取永久阻塞
- closed的chan写入 panic
- waitq从队列头出队
- 关闭nil或closed的chan会panic
- select使用的chan是非阻塞的，revc和send不需要执行gopark暂停当前g
- send中唤醒recvq中的g，recv唤醒sendq中的g
## channel

### 概览
- chan中元素大小不能大于2^16()64k
- hchanSize： 96
### 函数
- makechan：
- > 对于长度为0的chan，调用mallocgc分配hchanSize的空间；c.buf指向nil
- > elem中没有指针，则mallocgc分配hchanSize+elem.size*size的连续空间；设置c.buf指向hchanSize偏移后的地址
- > 否则：new(hchan，使用mallocgc为c.buf分配空间
- > 设置elem等相关参数
- chansend1： c <- x的实现，block参数为true
- chansend： 在select场景下，不可写，则返回false
- > 若chan为nil，若select(block为false)则返回false，否则调用gopark阻塞g
- > 若为select（block=false），未关闭且buf满了，则返回false
- > 加锁
- > 若chan 关闭，则panic
- > 从recvq出队一个sudog，若sudog不等于nil，则调用send直接发送消息到对应的sudog,返回true
- > 若qcount< datasize,表示有buf未满：获取sendx位置的指针，调用typedmemmove为该位置赋值，sendx++，qcount++,返回true
- > 若是select，则直接返回false
- > 无法直接写入，则acquireSudog获取sudog，填充elem和g等，并设置g.waiting=sudog,gp.param=nil,将sudog放入sendq
- > 调用gopark，阻塞g,并在调用过程中解锁chan
- > KeepAlive??
- > 被唤醒，清空g的标志，释放sudog
- send：recvq的sudog和elem的值
- > 若sudog的elem不为nil，调用sendDirect，并设置elem为nil
- > 获取sudog的g，设置g.param=sudog,设置sudog.success=true,调用goready唤醒g
- sendDirect：将数据copy到revc dog的地址：调用typeBitsBulkBarrier对所有指针执行内存屏障，调用memmove，将elem放到sudog的elem地址
- acquireSudog:获取或新建一个sudog
- > 优先从p.sudogcache获取
- > p.sudogcache不够，则从sched.sudogcache获取
- > 否则new一个
- closechan: 关闭chan
- > 设置closed=1
- > 循环遍历出队c.recvq，nil则break: 设置gp.param，设置sg.succes=false，将sudog对应的g加入glist
- > 遍历出队 c.sendq，直到nil：同上
- > 遍历glist，依次调用goready，唤醒g
- chanrecv1: <- c 
- chanrecv2： 返回bool
- chanrecv：
- > c == nil，若select则返回false，否则gopark挂起g
- > 若select且chan为空，若chan未关闭，则返回false，false，若chan为空，则清空elem的指针，返回true，false
- > 加锁
- > 若chan 关闭且为空，解锁，清理elem，返回true，false
- > 否则 sendq出队，sudog不为nil，调用recv，返回true，true
- > 若c.qcount > 0，根据recvx获取buf指针，调用typedmemmove赋值buf值到elem，typedmemclr清理buf对应内存，c.recvx++ c.qcount--，返回true，true
- > 若select，解锁，返回false，false
- > 若没有待接收的elem，则获取sudog，赋值当前g，将sudog入recvq，调用gopark挂起g
- > 清理sudog参数并回收，清理g参数，返回true，true
- recv：
- > 若无缓冲chan，则recvDirect，从sender copy数据
- > 否则，获取recvx对应的指针，将数据copy给elem，将sender的数据copy到buf，c.recvx++
- > goready唤醒sender对应的g
- recvDirect：从sender dog cp数据到返回指针
## select 阻塞直到其中某个case就绪

### 结构
```
type scase struct {
	c    *hchan         // chan
	elem unsafe.Pointer // data element
}
```
### 函数
- selectnbsend： 对应select中的写case，调用chansend，block参数为false
- selectnbrecv： 对应select的读case，调用chanrecv，block参数为false
- selectnbrecv2：对应case v, ok = <-c:
- selectgo： 实现select状态转换;返回：返回选择的case的索引和是否读取到数据；入参：cas0指向[ncases]scase，order0指向[2*ncases]uint16，ncases<=65536
- > nsend和nrecv表示读和写的个数，相加=ncase,scase=cas0
- > pollorder对应order0[0:ncase]:控制遍历case的顺序
- > lockorder对应order0[ncase:2*ncases]：控制加锁顺序
- > 
