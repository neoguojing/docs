# 运行时类型

## interface
- 使用空interface转换类型，有一个量级的性能开销；
- 使用指针执行interface赋值，会提升性能，当时字符串类型比整形开销要大
```
<!-- 如下转换会发生panic -->
func main() {
	var i int8 = 1
	read(i)
}

//go:noinline
func read(i interface{}) {
	n := i.(int16)
	println(n)
}
```
## 源码
- runtime.type.go:_type
- reflect.type.go:rtype
- ../cmd/compile/internal/gc/reflect.go:dcommontype
```
type _type struct {
	size       uintptr   //字节对齐后的类型的字节数
	ptrdata    uintptr // 指针截止的长度位置；40表示前40个字节内包含指针，之后再没有指针
	hash       uint32
	tflag      tflag
	align      uint8
	fieldAlign uint8
	kind       uint8  // int8, int16, bool, etc
	// function for comparing objects of this type
	// (ptr to object A, ptr to object B) -> ==?
	equal func(unsafe.Pointer, unsafe.Pointer) bool
	gcdata    *byte //bitmap，记录相应字段是否是指针，每个bit表示一个8byte（指针大小）的内存
	str       nameOff
	ptrToThis typeOff
}

type arraytype struct {
	typ   _type
	elem  *_type
	slice *_type
	len   uintptr
}

type chantype struct {
	typ  _type
	elem *_type
	dir  uintptr
}

type slicetype struct {
	typ  _type
	elem *_type
}

type functype struct {
	typ      _type
	inCount  uint16
	outCount uint16
}

type ptrtype struct {
	typ  _type
	elem *_type
}

type structfield struct {
	name       name
	typ        *_type
	offsetAnon uintptr
}

type structtype struct {
	typ     _type
	pkgPath name
	fields  []structfield
}

<!--  用户态-->
type emptyInterface struct {
   typ  *rtype            // word 1 with type description
   word unsafe.Pointer    // word 2 with the value
}
```

## interface 实现
```
<!--  没有方法的interface{}-->
type eface struct {
	_type *_type  //实际类型
	data  unsafe.Pointer //指向数据
}


<!--  有方法的interface-->
type iface struct {
	tab  *itab
	data unsafe.Pointer
}

type itab struct {
	inter *interfacetype //interface类型
	_type *_type //接口实际类型
	hash  uint32 // copy of _type.hash. Used for type switches.
	_     [4]byte
	fun   [1]uintptr //接口实现列表，fun[0]==0 means _type does not implement inter.
}

type interfacetype struct {
	typ     _type
	pkgpath name
	mhdr    []imethod //接口方法列表，字典序
}

type imethod struct {
	name nameOff //方法名称
	ityp typeOff //方法描述
}

```

## 引用
- https://www.cnblogs.com/jiujuan/p/12653806.html
