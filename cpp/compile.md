# 编译问题

## : undefined reference to XXX @GLIBCXX_3.4.21'
gcc版本老导致
一般为
strings /usr/lib/libstdc++.so.6 | grep GLIBCXX


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
```
