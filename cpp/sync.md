# 同步 c++11

## 多个锁，要保证上锁的一致性，否则会死锁

## condition_variable

std::condition_variable 对象的某个 wait 函数被调用的时候，它使用 std::unique_lock(通过 std::mutex) 来锁住当前线程。当前线程会一直被阻塞，
直到另外一个线程在相同的 std::condition_variable 对象上调用了 notification 函数来唤醒当前线程。

std::condition_variable 对象通常使用 std::unique_lock<std::mutex> 来等待，如果需要使用另外的 lockable 类型，
可以使用 std::condition_variable_any 类

notify_one()

唤醒某个等待(wait)线程。如果当前没有等待线程，则该函数什么也不做，如果同时存在多个等待线程，则唤醒某个线程是不确定的(unspecified)。

## unique_lock　细粒度锁

支持lock unlock方法

默认上锁

析构是根据状态解锁

可以移动 不能复制

## lock_guard

提供自动为互斥量上锁和解锁

局部区域的保护

不可移动，不能复制

## std::mutex

用于保护共享数据不会同时被多个线程访问的类，它叫做互斥量
std::mutex，最基本的 Mutex 类。
std::recursive_mutex，递归 Mutex 类。
std::time_mutex，定时 Mutex 类。
std::recursive_timed_mutex，定时递归 Mutex 类。

## call_once
 std::once_flag oc; // global flag
 
 std::call_once(oc,initialize)
