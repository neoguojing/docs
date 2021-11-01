# 技巧

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
- - https://medium.com/a-journey-with-go/go-fuzz-testing-in-go-deb36abc971f
