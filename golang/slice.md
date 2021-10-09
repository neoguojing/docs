# slice

## 总结

## 结构
```
type slice struct {
	array unsafe.Pointer
	len   int
	cap   int
}

type notInHeapSlice struct {
	array *notInHeap
	len   int
	cap   int
}

type notInHeap struct{}
```

## 函数
- makeslicecopy
- makeslice: 创建slice，校验内存释放溢出，调用mallocgc；实际分配的是elem.size * cap大小的内存
- makeslice64： 同上
- growslice：处理append时的内存增加；新slice的len为old.len
- > 若传入的cap大于2倍old.cap,则     newcap=传入cap
- > 若old.cap < 1024 则              newcap= 2*old.cap
- > 否则：old.cap < 传入cap,，则     newcap= old.cap+old.cap/4
- > 针对type.size 为1，为指针大小和2的整数倍等场景计算长度
- > 有指针和无指针不同处理，调用mallocgc
- > 调用memmove，返回slice
- slicecopy：width参数时type.size即元素的大小,使用memmove，移动n，n为fromLen和toLen中较小的一个
