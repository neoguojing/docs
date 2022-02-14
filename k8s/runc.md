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
### selinux linux系统安全策略控制
- label：[user]:[role]:[type]:[sec-level]

### IntelRdt
- 资源访问技术，主要管理云环境下的L3缓存和内存带宽，防止L3缓存的不均衡访问；

### seccomp 以限制进程对系统调用的访问，从系统调用号，到系统调用的参数，都可以检查和限制
- SECCOMP_MODE_STRICT： 进程只能访问read,write,_exit,sigreturn系统调用
- SECCOM_MODE_FILTER：通过定义规则访问；

### sd_notify
- linux 想systemd进程报告当前服务启动状态等信息的函数，runc中对应notifySocket，对应目录/run/notify/pid/notify/notify.sock

### criu 在用户空间，控制进程的checkpoint和restore
- 程序或容器的热迁移
- 从/proc目录收集进程信息，并保存到磁盘；复杂

### proc知识
- /proc/self/exe： 指向当前运行进程的软链接
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
- > 设置pid文件，加载config.json
- > 初始化sd_nofity socket
- > createContainer 创建容器
- > runner设置和调用Run
- createContainer 输入容器id和配置文件;调用
- > 调用libcontainer的new和factory构建容器
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
	OOMScoreAdj *int         //oom 配置调整
	SelinuxLabel string
}
 ```

## libcontainer
- New：构建LinuxFactory，设置根、路径信息等
- LinuxFactory ：实现Factory
- linuxContainer：实现Container
- > Create:创建容器：校验配置和根文件
- > cgroup manager 创建
- > 构建linuxContainer，
- > NewIntelRdtManager构建RDT
- > 设置状态为stopped
- Factory ：接口
- Container ： 容器
- manager.New
- NewIntelRdtManager:
