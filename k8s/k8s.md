# k8s

## 基本概念
### 控制平面
- kube-apiserver
- etcd
- kube-scheduler
- kube-controller-manager
- cloud-controller-manager ： 应用于云服务商
### node
- kubelet
- kube-proxy 
- Container runtime : containerd, CRI-O
### 插件 使用k8s资源实现集群特性
- DNS ： 
- Web UI (Dashboard)
- Container Resource Monitoring
- Cluster-level Logging
### 对象 record of intent(记录意图)
- spec： 记录期望状态
- status： 当前状态，由系统提供
- 详细文档：https://github.com/kubernetes/community/blob/master/contributors/devel/sig-architecture/api-conventions.md
#### 必要字段
- apiVersion
- kind ： 对象类型
- metadata： 帮助唯一区分对象；name ，UID 和 namespace
- spec
- 格式参考：https://kubernetes.io/docs/reference/kubernetes-api/
#### 如何区分对象
- 命名方式：name-uuid
- uuid由系统产生，全局唯一： ISO/IEC 9834-8  ITU-T X.667
#### namespace
- namespace：用于区分不同用户：
- kube-system ： 系统创建的对象
- kube-public ： 所有公用的对象
- kube-node-lease： 维护lease 对象为每个节点
- 不在namespace下的资源：node，storageClass等
- kubernetes.io/metadata.name： label标记namespace的名称
#### 标签和选择器
- https://kubernetes.io/docs/concepts/overview/working-with-objects/common-labels/： 推荐的label
- 用于组织和选择对象的子集
- 由prefix/key: value 构成prefix可以省略
- kubernetes.io/ ，k8s.io/ 前缀保留核心组件使用
- 选择起类型：1.等价；2.集合； 
- 选择多个标签用“,”分割
- 
```
metadata:
  name: label-demo
  labels:
    environment: production
    app: nginx
    
selector:
    component: redis
    matchLabels:
      component: redis
    matchExpressions:
      - {key: tier, operator: In, values: [cache]}
      - {key: environment, operator: NotIn, values: [dev]}
```
#### Annotations 注解
- 用于在对象的metadata，添加自定义的属性
```
metadata:
  name: annotations-demo
  annotations:
    imageregistry: "https://hub.docker.com/"
```
#### 字段选择器
- kubectl get pods --field-selector status.phase=Running
- 通用字段：metadata.name=my-service，metadata.namespace!=default

#### Finalizers
- 通知k8s等待知道一定条件，才销毁对象；此时对象处于terminating状态
- 用于进行垃圾回收
- k8s有内置finalizers，用户可自定义
- kubernetes.io/pv-protection： 阻止PersistentVolume被删除

#### 父对象和子对象
- ReplicaSet  是pod的父对象
- metadata.ownerReferences： 指向父对象
- ownerReferences.blockOwnerDeletion： 阻止删除父对象，默认true

### 集群概念
#### node
- 通过"kind": "Node"的yml手动注册，metadata.name为ip地址
- 通过kubelet 的--register-node 来实现自动注册
- kubectl cordon 使得node不可被调度
- 节点状态包括：地址，状态，容量和是否可分配
##### 心跳： 
- 1.kubelet更新节点的.status 文件，5分钟更新一次；
- 2. 每个节点维护一个lease，10s更新一次
##### node controler
- 分配CIDR地址
- 保持内部node列表的更新
- 监听节点健康状态：1.更新状态；2.若5分钟不在线，则启动eviction驱逐pod
- --node-eviction-rate : 默认0.1/s,每10s不会驱逐超过1个node上的pod
##### Graceful node shutdown
-  v1.21引入
-  shutdownGracePeriod和shutdownGracePeriodCriticalPods，控制关闭pod的时间
-  依赖于systemd inhibitor locks 
-  PriorityClass：控制 graceful node shutdown的优先级
##### swap
- cgv1支持swap
```
memorySwap:
  swapBehavior: LimitedSwap
```
#### 垃圾回收
- 包含：pod终止，job完成，没有owner的对象，未使用的image和镜像，声明了storageclass的pv，和node lease等
##### 瀑布删除
- 前台： 设置deletionTimestamp，设置finalizers=foregroundDeletion，删除依赖，删除owner
- 后台： 删除owner，k8s后台删除依赖
##### 容器和镜像删除
- 镜像： 磁盘使用高于HighThresholdPercent，通过 image manager，删除最早未使用的镜像，知道磁盘利用率达到LowThresholdPercent
- 容器： 删除最早的
### 容器
#### 镜像
- pull策略： IfNotPresent，Always，Never
- ImagePullBackOff： 容器pull失败，5分钟后重试
- 多镜像索引： pause-amd64，加上arch后缀，实现不同架构获取不同镜像
#### 运行时类
- 用于选择容器的运行时配置；运行时配置用于启动pod容器
- 1. 配置运行时实现；2.配置对应的runtimeclass资源
```
apiVersion: node.k8s.io/v1
kind: RuntimeClass
metadata:
  name: myclass 
handler: myconfiguration 
---
apiVersion: v1
kind: Pod
metadata:
  name: mypod
spec:
  runtimeClassName: myclass
```
#### 生命周期钩子
- PostStart ： 钩子在镜像创建后立马执行
- PreStop： 在容器terminated后立即执行

