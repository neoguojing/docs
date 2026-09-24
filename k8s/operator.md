## 把一个业务领域的“期望状态”声明成 Kubernetes Resource，然后通过 Controller 持续把实际状态拉回期望状态。
## Operator = Custom Resource + Controller


```
                  用户
                   │
                   │ Task
                   ▼
              Management
                Service
                   │
                   │ create
                   ▼
              ┌─────────┐
              │  AIJob  │
              │  CR     │
              └────┬────┘
                   │
              desired state
                   │
                   ▼
        ┌────────────────────┐
        │   AIJob Operator   │
        │                    │
        │ Watch              │
        │ Reconcile          │
        │ Status             │
        │ Recovery           │
        └─────────┬──────────┘
                  │
                  │ create
                  ▼
                Pod
                  │
                  ▼
              Scheduler
                  │
                  ▼
              GPU Node
                  │
                  ▼
              Kubelet
                  │
                  ▼
               Worker
```

可以。后面我会采用 **“概念定义 → 最小示例 → 一句话理解”** 的紧凑格式，减少大段解释和重复架构图。

例如刚才这部分可以压缩成下面这样：

## Kubernetes Operator 核心概念

```text
Kubernetes API
│
├─ Resource      资源类型
├─ Object        资源实例
├─ Metadata      对象身份/关联信息
├─ Spec          期望状态
└─ Status        实际状态
      │
      ▼
     CRD          定义新的 Resource
      │
      ▼
      CR          Resource 的具体实例
      │
      ▼
  Controller
      │
      ├─ Watch       监听资源变化
      ├─ Event       资源变化事件
      ├─ Queue       待处理对象队列
      └─ Reconcile   调整实际状态
              │
              ▼
        Child Resource
          ├─ Pod
          ├─ Job
          ├─ Deployment
          └─ Service
```

### 1. Resource

**定义：** Kubernetes API 中的一种资源类型。

```yaml
apiVersion: v1
kind: Pod
```

常见 Resource：

```text
Pod / Job / Deployment / Service / ConfigMap / Node
```

> Resource = “有什么类型的东西”。

---

### 2. Object

**定义：** Resource 的一个具体实例。

```yaml
kind: Pod
metadata:
  name: worker-001
```

这里：

```text
Pod       → Resource 类型
worker-001 → Object 实例
```

> Object = “具体是哪一个”。

---

### 3. Metadata

**定义：** 描述 Object 身份、标签和关联关系。

```yaml
metadata:
  name: worker-001
  namespace: ai
  labels:
    app: inference
    task: task-001
```

常用字段：

```text
name            对象名称
namespace       命名空间
labels          查询/选择对象
annotations     附加信息
ownerReferences 父子资源关系
```

> Metadata = “这个对象是谁、属于谁、怎么查”。

---

### 4. Spec

**定义：** 用户声明的 **Desired State（期望状态）**。

```yaml
spec:
  containers:
    - name: worker
      image: face-worker:v1
```

AIJob：

```yaml
spec:
  image: face-worker:v1
  gpu: 1
  cpu: "4"
```

> Spec = “我希望系统变成什么样”。

---

### 5. Status

**定义：** Controller 记录的 **Actual State（实际状态）**。

```yaml
status:
  phase: Running
  workerPod: worker-001
```

核心关系：

```text
Spec   = 我要什么
Status = 现在是什么
```

---

### 6. CRD

**Custom Resource Definition**

**定义：** 向 Kubernetes 注册一种新的 Resource 类型。

```yaml
kind: CustomResourceDefinition

spec:
  group: ai.example.com
  names:
    kind: AIJob
    plural: aijobs
```

安装 CRD 后：

```bash
kubectl get aijobs
```

Kubernetes 就认识：

```text
AIJob
```

> CRD = “定义一种新资源”。

---

### 7. CR

**Custom Resource**

**定义：** CRD 定义出来的一个具体实例。

```yaml
apiVersion: ai.example.com/v1
kind: AIJob

metadata:
  name: face-task-001

spec:
  image: face-worker:v1
  gpu: 1
```

关系：

```text
CRD
 │
 │ 定义
 ▼
AIJob Resource
 │
 ├─ face-task-001
 ├─ face-task-002
 └─ ocr-task-001
```

