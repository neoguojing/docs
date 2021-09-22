# panic/defer/recover

## 架构
```
panic只出现在栈上

type _panic struct {
	argp      unsafe.Pointer // 
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

- gorecover ： recover关键字的实现,必须在defer中
- > 获取当前g的panic,g未退出，recovered=false，argp 指向最上层的defer的参数，则recovered=true，返回panic指针
- > 否则返回nil
- gopanic：panic关键字的实现
- > 创建一个panic，设置arg，link指向g当前的panic；重新赋值g.panic
- > addOneOpenDeferFrame ??
- > 循环：
- > gp._defer == nil 则退出循环
- > 
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
- freedefer： 在堆上无需释放；可能归还一半defer到全局pool；defer放入p-per缓冲
