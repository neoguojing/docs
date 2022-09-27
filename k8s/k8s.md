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
- 强制终结： --force + --grace-period=0 ，api pod对象首先被删除；
- 终结pod的垃圾回收：terminated-pod-gc-threshold 超越时， 执行pod清理
- hostname是pod的 metadata.name
#### init container
- 在app容器运行之前运行
- 多个容器串行执行，且必须是一个任务
- 和正常容器的区别是：不支持probe，资源配置也有区别
- pod重启，init 容器重新执行
- 修改init配置，会重启pod
- 修改init镜像不会重启pod
```
apiVersion: v1
kind: Pod
metadata:
  name: myapp-pod
  labels:
    app.kubernetes.io/name: MyApp
spec:
  containers:
  - name: myapp-container
    image: busybox:1.28
    command: ['sh', '-c', 'echo The app is running! && sleep 3600']
  initContainers:
  - name: init-myservice
    image: busybox:1.28
    command: []
```
#### 干扰项
- 非主动干扰： 硬件和操作系统错误等
- 主动干扰： 删除deploy或pod等
##### 解决干扰
- PodDisruptionBudget：pdb,用于限制同时down的应用副本的数量，解决主动干扰问题
- kubectl drain : 标记node失联，驱逐该node上的所有pod，周期性的删除pod
- 区分集群关联员和应用管理员
#### 临时的容器
- 没有端无需设置资源
- 口和probe
- ephemeralcontainers声明该容器
#### 用户空间
- 用于将容器中的用户和主机用户做映射，即容器以root用户运行，映射到主机上是非root用户（容器在主机上没有root权限）
- 保证pod仅在该用户空间有效
- pod.spec.hostUsers
- kubelet选择pod将要映射的UID/GID，并抱枕唯一性
- runAsUser，runAsGroup，fsGroup使用容器内的user
#### downward API
- 用于暴露pod和容器的字段
### 工作负载资源
#### Deployments
- 负责更新和配置pod和 ReplicaSets
- 重要组成部分
- > .spec.replicas 
- > .spec.selector: deploy 如何选帧pod进行管理
- > template: 包含.metadata.labels.app,.template.spec(容器描述)，.spec.template.spec.containers[0].name
- pod-template-hash： 内部标签，确保deploy的ReplicaSets不会重复
- 确保最少75%的pod是up的
- deployment创建后，直接创建一个ReplicaSet负责启动期望数量的pod
#### ReplicaSet
- 用于确保pod的数量
- 通过metadata.ownerReferences，关联相关pod
#### StatefulSets
- 保证pod的顺序性和唯一性，每个pod都有唯一的标识
- 唯一的网络标识，持久化存储
- 删除pod，不会删除volume,需要手动删除
- 需要一个headless service
- 优先scale而非删除
- volumeClaimTemplates ：申明pvc
- pod标签组成
- > Ordinal Index：0-N
- > Stable Network ID:$(statefulset name)-$(ordinal)
- >Stable Storage:
- > statefulset.kubernetes.io/pod-name : controler 标签
- .spec.podManagementPolicy： OrderedReady 或者 Parallel
#### DaemonSet
- 自动在所有节点部署该pod
- 用于每个节点的日志，监控和存储等
- 必要字段：.spec.template .spec.selector
#### job
- job类型：非并行job，并行job.spec.completions，并行job+工作队列.spec.parallelism
#### CronJob
- 时间由controller manage决定
- startingDeadlineSeconds： 在此时间段内统计到了时间未执行的job
#### ReplicationController
- 用于保证指定数量的pod运行和可用，类似一个supervisor
- 可以保证多个node上的多个pod
```
apiVersion: v1
kind: ReplicationController
```
### 服务/网络
#### 服务
- 解耦调用者和对应的pod
- 虚拟ip，cluaster ip
- targetPort： 用于绑定pod的port
···
 targetPort: http-web-svc
