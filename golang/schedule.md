# 调度
## 总结
- 启动流程： osinit -> scheduleinit -> newproc(runtime.main)
- main函数： 启动sysmon，执行init，启动gc，执行main.main
- schedule：挑选一个g运行：优先执行标记g，间歇的从本地全局runq获取g，从其他p偷g，
- execute：调度g在当前m上直接运行
- goschedImpl：设置g为_Grunnable，放入全局运行队列，执行schedule调度
- 启动时根据cpu，创建足够的p
- 模板线程：单独存在的，不绑定p；
### g相关
- gopark ：暂停当前g，切换g状态为_Gwaiting，解绑m和g，execute调度当前g或者schedule执行下一轮调度
- goready：系统栈设置g状态_Grunnable，g放入当前p的runnext使得下一轮调度，获取一个m取执行p
- dropg ： 设置g和m的参数，解绑g和m
- newproc ：创建和运行g
- malg：分配g
### p相关
- handoffp： 从系统调用或locked M中解放p，是p执行其他工作；运行队列，gc，netpoll等
- acquirep ： 绑定p和m，并清理mcache
- releasep： 解绑p和m，设置状态为_Pidle
- wakep：当g变为runnable时，运行一个p取执行g
- wirep(p) : 绑定当前g的p和m，设置p状态为_Prunning
- runqput： next为true则放入p.runnext,下一轮调度；否则放入队列尾部；队列满了，则放入全局队列

### m相关
- mPark ： 休眠m对应的线程，futex休眠
- mstart: 进程启动时设置g0，初始化信号量等
- stopm：将m放入空闲队列，并休眠m，唤醒后绑定p和m
- startlockedm：调用锁定的m取运行锁定的g，主动释放p
- stoplockedm：休眠m，唤醒后绑定nextp与m
- acquirem/releasem：对m.locks加一或者减一
- startm：从空闲m队列或新建一个m与p绑定（nmp.nextp=p），并唤醒m
- gcstopm：解绑和休眠当前m，通常用于stw的目的
## trace
- 调用栈跟踪：level分为0，1，2，all 和crash
- gotraceback 设置调用栈级别

## 流程：

- mstart：主要是设置g0.stackguard0，g0.stackguard1。
- mstart1：调用save保存callerpc和callerpc到g0.sched。然后调用schedule开始调度循环。
- schedule：获得一个可执行的g。下面用gp代指。
- execute(gp *g, inheritTime bool)：绑定gp与当前m，状态改为_Grunning。
- gogo(buf *gobuf)：加载gp的上下文，跳转到buf.pc指向的函数。
- 执行buf.pc指向函数。
- goexit->goexit1：调用mcall(goexit0)。
- mcall(fn func(*g))：保存当前g（也就是gp）的上下文；切换到g0及其栈，调用fn，参数为gp。
- goexit0(gp *g)：清零gp的属性，状态_Grunning改为_Gdead；dropg解绑m和gp；gfput放入队列；schedule重新调度

## 全局变量
- allm ：m列表的头
- allgs：全局g数组
- mainStarted：main对应的m是否启动，一般在runtime.main函数中设置为true
- m0：
- g0：
- RegSize：8
- MinFrameSize： 0；最小栈帧
- SpAlign： 1
- PCQuantum ：1
- mainStarted: main对应的M已经启动
- runningPanicDefers: 非0，表示正在执行panic的defer
- panicking：非0，程序崩溃因为一个不可恢复的panic


## 调度参数：
- maxmcount = 10000： 最大m个数，默认10000
- m.spinning：m未找到一个可运行的p，在积极寻找一个work
- nmspinning：在空转的m个数
- pidle: 空闲的p，npidle统计个数
- midle：空闲的m，nmidle控制个数
- nmidlelocked： 被g锁定的等待运行的m的个数
- lastpoll: 0 表示正在调用netpoll

