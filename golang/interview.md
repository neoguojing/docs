# interview

## defer

```
func call() (err error) {
	defer func(){
		fmt.Println("err is ",err )
	}()
	return errors.New("aaaaa")
}

func main() {
	call() //err is  aaaaa
}

```

## interface
- interface{}类型的变量包含了2个指针，一个指针指向值的在编译时确定的类型，另外一个指针指向实际的值；只有当类型和值都是nil的时候，才等于nil
- nil 是一个预定义常量，没有类型，是一个值
- 整形、bool和字符串的0，false，"",都会转换为nil；
- error 为一种特殊的interface:所以其nil判定和interface一样
```
type error interface{
  Error string
}
```

```
var s []string
s == nil  //true
isNil(s)
func isNil(i interface{}) {
  i == nil  //false i=&{[]string,slice{nil,0,0}}  type和值均非0值
}
```
```
a := (*interface{})(nil)
var b interface{} = (*interface{})(nil)
a == nil //true 空指针
b == nil //false {*interface{},0}
```

```
type ErrorImpl struct{}
func (e *ErrorImpl) Error() string {
   return ""
}
var ei *ErrorImpl
var e error
func ErrorImplFun() error {
   return ei
}

func ErrorImplFun1() *ErrorImpl {
   return ei
}
func main() {
   f := ErrorImplFun()
   fmt.Println(f == nil)  //f是interface类型 = {*ErrorImpl,0},类型非nil，所以未false
    f = ErrorImplFun1()   //f非inteface类型，是一个空指针，和nil相等
}
```

### slice

```
 slice := []string{"wo", "shi", "shui"}
for index, _:= range slice {
  println(&value) //地址永远一样，用这个指针赋值会得到一样的结果
  println(&slice[index]) //这个是正确的
}
```
