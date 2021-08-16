# 栈

## 定义：

- _StackCacheSize：32 * 1024
- stackpool ： 全局空闲栈池；linux下为长度为4的数组，依次缓存大小2k 4k 8k 16k的空闲span
- stackLarge： 大栈的池子保存48-13个大小的spanlist？
- FixedStack: linux下为2kb
- 可缓存的空闲栈大小为:2kb\4kb\8kb\16kb,大于16kb的栈则直接分配