> CRD 是“类型”，CR 是“对象”。
#### Kubernetes Go Client代码差创建CR
```
job := &aiv1.AIJob{
    ObjectMeta: metav1.ObjectMeta{
        Name: "face-task-001",
    },
    Spec: aiv1.AIJobSpec{
        Image: "face-worker:v1",
        GPU:   1,
    },
}

client.Create(ctx, job)
```
---

### 8. Controller

**定义：** 负责观察 Resource，并通过 Reconcile 让实际状态接近期望状态的程序。

```go
func (r *AIJobReconciler) Reconcile(
    ctx context.Context,
    req ctrl.Request,
) (ctrl.Result, error) {

    var job AIJob

    if err := r.Get(
        ctx,
        req.NamespacedName,
        &job,
    ); err != nil {
        return ctrl.Result{}, client.IgnoreNotFound(err)
    }

    // 检查 Worker
    // 不存在 → 创建 Worker

    return ctrl.Result{}, nil
}
```

> Controller = “负责把 Spec 变成现实”。

---

### 9. Watch

**定义：** Controller 监听 Kubernetes Resource 的变化。

```go
For(&AIJob{}).
Owns(&corev1.Pod{})
```

表示：

```text
Watch AIJob
Watch AIJob 创建的 Pod
```

例如：

```text
AIJob Created
      ↓
    Watch
      ↓
 Reconcile
```

> Watch = “有什么变化通知我”。

---

### 10. Event

**定义：** Resource 发生变化产生的事件。

例如：

```text
AIJob Created
AIJob Updated
Pod Created
Pod Deleted
Pod Running
Pod Failed
```

典型流程：

```text
Event → Queue → Reconcile
```

> Event = “发生了什么变化”。

---

### 11. Queue

**定义：** Controller 等待处理的对象队列。

```text
┌─────────────────────┐
│ Reconcile Queue     │
├─────────────────────┤
│ AIJob/task-001      │
│ AIJob/task-002      │
│ AIJob/task-003      │
└─────────────────────┘
```

> Queue = “哪些对象等着处理”。

---

### 12. Reconcile

**定义：** 比较 **Desired State** 和 **Actual State**，执行必要操作。

```text
Spec
 │
 │ Desired = Worker 1
 ▼
Controller
 │
 │ 检查
 ▼
Actual = Worker 0
 │
 │ 不一致
 ▼
Create Pod
 │
 ▼
Actual = Worker 1
```

如果已经一致：

```text
Desired = 1
Actual  = 1
    ↓
Nothing to do
```

如果 Pod 被删除：

```text
Desired = 1
Actual  = 0
    ↓
Reconcile
    ↓
Create Pod
```

> Reconcile = “把现实拉回目标”。

---

### 13. Child Resource

**定义：** Controller 创建并管理的下游 Resource。

你的场景：

```text
AIJob
 │
 ├── Pod
 ├── Job
 ├── Deployment
 └── Service
```

例如：

```text
AIJob
  │
  │ Operator
  ▼
Pod
```

或者：

```text
AIJob
  │
  ▼
Job
  │
  ▼
Pod
```

---

## 14. 你的 AI Worker 场景完整对应

```text
Management Service
        │
        │ Create AIJob
        ▼
     AIJob CR
        │
        │ Watch
        ▼
 AIJob Controller
        │
        │ Reconcile
        ▼
     Worker Pod
        │
        │ GPU=1
        ▼
 Kubernetes Scheduler
        │
        ▼
     GPU Node
```

对应关系：

| Kubernetes 概念  | 你的 AI 系统                  |
| -------------- | ------------------------- |
| CRD            | AIJob 类型定义                |
| CR             | 一个具体 AI Task              |
| Spec           | GPU/CPU/Image 等需求         |
| Status         | Pending/Running/Succeeded |
| Controller     | AI Worker Operator        |
| Watch          | 监听 Task/Worker            |
| Event          | Task/Pod 状态变化             |
| Queue          | 待 Reconcile 的 Task        |
| Reconcile      | 确保 Worker 存在且状态正确         |
| Child Resource | Worker Pod/Job            |
| Scheduler      | 选择 GPU Node               |

### 最重要的 5 句话

```text
CRD        = 定义“有什么资源”
CR         = 创建“一个具体资源”
Spec       = 声明“我想要什么”
Status     = 记录“现在是什么”
Controller = 不断把“现在”拉向“我想要的”
```

后面继续讲 **Operator / Kubebuilder** 时，我会保持这种紧凑格式：**概念 + 最小 YAML/Go + 关键原理**，避免为了完整而把篇幅拉得太长。
