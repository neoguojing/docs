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


## 调度参数：
- sched.maxmcount = 10000： 最大m个数，默认10000
- m.spinning：m未找到一个可运行的p，在空转
- sched.nmspinning：在空转的m个数

## 重要函数
- wakep:唤醒一个p去执行g
- startm：调度一个m去运行p，必要时创建一个新m
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
- gfget：从p.gFree获取空闲g，否则schedt.gFree获取g
- malg ：创建新的g，参数为栈大小；新建g对象，调用systemstack分配栈空间
- func newproc(siz int32, fn *funcval)： 进程启动时执行，再系统栈创建一个正在运行的g；
- > 获取参数指针：&fn+64；获取g，获取pc
- > 系统栈：调用newproc1创建新的g，调用runqput将g放入p
- func newproc1(fn *funcval, argp unsafe.Pointer, narg int32, callergp *g, callerpc uintptr) *g ：
- > 调用gfget获取可重用g，失败则调用malg，调整状态为_Gdead，添加到allgs列表
- > 将参数放入栈帧，构建stkmap，计算sp，栈顶指针
- > 初始化g：newg.sched.sp=newg.stktopsp，newg.sched.pc调整为指向goexit的pc，newg.sched.g为newg，newg.gopc为上一级函数pc，newg.ancestors：祖先信息，newg.startpc为当前函数pc
- > 切换状态为_Grunnable，设置goid，返回newg

## p
### 函数
- runqput： 将g放入p末尾，p满了，则放入全局p
- 
