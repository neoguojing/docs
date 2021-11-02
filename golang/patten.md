# 设计模式

##错误处理
```
type Reader struct {
    r   io.Reader
    err error   //内部错误包装
}
```
### 错误包装
github.com/pkg/errors
```
errors.Wrap(err, "read failed")
errors.Cause(err)
```

## build模式&&option 

主要解决类型够着函数问题

```
func NewServer(addr string, port int, conf *Config) (*Server, error)
```
解决方案1 建造者模式
```
//使用一个builder类来做包装
type ServerBuilder struct {
  Server
}

func (sb *ServerBuilder) Create(addr string, port int) *ServerBuilder {
  sb.Server.Addr = addr
  sb.Server.Port = port
  return sb
}

func (sb *ServerBuilder) WithProtocol(protocol string) *ServerBuilder {
  sb.Server.Protocol = protocol 
  return sb
}

func (sb *ServerBuilder) WithMaxConn( maxconn int) *ServerBuilder {
  sb.Server.MaxConns = maxconn
  return sb
}

func (sb *ServerBuilder) Build() (Server) {
  return  sb.Server
}
```

解决方案2，option

```
type Option func(*Server)

func Protocol(p string) Option {
    return func(s *Server) {
        s.Protocol = p
    }
}

func NewServer(addr string, port int, options ...func(*Server)) (*Server, error) {
  srv := Server{
    Addr:     addr,
    Port:     port,
  }
  for _, option := range options {
    option(&srv)
  }
  return &srv, nil
}
```

## 泛型

https://coolshell.cn/articles/21179.html

## pipline

实现shell 命令类似功能 echo $nums | sq | sum

元型
```
func echo(nums []int) <-chan int {
  out := make(chan int)
  go func() {
    for _, n := range nums {
      out <- n
    }
    close(out)
  }()
  return out
}

func sq(in <-chan int) <-chan int {
  out := make(chan int)
  go func() {
    for n := range in {
      out <- n * n
    }
    close(out)
  }()
  return out
}

func prime(in <-chan int) <-chan int {
  out := make(chan int)
  go func ()  {
    for n := range in {
      if is_prime(n) {
        out <- n
      }
    }
    close(out)
  }()
  return out
}

//调用
for n := range sum(sq(echo(nums))) {
  fmt.Println(n)
}
```

进阶
```
type EchoFunc func ([]int) (<- chan int) 
type PipeFunc func (<- chan int) (<- chan int) 


func pipeline(nums []int, echo EchoFunc, pipeFns ... PipeFunc) <- chan int {
  ch  := echo(nums)
  for i := range pipeFns {
    ch = pipeFns[i](ch)
  }
  return ch
}

//调用
var nums = []int{1, 2, 3, 4, 5, 6, 7, 8, 9, 10}    
for n := range pipeline(nums, sq, sum) {
    fmt.Println(n)
}
```

## fan in/out

串行输出，并发计算，串行输出

质数求和示例
```
func merge(cs []<-chan int) <-chan int {
  var wg sync.WaitGroup
  out := make(chan int)

  wg.Add(len(cs))
  for _, c := range cs {
    go func(c <-chan int) {
      for n := range c {
        out <- n
      }
      wg.Done()
    }(c)
  }
  go func() {
    wg.Wait()
    close(out)
  }()
  return out
}

func main() {
  nums := makeRange(1, 10000)
  in := echo(nums)

  const nProcess = 5
  var chans [nProcess]<-chan int
  for i := range chans {
    chans[i] = sum(prime(in))
  }

  for n := range sum(merge(chans[:])) {
    fmt.Println(n)
  }
}
```

## 监控模式
```
type Item = int

type waiter struct {
	n int
	c chan []Item
}

type state struct {
	items []Item
	wait  []waiter
}

type Queue struct {
	s chan state
}

func NewQueue() *Queue {
	s := make(chan state, 1)
	s <- state{}
	return &Queue{s}
}

func (q *Queue) Put(item Item) {
	s := <-q.s
	s.items = append(s.items, item)
	for len(s.wait) > 0 {
		w := s.wait[0]
		if len(s.items) < w.n {
			break
		}
		w.c <- s.items[:w.n:w.n]
		s.items = s.items[w.n:]
		s.wait = s.wait[1:]
	}
	q.s <- s
}

func (q *Queue) GetMany(n int) []Item {
	s := <-q.s
	if len(s.wait) == 0 && len(s.items) >= n {
		items := s.items[:n:n]
		s.items = s.items[n:]
		q.s <- s
		return items
	}

	c := make(chan []Item)
	s.wait = append(s.wait, waiter{n, c})
	q.s <- s

	return <-c
}
```
