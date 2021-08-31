# 调度
# trace
- 调用栈跟踪：level分为0，1，2，all 和crash
- gotraceback 设置调用栈级别

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
- sched.maxmcount = 10000： 最大m个数，默认10000
- m.spinning：m未找到一个可运行的p，在积极寻找一个work
- sched.nmspinning：在空转的m个数
- sched.pidle: 空闲的p，npidle统计个数
- sched.midle：空闲的m，nmidle控制个数
- 

## 重要函数
- notesleep
- notewakeup
- wakep:唤醒一个p去执行g
- schedule()
- execute(): 调度g在当前m执行
- sysmon()
- main: 必须在m0上执行
- > 设置栈全局变量,
- > 调用newm，执行sysmon()函数
- > lockOSThread，设置m0参数和全局参数
- > doInit: 调用init函数
- > 调用gcenable: 启动bgsweep 和bgscavenge
- > doInit: 调用main init函数
- > 执行自定义main函数
- > 处理runningPanicDefers：Gosched
- > 处理panicking:  gopark
- dolockOSThread：锁死m和g，相当于对_g_.m.lockedg 和_g_.lockedm同时设置值
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

## g

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
- gopark： 暂停go
- > 禁止抢占，获取g状态，设置mp.waitlock 和mp.waitunlockf等
- > 恢复抢占，调用mcall(park_m)，切换到g0执行
- park_m：g0执行体
- > casgstatus切换g的状态由_Grunning到_Gwaiting
- > dropg：设置g_.m.curg.m=nil，g_.m.curg=nil，解除g和m的绑定关系
- > waitunlockf不为空，则执行waitunlockf，返回false，则切换状态为_Grunnable，执行execute，
- > 否则，调用schedule()
## p
### 函数
- pidleput： 将p放入空闲列表
- runqput： 将g放入p末尾，p满了，则放入全局p
- handoffp： 将p从执行系统调用或者加锁的m种解除 
- > runqempty不为空或者sched.runqsize不为0，调用startm调度一个m执行q
- > gcBlackenEnabled != 0 && gcMarkWorkAvailable(_p_) 调用startm调度一个m执行q
- > 没有空闲的m， 调用startm调度一个m执行q
- > ？？

## m

### 结构体
- newmHandoff： m的列表，列表里的m都没有绑定os thread，通过 m.schedlink构建列表
- m.freeWait:0则表示可以安全停止g0和释放m
- m.park: 暂停等待的指针

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
- > mp.g0.stack.hi 获取栈顶指针
- > 调用系统调用clone创建线程
- allocm: 分配一个m，不绑定任何thread
- > 当前g.m.p == 0 ,则acquirep，临时借用一个p
- > sched.freem != nil,则释放m，回收系统栈stackfree
- > new(m) 和mcommoninit
- > mp.g0 初始化
- > releasep,释放m.p指向的p
- mstart: 进程启动时调用，启动m，开始执行调度;
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
- acquirep: 绑定p到m
- releasep: 解绑p和m
- mPark: 线程park自身的唯一途径；
- > 死循环： 调用notesleep休眠线程，调用mDoFixup
