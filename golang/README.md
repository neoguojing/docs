# 源码导读 1.16

- AUTHORS：Golang官方作者清单
- CONTRIBUTING.md：加入贡献者队列的指导文件
- CONTRIBUTORS：第三方贡献者清单
- LICENSE：授权协议
- PATENTS：专利
- README.boringcrypto.md：因为Golang是Google发布的，这是针对Google内部研究分支的说明
- README.md：说明文件，大家都明白，每个开源库都有
- SECURITY.md：安全政策
- api：Golang每个版本的功能列表归档文件，下面有具体介绍
- doc：Golang文档说明，和官方文档相同，可以离线查看
- favicon.ico：浏览器页签左边的图标，一般放在网站根目录，就长这样
- file
- lib：看起来像是库文档模板，里面列举了time包的说明
- misc：汇集了Go语言相关的IDE、插件、cgo测试程序、示例等乱七八糟的东西
- robots.txt：主要用来控制各大搜索引擎爬虫的爬取规则
- src：Golang核心实现都在这里，下面详细讲述
- test：Golang单元测试程序，通过查看测试程序可以学习到golang的用法和特性

## 汇编

- SB(static base)：内存的地址，foo(SB)是一个由foo这个名字代表的内存地址
- FP(frame pointer)：0(FP)就是第一个参数，8(FP)就是第二个(64位机器)
## runtime 

- src/runtime/malloc.go   mallocgc等内存分配函数
- src/runtime/runtime2.go gmp实现函数
- src/runtime/mheap.go 堆内存实现函数
- src/runtime/proc.go 运行时入口: schedinit
- src/runtime/lockrank.go 运行时锁的排序定义

### runtime 启动流程 src/runtime/asm_amd64.s
- _rt0_amd64_linux
- _rt0_amd64
- rt0_go 
- 保存函数参数
- 保存g0到寄存器，并设置栈
- 发现系统cpu类型，设置相关参数
- cgo处理
- 设置tls
- 设置g0和m0
- runtime·osinit
- runtime·schedinit
- runtime·newproc(runtime.main)
- runtime·mstart -> runtime.schedule

### 常用算法
- func Ctz64(x uint64) int : 计算小端字节序末尾0的个数
- func TrailingZeros64(x uint64) int ：同上
### 常用汇编函数

> mcall, systemstack, asmcgocall是基于asm_arm64.s汇编文件
> 主要使用两个寄存器： SP ：存放栈顶地址   PC：下一个要执行的指令的地址
- func mcall(fn func(*g) ： 保存当前的g的上下文，切换到g0的上下文，传入函数参数，跳转到函数代码执行
- func systemstack(fn func()) ：切换到系统栈，由g0执行fn
- func asmcgocall(fn, arg unsafe.Pointer) int32 ： cgo代码执行器
- func syscall(fn, a1, a2, a3 uintptr) (r1, r2 uintptr, err Errno)  Syscall6： 系统调用
- func memclrNoHeapPointers(ptr unsafe.Pointer, n uintptr)： 清0 ptr开始的n字节数据，memclr_amd64.s


