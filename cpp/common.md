# 通用知识

## 信号处理
- SIGPIPE   tcp通道单向关闭，协议栈返回rst，linux产生   signal(SIGPIPE, SIG_IGN); 屏蔽方法
- SIGINT    ctrl+c信号
- SIGSEGV   是当一个进程执行了一个无效的内存引用，或发生段错误时发送给它的信号
- SIG_IGN   忽略
- SIG_DFL   默认信号处理程序
- SIGABRT   多次free导致的SIGABRT、abort、assert

## move
 解决问题：临时变量的copy
 
  - func("some temporary string"); //初始化string, 传入函数, 可能会导致string的复制

  - v.push_back(X()); //初始化了一个临时X, 然后被复制进了vector

  - a = b + c; //b+c是一个临时值, 然后被赋值给了a

  - x++; //x++操作也有临时变量的产生

  - a = b + c + d; //c+d一个临时变量, b+(c+d)另一个临时变量
  
(这些临时变量在C++11里被定义为rvalue, 右值, 因为没有对应的变量名存他们)

(同时有对应变量名的被称为lvalue, 左值)

### 语义
  - (1)成员变量内部的指针指向"temporary str1"所在的内存

  - (2)临时变量内部的指针指向成员变量以前所指向的内存

  - (3)最后临时变量指向的那块内存再被回收
  ```
  void set(string && var1, string && var2){
    //avoid unnecessary copy!
    m_var1 = std::move(var1);  
    m_var2 = std::move(var2);
  }
  A a1;
  //temporary, move! no copy!
  a1.set("temporary str1","temporary str2");
  ```
  
## forward

```
template<typename T1, typename T2>
void set(T1 && var1, T2 && var2){
  m_var1 = std::forward<T1>(var1);
  m_var2 = std::forward<T2>(var2);
}

//when var1 is an rvalue, std::forward<T1> equals to static_cast<[const] T1 &&>(var1)
//when var1 is an lvalue, std::forward<T1> equals to static_cast<[const] T1 &>(var1)

```
##　ptr

- shared_ptr

只要将 new 运算符返回的指针 p 交给一个 shared_ptr 对象“托管”，就不必担心在哪里写delete p语句——实际上根本不需要编写这条语句，
托管 p 的 shared_ptr 对象在消亡时会自动执行delete p。而且，该 shared_ptr 对象能像指针 p —样使用，
即假设托管 p 的 shared_ptr 对象叫作 ptr，那么 *ptr 就是 p 指向的对象。