···
- 没有选择器的service：1.用于抽象集群外的资源；2.抽象其他service；
- 通过自定义Endpoints对象，绑定无选择器serivec：数量不能超过1000
```
apiVersion: v1
kind: Endpoints
metadata:
  # the name here should match the name of the Service
  name: my-service
subsets:
  - addresses:
      - ip: 192.0.2.42   //不能是虚拟ip
    ports:
      - port: 9376
```
- proxy: 不适用DNS的原因是：DNS不支持ttl
- service.spec.sessionAffinity： 设置session保持
- spec.externalTrafficPolicy： 设置外部流量转发规则,cluster:向集群中可用的endpoint转发；local：只向本机转发
- pec.internalTrafficPolicy：设置内部流量转发规则，同上
- 服务发现：1.环境变量{SVCNAME}_SERVICE_HOST + {SVCNAME}_SERVICE_PORT（服务需要先于pod建立）；2.DNS
- headless： .spec.clusterIP = None；proxy和负责均衡不会处理这类service；
- > 有选择器的： endpoints controller 创建Endpoints，并修改DNS配置使得A记录指向pod
- > 无选择器：DNS会记录 ExternalName-type Services，或A记录记录和service同名的endpoint对象
- 类型
- > ClusterIP 默认类型，只支持集群内访问
- > NodePort: 通过nodeip+nodeport 可以在集群外部访问；--nodeport-addresses指定特定ip；
- > LoadBalancer:  云提供的负载均衡器；spec.loadBalancerClass设置自定义负载均衡器
- > ExternalName: 将service映射到externalName（dns记录），使用一个CNAME记录该值，需要DNS
```
spec:
  type: ExternalName
  externalName: my.database.example.com
```
- externalIPs: 声明外部ip,通过externalIP:port从外部访问服务；不归k8s管理
- 虚拟ip实现：

#### 流量拓扑
- 让流量转发依照node的网络拓扑进行
- 默认情况下转发到ClusterIP/NodePort服务的流量会转发到任意的endpoint
- topologyKeys： 配置node lable，让流量转发优先跑到相关node；kubernetes.io/hostname  topology.kubernetes.io/zone
- 依赖ServiceTopology特性
```
topologyKeys:
    - "kubernetes.io/hostname"
```
#### DNS
- 为service和pod创建dns记录，每个service在dns中都有一条记录
- 包含DNS pod和service
- 服务名称存储于pod的/etc/resolv.conf
- Service（除去headless）： 在DNS里会分配一条A 或者 AAAA记录，解析服务名获取到的是cluster ip
- Headless（没有clusterip）：同样有一条A 或者 AAAA记录，解析服务名，获取到的是pod ip集合，客户端自己做负载均衡
- SRV 记录： 服务于service或者headless
- > my-port-name._my-port-protocol.my-svc.my-namespace.svc.cluster-domain.example
- 每个pod都会有一条记录：od-ip-address.my-namespace.pod.cluster-domain.example
- pod就绪时必须包含一条dns记录
- pod dns策略
- > Default: pod遵循node的dns策略
- > ClusterFirst: 优先遵循cluster的dns策略
- > ClusterFirstWithHostNet
- > None: 必须在dnsConfig 申明策略
- pod dns配置 dnsConfig,记录在pod的/etc/resolv.conf中
- > nameservers : 一系列的ip地址
- > searches: 一系列dns域名
```
 dnsConfig:
    nameservers:
      - 1.2.3.4
    searches:
      - ns1.svc.cluster-domain.example
      - my.dns.search.suffix
    options:
      - name: ndots
        value: "2"
      - name: edns0
```
#### ingress
- 一个API对象，管理外部访问内部Service
- 流量路由由ingress规则控制
- .spec.defaultBackend 默认路由
- Resource backend ：是对k8s资源的引用，必须在同一个命名空间
```
- path: /icons
            pathType: ImplementationSpecific
            backend:
              resource:
                apiGroup: k8s.example.com
                kind: StorageBucket
                name: icon-assets
```
- Ingress class: 包含ingress controller的配置，用于区分不同的controller；scope: Cluster用于控制ingree的作用空间
- defaultBackend： 设置默认后端
#### ingress controller
- 用于执行ingree，提供负载均衡，边界路由等
- .spec.defaultBackend ## opporator开发
#### EndpointSlices
- 用于跟踪network endpoints，不能超过100个
- 用于应对规模巨大的endpoint
- k8s为有选择器的service自动创建一个对象；
- EndpointSlices中的对象必须有一个合法的dns 子域名，
- 里面的对象，可以包含node和zone信息
- endpointslice.kubernetes.io/managed-by： 标签，指定slice的管理者
- 拥有者是Service
#### 服务内部网络策略
- .spec.internalTrafficPolicy
- > Local: 告诉kubeproxy只使用node本地的endpoint
#### 基于网络拓扑的路由
- 在计算service的endpoint时，会注入对应的zone和region信息
- kubeproxy消费对应信息，影响路由
- service.kubernetes.io/topology-aware-hints设置为auto

