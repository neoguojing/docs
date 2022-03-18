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
- > deploymentInformer：以下均基于NewSharedIndexInformer实现
- > replicaSetInformer
- > podInformer
- >
- DeploymentController.Run:执行
