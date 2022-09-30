# 如何扩展 k8s

##  自定义资源
- resource： 是可扩展的api，在默认的k8s中，不是必须的，用于存储和获取结构化数据
- controllers： 提供申明式api，保证期望状态和当前状态同步

### 声明式api
- api由多个相关的对象构成
- 会定义应用的配置
- 更新不是频繁
- 需要被人读和写
- 需要CRUD
- 不需要事物

### 如何添加客户资源
- CRD= CustomResourceDefinitions
- API Aggregation 

### CRD
- 定义一个CRD对象
- k8s api处理自定义资源的存储
- 不需要编写自己的api server
- 灵活性不够
- https://github.com/kubernetes/sample-controller
- 规模较小

###  API server aggregation
- 自定义api server

## 插件
- 包含：CSI插件，设备插件，网络插件
### 网络插件
- 需遵循CNI 接口
- CNI插件的申明由容器运行时执行
- CNI loopback plugin.
- portmap插件： 管理hostPort
- Support traffic shaping： bandwidth 

### 设备插件


### oprerator
- https://operatorframework.io/
