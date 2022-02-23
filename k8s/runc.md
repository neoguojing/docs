# runc

## 概念
### 系统调用
- execve(2):执行一个二进制文件，新的执行体将覆盖调用他的进程的堆栈和其他区域；完全取代运行
- clone2
- clone ： 提供一系列的参数，确认于父进程共享哪些资源；支持fork和vfork的所有功能；
- unshare(CLONE_NEWUSER)：为当前进程创建新的user namespace
- fork: 创建子进程，子进程和父进程共享页帧等信息，均为只读；当父或子改变某页，则产生page fault，复制该页；不共享虚拟地址；子进程会copy数据段和代码段等；执行顺序不确定
- vfork： 创建线程，和父亲共享物理以及虚拟地址；且父进程被挂起；子进程优先执行
- prctl(PR_SET_DUMPABLE, 0, 0, 0, 0)：设置进程不可dump，prctl系统调用用于改变进程参数
- setjmp: 保存当前进程上下文到结构体
### capability 非root的权限管理
- 进程权限
- 文件权限
### namespace
- 共7种：ips，pid，mount，network,uts,cgroup,users
- /proc/pid/ns: 列出当前进程所属的命名空间
- clone(CLONE_NEWUSER):创建子进程，并创建user命名空间
- setns(fd，flag)：将当前进程加入到fd对应的namespace，fd为 /proc/pid/ns/ips等对应的文件
- unshare(flag):创建namespace，然后将当前进程加入到该命名空间
- /proc/pid/uid_map,gid_map 设置ns内外的用户映射
### cgroup
把一个cgroup目录中的资源划分给它的子目录，子目录可以把资源继续划分给它的子目录，为子目录分配的资源之和不能超过父目录，进程或者线程可以使用的资源受到它们委身的目录的限制
- tasks目录保存线程id
- proc文件保存进程id
- v2：1.只能绑定到文件层级的叶子节点；2.root可以授权普通用户管理cgroup的权限；3.线程模式
- > unifid模式：/sys/fs/cgroup/user.slice/user-1001.slice/session-1.scope
- v1为不同的线程指定不同的memory cgroup没有意义
- 9大子系统
- unified：叶子节点管理task，中间节点只允许资源控制
#### 子系统 lssubsys -a
- blkio ：为块设备设定输入/输出限制
- cpu ：提供对cpu调度的限制，影响调度器的参数，将进程容原调度队列解绑，发送到新的cgroup队列
- cpuacct ：cpu使用报告
- cpuset： cpu核和内存节点的分配
- devices ：是否允许访问设备
- memory ：提供内存限制和提供内存报告，oom配置等
- net_cls ：等级识别符（classid）标记网络数据包于linux流量控制配合
- pids：功能是限制cgroup及其所有子孙cgroup里面能创建的总的task数量
- net_prio
- perf_event
- rdma：RDMA让计算机可以直接存取其他计算机的内存，而不需要经过处理器的处理
#### freezer子系统 :这个子系统挂起或者恢复 cgroup 中的任务
- 成批启动/停止任务，以达到及其资源的调度
- 强制一组任务进入静默状态：用于读取器运行时信息
- 父任务冻结，则子节点均冻结
- 状态包括：冻结和解冻，以及中间状态冻结中
### cgroup driver
#### cgroupfs:cgroup接口封装，默认/sys/fs/cgroup，是一种虚拟文件系统
```
mkdir /sys/fs/cgroup/cpuset/demo
echo 0 > /sys/fs/cgroup/cpuset/demo/cpuset.cpus //选择cpu号码
echo 0 > /sys/fs/cgroup/cpuset/demo/cpuset.mems //内存号码
echo pid >  /sys/fs/cgroup/cpuset/demo/tasks
```
#### systemd:以pid=1运行，实现了cgroup接口，k8s默认使用
- 支持通过dbus进行进程间通信
- sd_notify：linux 想systemd进程报告当前服务启动状态等信息的函数，runc中对应notifySocket，对应目录/run/notify/pid/notify/notify.sock
- notifySocket：从systemd中读取同期信息转发给notifySocketHost
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
- RLIMIT_NOFILE: 打开文件的限制数量
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



### criu 在用户空间，控制进程的checkpoint和restore
- 程序或容器的热迁移
- 从/proc目录收集进程信息，并保存到磁盘；复杂

