# slice

## 总结
- 内存如何增长：当前cap小于1024，则新空间翻倍；当前cap大于1024，则扩容1/4
- 编译器自动计算len和cap的长度，没有对应函数
- 内存清理：调用runtime.memclrNoHeapPointers或runtime.memclrHasHeapPointers
- 创建加速：正常情况下，创建slice会先清理内存；在make+copy时，则不清理内存，直接copy
- copy重叠问题：
```
<!-- 将a，copy到从第二个地址开始的 
从前向后copy：会有内存重叠问题
memmove实现了从后向前copy，解决该问题-->
func main() {
   a := []byte("hello")
   copy(a[2:], a)
}
```
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
