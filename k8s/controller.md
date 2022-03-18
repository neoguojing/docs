# controller
- sharedInformerFactory 实现SharedInformerFactory接口
- ControllerClientBuilder接口：实现方式：1.SimpleControllerClientBuilder 2.NewDynamicClientBuilder 用于构建client的集合访问不同的url
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
- DeploymentController.Run:执行
- >WaitForNamedCacheSync :等待各个controller的同步状态HasSynced()为true；
- > 启动k个工作线程：
- > 工作线程从queue（工作队列）获取key,并调用状态同步函数同步状态。
