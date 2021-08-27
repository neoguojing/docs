# 栈
## 栈原理
- 栈帧：函数执行的环境
- 从高地址向低地址延伸，向下生长
- 每个函数的每次调用，都有它自己独立的一个栈帧
- 寄存器ebp指向当前的栈帧的底部（高地址）
- 寄存器esp指向当前的栈帧的顶部（低址地）
```
函数调用大致包括以下几个步骤：
参数入栈：将参数从右向左依次压入系统栈中
返回地址入栈：将当前代码区调用指令的下一条指令地址压入栈中，供函数返回时继续执行（EIP寄存器的值，PC值）
代码区跳转：处理器从当前代码区跳转到被调用函数的入口处
栈帧调整：
具体包括保存当前栈帧状态值，已备后面恢复本栈帧时使用（EBP入栈）
将当前栈帧切换到新栈帧。（将ESP值装入EBP，更新栈帧底部）
给新栈帧分配空间。（把ESP减去所需空间的大小，抬高栈顶） 被调用函数可以根据sp的值计算参数的地址
```
![栈示意](https://pic2.zhimg.com/1314ce0c49d0a1e2800e23ca3d5cdd75_r.jpg?source=1940ef5c)

## 概念：
- pc： 程序计数器，CPU中用于存放下一条指令地址的寄存器
- sp: 栈帧的栈顶指针
- getcallerpc(): 返回调用该函数的函数的调用者的pc，编译器实现，及上级的上级函数
- getcallersp(): 返回调用该函数的函数的调用者的sp，编译器实现
## 定义：
- _StackCacheSize：32 * 1024
- stackpool ： 全局空闲栈池；linux下为长度为4的数组，依次缓存大小2k 4k 8k 16k的空闲span
- stackLarge： 大栈的池子保存48-13个大小的spanlist，缓存>=32k的空闲span
- stackCache：小于32的栈缓冲,0:缓存2k，1：缓存4k，2：缓存8k，3：缓存16k，缓存stack结构体
- _FixedStack0：linux 下为_StackMin 2048
- _FixedStack1： 2047
- _FixedStack2：2047
- _FixedStack3：2047
- _FixedStack4 2047
- _FixedStack5 2047
- _FixedStack6 2047
- FixedStack: linux下为2kb 2048
- 可缓存的空闲栈大小为:2kb\4kb\8kb\16kb,大于16kb的栈则直接分配
- _StackSystem: 在linux下面为0；windows不适用分裂栈，大小为512*64
- _StackMin ： 2k
- _StackGuard: 保护区大小，常量Linux上为928字节
- stackguard0: stack.lo+StackGuard,用于stack overlow的检测,设置为StackPreempt触发抢占
- stackguard1：在g0和gsignal 中是stack.lo+StackGuard，否则为~0
- StackPreempt： 1314触发抢占
- _NumStackOrders： 4
- maxstacksize: 1GB

## 结构体
```
type stack struct {
	lo uintptr
	hi uintptr //栈顶指针，最开始分配：函数参数大小+4*RegSiz，将参数填入 
}
```
## 重要函数
- stackalloc: 必须在系统栈上执行;不能栈分裂
```
1. 若debug.efence != 0 || stackFromSystem != 0 ，则从调用sysAlloc，从操作系统分配内存，返回stack结构体
2. 若分配的栈小于32kb，则计算order=log_2(size/FixedStack),
    若m不存在p或在gc期间，则直接从stackpool分配，调用stackpoolalloc(order)
    否则从p的本地mcache分配，分配成功则从stackcache[order].list拿掉该节点；分配失败，则调用stackcacherefill，填充stackcache
3. 若大于等于32k，计算页数（除以8192），计算log2npage=log2页数，
  若stackLarge[log2npage].free有空闲内存，则获取，并移除该列表;
  否则从mheap.allocManual分配空间，失败则抛出异常
4.返回分配的栈空间
```
- stackcacherefill：从stackpool获取空闲内存，调用stackpoolalloc，更新stackcache[order]的列表
- stackpoolalloc：根据stackpool[order]获取对应span;
- > 若span.first==nil，调用 mheap_.allocManual分配4页内存大小（32k）的mspan；更新mspan.manualFreeList;插入stackpool[order].item.span列表
- > 从span.manualFreeList获取内存，并更新manualFreeList和allocCount
- > 返回分配的span
- morestack_noctx ：
# 引用
- https://www.zhihu.com/question/22444939
