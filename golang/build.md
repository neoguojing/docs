# golang编译相关

## build选项

-tag

自定义 条件编译选项,只能备build命令识别
```
// +build dev

go build -tags dev
```

-ldtags

命令行传递编译参数

```
var version string

go build -ldflags '-X main.version="dev"'
```

## 默认条件编译
```
这个将限制此源文件只能在 linux/386或者darwin/386平台下编译
 // +build linux darwin  
 // +build 386  
 
 文件后缀限制
 mypkg_freebsd_arm.go // only builds on freebsd/arm systems  
 mypkg_plan9.go       // only builds on plan9  
```

- //go:nosplit : 表明函数不能在分裂栈上使用,不需要做栈溢出检查

## vender

vendor搜索方式 vendor包的搜索方式为：自包引用处，从其所在文件夹查询是否有vendor文件夹包含所引用包；若没有，然后从其所在文件夹的上层文件夹寻找是否有vendor文件夹包含所引用包，若没有，则再搜索上层文件夹的上层文件夹...，直至搜索至$GOPATH/src并搜索完成时止。

当vendor存在嵌套时，若不同的vendor文件夹包含相同的包，且该包在某处被引用，寻找策略仍遵循如上规则。即从包引用处起，逐层向上层文件夹搜索，首先找到的包即为所引，也就是从$GOPATH/src来看，哪个vendor包的路径最长，使用哪个

## mod

## 交叉编译

CGO_ENABLED=0 GOOS=linux GOARCH=amd64 go build hello.go

GOOS：目标操作系统

GOARCH：目标操作系统的架构

CGO_ENABLED=0 使用C语言编译器
CGO_ENABLED=1，CGO编译器
