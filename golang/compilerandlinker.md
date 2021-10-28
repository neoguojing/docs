# compiler
- go tool compile -S -l main.go ： 查看编译时的指令地址和目标文件
- go tool objdump main.o： 解封装
- go tool nm main.o： 查看符号表
- -l：避免inline
## 编译器计算的值
- len
- cap
- unsafe.Sizeof
- unsafe.Alignof
## .o文件
- 符号以16进制fe开头
- 
# linker
- 为所有section和指令分配虚拟地址空间
- objdump -h 二进制文件：查看section信息
- objdump -d：查看指令信息
## 重定向
- linker为外部符号重新分配地址
- 计算符号的整体偏移，即可定位指令的位置
## 引用
- https://medium.com/a-journey-with-go/go-goroutine-and-preemption-d6bc2aa2f4b7
- https://medium.com/a-journey-with-go/go-object-file-relocations-804438ec379b
- https://pkg.go.dev/cmd/internal/objabi
