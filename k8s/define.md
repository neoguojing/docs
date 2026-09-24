为了直观理解这些概念在日常配置中的位置，下面将它们分为 **原生工作负载（Deployment/Pod）**、**扩展模型（CRD/CR）** 以及 **权限控制（RBAC）** 三类具体的 YAML 结构。

---

### 1. 原生资源对象：Spec、Status、OwnerReference 与 Finalizer

执行 `kubectl get pod <name> -o yaml` 时，会看到完整带运行时字段的 YAML。以下标注了各关键概念所在的位置：

```yaml
apiVersion: v1
# 【Resource】对应核心 API 的 pod 类型
# 【Object】当前这个具体的实例：web-app-79844877f-abcd
kind: Pod
metadata:
  name: web-app-79844877f-abcd
  namespace: default

  # 【Finalizer】防止资源被物理误删；清空该数组前，资源处于 Terminating 状态
  finalizers:
    - kubernetes.io/pvc-protection

  # 【OwnerReference】声明父子从属关系；父级删除时，负责级联清理该 Pod
  ownerReferences:
    - apiVersion: apps/v1
      kind: ReplicaSet
      name: web-app-79844877f
      uid: a1b2c3d4-e5f6-7890-abcd-1234567890ab
      controller: true
      blockOwnerDeletion: true

# 【Spec】用户声明的“期望状态”（Desired State）
spec:
  containers:
    - name: nginx
      image: nginx:1.25
      ports:
        - containerPort: 80
  restartPolicy: Always

# 【Status】由 Controller / kubelet 观测后回写的“实际状态”（Actual State）
# 运维通常不在 apply 文件里手写 status，它由系统自动更新
status:
  phase: Running
  podIP: 10.244.1.42
  containerStatuses:
    - name: nginx
      ready: true
      restartCount: 0

```

---

### 2. 定制与扩展：CRD、CR 与 Operator

这是 Operator 模式的基础。先通过 CRD 注册 API 规范，再通过 CR 创建实际业务对象。

#### ① CRD (CustomResourceDefinition)

平台管理员或 Helm Chart 应用的元数据，通知 API Server 注册新的 Resource：

```yaml
# 【CRD】声明一个集群级的新资源规范
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  name: redisclusters.database.example.com
spec:
  group: database.example.com        # API 组
  versions:
    - name: v1alpha1
      served: true
      storage: true
      schema:
        openAPIV3Schema:             # 校验规则
          type: object
          properties:
            spec:
              type: object
              properties:
                nodes:
                  type: integer
                version:
                  type: string
  scope: Namespaced
  names:
    plural: redisclusters
    singular: rediscluster
    kind: RedisCluster               # 定义出来的 CR 资源类型名称
    shortNames:
      - rdc

```

#### ② CR (CustomResource) 与 Operator 交互

业务运维人员提交给 API Server 的具体实例：

```yaml
# 【CR】通过上面 CRD 衍生出来的具体对象实例
# 【Operator】后台运行的独立进程（监听 RedisCluster 的变化，负责执行 Reconcile 逻辑）
apiVersion: database.example.com/v1alpha1
kind: RedisCluster
metadata:
  name: prod-redis-cluster
  namespace: db
  # Operator 注册的 Finalizer，用于在物理删除前备份 RDB 或解除哨兵注册
  finalizers:
    - redis.example.com/backup-before-delete
spec:
  # 用户期望：6 个节点，版本 7.0
  nodes: 6
  version: "7.0.12"
status:
  # Operator 调谐后写入的实际集群状态
  clusterStatus: "OK"
  syncedNodes: 6

```

---

### 3. 控制面权限：RBAC 配置

Controller 或 Operator 需要使用特定的 ServiceAccount 访问 Kubernetes API Server，RBAC 用于约束其权限范围：

```yaml
# 【RBAC: Role / ClusterRole】定义 Controller 能够操作哪些 Resource 及动作
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: redis-operator-role
rules:
  - apiGroups: ["database.example.com"]
    # 允许操作自定义的 Resource
    resources: ["redisclusters", "redisclusters/status"]
    verbs: ["get", "list", "watch", "create", "update", "patch"]
  - apiGroups: [""]
    # 允许操作基础的 Pods、Services，供调谐时调度底层实例
    resources: ["pods", "services", "endpoints"]
    verbs: ["get", "list", "watch", "create", "update", "delete"]
  - apiGroups: [""]
    # 【Event】允许 Controller 抛出事件以供运维通过 describe 查看排障
    resources: ["events"]
    verbs: ["create", "patch"]
---
# 【RBAC: ClusterRoleBinding】将角色绑定到 Controller 的 ServiceAccount
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: redis-operator-binding
subjects:
  - kind: ServiceAccount
    name: redis-operator-sa
    namespace: operator-system
roleRef:
  kind: ClusterRole
  name: redis-operator-role
  apiGroup: rbac.authorization.k8s.io

```