## 重要函数
- Gosched：不会挂起当前g，只是放入全局queue：切换到g0，执行gosched_m；
- gosched_m：调用goschedImpl
- goschedImpl： 
- > g的状态切换到_Grunnable
- >dropg 解绑g和m
- > 将g放入全局queue
- > 调用schedule
- globrunqget：从全局runq获取g
- checkTimers: 定时器检查
- checkdead()：检查死锁
- notesleep
- notewakeup
- wakep(p):唤醒一个p去执行g，修改m和p的绑定关系：_g_.m.p = p，  _p_.m.set(_g_.m)，_p_.status = _Prunning
- acquirep(*p): 绑定p到m,调用wakep，调用mcache.prepareForSweep尝试清理
- releasep(*p): 解绑p和m: _g_.m.p = 0， _p_.m = 0 ，_p_.status = _Pidle
- schedule(): 一轮调度，找到一个g并执行，永远不返回
- > 获取当前g
- > 若当前g与m绑定，则执行stoplockedm停止m，调用execute调度当前g去执行，永不返回
- > sched.gcwaiting != 0，则停止当前m，重新调度
- > 检查定时器checkTimers，定义待调度gp
- > 在标记阶段，则获取一个对于当前p的标记线程gp
- > 若gp==nil，则每_g_.m.p.ptr().schedtick%61 == 0 从全局runq获取一个g
- > 若gp==nil,从本地runq获取一个g
- > 若gp==nil，findrunnable获取g
- > _g_.m.spinning,resetspinning，启动一个新的spinning m
- > 判断gp是否可被调度，sched.disable.user，否则放入待调度队列，重新调度
- > gp和某个m锁定，startlockedm调度m去执行锁定的g，重新调度
- > 调用execute
- execute(): 调度g在当前m执行，设置g的参数，调用gogo
- func gogo(buf *gobuf):汇编：加载newg的上下文，跳转到gobuf.pc指向的函数
- findrunnable： 从其他p获取g，从本地和全局runq获取g，从poll获取g
- startlockedm：调度m去执行锁定的g
-> 
- stoplockedm： 停止当前锁定一个g的m，直到g可运行
- > 解除m绑定的p，mPark休眠自身，唤醒后调用acquirep
- sysmon()
- main: 必须在m0上执行
- > 设置栈全局变量,
- > 调用newm，执行sysmon()函数
- > lockOSThread，设置m0参数和全局参数，锁定main g到m
- > doInit: 调用init函数
- > 调用gcenable: 启动bgsweep 和bgscavenge
- > doInit: 调用main init函数
- > 执行自定义main函数
- > 处理runningPanicDefers：Gosched
- > 处理panicking:  gopark
- dolockOSThread：锁死m和g，相当于对_g_.m.lockedg 和_g_.lockedm同时设置值
- goexit：汇编函数，rerurn是写入调用栈，调用goexit1
- goexit1：切换到g0，执行goexit0
- goexit0：设置g的参数，调用dropg解绑g和m，调用gfput放置g到空闲列表，调用schedule()
## 启动流程schedinit

- 初始化锁
- moduledataverify： 静态文件检查
- stackinit 栈初始化
- mallocinit 分配器初始化
- fastrandinit : 生产随机数，从linux：AT_RANDOM获取或者从系统/dev/urandom\x00中读取
- mcommoninit ：初始化g的m
- cpuinit ： cpu初始化
- alginit： 算法初始化，AES
- modulesinit： 模塊初始化，主要用于动态库和plugin模式
- typelinksinit： 模块类型map初始化
- itabsinit ： 模块？？？
- sigsave ：保存当前线程的信号到p中
- goargs/goenvs： 初始化arg和env
- parsedebugvars： 解析debug参数，如madvdontneed等
- gcinit： gc初始化

## mcommoninit 启动m初始化
- 若当前g不等于g.m.g0,则callers-> gentraceback？？
- sched.lock 上锁
- 为m分配id
- m.fastrand初始化
- mpreinit： 创建新的g，更新m.gsignal
- 将当前m挂到allm列表上
- 若是cgo调用，则更新mp.cgoCallers

