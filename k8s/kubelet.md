# kubelet
- remoteRuntimeEndpoint = "unix:///var/run/dockershim.sock" : runtime的unix sock
- DockershimRootDirectory:   "/var/lib/dockershim",
- CRI：容器运行时接口：dockershim，containerd等
- dockershim：负责初始化pause容器的网络namespace
- pause容器：内置永远阻塞的进程，负责占用一个网络namespace，新建容器执行docker run --net=none加入pause的namesapce
- CNI : 容器网络接口：内置插件如p2p，bridge,负责创建pause容器网络设备eth0，并绑定IP，第三方插件flannel,calico，解决容器跨主机通信问题
- Network Namespace的网络栈包括：网卡（Network interface）、回环设备（Loopback Device）、路由表（Routing Table）和iptables规则。
- kube-controller-manager/app/plugins.go 
- init container：在app容器运行前运行的容器，用于装备工具或者启动脚本等不在app容器运行的容器；只能一个一个的运行，运行失败整个pod失败；运行需要退出
- > 主要用于访问secret和进行一些不安全的操作
- > deploy中申明initContainers
- > 重启需要重新执行
- readiness：用户自定义状态：通过spec的readinessGates注入
- probe：
- > livenessProbe: 容器是否运行，false则kill pod
- > readinessProbe : 容器是否可以响应请求，false则从service移除改pod
- > startupProbe: 容器内的进程是否启动，false则kill pod
- 
## CNI 原理
- CNIBinDir:   "/opt/cni/bin",
- CNIConfDir:  "/etc/cni/net.d",
- CNICacheDir: "/var/lib/cni/cache"
### 流程
- 创建veth虚拟网卡对，绑定pod和eth0
- 为veth分配IP，先按node网络段划分，然后容node拿到ip
- 配置pod的路由
- CNI进程负责监听pod和node的ip
- 通过overlay或者VPC路由或者OSG路由打通node直接的通信网络
- 打通pod到node的网络：linux路由表、fdb，ovs等
## 问题
- featureGate ：是什么？ k8s.io/component-base/featuregate/feature_gate.go
- 
## depend
- VolumePlugins
- DynamicPluginProber
- DockerOptions

## 启动
- 构建kubeclient:从apiserver获取pod信息
- 构建EventClient
- 构建HeartbeatClient
- 获取cgroup：1.pod使用的cgroup，2.kubelet的cgroup，3.runtime使用的cgroup,4.系统cgroup
- NewMainKubelet:kubelet对象创建
- > 构建nodeInformer
- > serviceInformer
- > Kubelet对象创建
- > secretManager
- > configMapManager
- > livenessManager
- > startupManager
- > podManager：实现：basicManager:维护内存pod集合，以及提供secret，configmap的manager；实现pod的增删改查；
- > statusManager： 维护kubeclient、pod状态集合；Start启动之后，监听podStatusChannel，向apiserver同步状态；
- > runtimeClassManager
- > kubeGenericRuntimeManager
- > CRIStatsProvider
- > GenericPLEG
- > realContainerGC
- > ImageGCManager
- > probeManager
- > tokenManager
- > VolumePluginMgr
- > pluginManager
- > volumeManager
- > podWorkers: ；实现podWorkers，每个pod一个协程，维护pod对象集合，工作队列和pod状态缓存等；由syncloop调用dispatchWork调用UpdatePod接口，为每个新建的pod启动一个managePodLoop协程，用于监听UpdatePodOptions管道执行相关状态同步；
- > nodeLeaseController
- > admitHandlers
- > softAdmitHandlers
- > shutdownManager
- syncLoop 主循环：
- > 监听pod配置变化（来源于NewSourceApiserver）：处理pod的add/update/delete/RECONCILE等
- > 监听pleg chan，处理pod状态的同步和删除
- > 每秒中检查pod同步状态
- > 没2s检查处理pod的cleanup
- > 监听livenessManager的chan，执行pod状态同步

