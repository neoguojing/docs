# kubelet
- remoteRuntimeEndpoint = "unix:///var/run/dockershim.sock" : runtime的unix sock
- DockershimRootDirectory:   "/var/lib/dockershim",
- CRI：容器运行时接口：dockershim，containerd等
- dockershim：负责初始化pause容器的网络namespace
- pause容器：内置永远阻塞的进程，负责占用一个网络namespace，新建容器执行docker run --net=none加入pause的namesapce
- CNI : 容器网络接口：内置插件如p2p，bridge,负责创建pause容器网络设备eth0，并绑定IP，第三方插件flannel,calico，解决容器跨主机通信问题
- Network Namespace的网络栈包括：网卡（Network interface）、回环设备（Loopback Device）、路由表（Routing Table）和iptables规则。
- kube-controller-manager/app/plugins.go 
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
## 容器创建
- Kubelet 通过 CRI 接口(gRPC) 调用 dockershim（内嵌在kubelet代码中）
- 请求发送给Docker Daemon
- 请求发送给containerd 
- 调用OCI接口创建一个 containerd-shim进程负责创建容器，并成为容器进程的父进程，负责采集容器状态等上报给containerd 
- 调用runC负责创建容器，runc在容器创建完成之后退出；
## PodSandbox
- RunPodSandbox 
