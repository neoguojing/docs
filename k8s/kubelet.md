# kubelet
- remoteRuntimeEndpoint = "unix:///var/run/dockershim.sock" : runtime的unix sock
- DockershimRootDirectory:   "/var/lib/dockershim",
- CNIBinDir:   "/opt/cni/bin",
- CNIConfDir:  "/etc/cni/net.d",
- CNICacheDir: "/var/lib/cni/cache",

- kube-controller-manager/app/plugins.go 

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
