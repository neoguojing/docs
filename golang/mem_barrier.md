# 运行时内存屏障
## 内存屏障相关
- mbarrier.go
- wbBufEntries：256
- wbBufEntryPointers： 每个写屏障写入buf的指针数量
- shade：查找对象findObject，将对象greyobject
- gcphase == _GCmark || gcphase == _GCmarktermination 时开启写屏障

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
- memclrHasPointers：清空有指针的内存
- bulkBarrierPreWrite： 为区域内的所有指针执行内存屏障
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