#### 网络策略
- 用于控制3-4层的流量，告诉一个pod要如何和其他网络实体通信
- 依赖于网络插件，提供一个controller；NetworkPolicy本身是一种资源
- policyTypes: 
- > ingress定义入口策略，允许进入的CIDR范围，命名空间选择器和pod选择器，进入的端口
- > egress : 定义出口策略，CIDR和端口

####ipv4/v6 双栈

### 存储

#### 卷
- 解决持久化和文件共享的问题
- 瞬间卷的生命周期和pod一致
- 卷：是一个目录，可以被pod中的容器访问
- .spec.volumes： 卷定义
- .spec.containers[*].volumeMounts：卷绑定在容器的什么目录
- 卷类型：cephfs，configMap，emptyDir（生命周期同pod）
- > hostPath: 绑定宿主机的文件或文件夹到容器里；可以是dev，socket等等文件或者文件夹，最好是readonly，DirectoryOrCreate/FileOrCreate
- > local: 仅支持静态持久卷；与hostpath类似；node不健康的情况下，相关volume不可用；数据容易丢失；必须设置nodeAffinity
- > gitRepo: 
```
 gitRepo:
      repository: "git@somewhere:me/my-git-repository.git"
      revision: "22f1d8406d464b0c0874075539c1f2e96c253775"
```
- > secret : 基于tmpfs实现，不会落盘
- subPath： 在一个pod内共享一个卷给多个用户，不推荐生产使用
- > volumeMounts.subPath
```
 volumeMounts:
      - mountPath: /var/www/html
        name: site-data
        subPath: html
```
- 根目录：/var/lib/kubelet
- emptyDir/hostPath 在不同的容器之间隔离
- 绑定传播： 在不同的容器之间共享数据卷
- > .volumeMounts.mountPropagation
- > none： 不接受子目录重新挂载，容器卷对于host不可见
- > HostToContainer: 宿主机的目录中挂载新的子目录，则容器内可见；参见linux rslave
- > Bidirectional: 同上，同时容器内创建的目录对宿主机可见；参见linux rshare
- > 开启该特性，需要在docker的systemctl中设置MountFlags=shared
#### 持久卷 抽象存储是如何提供和被消费的
- PersistentVolume： 
- > 一块存储，已经被管理员或者storage class提供；和node资源类似；生命周期独立于pod
- > 静态分配: 管理员手动分配
- > 动态分配：基于StorageClasses；需要启动--enable-admission-plugins；默认DefaultStorageClass
- > pv被删除时也不会立刻删除，直到没有任何pvc绑定
- > kubernetes.io/pvc-protection : finalizer
- > spec:capacity,volumeMode,accessModes，storageClassName，nodeAffinity
- > volumeMode: Filesystem(后台是块设备，k8s会创建一个文件系统)，Block（直接使用，速度块，但是需要自定义如何操作和管理存储）
- accessModes: 
- > ReadWriteOnce : 仅被一个node挂载，可读写，允许同node的pod共享
- > ReadOnlyMany: 被多个node挂载，只读
- > ReadWriteMany: 被多个node挂载，读写
- > ReadWriteOncePod: 只能被一个pod挂载和读写，只要csi支持，
- PersistentVolumeClaim： 
- > 是一个用户的存储请求；等价于pod的角色定位；允许用户消费抽象的存储资源
- > controller 根据pvc请求，去pv寻找合适的资源，做绑定，失败则处于unbound状态
- > pvc被用户删除时，不会立即删除，直到没有任何pod使用
- > kubernetes.io/pv-protection: finalizer
- > spec:capacity,volumeMode,accessModes，storageClassName，matchLabels
- > 沒有storageClassName的使用默認的class；默認class未開啓，則使用PV中申明的class
- > 需要和使用者在同一個命名空間
- 
- 回收策略：
- > 保留： 只支持手动改造资源
- > Delete ： 删除pv对下以及相关的外部存储资产
- > Recycle: 降级
- 预定PV
```
apiVersion: v1
kind: PersistentVolume
metadata:
  name: foo-pv
spec:
  storageClassName: ""
  claimRef:
    name: foo-pvc
    namespace: foo
  ...
```
- PV扩容
- > storage class 中的allowVolumeExpansion为true
- > csi默认支持扩容；仅支持XFS，Ext3和ext4
- > 就绪的pvc支持扩容
- > 扩容失败，controller会不断重试，直到管理员介入；
- > 失败恢复：1.设置pv为Retain；2.删除PVC（不会丢失数据）；3.删除pv的claimRef；4.重建更小的PVC；5.重置PV的reclaim
#### 快照生成和恢復
- 僅樹外的CSI插件支持
- 在pvc中指定數據源爲snapshot
```
dataSource:
    name: new-snapshot-test
    kind: VolumeSnapshot
    apiGroup: snapshot.storage.k8s.io
```
- dataSourceRef: 指定卷寄居者
#### 投影卷
- secret
- downwardAPI
- configMap
- serviceAccountToken

