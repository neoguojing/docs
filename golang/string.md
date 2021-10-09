# string

## 总结
- string:是一个壳，里面存储了指针
## 结构
```
type stringStruct struct {
	str unsafe.Pointer
	len int
}
```
## 函数
- gostringnocopy： 字符串构建（无内存copy），先从byte指针转换为stringStruct，再转换为string
- concatstrings： 字符串拼接，首先计算字符串的总长度l，调用rawstringtmp，并遍历copy多有字符串
- rawstringtmp：
- rawstring： 调用mallocgc分配内存，
