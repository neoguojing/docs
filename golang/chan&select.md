# chan和select

## 总结
- nil chan写入/读取永久阻塞
- closed的chan写入 panic
- waitq从队列头出队
- 关闭nil或closed的chan会panic
- 关闭chan：将recvq和sendq的g，放入列表，依次调用goready唤醒
- makechan：对于没有指针的chan，只分配一次内存，否则分配两次
- send：总是优先从recvq获取g，直接发送数据；否则写入buf；buf满才挂起
- recv： 优先从sendq的g获取数据；否则从buf获取；buf空则挂起
- select使用的chan是非阻塞的，revc和send不需要执行gopark暂停当前g
- send中唤醒recvq中的g，recv唤醒sendq中的g，send和recv会唤醒select的g
- select：加锁按照lockorder正向遍历加锁，解锁则反向遍历解锁
- select：lockorder按照hchan的地址从小到大顺序排序
- select流程：
- > 1.优先处理已经挂起的操作，即在recvq和sendq中挂起的sudog，每次一个，成功返回
- > 2.若select非阻塞，可能是default，则解锁，索引为-1，执行返回动作
- > 3.以lockorder将所有的case，通过sudog建立列表（isSelect为true），并挂到sendq/recvq，现在所有case都放在了等待队列里,可以被1处理，gopark挂起go，挂起之后会解锁
- > 4.select g被send或者recev等操作唤醒，并会在g.param中加入对应的sudog，加锁，以lockorde遍历所有sudog出队，解锁，返回被唤醒的case的索引，
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
- waitq.dequeue:从队列头取一个sg，若是在select中要判断sg的isSelect和竞争sgp.g.selectDone，竞争失败则continue
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
- > nsend和nrecv表示读和写的个数，相加=ncase,scases=cas0
- > pollorder对应order0[0:ncase]:控制遍历case的顺序，值大于nsend为recv case否则为send case
- > lockorder对应order0[ncase:2*ncases]：控制加锁顺序
- > 遍历scases，过滤nil的hchan，通过随机函数计算新的pollorder,norder<=ncases
- > 遍历lockorder，堆排序，按照hchan的地址大小排序，顺序存储于lockorder
- > sellock对所有chan按照lockorder，加锁
- > 遍历pollorder：
- > 值大于nsends，表示为recv chan阻塞，尝试从sendq出队sudog，不为nil，则跳转的recv，若c.qcount > 0，则跳转到bufrecv，若 c.closed != 0 ，跳转到rclose
- > 否则，分辨跳转到sclose，send和bufsend
- > 非阻塞，则解锁selunlock，索引置为-1，跳转搭到retc
- > 按照lockorder，为每个case创建sudog，分别加入sendq和recvq，并在gp.waiting 构建sudog列表
- > gopark 挂起当前g
- > 被唤醒：sellock加锁,获取唤醒g的sudog
- > 切断gp.waiting的sudog列表
- > 遍历lockorder，出队所有sudog，返回唤醒的chan的case索引
- > bufrecv:从buff recv，从buff取出数据付给scase.elem，selunlock,跳转到retc
- > bufsend:从scase.elem，放入buff,解锁，跳转到retc
- > recv:调用recv，从sender sg获取data写入scase.elem.跳转到retc
- > rclose: 从关闭的chan 读数据，解锁，清理scase.elem,跳转到retc
- > send: 调用send，将scase.elem发送给receiver的sg
- > retc:返回case的索引和recvOK
- >sclose: 解锁，panic，表示向关闭的chan 发送数据
- sellock：对scases按照lockorder加锁,正序遍历
- selunlock：倒叙遍历lockorder，依次解锁hchan.lock
