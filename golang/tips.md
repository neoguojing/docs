# 技巧

## 传指针还是copy结构体？
- 返回指针，变量会逃逸到堆上，产生大量需要gc的内存，加大了gc压力，进一步降低了程序性能
- 返回结构体copy,遍历在栈上被copy，不影响gc
```
func byCopy() S //yes

func byPointer() *S 

func (s S) stack(s1 S) {}  //yes

func (s *S) heap(s1 *S) {}
```
- go自有类型返回值比较好
- 引用类型避免使用指针： slice, map, interface, function and channel types；被设计在栈上不会跑到堆上，减小压力
- 不确定是否需要使用指针？使用指针
- UnmarshalXX一般需要指针
- File 包全部返回指针，因为需要共享
- Time包指针和结构体混合
```
func (t Time) Add(d Duration) Time
func Now() Time
func (t *Time) UnmarshalBinary(data []byte) error {
func (t *Time) GobDecode(data []byte) error {
func (t *Time) UnmarshalJSON(data []byte) error {
func (t *Time) UnmarshalText(data []byte) error {
```
## 数值转换
- 大int转小int截断，然后按照值和目标类型重新解析
- 小int转大int，执行符号扩展//c++是直接copy
## init函数
- 在main函数之前执行
- initdone：确保init只被执行一次；初始为1，所有init执行完之后赋值为2
- 先执行其他包导入的init
- 执行本包的init
- 本包中多个init，按照申明顺序执行
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
- https://www.ardanlabs.com/blog/2017/06/design-philosophy-on-data-and-semantics.html