### proc知识
- /proc/self/exe： 指向当前运行进程的软链接
- /proc/cgroups： 列出系统支持的控制器
### tty的创建
- open /dev/tty
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
## 基本流程
- 1. 运行命令的runc create id 创建进程
- 2.执行startContainer：其中createContainer创建容器对象
- 3.runner.run：调用newProcess 创建进程对象；
- 4.linuxContainer.Start:创建父进程和启动
- 5.newParentProcess/newInitProcess：创建初始化进程信息：包含父进程信息和子进程信息
- 6.initProcess.start :启动父进程 runc 参数路径+init；
- 7.执行init.go中导入的github.com/opencontainers/runc/libcontainer/nsenter，执行nsenter 注册的init，然后执行nsexec
```
extern void nsexec();
void __attribute__((constructor)) init(void) {
	nsexec();
}
nsexec：为一个状态机，最终状态才会退出到go runtime执行
1. 此时为父进程，首先clone一个子进程
2. 子进程再clone一个子进程
3. 最终的子进程执行完，进入go runtime
```
- 8.init.go的init函数：设置cpu为1，并锁死线程，进入StartInitialization
- 9.factory.StartInitialization: 创建linuxStandardInit，调用其init函数，最终调用unix.Exec执行目标容器运行时进程，替换当前进程；
- 
## 默认值
- _LIBCONTAINER_LOGPIPE：父子进程的log pipe环境变量
- _LIBCONTAINER_INITPIPE ：初始化管道
- process.root：rootfs
- factory.root:由-root指定，保存容器状态,是宿主机的目录
- Container.root: factory.root + 容器id，是宿主机的目录
- cgroups.Manager.Path:保存 子系统：路径的映射 如"/sys/fs/cgroup/blkio/user.slice/my1"
- 容器父进程为：runc
- Container.initProcess: cmd保存了父进程（信息 执行runc init），process保存了子进程信息（sleep 5）
## 关键函数
- startContainer：核心函数：传入context和动作：创建，启动和恢复；负责启动和运行容器
- > 设置pid文件，加载config.json
- > 初始化sd_nofity socket
- > createContainer 创建容器：创建cgroup manager 和 rdt manager等，返回container对象
- > runner设置和调用Run
- runner.Run容器的运行：入参为进程配置信息
- > newProcess创建进程,并填充进程信息： 包括cap和rlimit信息
- > newSignalHandler:新建信号处理，处理SIGCHLD和SIGWINCH
- > setupIO：主要是创建tty和设置管道，将os的三个管道，指向容器进程本身
- > 依据不同状态分别调用Container的start run 和Restore
- > createPidFile:创建pid
- > 启动信号处理循环
- > 销毁runner，返回容器状态
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
- New：构建LinuxFactory，建立根文件夹，设置criu,初始路径和参数等
- LinuxFactory ：实现Factory
- > Create接口：执行结果返回一个：linuxContainer对象
```
1.用容器id创建容器根目录，修改根路径的权限；
2.创建cgroup manager：分为cgroup v1/v2 和是否使用systemd的组合；systemd需要dbus通信；cm：包含配置，路径或dbus；
3.构建linuxContainer;
4.NewIntelRdtManager构建RDT
5.设置状态为stopped
6.校验是否冻结状态
```
- > 
- linuxContainer：实现Container
- Factory ：接口
- Container ：容器接口
- > Start:1.创建命名FIFO管道exec.fifo命名；2.创建父进程（父子通信socket，父子日志管道，用于启动真正执行的进程）；3.启动父进程：cmd.Start执行runc init；execSetns设置ns；WriteCgroupProc设置将pid写入对应cgroup的proc文件；pid写入rtd文件；setupRlimits：unix.Prlimit设置limit；
- > Restore: 收集信息，调用criu命令执行转储
- > Run: 相比start，多了exec函数的执行（一个死循环）：1.监听fifo队列，执行热舞；2.没100ms检查进程状态
- > Pause/Resume:调用cgroupManager的Freeze操作
- manager.New
- NewIntelRdtManager:
- NewUnifiedManager：unifd v2 cgourp
- fs2.NewManager：创建v2的cgroup,构建manager对象实现Manager接口
- newInitProcess： 父进程（runc）：其中会调用newInitProcess
- newInitProcess：创建容器进程:

### nsexec 启动容器进程
#### 流程
- 1.从_LIBCONTAINER_INITPIPE获取pipe，从pipe获取配置并解析
- 2.clone 源二进制文件确保隔离（/proc/self/exe）
- 3.更新oom_score_adj， /proc/self/oom_score_adj
- 4.设置不可dump，建立和子进程和孙子进程通信的unix sock
- 5.setjump保存当前上下文：第一次返回0；
- 6.STAGE_PARENT： 父进程执行
- > 设置进程名；clone子进程，子进程执行longjmp跳转到第5步，并返回值1
- > 循环read和孩子的通信socket；
- > 向子进程更新配置用户配置：/proc/childpid/setgroups，/proc/%d/uid_map，/proc/%d/gid_map；并发送回执
- > 子孙进程创建通知：读取子孙进程pid，发送回执，并通过_LIBCONTAINER_INITPIPE通知runc
- > 通知子进程挂载目录：setns(/proc/childpid/ns/mnt, CLONE_NEWNS)为子进程新建命名空间，并通过unix fd一条一条发送mount配置给子进程；为父进程新建ns；，并发送回执；
- > 子进程操作完成，结束循环
- > 监听孙进程状态，直到孙进程完成
- 7.STAGE_CHILD： 子进程中：
- > 为子进程本身设置ns；unshare(CLONE_NEWUSER)为进程创建新的user namespace；
- > 设置进程本身可dump，等待进程父进程帮忙设置用户信息；设置步可dump；
- > setresuid(0, 0, 0):使子进程在本user namespace中成为root
- > unshare(config.cloneflags & ~CLONE_NEWCGROUP):创建其他ns
- > 向父亲请求挂载路径，并接受
- > clone 子孙进程并传递参数2,发送孙进程id给父亲；
- > 发送结束消息，并退出进程
- 8：STAGE_INIT
- > 收到父进程信号，开始工作
- > 设置本身为root：setuid(0)；setgid(0)
- > 创建新unshare(CLONE_NEWCGROUP)cgroup
- > 向父进程发送完成信号信号
- > 返回
### cgroup
- systemd：模块；包含：dbus
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
