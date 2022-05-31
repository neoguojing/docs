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
- runtimeClass：主要用来解决多个容器运行时混用的问题
- > http://dockerone.com/article/9990
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
- > secretManager：使用kubeClient从apiserver获取对应的secret
- > configMapManager：使用kubeClient从apiserver获取
- > livenessManager：在worker中使用，根据worker中的状态为livenessManager设置值（赋值给resultsManager）
- > startupManager：在worker中使用，根据worker中的状态为livenessManager设置值（赋值给resultsManager）
- > podManager：实现：basicManager:维护内存pod集合，以及提供secret，configmap的manager；实现pod的增删改查；
- > statusManager： 维护kubeclient、pod状态集合；Start启动之后，监听podStatusChannel，向apiserver同步状态；
- > runtimeClassManager：启动infomer，监听runtimeClass，并返回Handler
- > kubeGenericRuntimeManager: 参加容器运行时
- > CRIStatsProvider ： 从cadvisor获取node状态和CRI获取容器状态
- > GenericPLEG： pleg pod生命周期管理
- > realContainerGC：封装容器GC，底层调用kubeGenericRuntimeManager实现
- > ImageGCManager：负责清理image
- > probeManager：管理liveness，readness和startUp；监听readness和startup，将事件写入statusManager
- > tokenManager：从APiserver获取token，并缓存
- > VolumePluginMgr： 参见卷管理
- > pluginManager：插件管理，监听文件夹变化，并启动协调器，协调状态
- > volumeManager ：参见卷管理
- > podWorkers: ；实现podWorkers，每个pod一个协程，维护pod对象集合，工作队列和pod状态缓存等；由syncloop调用dispatchWork调用UpdatePod接口，为每个新建的pod启动一个managePodLoop协程，用于监听UpdatePodOptions管道执行相关状态同步；
- > nodeLeaseController
- > admitHandlers： 监听node是否关闭，关闭的话拒接所有pod
- > softAdmitHandlers ： 注册的一系列软handler，在启动pod的时候调用
- > shutdownManager： 处理节点关闭事件
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
#### ContainerManager：管理机器上的容器
- setupNode：
- > cgroupManager
- InternalContainerLifecycle：负责在容器启动或停止后执行动作，主要包括添加和删除CPU和维护pod拓扑
- > PreStartContainer: 调用cpuManager.AddContainer添加CPU资源；调用topologyManager.AddContainer添加拓扑
- > PostStopContainer: 从拓扑上移除容器
- cpuManager： 启动循环，根据policy协调容器的资源状态
- > 调用runtimeClient.UpdateContainerResources为容器添加cpu资源
- topologyManager：通过map记录容器和pod的映射关系，实现是在scope类中实现的；
- cgroupManager：提供Create和Update等接口，负责创建和更新cgroup;
- > 资源类型包括：内存/cpu相对份额/cpu在某个时间段内的可用时间/数据页映射限制/进程数限制
- > 底层使用runc/libcontainer/cgroups管理cgroup的创建和更新等操作
- qosContainerManager：控制服务质量
- > 分为：Guaranteed/Burstable/BestEffort
- > 调用cgroupManager，完成qos cgroup的创建和更新
### syncPod pod状态同步主逻辑：

#### SyncPodKill状态处理：
- statusManager.SetPodStatus 设置状态
- Kublete.killPod:kubeGenericRuntimeManager.KillPod依次此执行: 
- > internalLifecycle.PreStopContainer:不作任何事
- > runtimeService.StopContainer
- > runtimeService.StopPodSandbox
#### pod创建和更新：
- statusManager.SetPodStatus： 设置状态
- runtimeState.networkErrors： 网络plugin就绪
- makePodDataDirs： 创建pod目录：pod目录，卷目录和插件目录
- volumeManager.WaitForAttachAndMount： 等待卷挂载完成
- containerRuntime.SyncPod：同步pod状态
#### 容器运行时
- kubeGenericRuntimeManager： 即containerRuntime，集成了image，gc，健康检查等功能,实现了Runtime接口，包含同步和killpod，和image管理等接口
- > SyncPod：检查状态变更，kill sanbox和其他容器；创建sandbox（需要的话），瞬息容器，初始化容器和正常容器;
- 停止sandBox：runtimeClient.StopPodSandbox等接口执行动作；
- 停止容器：同SyncPodKill
- 创建sandBox：调用runtimeClient.RunPodSandbox
- 启动容器: 
- > 1.pull镜像；
- > 2.runtimeClient.CreateContainer 
- > 3.internalLifecycle.PreStartContainer；
- > 4.runtimeService.StartContainer
- > 5.执行post hook
#### docker客户端和dockershim
- unix:///var/run/docker.sock ： runtimeClient连接地址
- unix:///var/run/dockershim.sock: RemoteRuntimeService连接地址
- PreInitRuntimeService: 根据配置决定选用容器运行时：
- > 若为docker，则启动runDockershim
- > 创建remoteRuntimeService，和server通信
- runDockershim： 启动dockershim
- > NewDockerService：创建docker服务，初始化网络插件，并启动网络插件管理服务
- > NewDockerServer: 创建docker服务端，启动grpc和http服务
### 插件管理：
- pluginManager：CSI和Device
- > 在kubelet启动时，注册CSIPlugin和DevicePlugin回调
- > 1.监听文件系统通知，处理创建和删除事件，并缓存相关目录
- > 2. reconcile,注册或者去注册相关的插件，最终调用注册的函数处理：分别为RegistrationHandler和deviceManager
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

### GC
- containerGC ： pod垃圾回收
- > 依次：驱逐容器，驱逐sandbox，删除日志目录
- realImageGCManager：image清理
- > 每5分钟，更新内存记录的使用种的image，每30s更新缓存
- > GarbageCollect: 统计磁盘用量，当达到阈值，则清理一定量的image
- > 真正的清理动作有kubeGenericRuntimeManager调用CRI完成
## 容器创建
- Kubelet 通过 CRI 接口(gRPC) 调用 dockershim（内嵌在kubelet代码中）
- 请求发送给Docker Daemon
- 请求发送给containerd 
- 调用OCI接口创建一个 containerd-shim进程负责创建容器，并成为容器进程的父进程，负责采集容器状态等上报给containerd 
- 调用runC负责创建容器，runc在容器创建完成之后退出；

## PodSandbox
- RunPodSandbox 
