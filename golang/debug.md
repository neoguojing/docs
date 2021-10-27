# debug

## 编译
- 禁止内联和优化
> go build -gcflag "-N -l"
- 生成汇编：go tool compile -S main.go
- 打印边界检查结果：-gcflags="-d=ssa/check_bce/debug=1"
- 打印内存逃逸：-gcflags="-m"
## 协程泄露：
- 协程以为某些原因阻塞，导致协程越来越多，最终导致系统崩溃
- https://github.com/uber-go/goleak

## vscode 

shift + ctrl + p 创建task
task.json
```


{
    // See https://go.microsoft.com/fwlink/?LinkId=733558
    // for the documentation about the tasks.json format
    "version": "2.0.0",
    "tasks": [
        {
            "label": "build",
            "command": "make",
            "args": ["build","nolic=1 host=amd64 device=none withocr=1 withmodel=1"],
            "type": "shell"
        },
        {
            "label": "clean",
            "command": "make",
            "args": ["clean"],
            "type": "shell"
        }
    ]
}
```
launch.json
```
{
    // Use IntelliSense to learn about possible attributes.
    // Hover to view descriptions of existing attributes.
    // For more information, visit: https://go.microsoft.com/fwlink/?linkid=830387
    "version": "0.2.0",
    "configurations": [
        {
            "name": "Launch",
            "type": "go",
            "request": "launch",
            "mode": "exec",
            "remotePath": "",
            "program": "${workspaceFolder}/${workspaceRootFolderName}",
            "cwd": "${workspaceFolder}",
            "env": {
                "GOPATH":"",
                "LD_LIBRARY_PATH" :""
            },
            "args": ["-config ${workspaceFolder}/config/config.json"],
            "showLog": true,
            "preLaunchTask": "build"
        }
    ]
}
```

## 需要传递编译参数的debug配置(亲测)

```
{
    // Use IntelliSense to learn about possible attributes.
    // Hover to view descriptions of existing attributes.
    // For more information, visit: https://go.microsoft.com/fwlink/?linkid=830387
    "version": "0.2.0",
    "configurations": [
        {
            "name": "Launch",
            "type": "go",
            "request": "launch",
            "mode": "debug",
            "remotePath": "",
            "port": 2345,
            "host": "127.0.0.1",
            "program": "${workspaceFolder}/cmd/${workspaceRootFolderName}", //main函数不在当前目录
            "cwd": "${workspaceFolder}",
            "env": {
                "GOPATH":"",
                "LD_LIBRARY_PATH" :"",
                "CGO_LDFLAGS":""
            },
            "output": "${workspaceFolder}/${workspaceRootFolderName}",    //bin文件输出到主目录.幷指定名称
            "args": [
                "-config",
                "config.json",
            ],
            "showLog": true,
            "buildFlags": "-tags 'dev'"             //传递编译参数给dlv->go build
        }
    ]
}
```
