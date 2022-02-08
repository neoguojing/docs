# panic/defer/recover

## 概要
- recover函数会校验是否存在defer
- panic函数执行后，会挂载panic结构体到g，同时遍历g的defer，依次执行defer，中间遇到recover函数则设置标记为true，切换到系统栈调用gogo切换到defreturn函数继续执行；否则打印panic信息，退出进程
- 编译器在每个函数末尾植入1个或多个deferreturn（个数和defer个数相同），检测是否有defer需要执行
- panic发生时，需要记录当前栈帧的pc和sp，一遍在recover之后恢复到正常流程；pc实际指向当前函数末尾的runtime.deferreturn；
- panic函数后面的代码不会被执行；panic在deferreturn前面被执行
- Goexit: 会触发panic，但是设置一个标记goexit = true，以便运行时不会执行recover
- defer改写return结果：如下返回值时一个已命名的参数
```
这种情况下，defer会改写返回值
func namedParam() (x int) {
   x = 1
   defer func() { x = 2 }()
   return x
}
```
## 架构

panic只出现在栈上
```
type _panic struct {
	argp      unsafe.Pointer // 指向defer参数的指针
	arg       interface{}    //panic参数
	link      *_panic        // 链接之前的panic
	pc        uintptr        //程序指针
	sp        unsafe.Pointer // 栈指针
	recovered bool           // panic是否结束
	aborted   bool           // panic是否被丢弃
	goexit    bool 
}

type _defer struct {
	siz     int32 // includes both arguments and results
	started bool
	heap    bool
	openDefer bool
	sp        uintptr  // sp at time of defer
	pc        uintptr  // pc at time of defer
	fn        *funcval // can be nil for open-coded defers
	_panic    *_panic  // panic that is running defer
	link      *_defer

	// If openDefer is true
	fd   unsafe.Pointer // funcdata for the function associated with the frame
	varp uintptr        // value of varp for the stack frame
	framepc uintptr
}
```
- gp._panic
- p.deferpool
- sched.deferpool
## 函数
- panicIndex：数组越界触发的panic
- recovery：处理gorecover函数执行后恢复正常流程的问题
- gorecover ： recover关键字的实现,必须在defer中
- > 获取当前g的panic,g未退出，recovered=false，argp 指向最上层的defer的参数，则recovered=true，返回panic指针
- > 否则返回nil
- gopanic：panic关键字的实现
- > 创建一个panic，设置arg，link指向g当前的panic；插入g.panic列表头部（执行panic函数，就在g的panic头部加入新的panic）
- > addOneOpenDeferFrame ??
- > 循环：遍历g.defer
- > gp._defer == nil 则退出循环
- > 若d.started，表示该defer已经开始执行了，则设置d._panic.aborted = true，设置gp._defer = d.link，释放当前defer,遍历下一个defer
- > 否则设置started=true，设置d._panic为当前panic，调用reflectcall执行defer函数体，gp._defer = d.link，释放当前defer
- > 若p.recovered=true（defer中有recover函数，gorecover已经执行），则调用recovery执行gogo继续调度,不返回
- > 循环结束,表示没有recover
- > 打印panic信息，调用fatalpanic，退出进程
- fatalpanic : 不可恢复panic实现
- > 系统栈调用，startpanic_m，返回true则打印panic信息，调用dopanic_m打印栈信息
- > 系统调用exit 退出进程
- startpanic_m：处理panic下的m的状态
- > m加锁，阻止递归panic
- > g_.m.dying为0，则设置值为1，调用freezetheworld进行stw，返回true
- > g_.m.dying为1,返回false
- > 否则直接调用exit，退出
- dopanic_m： 打印栈信息
-  
- reflectcall： 使用传入的参数列表，调用一个函数，汇编实现
- freedefer： 在堆上无需释放；可能归还一半defer到全局pool；defer放入p-per缓冲
- deferreturn: 编译器插入该函数到任何有defer的函数末尾；在函数执行完毕，会执行该函数
- > 检测是否存在defer函数，若存在，则调用jmpdefer，执行defer
- jmpdefer: 跳转到deffer函数执行
- newdefer: 分配一个defer
- deferproc：defer关键字的实现
- > 创建一个defer，pc和sp指向调用者的pc和sp
- > defer函数参数紧跟在defer结构体后面

## 引用
- https://blog.csdn.net/u010853261/article/details/102761955
- https://blog.csdn.net/shidantong/article/details/106341159
