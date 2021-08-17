# 调度

## 启动流程schedinit

- 初始化锁
- moduledataverify： 静态文件检查
- stackinit 栈初始化
- mallocinit 分配器初始化
- fastrandinit : 生产随机数，从linux：AT_RANDOM获取或者从系统/dev/urandom\x00中读取
- mcommoninit ：初始化g的m
- cpuinit ： cpu初始化
- alginit： 算法初始化，AES
- modulesinit： 模塊初始化，主要用于动态库和plugin模式
- typelinksinit： 模块类型map初始化
- itabsinit ： 模块？？？
- sigsave ：保存当前线程的信号到p中
- goargs/goenvs： 初始化arg和env
- parsedebugvars： 解析debug参数，如madvdontneed等
- gcinit： gc初始化
