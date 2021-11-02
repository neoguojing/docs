# 技巧
## golang 包
- runtime.SetFinalizer :为变量设置析构函数，在gc回收对象时执行
-> 不保证在程序退出时执行
-> 会影响到系统性能
-> finalizer在一个g中串行执行
-> runtime.SetFinalizer(p, nil):主动移除finalizer
- runtime.KeepAlive: 保证在该函数调用之前，gc不会回收相关变量；通过编译器嵌入代码保证gc不会回收

## 日志包 
- ZAP：有点：1.避免interface，强类型；2.避免json编码
- Zerolog：性能更强
## ORM
- ent
## fuzz 测试
- https://github.com/google/gofuzz
## 聚合g中的多个错误
-  go-multierror 效率更高
-  multierr
## 一个错误，多个g
- errgroup
## 2d游戏渲染库
- https://ebiten.org/
## 别名
```
type T2 struct {}

// T1 is deprecated, please use T2
type T1 = T2

func main() {
   var t T1
   f(t)
}

func f(t T1) {
   // print main.T2
   println(reflect.TypeOf(t).String())
}
```
- 升级代码，兼容性更强
- 可读性更强
- 别名的转换在编译时候进行，不影响代码执行

## 引用
- https://medium.com/a-journey-with-go/go-fuzz-testing-in-go-deb36abc971f
- https://medium.com/a-journey-with-go/go-how-zap-package-is-optimized-dbf72ef48f2d
