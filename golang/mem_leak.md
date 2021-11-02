# 内存泄露
- vet 工具可以检查泄露
- context没有cancel导致的泄露
## pprof
堆栈
heap profile: 96: 1582948832(当前分配 byte) [21847: 15682528480(历史总分配内存)] @ heap/1048576

## cgo

## goroutine 泄露

## grpc

transport.newBufWriter 
bufio.NewReaderSize

以上内存未释放: 连接未正常关闭

