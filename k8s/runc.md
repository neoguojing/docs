# runc

## 概念
### rlimit
在Linux环境编程下，可以具体的限制一个进程对资源的使用，当进程尝试超过资源使用的限制：
- 它可能会收到一个信号，
- 是因资源而失败的系统调用
每个进程最初的获得的限制来自父进程，但是后来可以更改这个限制。
有两个关于资源限制的概念：
- current limit：为系统规定的上限，也叫做"soft limit"，因为进程通常将被限制在这个范围内；
- maxinum limit：为一个进程被允许建立current limit的最大值，也叫做"hard limit"，因为一个进程无法避开它，一个进程必须低于自己的maxinum limit，且只有超级用户可能提高它的maxinum limit。
### oom
- oom_score：内存不足之后杀死oom_score最大的进程；打分标准：1.占用内存；2.oom_score_adj的值
- oom_score_adj：[-1000,1000],正数表示分数会增加；负数表示分数会相应减少；-1000则表示不杀死该进程；0为默认值
```
cat /proc/pid/oom_score_adj
cat /proc/pid/oom_score
```
## 基本使用
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
	ApparmorProfile string   //AppArmor是一个高效和易于使用的Linux系统安全应用程序,对容器进行保护；此处提供profile
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
