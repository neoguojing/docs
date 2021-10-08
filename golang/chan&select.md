# chan和select

- nil chan写入永久阻塞
- closed的chan写入 panic
- waitq从队列头出队
- 

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
- > 若qcount< qcount,表示有buf未满：获取sendx位置的指针，调用typedmemmove为该位置赋值，sendx++，qcount++,返回true
- > 若是select，则直接返回false
- > 无法直接写入，则acquireSudog获取sudog，填充elem和g等，并设置g.waiting=sudog,gp.param=nil,将sudog放入sendq
- > 调用gopark，阻塞g,并在调用过程中解锁chan
- > KeepAlive??
- > 被唤醒，清空g的标志，是否sudog
- send：recvq的sudog和elem的值
- > 若sudog的elem不为nil，调用sendDirect，并设置elem为nil
- > 获取sudog的g，设置g.param=sudog,设置sudog.success=true,调用goready唤醒g
- sendDirect：调用typeBitsBulkBarrier对所有指针执行内存屏障，调用memmove，将elem放到sudog的elem地址
- acquireSudog:获取或新建一个sudog
- > 优先从p.sudogcache获取
- > p.sudogcache不够，则从sched.sudogcache获取
- > 否则new一个
- closechan: 关闭chan
- > 设置closed=1
- > 循环遍历出队c.recvq，nil则break: 设置gp.param，设置sg.succes=false，将sudog对应的g加入glist
- > 遍历出队 c.sendq，直到nil：同上
- > 遍历glist，依次调用goready，唤醒g
## select