## sudog 

### 函数
- acquireSudog ： 申请sudog
- > acquirem
- > pp.sudogcache 为空则尝试从schedt.sudogcache获取cap/2的sudog，否则new(sudog)挂到p的sudogcache
- > 从 pp.sudogcache末尾取一个sudog返回
- releaseSudog ： 释放sudog
## g

### 变量
- lockedm： 锁定的m，dolockOSThread有调用
- startpc： g 函数执行地址
- g.param: 用于在gopark和goready之间传递数据，如sudog

### 函数：
- getg： 从TLS（线程本地缓存）获取当前运行的指向g结构体的指针
- gfget：从p.gFree获取空闲g，否则schedt.gFree获取g
- malg ：创建新的g，参数为栈大小；新建g对象，调用systemstack分配栈空间
- func newproc(siz int32, fn *funcval)： 进程启动时执行，再系统栈创建一个正在运行的g，执行runtime.main；
- > 获取参数指针：&fn+64；获取g，获取pc
- > 系统栈：调用newproc1创建新的g，调用runqput将g放入p
- func newproc1(fn *funcval, argp unsafe.Pointer, narg int32, callergp *g, callerpc uintptr) *g ：
- > 调用gfget获取可重用g，失败则调用malg，调整状态为_Gdead，添加到allgs列表
- > 将参数放入栈帧，构建stkmap，计算sp，栈顶指针
- > 初始化g：newg.sched.sp=newg.stktopsp，newg.sched.pc调整为指向goexit的pc，newg.sched.g为newg，newg.gopc为上一级函数pc，newg.ancestors：祖先信息，newg.startpc为当前函数pc
- > 切换状态为_Grunnable，设置goid，返回newg
- goparkunlock：对gopark的封装，强调了解锁操作
- gopark： 暂停go ，并释放g获得的锁，同goparkunlock
- > 禁止抢占，获取g状态，设置mp.waitlock 和mp.waitunlockf等
- > 恢复抢占，调用mcall(park_m)，切换到g0执行
- park_m：g0执行体
- > casgstatus切换g的状态由_Grunning到_Gwaiting
- > dropg：设置g_.m.curg.m=nil，g_.m.curg=nil，解除g和m的绑定关系
- > waitunlockf不为空，则执行waitunlockf，释放g获取的锁，返回false，则切换状态为_Grunnable，执行execute，
- > 否则，调用schedule()
-  goready(gp)：系统栈调用ready
- func ready(gp *g, traceskip int, next bool) ：
- > 设置当前g为_Grunnable
- > runqput放入本地p
- > wakep尝试调度
## p

### 变量：
- preempt：false表示不参与调度
- _Pidle = iota
- _Prunning
-	_Psyscall
- _Pgcstop
-	_Pdead
-	
### 函数
- pidleput： 将p放入空闲列表
- runqput： 将g放入p末尾，p满了，则放入全局p
- handoffp： 将p从执行系统调用或者加锁的m种解除 ，会尝试调用startm用一个新的m去运行p
- > p的runqempty不为空或者sched.runqsize不为0，调用startm调度一个m执行q
- > gcBlackenEnabled != 0 && gcMarkWorkAvailable(_p_) 调用startm调度一个m执行q
- > 没有空闲的m和p， 调用startm调度一个m执行q
- > gc需要stw，则停止p（_Pgcstop）
- > p在等待sched.safePointFn函数的执行，则执行sched.safePointFn，尝试唤醒sched.safePointNote
- > 全局q队列不为空，调用startm运行p
- > 当前p为最后一个在运行的p，且当前没有g在调用netpoll，则需要唤醒一个m运行当前p，进行netpoll调用
- > pidleput，将p放入空闲列表
- > wakeNetPoller:调用netpollBreak唤醒运行netpoll的线程
- wakep()
- > sched.npidle没有空闲p则返回
- > 调用startm，调度一个m运行p
- acquirep:将p和当前m绑定，调用wirep
- releasep：解绑p和当前的m
- wirep：设置m.p为穿入p，设置p.m为当前m，设置p为_Prunning
- 
## m
### 状态
- spinning： 空转状态，本地工作结束，没有从global工作队列和netpoller获取到工作；
- park： 若处于spinning的未找到work，则park自己

