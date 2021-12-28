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
- 遍历slice时不能修改
- 删除元素： append(this.data[:i], this.data[i+1:]...)；不能删除最后一个元素，因此最后i要做判定
- 插入元素：新建一个slice保存插入位置之后的元素
```
rear = append(rear, this.data[j+1:]...)
this.data = append(this.data[:j], rear...)
```
## 题目
- ret = ret[:j] ret = append(ret, asteroids[i]) （是覆盖原有元素吗？）
- 1. 初始时，len等于cap
```
a := make([]int,1)
fmt.Println(len(a),cap(a)) //1,1
a = []int{1,2,3}
fmt.Println(len(a),cap(a)) //3,3
a = append(a,4)
fmt.Println(len(a),cap(a)) //4,6
a = a[:2]
fmt.Println(len(a),cap(a)) //2,6
a = append(a,5)
fmt.Println(len(a),cap(a)) //3,6
```
- 2 传参：slice传值是值copy，内存里两个变量指向同一块内存；len和cap的变化不会反应到外面；当append扩容时，扩容后的数组空间和外面的数组是两个空间；当append没有扩容，但是len变化，新添加的元素也不会在外面被看到，因为外面的len没有更新
```
func main() {
	a := make([]int,1)
	change(a)
	fmt.Println("out",a,len(a),cap(a)) //[0] 1 1
}

func change(a []int) {
	a = append(a,2)
	fmt.Println("in",a,len(a),cap(a)) [0,2] 2,2
	
}
```
- 3 删除末尾值的问题: 
-> 删除末尾值，改变了slice的长度，在heap中，为了保证len的准确性，则需要通过指针赋值
```
func (h *IntHeap) Pop() interface{} {
    old := *h
    n := len(old)
    x := old[n-1]
    *h = old[0 : n-1]
    return x
}
```
-> 更复杂的场景: 回调中使用重复使用slice的值：因为len或者扩容导致的内存空间变化，直接使用会导致值混乱；赋值使用内存copy比较合适
```
func test(one []string,ret *[][]string) {
	if len(*ret) == 5{
	    return 
	}
   	*ret = append(*ret, one) //此时one的len= k
	for i:=0;i<2;i++ {
		one = append(one, fmt.Sprintf("%d",i))
		test(one,ret) //此时one的len= k
		one = one[:len(one)-1]  //此时one的len = k-1
	}
}

// 每轮one的值
[]
[0]
[0 0]
[0 0 0]
[0 0 0 0]
//最终ret的值于放入ret的值不符合
[[] [1] [0 1] [0 0 1] [0 0 1 1]]
 ```
- 4 切片问题: 切片索引最多可以取到数组的长度；如下：4可以，但5就会越界
```
a := []int{1,2,3,4}
a[4:4] > []
a[:4] > [1,2,3,4]
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
