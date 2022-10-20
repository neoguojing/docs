# 关键项目

## 扩展
- https://operatorhub.io/

### operator-sdk
- https://github.com/operator-framework/operator-sdk
- operator 开发脚手架
- https://github.com/kubernetes-sigs/controller-runtime
- controller-runtime：operator-sdk依赖的底层库
- > 用于创建Operators，Workload api，Configuration APIs，Autoscalers
- > client: 读写k8s对象
- > Cache： 从本地缓存中读取数据； 集成了client和Informer（包含事件handler操作和存储Index操作）
- > Manager：创建Controller
- > Controller： 依据工作队列的模式实现（ratelimitqueue）
- > Webhook： 认证，用于扩展k8s api
- > Reconciler： 一个函数，被controler调用
- > Source：Controller的一个参数，事件流？
- > EventHandler：Controller的参数，处理事件
- 

## 网络扩展
- https://github.com/containernetworking/cni
- https://github.com/containernetworking/plugins
- https://github.com/flannel-io/flannel
- https://github.com/projectcalico/calico
- https://github.com/kubernetes/dns

## 存储扩展
- https://github.com/kubernetes-sigs/sig-storage-lib-external-provisioner
- https://github.com/jooho/nfs-provisioner-operator
- hostPath: pkg/volume/hostpath/host_path.go
- csi: pkg/volume/csi/csi_plugin.go
- csi: github.com/container-storage-interface/spec/lib/go/csi/csi.pb.go
- https://github.com/ceph/ceph-csi
![volume class](https://img-blog.csdnimg.cn/img_convert/0aaf388e02c6bd9beaec2cefff89c858.png)


## 调度扩展
- https://github.com/kubernetes-sigs/scheduler-plugins
