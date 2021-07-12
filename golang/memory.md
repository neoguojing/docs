# memory

## 概念
- small 分配器处理小于32kb的70个大小的内存分类
- 页可以被分裂为一个固定大小的许多对象的集合
- GOOS: 操作系统- darwin, freebsd, linux等
- GOARCH: 架构-386, amd64, arm, s390x 等

## 分配器

- fixalloc: 用于分配非堆对象,这些对象是分配器本身要使用的内存
- mheap: golang的堆, 以page(8kb)为单位管理内存 
- mspan: 堆中一系列在使用的内存
- mcentral: 固定大小的mspan集合
- mcache: 有空闲的mspan线程本地缓存
- mstats: 统计内存分配

## 各架构内存bit数
- 符号扩展: 有符号数短数扩展到长数,扩展部分的值为之前数的最高位
- zero扩展: 无符号短数扩展为长数
- amd64 : 48必填地址位,有符号扩展到64; 支持57bit,当前仅linux 支持
- arm64 : 48bit地址
- ppc64, mips64, and s390x 支持64bit寻址


## 参数
- FixedStack: linux下为2kb
- 可缓存的空闲栈大小为:2kb\4kb\8kb\16kb,大于16kb的栈则直接分配