### ListenAndServe
- 提供：/pods /metrics /stat 等接口
- 提供debug接口
### pod管理
- 操作：SyncPodSync，SyncPodUpdate，SyncPodCreate，SyncPodKill
- 状态： Pending，Running，Succeeded（所有容器正常退出），Failed（所有容器退出，但是退出时有异常），Unknown（pod状态无法获取）
- mirrorpod：非apiserver来源的pod（static pod），需要做一个镜像和原本staticpod一样，mirror pod的状态回上报apiserver
```
type Pod struct {
	metav1.TypeMeta `json:",inline"` //kind和apiversion
	metav1.ObjectMeta //对象元信息：name/namespace/uid/lables等信息
	Spec PodSpec //期望状态的描述：数据卷/容器/node信息/调度策略信息等
	Status PodStatus //状态信息：pod状态/ip信息/容器状态信息等
}
```
- HandlePodAdditions
- > 启动podworker协程，读取UpdatePodOptions管道的内容，调用syncPod做状态同步
- HandlePodSyncs: 从podmanager获取所有pod，调用podWorkers.UpdatePod，将操作信息写入对应podworker的UpdatePodOptions管道，处理同上
### syncPod pod状态同步主逻辑：
#### SyncPodKill状态处理：
- statusManager.SetPodStatus 设置状态
- Kublete.killPod:kubeGenericRuntimeManager.KillPod依次此执行: 
- > internalLifecycle.PreStopContainer
- > runtimeService.StopContainer
- > runtimeService.StopPodSandbox
#### pod创建和更新：
- statusManager.SetPodStatus： 设置状态
- runtimeState.networkErrors： 网络plugin就绪
- makePodDataDirs： 创建pod目录：pod目录，卷目录和插件目录
- volumeManager.WaitForAttachAndMount： 等待卷挂载完成
- containerRuntime.SyncPod：同步pod状态

### 卷管理
- VolumeManager：负责管理一系列的loop，异步实现Volume状态同步
- reconciler： 实现Volume状态同步
- pkg/volume/ ： 实现了各种存储包括：rdb,nfs等插件，均实现了VolumePlugin接口
- VolumePlugin：卷插件接口，定义Init，NewMounter（返回Mounter接口）；
- VolumePluginMgr：负责管理插件，提供依据名称等查找VolumePlugin的函数
- OperationExecutor： 定义一系列卷attach/mount操作，底层依赖pkg/volume/的实现；
- Mounter: 定义Mount/UnMount等接口，底层使用mount命令或者systemd-run --description=... --scope -- mount -t 实现挂载
- operationExecutor：定义AttachVolume/MountVolume等接口，依赖NestedPendingOperations和OperationGenerator实现
- NestedPendingOperations：处理pending的op，防止多个相同的操作执行，主要包含operation数组；定义Run和Wait等接口
- OperationGenerator：生成处理函数，和operationExecutor解绑；依据Volume名称和Node等生成操作函数，作为NestedPendingOperations函数的入参，被调用

#### 卷管理流程：
- 启动kublet时启动：VolumeManager
- VolumeManager启动：reconciler ，DesiredStateOfWorldPopulator，volumePluginMgr三个loop
- volumePluginMgr：插件管理，启动CSIInformer，并执行等待同步；csiPlugin使用该Informer接收消息
- DesiredStateOfWorldPopulator：定期执行findAndAddNewPods和findAndRemoveDeletedPods，同步pod中的volume信息；依赖podManager
- reconciler： 定期的处理unmountVolumes，mountAttachVolumes和unmountDetachDevices
- > 以上操作均调用operationExecutor的对应接口
- operationExecutor：
- > 调用operationGenerator生成generatedOperations（为了解耦）
- > 调用pendingOperations.Run方法，
- operationGenerator： 
- > 调用volumePluginMgr，寻找对应插件；
- > 调用volumePlugin的New方法创建插件，
- > 返回OperationFunc：调用plugin执行TearDown等操作
- nestedPendingOperations：启动go程，调用generatedOperations.Run
- GeneratedOperations.run:会执行OperationFunc
- 
## 容器创建
- Kubelet 通过 CRI 接口(gRPC) 调用 dockershim（内嵌在kubelet代码中）
- 请求发送给Docker Daemon
- 请求发送给containerd 
- 调用OCI接口创建一个 containerd-shim进程负责创建容器，并成为容器进程的父进程，负责采集容器状态等上报给containerd 
- 调用runC负责创建容器，runc在容器创建完成之后退出；

## PodSandbox
- RunPodSandbox 
