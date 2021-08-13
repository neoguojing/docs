# 锁

## 原子操作：
- lock指令： lock前缀的指令，会锁cpu总线
- xchg指令： 原子操作，将新值存入变量，并返回旧值；锁内存总线
- 原子类型： int类型，禁止寄存器缓存
- cmpxchg： 变量值与旧值比较，相同则用新值覆盖，并返回旧值
- barrier()：空操作，告诉编译器这里有一个内存屏障，放弃在寄存器中的暂存值，重新从内存中读入

## 系统调用或汇编
- runtime·osyield：linux系统调用sched_yield，主动放弃cpu，当前进程或者线程进入调度队列等待重新调度
- runtime·procyield： 执行30次pause指令；pause提醒cpu代码是个循环等待，避免循环代码导致的可能的内存顺序违规导致的性能下降；降低耗电；执行一个预定义的延迟；

## 运行时锁


## 用户锁


## 引用
https://blog.csdn.net/sydhappy/article/details/115500346
