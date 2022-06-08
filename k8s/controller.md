# controller
- sharedInformerFactory 实现SharedInformerFactory接口
- ControllerClientBuilder接口：实现方式：1.SimpleControllerClientBuilder 2.NewDynamicClientBuilder 用于构建client的集合访问不同的url
- kube-controller-manager
- cloud-controller-manager：云服务商的manager
- Controller manager metrics 默认监听在 kube-controller-manager 的 10252 端口
- 控制器分为：必须启用，默认启动和默认未启动

## 高可用
- --leader-elect=true
- 主节点调用StartControllers()
- 其他节点只运行选主算法
- 实现了两种资源锁（Endpoint 或 ConfigMap
- 通过更新资源的 Annotation（control-plane.alpha.kubernetes.io/leader），来确定主从关系

## 高性能
- Informer 提供了基于事件通知的只读缓存机制，可以注册资源变化的回调函数，并可以极大减少 API 的调用
- 使用方法
## 节点驱逐：
- --node-eviction-rate=0.1：每10s一个节点
- Normal：所有节点都 Ready，默认速率驱逐
- PartialDisruption：即超过33% 的节点 NotReady 的状态，则减慢速率
- FullDisruption：所有节点都 NotReady，返回使用默认速率驱逐。但当所有 Zone 都处在 FullDisruption 时，停止驱逐。
## manager 启动流程 cmd/kube-controller-manager/app/controllermanager.go
- NewControllerManagerCommand 
- Run：创建ControllerContext
- StartControllers ： 启动所有注册的controller
- NewControllerInitializers： 注册所有启动时需要的controller
- 
## deployment controller
- startDeploymentController：启动
- NewDeploymentController：
- > client
- > eventRecorder
- > queue: 工作队列：默认使用NewNamedRateLimitingQueue
- > 为deploymentInformer、replicaSetInformer、podInformer 注册回调函数
- > 注册dLister、rsLister、podLister等，并更新同步状态
- DeploymentControlle.Run:执行
- >WaitForNamedCacheSync :等待各个controller的同步状态HasSynced()为true；
- > 启动k个工作线程：
- > 工作线程从queue（工作队列）获取key,并调用状态同步函数同步状态。
- DeploymentController.worker : 工作协程
- > processNextWorkItem
- > queue.Get
- > syncHandler: 处理状态变更
- > handleErr：处理错误
- syncHandler：状态变更函数,实际执行syncDeployment
- > 从dLister中获取v1.Deployment对象
- > getReplicaSetsForDeployment 获取deployment拥有的rs
- > getPodMapForDeployment获取deployment拥有的pod
- > DeletionTimestamp被设置，表示正在被删除；执行syncStatusOnly
- > checkPausedConditions检查暂停状态；若暂停，则sync同步状态
- > isScalingEvent若是扩容事件；sync更新状态
- > depolyment更新策略：1.rolloutRecreate 2.rolloutRolling
