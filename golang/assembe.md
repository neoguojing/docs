# 汇编
- JLT 71:小于0的话跳转到指令71
- JMP 跳转到指令并执行
- MOVQ src dst
- autotmp_*自动生成的变量名
- INCQ加1
- 

## loop如何转换为汇编
```
func main() {
   l := []int{9, 45, 23, 67, 78}
   t := 0

   for _, v := range l {
      t += v
   }

   println(t)
}

伪汇编代码
func main() {
   l := []int{9, 45, 23, 67, 78}
   t := 0
   i := 0

   var tmp int

   goto end
start:
   tmp = l[i]
   i++
   t += tmp
end:
   if i < 5 {
      goto start
   }

   println(t)
}

// 为抢占做的优化
func main() {
   l := []int{9, 45, 23, 67, 78}
   t := 0
   i := 0

   var tmp int
   p := uintptr(unsafe.Pointer(&l[0]))

   if i >= 5 {
      goto end
   }
body:
   tmp = *(*int)(unsafe.Pointer(p))
   p += unsafe.Sizeof(l[0])
   i++
   t += tmp
   if i < 5 {
      goto body
   }
end:
   println(t)
}
```
-  AX寄存器保存当前loop的位置
-  cx寄存器保存当前t的值
