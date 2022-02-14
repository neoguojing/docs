# runc

## 概念
### cgroup
把一个cgroup目录中的资源划分给它的子目录，子目录可以把资源继续划分给它的子目录，为子目录分配的资源之和不能超过父目录，进程或者线程可以使用的资源受到它们委身的目录的限制
- tasks目录保存线程id
- proc文件保存进程id
- v2：1.只能绑定到文件层级的叶子节点；2.root可以授权普通用户管理cgroup的权限；3.线程模式
- > unifid模式：/sys/fs/cgroup/user.slice/user-1001.slice/session-1.scope
- v1为不同的线程指定不同的memory cgroup没有意义
-
### cgroup driver
- cgroupfs:cgroup接口封装，默认/sys/fs/cgroup，是一种虚拟文件系统
```
mkdir /sys/fs/cgroup/cpuset/demo
echo 0 > /sys/fs/cgroup/cpuset/demo/cpuset.cpus //选择cpu号码
echo 0 > /sys/fs/cgroup/cpuset/demo/cpuset.mems //内存号码
echo pid >  /sys/fs/cgroup/cpuset/demo/tasks
```
- systemd:以pid=1运行，实现了cgroup接口，k8s默认使用
```
systemd-cgls 查看cgroup层级
systemctl set-property cron.service CPUShares=100 MemoryLimit=200M
```
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
- /proc/cgroups： 列出系统支持的控制器
- 
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
- NewUnifiedManager：unifd v2 cgourp
- fs2.NewManager：创建v2的cgroup,构建manager对象实现Manager接口

### cgroup
```
type Manager interface {
	// Apply creates a cgroup, if not yet created, and adds a process
	// with the specified pid into that cgroup.  A special value of -1
	// can be used to merely create a cgroup.
	Apply(pid int) error

	// GetPids returns the PIDs of all processes inside the cgroup.
	GetPids() ([]int, error)

	// GetAllPids returns the PIDs of all processes inside the cgroup
	// any all its sub-cgroups.
	GetAllPids() ([]int, error)

	// GetStats returns cgroups statistics.
	GetStats() (*Stats, error)

	// Freeze sets the freezer cgroup to the specified state.
	Freeze(state configs.FreezerState) error

	// Destroy removes cgroup.
	Destroy() error

	// Path returns a cgroup path to the specified controller/subsystem.
	// For cgroupv2, the argument is unused and can be empty.
	Path(string) string

	// Set sets cgroup resources parameters/limits. If the argument is nil,
	// the resources specified during Manager creation (or the previous call
	// to Set) are used.
	Set(r *configs.Resources) error

	// GetPaths returns cgroup path(s) to save in a state file in order to
	// restore later.
	//
	// For cgroup v1, a key is cgroup subsystem name, and the value is the
	// path to the cgroup for this subsystem.
	//
	// For cgroup v2 unified hierarchy, a key is "", and the value is the
	// unified path.
	GetPaths() map[string]string

	// GetCgroups returns the cgroup data as configured.
	GetCgroups() (*configs.Cgroup, error)

	// GetFreezerState retrieves the current FreezerState of the cgroup.
	GetFreezerState() (configs.FreezerState, error)

	// Exists returns whether the cgroup path exists or not.
	Exists() bool

	// OOMKillCount reports OOM kill count for the cgroup.
	OOMKillCount() (uint64, error)
}
```
