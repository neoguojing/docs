# plugin

- 模块化开发方法
- 将golang代码编译为so,在另外的golang代码中动态加载
- 
- 不同的 Go 版本会失败
- 不同的 GOPATH 失败
- 使用 vendor 文件夹失败
- 不同版本的公共依赖项 会失败

## 编译方法

```
目标文件必须使用main包

生成的包最好放在LD_LIBRARY_PATH下面

go build --buildmode=plugin -o pluginhello.so pluginhello.go
```

##　plugin包示例

最好使用接口方式统一 so和调用方的方法

查找包
```
var (
	gPluginMu   sync.Mutex
	gPlugin     *plugin.Plugin
)


func lookupPluginSymbol(sym string) (plugin.Symbol, error) {
  gPluginMu.Lock()
	defer gPluginMu.Unlock()

	return gPlugin.Lookup(sym)
}

type myAPI interface {
  Hello(string) err
}

func NewHello(in string) (myAPI,error) {
	sym, err := lookupPluginSymbol("NewHello")
	if err != nil {
		return nil, err
	}
	f, ok := sym.(func(string) (myAPI, error))
	if !ok {
		return nil, err
	}
	return f(in)
}


```
