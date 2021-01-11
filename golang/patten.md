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
