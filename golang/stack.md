# 栈

## 定义：

- _StackCacheSize：32 * 1024
- stackpool ： 全局空闲栈池；linux下为长度为4的数组，依次缓存大小2k 4k 8k 16k的空闲span
- stackLarge： 大栈的池子
