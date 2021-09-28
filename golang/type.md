# 运行时类型

## 源码
- runtime.type.go:_type
- reflect.type.go:rtype
- ../cmd/compile/internal/gc/reflect.go:dcommontype
```
type _type struct {
	size       uintptr
	ptrdata    uintptr // size of memory prefix holding all pointers
	hash       uint32
	tflag      tflag
	align      uint8
	fieldAlign uint8
	kind       uint8
	// function for comparing objects of this type
	// (ptr to object A, ptr to object B) -> ==?
	equal func(unsafe.Pointer, unsafe.Pointer) bool
	// gcdata stores the GC type data for the garbage collector.
	// If the KindGCProg bit is set in kind, gcdata is a GC program.
	// Otherwise it is a ptrmask bitmap. See mbitmap.go for details.
	gcdata    *byte
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
