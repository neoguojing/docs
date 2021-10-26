# 运行时屏障
## 总结
- 何时开启写屏障：2个阶段： gcphase == _GCmark || gcphase == _GCmarktermination 
- 
## 内存屏障相关
- mbarrier.go
- wbBufEntries：256
- wbBufEntryPointers： 每个写屏障写入buf的指针数量
- shade：查找对象findObject，将对象greyobject；
- gcphase == _GCmark || gcphase == _GCmarktermination 时开启写屏障
- slot ： 目标对象
- ptr: 指针

## 结构体
```
type wbBuf struct {
	next uintptr
	end uintptr
	buf [512]uintptr
}
var writeBarrier struct {
	enabled bool    // compiler emits a check of this before calling write barrier
	pad     [3]byte // compiler uses 32-bit load for "enabled" field
	needed  bool    // whether we need a write barrier for current GC phase
	cgo     bool    // whether we need a write barrier for a cgo check
	alignme uint64  // guarantee alignment so that compiler can use a 32 or 64-bit load
}

```

## 函数
- typedmemmove： 内存copy;map等会使用
- > 若需要内存屏障且typ.ptrdata != 0（对象有指针），调用bulkBarrierPreWrite
- > 调用memmove，执行内存copy
- memclrHasPointers：清空有指针的内存：调用bulkBarrierPreWrite和memclrNoHeapPointers
- bulkBarrierPreWrite： 为区域内的所有指针执行内存屏障
- > 未开启内存屏障，则返回
- > 查找dst指针对应的span
- > 若span == nil，表明是一个全局的（data或bss值），遍历data和bss，多dst在该区域内，则执行bulkBarrierBitmap;返回
- > 否则，若dst之前在堆中，现在不在了，则直接返回
- > 否则，dst在heap中，获取dst的heapBits：
- > 若src==0（内存清理）,以8字节遍历指针内存，若heapBits.isPointer,则将目标地址和0放入wbBuf，若wbBuf满了，执行wbBufFlush，
- > 否则,同上，不同的是将dst+i和src+i放入wbBuf
- bulkBarrierBitmap：设置data和bss的bitmap，掩码计算为1则放入wbbuf
- wbBuf.putFast:将新旧指针放入p.wbbuf
- wbBufFlush：
- wbBufFlush1: 将写屏障buf同步到gc work 
- setGCPhase：设置gcphase，设置写屏障的状态
- gcWriteBarrier: 汇编函数
- > 增加wbBuf.next的位置
- > 保存 val的值到wbBuf
- > 保存 ptr的值到wbbuf
- > 若队列满了，跳转到flush，保存寄存器值，调用runtime·wbBufFlush
```
<!-- 为代码 -->
if runtime.writeBarrier.enabled {
    runtime.gcWriteBarrier(ptr, val)
} else {
    *ptr = val
}
```

## 引用
- https://github.com/aceld/golang/blob/main/5%E3%80%81Golang%E4%B8%89%E8%89%B2%E6%A0%87%E8%AE%B0%2B%E6%B7%B7%E5%90%88%E5%86%99%E5%B1%8F%E9%9A%9CGC%E6%A8%A1%E5%BC%8F%E5%85%A8%E5%88%86%E6%9E%90.md
