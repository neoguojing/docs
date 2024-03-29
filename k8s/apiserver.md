# api server
- kubernetes/cmd/kube-apiserver

## 知识点
- 提供集群管理的 REST API 接口
- 提供其他模块之间的数据交互和通信的枢纽
-  https（默认监听在 6443 端口）和 http API（默认监听在 127.0.0.1 的 8080 端口

## 访问控制
- 认证：解决用户是谁的问题：
- > x509证书认证：生成证书和指定证书，启动时--client-ca-file指定证书
- > 静态 Token 文件: 启动时指定--token-auth-file，使用时客户端指定 Bearer Authorization 头
- > 引导token:api server启动--experimental-bootstrap-token-auth，Controller Manager 开启 TokenCleaner，kubeadmin启动时会自动创建token
- > 静态密码文件：API Server 启动时配置 --basic-auth-file=，客户端指定Basic Authorization头
- > openID 提供了 OAuth2 的认证机制
- 授权：解决用户可以做什么
- > ABAC: --authorization-policy-file=
- > webhook授权：-authorization-webhook-config-file
- > Node 授权:--authorization-mode=Node,RBAC --admission-control=...,NodeRestriction,...
- Admission Control：资源管理，对授权后的请求进一步验证，添加参数，对写操作有效
- > 准入控制插件：ServiceAccount，ResourceQuota，EventRateLimit
- > Initializers: 给资源执行策略或者配置默认选项，包括 Initializers 控制器和用户定义的 Initializer 任务，控制器负责执行用户提交的任务，并完成后将任务从 metadata.initializers 列表中删除。
- > 开发参考：https://github.com/kelseyhightower/kubernetes-initializer-tutorial
- > GenericAdmissionWebhook :种 Webhook 方式的准入控制机制，它不会改变请求对象，但可以用来验证用户的请求
- > PodNodeSelector : 用来限制一个 Namespace 中 Pod 所允许运行的 Node
- 

### RBAC授权
- 启动 kube-apiserver 时配置 --authorization-mode=RBAC
- ROLE：角色，一系列权限的集合，负责namespace下的权限管理
- ClusterRole：对namespace和集群以及非资源类api使用；使用aggregationRule 可以与其他Role聚合
- RoleBinding ：把角色（Role 或 ClusterRole）的权限映射到用户或者用户组
- ClusterRoleBinding ：让用户继承 ClusterRole 在整个集群中的权限
- 用户类型保存：User，Group和ServiceAccount
- 在kube-system下默认配置了许多系统Role
## 流程
- NewAPIServerCommand
- Run
