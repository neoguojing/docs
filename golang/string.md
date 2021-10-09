# string

## 总结
- string:是一个壳，里面存储了指针
- string是一个只读段，本身没有内存
- string和byte转换内存copy的原因：多g的竞争修改
- rune转换为string要经过utf8编码
## 结构
```
type stringStruct struct {
	str unsafe.Pointer
	len int
}
```
## 函数
- gostringnocopy： 字符串构建（无内存copy），先从byte指针转换为stringStruct，再转换为string
- concatstrings： 字符串拼接，首先计算字符串的总长度l，调用rawstringtmp 分配空间和返回string和[]byte，并遍历copy所有字符串在到返回的[]byte
- rawstringtmp：调用rawstring；在buf不为空的情况下，调用slicebytetostringtmp,可以省略内存分配
- rawstring： 调用mallocgc分配内存，分别构建string和slice返回
- slicebytetostringtmp： slice转string，无内存copy，直接赋值，使用场景为：
- >  m[T1{... Tn{..., string(k), ...} ...}] and m[string(k)] map
- >  "<"+string(b)+">" 字符串拼接
- >  string(b)=="foo" 比较
- slicebytetostring：slice to string，分配空间，构建stringStruct，并做内存copy
- stringtoslicebyte: string to byte ,分配空间，字节copy，会调用rawbyteslice
- rawbyteslice：分配空间mallocgc，构建slice并返回，
- stringtoslicerune：调用rawruneslice构建slice，遍历string，为slice赋值，返回
- rawruneslice：分配空间，rune大小为int32，构建slice
- slicerunetostring：[]rune to string:
- > 遍历[]rune,调用encoderune，将rune转换为[4]byte,并返回编码后的大小
- > 根据大小，构建string，调用rawstringtmp
- > 遍历[]rune,调用encoderune填充sting，返回
- encoderune：编码rune（32bit）为utf-8的[]byte，并返回编码大小;
- > rune小于=127，为ascci码，编码长度为1
- > rune<=2047,编码为两字节
- > rune在55296和57343之间为非法
- > rune <= 65535编码为3字节
- > 否则编码为4字节
- 
