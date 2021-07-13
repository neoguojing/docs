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


## runtime 

- src/runtime/malloc.go   mallocgc等内存分配函数
- src/runtime/runtime2.go gmp实现函数
- src/runtime/mheap.go 堆内存实现函数
- src/runtime/proc.go 运行时入口: schedinit
- src/runtime/lockrank.go 运行时锁的排序定义
