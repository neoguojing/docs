# runc
基本使用
```
# create the top most bundle directory
mkdir /mycontainer
cd /mycontainer

# create the rootfs directory
mkdir rootfs

# export busybox via Docker into the rootfs directory
docker export $(docker create busybox) | tar -C rootfs -xvf -

runc spec
修改如下参数：
"terminal": false and "args": ["sleep", "5"]

runc create mycontainerid

# view the container is created and in the "created" state
runc list

# start the process inside the container
runc start mycontainerid

# after 5 seconds view that the container has exited and is now in the stopped state
runc list

# now delete the container
runc delete mycontainerid
```
## 关键函数
- startContainer：核心函数：传入context和动作：创建，启动和恢复；负责启动和运行容器
- > 启动容器依赖根文件系统和配置文件
- createContainer 
- Spec : 配置文件：
 ```
 Version string 版本
	Process *Process 进程配置
	Root *Root 根文件系统
	Hostname string 容器名称
	Mounts []Mount 挂载点
	Hooks *Hooks 控制容器声明周期的事件处理回调
	Annotations map[string]string 源数据
	Linux *Linux linux系统配置
	Solaris *Solaris
	Windows *Windows 
	VM *VM 
  
  type Process struct {
	Terminal bool 
	ConsoleSize *Box
	User User
	Args []string
	CommandLine string
	Env []string
	Cwd string  //当前工作目录
	Capabilities *LinuxCapabilities // linux必须包含的能力
	Rlimits []POSIXRlimit //资源限制
	NoNewPrivileges bool 
	ApparmorProfile string
	OOMScoreAdj *int
	SelinuxLabel string
}
 ```

## libcontainer
- New
- LinuxFactory ：实现Factory
- linuxContainer：实现Container
- Factory ：接口
- Container ： 容器
