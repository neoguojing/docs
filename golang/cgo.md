# cgo

## 编译其他语言共享的so
- go build -o main.so -buildmode=c-shared main.go.
```
//export NewProperty
func NewProperty(b int, p int, s int, t int) C.property {
   // business logic

   return C.property{
      bedroom:   C.int(b),
      price:     C.int(p),
      size:      C.int(s),
      type_id:   C.int(t),
   }
}
```
