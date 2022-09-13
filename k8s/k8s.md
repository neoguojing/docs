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

