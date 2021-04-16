# 编译问题

## : undefined reference to XXX @GLIBCXX_3.4.21'
一般为gcc或g++版本不一致导致的
以下命令可以查看so编译对应的gcc版本
```
strings *.so | grep  GCC

GCC_3.0
GCC: (Ubuntu 7.5.0-3ubuntu1~18.04) 7.5.0
_Unwind_Resume@@GCC_3.0
```


## 多版本gcc或g++切换

```
// 安装
apt install g++-5.4 
apt install gcc-5.4 

//设置
update-alternatives --install /usr/bin/g++ g++  /usr/bin/g++-5   100
update-alternatives --install /usr/bin/gcc gcc /usr/bin/gcc-5  100

//切换版本
update-alternatives --config gcc
update-alternatives --config g++

//重载库文件
ldconfig -v
```