### 结构体
- newmHandoff： m的列表，列表里的m都没有绑定os thread，通过 m.schedlink构建列表
- freeWait:0则表示可以安全停止g0和释放m
- m.park: 暂停等待的指针
- lockedg: 保存g的指针，在dolockOSThread有使用，明确m加锁的g
- preemptoff: !="" 则不允许抢占
- spinning：m是否处于空转状态
- blocked： 线程被notesleep睡眠了

### 重要函数
- startm：调度一个m去运行p，必要时创建一个新m
- > 加锁，防止抢占
- > 传入的p==nil.尝试从sched.pidle获取一个p，依旧为nil，则返回
- > 尝试从sched.midle获取一个空闲m，若m==nil，则调用newm，创建一个m
- > 设置新m的spinning，nextp，并调用notewakeup（futexwakeup），唤醒m
- newm: (fn func(), _p_ *p, id int64) ：
- > allocm分配一个m
- > 设置doesPark=true,nextp绑定p，sigmask设置
- > newosproc为m创建线程
- func newosproc(mp *m)：创建操作系统线程
- > mp.g0.stack.hi 获取栈底指针
- > 调用系统调用clone创建线程，运行mstart
- allocm: 分配一个m，不绑定任何thread
- > 当前g.m.p == 0 ,则acquirep，临时借用一个p
- > sched.freem != nil,则释放m，回收系统栈stackfree
- > new(m) 和mcommoninit
- > mp.g0 初始化
- > releasep,释放m.p指向的p
- mstart: m启动时调用，启动m，开始执行调度;
- > 获取g，设置stack信息,当前g为g0
- > 调用mstart1
- > mexit
- mstart1:
- > 获取g，当前g为g0,保存调用者的返回pc和栈顶指针
- > 调用minit，minitSignals初始化信号量，为启动m设置tid（线程id）
- > 若是m0，则调用mstartm0，初始化信号量initsig
- > schedule()
- minitSignals:minitSignalStack 设置信号量栈；minitSignalMask：调用系统调用sigprocmask，设置信号掩码
- mexit： 退出当前m
- > 若是m0，handoffp，解除p与m的绑定，mPark休眠 线程
- > 否则，释放信号栈，将m从allm列表删除，m挂到freem，handoffp解除p绑定，退出线程，然后设置freeWait=0
- mPark: 线程park自身的唯一途径；
- > 死循环： 调用notesleep休眠线程，调用mDoFixup
- stopm： 停止当前m直到一个新的work就绪
- mput ： m放入sched.midle
- startlockedm: 调度锁定的m去执行锁定的g，解绑p，赋值mp.nextp=p，唤醒m.park,stopm停止m
### 模板线程m
```
<!-- 被newm使用，用于在os线程无法安全创建线程的时候 -->
var newmHandoff struct {
	lock mutex
	newm muintptr //指向一个m结构体列表
	waiting bool //表示需要唤醒
	wake    note
	haveTemplateThread uint32 //1表示templateThread已经启动
}
```
- startTemplateThread：1.main中使用cgo的时候，用于在c环境下创建线程；2.LockOSThread：启动一个
- > 启动newm(templateThread, nil, -1)
- templateThread：用于启动其他os线程，在其他线程状态不对的情况下
- > 增加一个系统线程计数，检查死锁
- > 进入循环：
- > 从newmHandoff.newm获取所有m，调用newm1为m创建os线程
- > 调用notesleep，休眠templateThread
## 引用
- https://blog.csdn.net/zdy0_2004/article/details/106392885
