# compile and linker
- 创建二进制文件目录
- compile 工具打包和并发编译
- linker链接和打包二进制
- 二进制赋值和删除临时目录
- 二进制包大小对性能的影响:1.使cpu cache命中率下降；2.内存大小和加载慢
# compiler
- go tool compile -S -l main.go ： 查看编译时的指令地址和目标文件
- go tool objdump main.o： 解封装
- go tool nm main.o： 查看符号表
- -l：避免inline
- -x：打印编译信息
- -X：编译时设置符号表的值
- -w： 删除debug信息
## 编译器计算的值
- len
- cap
- unsafe.Sizeof
- unsafe.Alignof
## .o文件
- 符号以16进制fe开头
- 
# linker 打包外部依赖包生成
- 为所有section和指令分配虚拟地址空间
- objdump -h 二进制文件：查看section信息
- objdump -d：查看指令信息
- 在生成binary的最后阶段介入，最耗时的阶段
- linker包括：clang，gcc或者内部go连接器
- 内部linker：1.压缩dwarf信息；2引用c代码，打包插件，arm平台等不能使用
## 功能
- 打包外部依赖包生成
- 删除未使用的依赖
- 生成DWARF debug信息
- 生成pc表，符号表和函数表
- 重定向
## 重定向
- linker为外部符号重新分配地址
- 计算符号的整体偏移，即可定位指令的位置
## 引用
- https://medium.com/a-journey-with-go/go-goroutine-and-preemption-d6bc2aa2f4b7
- https://medium.com/a-journey-with-go/go-object-file-relocations-804438ec379b
- https://pkg.go.dev/cmd/internal/objabi
- https://pkg.go.dev/debug/dwarf
- https://dwarfstd.org/doc/dwarf-2.0.0.pdf