### 工作负载 workload
#### pod
- 阶段：pending，running，failed，Succeeded，Unknown
- status： Waiting，Running，Terminated
- condition： PodScheduled，PodHasNetwork，ContainersReady，Initialized，Ready
- readiness： 用于向pod注入自定的condition；由pod的status.condition字段获取；该字段不影响pod的condition
- probe：
- > livenessProbe: 报告容器是否在运行;失败会kill 容器执行重启；默认为成功； 容器内应用会自动崩溃，则不需要
- > readinessProbe： 容器是否准备回应请求，失败则从service移除对应endpppoint的ip，默认成功；可以用来帮助检测依赖服务是否正常
- > startupProbe: 容器内应用是否启动；失败则kill并重启；默认为成功；适用于初始化占用较长时间的应用；
- 终结流程
- > kubectl 删除pod，grace period 30s
- > api server 更新pod状态为Terminating；kubelet启动pod终结流程：1.执行prestop hook；2.发送TERM信号给容器的1号进程；
- > 同时控制平面删除endpoint中的pod对象，从Servic，Replicaset移除pod
- > grace period 超时，开始执行强制关闭；CRI发送SIGKILL，关闭容器里所有的进程；kubelet关闭pause容器
- > kubelet触发api server移除pod对象，并设置grace period为0
- > api server 删除pod对象
## opporator开发
https://zhuanlan.zhihu.com/p/246550722

## nodeport
- 流量转发给kube-proxy,kube-proxy下发路由规则给iptable，同时创建nodeport的端口监听
- 通过iptable 查看 KUBE-EXTERNAL-SERVICES，为nodeport的条目
- 高可用问题：1.pod不在这台机器，则需要转移流量；2.node机器挂掉，则表现为请求不可达，但是服务事实上可用的行为

## API 分组和版本

### 访问格式
- /apis/GROUP/VERSION/RESOURCETYPE：RESOURCETYPE如：namespaces ，pod，services
- /apis/GROUP/VERSION/RESOURCETYPE/NAME
- /apis/GROUP/VERSION/namespaces/NAMESPACE/RESOURCETYPE : 某个namesapce下的值

### list watch
- 首先使用list返回需要的资源,包含一个meta：resourceVersion；
- watch监听自resourceVersion之后的所有CUD事件；并返回对象
- watch连接中断，watch可以从最后一个resourceVersion开始重新监听
- watch连接中断，也可以重新执行list，然后watch
- etcd维持5分钟的历史版本数据；若list或watch时出现401 gone，即数据过期；需要清理本地缓冲，然后重新执行list 和watch
- watch拿到的resourceVersion可能时不连续的：因为有些版本当前watch并不关心
- 为了保持watch拿到最新的resourceVersion，减少list，实现bookmarks机制；服务端只通知最新的resourceVersion，而不返回数据；这样客户端就直到最新的resourceVersion；而不需要重新list

