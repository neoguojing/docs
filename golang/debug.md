# debug

禁止内联和优化
go build -gcflag "-N -l"

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