#### 臨時卷
- emptyDir
- configMap, downwardAPI, secret
- CSI ephemeral volumes:
- generic ephemeral volumes

#### storage class
- 用於描述存儲的類
- 包含： provisioner，parameters，reclaimPolicy
- 名字很重要，一經創建不能修改
- kubernetes-sigs/sig-storage-lib-external-provisioner/ kubernetes-sigs/sig-storage-lib-external-provisioner ：外部提供者的代碼目錄
- storage.k8s.io： 提供相關api
#### 動態卷提供者
- 提供按需的存儲分配
- 需要實現storage class接口
- 創建：1.創建StorageClass；2.
- 使用：在PVC中指定或者PV中指定
#### 快照卷
#### 快照卷類
#### CSI
#### 存儲容量
- 調度：pod未創建，聲明了使用CSI驅動的storage class；StorageCapacity爲true，這種情況下，調度器考慮卷的大小來選擇容器
#### 卷健康監控
- CSI volume health monitoring ，是CSI的一部分
- 依賴於：1.外部的外部的健康健康controller；2.kublete
https://zhuanlan.zhihu.com/p/246550722

### 調度、搶佔和驅逐
#### k8s调度
- 默认调度器
- 选取可用nodes，执行一系列函数，打分排序，挑选分数最高的运行pod
- 影响调度的因素有：资源需求，硬件、软件、策略，亲和性和反亲和性，数据本地化，内部流量引用等
- 过滤，打分
- 调度策略： KubeSchedulerConfiguration 对象指定策略配置文件 /etc/srv/kubernetes/kube-scheduler/kubeconfig
- 调度profile：用于配置不同阶段的插件，queueSort，preFilter，filter，postFilter，preScore，score，reserve，permit，preBind，bind，postBind，multiPoint
#### 过滤手段
- nodeSelector(pod)/node lable: 
- 亲和性和反亲和性：1.节点亲和性，用于选择节点；2.pod直接的亲和性和反亲和性：用于设置pod之间的限制
- 节点亲和性： 可以在pod和KubeSchedulerConfiguration对象中指定
```
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution://硬性要求
        nodeSelectorTerms:
        - matchExpressions:
          - key: topology.kubernetes.io/zone
            operator: In
            values:
            - antarctica-east1
            - antarctica-west1
      preferredDuringSchedulingIgnoredDuringExecution: //soft

```
- pod内部亲和性和反亲和性：
- > 在大于n00的集群中会显著拖慢调度的速度
- > operator: 包含：In NotIn Exists DoesNotExist
```
spec:
  affinity:
    podAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
    podAntiAffinity:
      preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 100
        podAffinityTerm:
          labelSelector:
            matchExpressions:
            - key: security
              operator: In
              values:
              - S2
          topologyKey: topology.kubernetes.io/zone //用于标记域
```
- namespaceSelector: 命名空间选择器
- nodeName： 直接通过nodename 选择
- pod调度的额外资源：即runtime消耗的资源，一般在RuntimeClass中使用overhead定义；调度器需要将overhead资源+pod本身请求的资源，来综合考虑调度
- pod拓扑扩展限制：限制pod在失败的域：zone，regions，nodes等中的传播
- > 
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

