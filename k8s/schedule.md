# kube-scheduler
- 监听 kube-apiserver，查询还未分配 Node 的 Pod，然后根据调度策略为这些 Pod 分配节点
- 因素：公平调度，资源高效利用，QoS，affinity 和 anti-affinity，数据本地化（data locality，内部负载干扰（inter-workload interference），deadlines

## 指定NODE节点的调度:适用于pod
- nodeSelector：只调度到匹配指定 label 的 Node 上
- > 首先给node打标签，然后再yaml中指定node
- nodeAffinity：功能更丰富的 Node 选择器，比如支持集合操作
- > 支持两种：必须满足条件和优选条件，在pod ymal中使用
- podAffinity：调度到满足条件的 Pod 所在的 Node 上
- >  支持podAffinity 和 podAntiAffinity

## 保证pod不被调度到节点
### taint：应用于Node对象
- NoSchedule/PreferNoSchedule/NoExecute（删除以及运行的pod）
- kubectl taint nodes node1 key2=value2:NoSchedule
###  toleration：应用于pod上

## 优先级调度，默认开启
- 先定义PriorityClass（非namespace资源）
- pod中指定：priorityClassName

## 自定义调度器：
- podSpec.schedulerName：pod中使用该标签选择调度器
- https://kubernetes.feisky.xyz/extension/scheduler

## 调度扩展
### 调度插件
- https://medium.com/@juliorenner123/k8s-creating-a-kube-scheduler-plugin-8a826c486a1
- KubeSchedulerConfiguration：使用这个对象配置调度器

## 原理
### predicates 过滤不符合条件的节点
- PodFitsHostPorts：端口冲突检测
- PodFitsResources：资源是否充足
- MatchInterPodAffinity：亲和性检测
- MatchNodeSelector：节点匹配
- PodToleratesNodeTaints
- CheckNodeMemoryPressure
- CheckNodeDiskPressure
### priority优先级排序，选择优先级最高的节点
- SelectorSpreadPriority:保证服务或rs分布在不同节点
- InterPodAffinityPriority：区域选择
- LeastRequestedPriority：优先调用到最少使用的节点
- BalancedResourceAllocation：平和节点的资源使用
- NodeAffinityPriority：节点亲和性选择
- TaintTolerationPriority
- 
