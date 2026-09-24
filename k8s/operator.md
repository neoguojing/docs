# Kubernetes Client-go 与 Operator 核心机制技术全景文档

---

## 模块一：核心概念体系与映射模型

Kubernetes 系统的本质是**基于 etcd 的声明式状态机**。各层概念的严格映射如下：

### 1. 概念字典与代码映射

| 概念 | 架构含义 | 代码/物理映射 | 核心职责 |
| --- | --- | --- | --- |
| **API Server** | 声明式状态总线与网关 | `kube-apiserver` 二进制进程 | 唯一与 etcd 交互的组件，提供认证、鉴权、准入控制及 MVCC 增量广播。 |
| **Resource** | 资源的抽象类型定义 | REST 路径（如 `/api/v1/pods`） | 类似关系型数据库的“表定义”（Class）。 |
| **Object** | 资源在运行时的具体实体 | etcd 内的一条 JSON/Protobuf 记录 | 类似数据表中的“某一行记录”（Instance），包含全局唯一 UID 与版本号。 |
| **Pod** | 最小原子运行单元 | Linux Namespace + Cgroups + Pause 容器 | 容器共享网络栈（IP、端口）与存储卷（Volume）的组合实体。 |
| **Deployment** | 声明式无状态版本控制器 | `DeploymentController`（运行在 Controller Manager） | 维护 ReplicaSet 的创建、扩缩与滚动发布（RollingUpdate/Rollback）。 |
| **CRD** | 用户自定义资源类型规范 | `apiextensions.k8s.io/v1` 的元类型 | 动态向 API Server 注册新的 Schema，扩展集群 API 能力。 |
| **CR** | CRD 的具体实例化对象 | 符合 CRD Schema 的业务声明 | 业务自定义配置的实际载体（如 `RedisCluster`）。 |
| **Operator** | 自动化运维领域专业控制器 | CRD + Custom Controller 进程组合 | 将运维工程师的领域知识（高可用、备份、拓扑重构）固化为代码的自治系统。 |
| **Reconcile** | 调谐状态收敛闭环 | `Reconcile(ctx, Request) (Result, error)` | 水平触发计算：“实际状态（Actual）$\rightarrow$ 期望状态（Desired）”。 |
| **Spec** | 用户声明的期望状态 | 对象 YAML 中的 `spec:` 字段 | 系统的输入基准，代表运维人员的意图。 |
| **Status** | 控制器记录的实际观测状态 | 对象 YAML 中的 `status:` 字段 | 系统的观测输出，由 Controller/kubelet 写入，不污染 `spec.generation`。 |
| **Event** | 对象生命周期的伴随事件 | `core/v1.Event` 资源 | 记录系统状态迁移的历史痕迹（生命周期约 1 小时，存 etcd）。 |
| **Watch** | HTTP/2 增量流式监听机制 | HTTP/2 Chunked Stream 接口 | 基于 etcd MVCC Revision 的长连接事件驱动通知。 |
| **OwnerReference** | 元数据父子依赖关系 | `metadata.ownerReferences` 数组 | 串联“子找父”（反向唤醒）与“父死子随”（GC 级联物理删除）。 |
| **Finalizer** | 异步清理拦截器 | `metadata.finalizers` 字符串数组 | 资源物理删除前的预检钩子，阻塞 etcd 物理抹除直到外部资源释放完毕。 |
| **RBAC** | 控制面权限模型 | Role, ClusterRole, Binding | 约束 Controller 的 ServiceAccount 能操作的 API 组、资源与动作（Verbs）。 |

---

## 模块二：List-Watch 机制与底层存储原理

List-Watch 是 Kubernetes 控制器实现**高性能、毫秒级响应、强一致性且低 etcd 压力**的底层通信支柱。

```
+────────────────+                +──────────────────────────────────────+                +────────────────+
| Controller     |                |             kube-apiserver           |                |      etcd      |
| (Reflector)    |                |                                      |                |                |
+────────────────+                +──────────────────────────────────────+                +────────────────+
        │                                    │                                                     │
        │ 1. List 全量 (RV="")                │                                                     │
        ├───────────────────────────────────►│ 查本地 watchCache / etcd ───────────────────────────►│
        │◄───────────────────────────────────┤ 返回全量对象快照 + RV=1000 ◄─────────────────────────┤
        │                                    │                                                     │
        │ 2. Watch 增量 (RV=1000)             │                                                     │
        ├───────────────────────────────────►│                                                     │
        │    (HTTP/2 长连接保持)               │ 注册 Watcher 到 watchCache (环形滑动窗口)              │
        │                                    │                                                     │
        │                                    │ 发生写事务，生成全局新 Revision (RV=1001)            │
        │                                    │◄────────────────────────────────────────────────────┤
        │ 3. Chunked 流式推送事件 (RV=1001)   │                                                     │
        │◄───────────────────────────────────┤                                                     │

```

### 1. 为什么必须同时使用 List 和 Watch？

* **仅用 List（轮询）**：轮询引发高延迟与 API Server/etcd 的 CPU 穿透；当集群对象达到数万级别时，系统会瞬间因序列化与网络 I/O 崩溃。
* **仅用 Watch（纯流式）**：分布式网络抖动、闪断或重启不可避免。纯增量模式在断连后将失去全局基准状态，无法自愈。
* **协同机制**：
* **List（全量基准）**：启动时调用 `List` 获取全量快照，同时拿到集群全局唯一的版本游标 `resourceVersion`（RV）。
* **Watch（低延迟增量）**：随后以该版本号发起 `Watch(RV)`，利用 HTTP/2 Chunked 长连接仅接收 `RV > 1000` 的后续增量事件。



### 2. etcd 层的实现：MVCC 与全局 Revision

* etcd v3 是基于多版本并发控制（MVCC）的键值数据库。底层通过 B-tree 维护 Key 与 `revision` 的映射，通过 BoltDB（B+ Tree）将 `revision` 作为主键写入磁盘。
* 每一个写事务（Create/Update/Delete）都会使全局单调递增的 `revision` 增加 1。
* etcd 的 Watch 机制基于该递增的 `revision` 进行二分查找，通过 gRPC Server-Streaming 将历史与新变动按序推回。

### 3. API Server 层的减负机制：watchCache 环形滑动窗口

若上千个节点的 Kubelet 和 Controller 直连 etcd，etcd 将因连接数过载而崩溃。

* API Server 在内存中为每种资源类型维护一个 **`watchCache`**，本质是一个固定大小的**环形缓冲区（Ring Buffer）**（如默认容量 100）。
* API Server 自身充当 etcd 的单个 Watch 客户端，将 etcd 广播的事件存入环形缓冲区。
* 外部 Client（Reflector）发起的 `Watch(RV=1000)` 直接打到 API Server 的内存滑动窗口中匹配。
* **高频考点：HTTP 410 Gone (Expired)**：
如果客户端断线时间过长，重新发起 `Watch(RV=1000)` 时，API Server 的滑动窗口或者 etcd 的历史已被 Compaction（周期压缩），最小可用版本已变成 1050。API Server 会立刻返回 **`410 Gone`**。Reflector 捕获 410 状态码后，会重置状态，**重新触发一次全量 List 刷新基准版本**。

---

## 模块三：Client-go 核心数据流水线（1~9 步逐层拆解）

本部分严格拆解 Client-go Informer 与 Custom Controller 数据流图中的 9 个执行阶段。

### 阶段 1：`1) List & Watch` —— 远端数据拉取

* **组件关系**：`Reflector` $\rightarrow$ `Kubernetes API Server`
* **底层机制**：`Reflector` 内部运行 `ListAndWatch()`。
1. 调用 `List` 接口获取该资源的全量对象快照，并获取响应 Header 中的 `resourceVersion`。
2. 携带该 `resourceVersion` 发起 HTTP/2 Chunked GET 请求，挂载长连接流，实时接收服务端下发的事件数据帧。



### 阶段 2：`2) Add Object` —— 写入增量时序缓冲池

* **组件关系**：`Reflector` $\rightarrow$ `DeltaFIFO`
* **底层机制**：Reflector 收到事件后，将其封装为 `Delta{Type, Object}`，调用 DeltaFIFO 的 `Add(obj)`、`Update(obj)` 或 `Delete(obj)` 方法。
* **DeltaFIFO 核心源码数据结构**：
```go
type DeltaFIFO struct {
    lock sync.RWMutex
    cond sync.Cond

    // 真正存放对象增量历史的 Map：Key 是 "namespace/name"
    // Value 是针对该对象的多次变更动作切片（保留微时序）
    items map[string]Deltas

    // 维护绝对先进先出时序的 Key 切片，保证 Key 唯一
    queue []string

    // 关联本地 Indexer 存储，用于 Resync 时检测远端已被物理删除的对象
    knownObjects KeyListerGetter
}

```


* **去重与时序并存机制**：
若短时间内针对同一个 Key 连续发生 `Added -> Updated -> Deleted`：
* `items["default/pod-a"]` 切片会依次追加这 3 个动作（微时序完整保留）；
* `queue` 切片在发现 `"default/pod-a"` 已存在时**不重复入队**。队列始终维持单一 Key 调度，数据体完整保留历史。



### 阶段 3：`3) Pop Object` —— 弹出增量事件

* **组件关系**：`DeltaFIFO` $\rightarrow$ `Informer`
* **底层机制**：Informer 启动的单协程循环 `processLoop()` 持续阻塞调用 `DeltaFIFO.Pop(processDeltas)`。每次弹出时，该 Key 下的整个 `[]Delta` 切片被一次性拔起，传给处理器。

### 阶段 4：`4) Add Object` —— 分发与持久化流转

* **组件关系**：`Informer` $\rightarrow$ `Indexer`
* **底层机制**：Informer 的 `processDeltas` 接收切片，根据 Delta 的动作类型（Added/Updated/Deleted/Sync）准备更新本地内存二级缓存（Indexer），以供外部业务无开销读取。

### 阶段 5：`5) Store Object & Key` —— 内存倒排索引落盘

* **组件关系**：`Indexer` $\rightarrow$ `ThreadSafeStore`
* **底层机制**：数据被写入受读写锁保护的 `ThreadSafeStore`。
```go
type threadSafeMap struct {
    lock     sync.RWMutex
    items    map[string]interface{} // 全量对象底表: Key -> Object 指针
    indexers Indexers               // 索引计算函数字典: map[索引名]IndexFunc
    indices  Indices                // 倒排索引表: map[索引名]map[索引计算值]sets.String(Keys)
}

```


* **落盘更新**：向主字典 `items["namespace/name"]` 存入对象指针。
* **倒排构建**：遍历所有注册的 `Indexers`（如 `MetaNamespaceIndexFunc`），将该 Key 插入到对应维度的倒排 Set 中（如 `indices["namespace"]["default"]`）。
* **读性能**：后续通过命名空间等条件过滤查询时，直接读内存倒排索引，O(1) 效率，彻底隔离对 API Server 的网络请求。



### 阶段 6：`6) Dispatch Event Handler functions` —— 跨越边界派发事件

* **组件关系**：`Informer` $\rightarrow$ `Resource Event Handlers`
* **底层机制**：这是从 **client-go 基础设施层** 跨越到 **开发者自定义 Controller** 的分水岭。
* 内存索引落盘完毕后，Informer 依据 DeltaType 触发外部注册的事件钩子：
* `Added` $\rightarrow$ 触发 `OnAdd(obj)`
* `Updated` $\rightarrow$ 触发 `OnUpdate(oldObj, newObj)`
* `Deleted` $\rightarrow$ 触发 `OnDelete(obj)`



### 阶段 7：`7) Enqueue Object Key` —— 纯 Key 去重入队

* **组件关系**：`Resource Event Handlers` $\rightarrow$ `WorkQueue`
* **底层机制**：在 `OnAdd` 等回调函数中，**绝不执行耗时业务逻辑**，而是调用 `cache.MetaNamespaceKeyFunc(obj)` 提取出纯字符串 Key（`"namespace/name"`），随后压入 RateLimiting WorkQueue。
* **WorkQueue 核心源码数据结构**：
```go
type Type struct {
    queue      []t             // 待出队处理的 FIFO 顺序切片
    dirty      set             // 正在等待被处理的所有 Key 集合 (去重防膨胀)
    processing set             // 当前正在被 Worker 调谐的所有 Key 集合 (防并发冲突)
    cond       *sync.Cond      // 条件变量：用于 Worker 空闲等待与唤醒
}

```


* **状态机运转与防并发控制**：
1. 外部调用 `queue.Add("pod-a")`：
* 若 `"pod-a"` 已在 `dirty` 集合中，直接丢弃（**去重机制**，避免 1 秒内连续 10 次更新引发 10 次排队）；
* 若 `"pod-a"` 已在 `processing` 集合中（说明当前正有 Worker 在调谐它），**强制不压入 `queue` 切片**！


2. 此时，即使后台有 8 个并发 Worker 协程，其他空闲 Worker 绝不会拿到 `"pod-a"`。**强制实现单资源全局单协程串行调谐，避免并发覆盖与脑裂。**
3. 调用 `cond.Signal()` 唤醒一个休眠中的 Worker。



### 阶段 8：`8) Get Key` & `Process Item` —— Worker 出队消费

* **组件关系**：`WorkQueue` $\rightarrow$ `Process Item` (Worker 协程)
* **底层机制**：
1. Worker 协程调用 `queue.Get()`。
2. 拿到 Key 后，WorkQueue 自动将其从 `dirty` 移入 `processing` 集合。
3. Worker 开启业务函数 `Reconcile(Key)`。
4. 调谐完毕后必须执行 `defer queue.Done(Key)` 将其从 `processing` 清除。如果处理期间又有新事件积压在该 Key 上（留存在 `dirty` 中），`Done()` 会在此时立刻将该 Key 重新推入 `queue` 切片，触发下一轮调谐。



### 阶段 9：`9) Get Object for Key` —— 本地只读零开销调谐

* **组件关系**：`Handle Object` (业务调谐) $\rightarrow$ `Indexer` $\rightarrow$ `ThreadSafeStore`
* **底层机制**：
* Worker 手里只有字符串 Key（`"default/my-app"`）。
* **水平触发与读写分离**：Worker **绝对不向 API Server 发送 GET 请求**，而是拿着该 Key 查询步骤 5 中已落盘的 **Indexer 本地缓存**。
* 零网络开销读取到最新的 Desired 期望规格（`Spec`），观察实际物理环境（`Actual`），计算差值；
* 若有差异，**穿透直写 API Server** 执行变更（创建 Deployment、更新规格），最终写回 `Status` 子资源。



---

## 模块四：Operator 完整生命周期六大阶段拆解

```
1. 注册引导期 ──► 2. 管道构建期 ──► 3. 缓存预热期 ──► 4. 事件派发与入队 ──► 5. 稳态调谐闭环 ──► 6. 销毁与 Finalizer
  (CRD/RBAC)      (Reflector/       (ListAndWatch     (EventHandler ->      (Worker 消费与     (DeletionTimestamp
                   DeltaFIFO)        Sync Indexer)     WorkQueue 状态机)      Reconcile 收敛)    级联清理)

```

### 阶段 1：定义与注册引导期（Registration & Bootstrap）

* **操作与细节**：
1. 运维通过 YAML 向集群提交 CRD 定义。API Server 的 `apiextensions-apiserver` 校验 OpenAPI v3 规则合法性。
2. API Server 在 etcd 中动态分配对应的 key-path：`/registry/<group>/<plural>/...`。
3. 配置 RBAC：授予 Operator 专用的 ServiceAccount 对主资源 CR 及衍生子资源（Deployment、Pod、Event 等）的读写权限。
4. Operator 以 Deployment 形式在工作节点上拉起运行。



### 阶段 2：管道构建期（Pipeline Wiring）

* **操作与细节**：
1. Operator 进程启动运行 `main.go`，实例化 `ctrl.Manager`。
2. 调用 `ctrl.NewControllerManagedBy(mgr).For(&MyRedis{}).Owns(&Deployment{}).Complete(r)`。
3. Manager 的统一 Cache 模块为 `MyRedis` 和 `Deployment` 注册 `SharedIndexInformer`，在底层自动创建 **Reflector**、**DeltaFIFO**、**ThreadSafeStore Indexer**。
4. 实例化内置指数退避和令牌桶限速器的 **RateLimitingQueue**。
5. 注册事件分发逻辑：主资源绑定 `EnqueueRequestForObject`；受管子资源绑定 `EnqueueRequestForOwner`。



### 阶段 3：缓存预热与冷启动对齐（Cold Start & Cache Sync）

* **操作与细节**：
1. `mgr.Start(ctx)` 触发所有注册的 Informer 异步运行 `informer.Run(stopCh)`。
2. Reflector 触发全量 `List()`，将全量数据以 `Sync` 动作灌入 DeltaFIFO。
3. Informer 的 `processLoop` 消费 DeltaFIFO，将全量快照写入本地 `Indexer` 倒排表。
4. **核心门禁：`cache.WaitForCacheSync(ctx)**`：
* 控制器的 Worker 协程被**绝对阻塞**，禁止出队。
* **踩坑考点**：若无该等待，Worker 启动后执行 `r.Get()` 查到的本地缓存全部为 `NotFound`，控制器会误判集群内所有子资源全被删除了，进而发起误删或重复创建风暴。





### 阶段 4：增量事件捕获、派发与 WorkQueue 入队（Watch & Enqueue）

* **操作与细节**：
1. Reflector 携带冷启动获取的最新 `resourceVersion` 建立 HTTP/2 长连接。
2. 运维执行修改（或子 Deployment 发生故障重启），API Server 下发流式变更事件。
3. Reflector 打包 `Delta` 压入 DeltaFIFO，Informer 弹出 Delta 更新本地 Indexer 倒排表。
4. **所有权反查（`EnqueueRequestForOwner`）**：
* 若受管的子 Deployment 发生变动，Handler 会解析子资源的 `metadata.ownerReferences`；
* 匹配找到所属父级 `kind: MyRedis`，提取父级的名字，拼接成 `"default/my-redis"` 放入 WorkQueue。


5. Key 进入 WorkQueue，经过 `dirty` 集合去重与 `processing` 隔离校验，唤醒空闲 Worker。



### 阶段 5：稳态调谐闭环（Reconcile Loop）

* **操作与细节**：
1. Worker 取出 Key，将其移入 `processing` 集合，执行用户代码 `Reconcile(ctx, req)`。
2. **水平触发调谐（Level-Triggered）**：
* 调用 `r.Get()` 读取**本地 Indexer 缓存**获取 Desired 期望状态；
* 读取子资源实际状态（Actual）；
* 水平比对差异：只计算当前状态差，不关心历史经历过多少次丢包或乱序；
* 若子 Deployment 缺失，调用 `r.Create()` 穿透直写 API Server，同时通过 `ctrl.SetControllerReference` 注入 OwnerReference；
* 若规格不符，调用 `r.Update()` 直写更新；
* 调用 `r.Status().Update()` 隔离更新 Status。


3. 调谐成功调用 `queue.Done()`，将 Key 移出 `processing`；若报错抛出 error，调用 `AddRateLimited()` 进行指数退避延迟重试。



### 阶段 6：注销与级联清理期（Deletion & Finalizer Hook）

* **操作与细节**：
1. 运维执行 `kubectl delete myredis my-sample`。
2. API Server 检测到 `metadata.finalizers` 数组非空，**阻止物理删除**，仅打上 `metadata.deletionTimestamp`，进入 `Terminating` 软删除状态。
3. Informer 感知该变动，将 Key 推入队列，Worker 进入 `Reconcile`。
4. 代码中拦截到 `!DeletionTimestamp.IsZero()`，执行业务外部清理逻辑（解绑云盘快照、下线物理节点等）。
5. 业务清理成功，调用 `controllerutil.RemoveFinalizer` 移除终结器并直写更新 API Server。
6. API Server 检测到 Finalizer 数组清空，从 etcd 彻底抹除该 CR。
7. Kubernetes 原生 **Garbage Collector Controller** 介入，沿 `ownerReferences` 引用链将下游的 Deployment $\rightarrow$ ReplicaSet $\rightarrow$ Pod 级联清理，实现无泄漏退出。



---

## 模块五：面试核心考点与攻防策略

### 考点 1：为什么不能用普通的 Go Channel 替代 WorkQueue？

* **普通 Channel 的缺陷**：
1. **无法去重**：同一对象 1 秒内连续 10 次更新会导致 Channel 堆积 10 个任务，引发重复调谐；WorkQueue 的 `dirty` 集合能实现 Key 级去重合并。
2. **并发脑裂**：多个并发 Worker 协程消费同一 Channel 时，同一资源的连续两个事件会被两个协程并行处理，造成状态读写覆盖冲突；WorkQueue 的 `processing` 集合保证**同一资源任意时刻仅能被单一 Worker 调谐**。
3. **缺乏防雪崩机制**：Channel 不具备指数退避能力；WorkQueue 组合了令牌桶与指数退避（`RateLimiter`），可平滑进行失败保护。



### 考点 2：为什么 Reconcile 必须设计为水平触发（Level-Triggered）？

* **边缘触发（Edge-Triggered）**：只对“状态切换瞬间”做出响应（如感知从 1 变 3）。一旦网络丢包错过了该事件，系统永远无法修复差值。
* **水平触发（Level-Triggered）**：只关心**当前这一瞬间系统处于什么状态**。Reconcile 执行时读的是本地缓存的最新全量快照，计算的是 `当前期望值 - 当前实际值` 的水位差。即使在网络抖动中丢失了前 99 次事件，只要第 100 次事件到达，系统仍会精准对齐到终态，具备自愈能力。

### 阶段 3：更新 Status 时频繁出现 `409 Conflict` 怎么解决？

* **成因**：API Server 依靠对象的 `metadata.resourceVersion` 实现乐观并发控制（OCC）。如果多个组件或协程同时更新同一对象，版本号不匹配就会返回 409 Conflict。
* **解法**：
1. 拆分更新流：严禁在更新 Spec 的同时修改 Status，必须使用 `r.Status().Update()` 隔离路径；
2. 无需 Panic：Reconcile 遇到 409 错误时，直接将 error 向上抛出，触发 WorkQueue 的 `AddRateLimited` 退避重试。下一轮调谐会拉取 etcd 最新 RV 自动收敛；
3. 复杂变更可使用基于 Patch 的 Server-Side Apply。



---

## 模块六：生产级 Operator 源码实现与生命周期阶段映射

本段源码完整实现了上述架构与生命周期模型，并通过内联注释精确标明了各代码块对应的阶段与底层技术细节：

```go
package controllers

import (
	"context"
	"fmt"
	"time"

	appsv1 "k8s.io/api/apps/v1"
	corev1 "k8s.io/api/core/v1"
	apierrors "k8s.io/apimachinery/pkg/api/errors"
	metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
	"k8s.io/apimachinery/pkg/runtime"
	ctrl "sigs.k8s.io/controller-runtime"
	"sigs.k8s.io/controller-runtime/pkg/client"
	"sigs.k8s.io/controller-runtime/pkg/controller/controllerutil"
	"sigs.k8s.io/controller-runtime/pkg/log"

	dbv1alpha1 "example.com/api/v1alpha1"
)

const redisFinalizer = "database.example.com/finalizer"

// MyRedisReconciler 调谐器主体
type MyRedisReconciler struct {
	// 【组件机制】底层包装了基于 Indexer 的只读 Cache Client 与直写 API Server 的 Non-cached Client
	client.Client
	Scheme *runtime.Scheme
}

// ==============================================================================
// 阶段 2 & 阶段 3：管道构建与缓存同步预热
// 包含：Reflector 初始化、DeltaFIFO 消费管道绑定、SharedIndexInformer 启动与 WaitForCacheSync
// ==============================================================================
func (r *MyRedisReconciler) SetupWithManager(mgr ctrl.Manager) error {
	return ctrl.NewControllerManagedBy(mgr).
		// 【阶段 2.1: 注册主资源 Informer】
		// 底层通过 handler.EnqueueRequestForObject 将变更事件的主键 Key (namespace/name) 推入 WorkQueue
		For(&dbv1alpha1.MyRedis{}).

		// 【阶段 2.2: 注册受管子资源 Informer】
		// 监听 Deployment 的变更，通过 handler.EnqueueRequestForOwner 反查 ownerReferences，
		// 提取父级 MyRedis 的 Key 压入队列，实现“子变唤醒父”
		Owns(&appsv1.Deployment{}).

		// 【阶段 3: 缓存对齐安全门禁】
		// Complete 组装 Worker 协程，并在底层自动调用 cache.WaitForCacheSync，
		// 必须等待本地 Indexer 倒排表从 DeltaFIFO 同步完成，才放行 Worker 开始出队调谐
		Complete(r)
}

// ==============================================================================
// 阶段 5 & 阶段 6：稳态调谐闭环与注销清理
// 包含：Indexer 读期望、水平触发比对、OwnerReference 依赖绑定、Finalizer 拦截释放
// ==============================================================================
func (r *MyRedisReconciler) Reconcile(ctx context.Context, req ctrl.Request) (ctrl.Result, error) {
	// 进入 Reconcile 前：Worker 协程已从 WorkQueue 的 queue 切片取出了 Key (req.NamespacedName)
	// 该 Key 在 WorkQueue 中已被标入 processing 集合，阻止其他 Worker 并发处理同一资源
	logger := log.FromContext(ctx)

	// --------------------------------------------------------------------------
	// 阶段 5.1：从本地 Indexer Cache 中读取 Desired 期望状态 (水平触发基准)
	// --------------------------------------------------------------------------
	redisCR := &dbv1alpha1.MyRedis{}
	// 【核心机制】：通过 ThreadSafeStore 内存倒排索引查询，O(1) 效率，绝对不打 API Server
	if err := r.Get(ctx, req.NamespacedName, redisCR); err != nil {
		if apierrors.IsNotFound(err) {
			// 资源已从 etcd 彻底抹除，返回 nil 终止后续流程，避免引发无意义重试
			return ctrl.Result{}, nil
		}
		// 读本地缓存异常，返回错误触发 RateLimitingQueue 的指数退避延迟重试 (AddRateLimited)
		return ctrl.Result{}, err
	}

	// --------------------------------------------------------------------------
	// 阶段 6：注销与级联清理期 (Finalizer 钩子拦截与物理删除)
	// --------------------------------------------------------------------------
	// 检查 metadata.deletionTimestamp 是否非空（代表 API Server 处于软删除拦截状态）
	if !redisCR.ObjectMeta.DeletionTimestamp.IsZero() {
		if controllerutil.ContainsFinalizer(redisCR, redisFinalizer) {
			logger.Info("阶段 6: 拦截到物理删除，执行外部依赖清理...")

			// 清理非 K8s 托管资源（如释放外部云盘快照、解绑云厂商负载均衡器等）
			if err := r.cleanUpExternalResources(redisCR); err != nil {
				// 清理失败，返回 error，WorkQueue 会进行退避重试，Finalizer 继续阻断 API Server 物理抹除
				return ctrl.Result{}, err
			}

			// 清理成功，从切片中移除 Finalizer 字符串
			controllerutil.RemoveFinalizer(redisCR, redisFinalizer)
			// 直写 API Server 提交变更
			if err := r.Update(ctx, redisCR); err != nil {
				return ctrl.Result{}, err // 出现并发冲突，自动重试
			}
			logger.Info("阶段 6: Finalizer 移除成功，等待 API Server 及 GC 控制器清理 etcd 记录")
		}
		// 处于删除流程中的资源不再向下执行常规调谐
		return ctrl.Result{}, nil
	}

	// 稳态挂载：确保处于正常运行状态的 CR 打上了 Finalizer 保护钩子
	if !controllerutil.ContainsFinalizer(redisCR, redisFinalizer) {
		controllerutil.AddFinalizer(redisCR, redisFinalizer)
		if err := r.Update(ctx, redisCR); err != nil {
			return ctrl.Result{}, err
		}
	}

	// --------------------------------------------------------------------------
	// 阶段 5.2：观测 Actual 实际状态，计算差值并进行收敛
	// --------------------------------------------------------------------------
	foundDeployment := &appsv1.Deployment{}
	deployName := client.ObjectKey{Namespace: redisCR.Namespace, Name: redisCR.Name + "-backend"}
	// 同样优先通过 Indexer 内存本地缓存检索目标 Deployment 是否存在
	err := r.Get(ctx, deployName, foundDeployment)

	if err != nil && apierrors.IsNotFound(err) {
		// 差值产生：期望有 Deployment，实际底层不存在 -> 执行创建
		newDeploy := r.buildDeploymentForRedis(redisCR)

		// 【核心机制】：在 Deployment 的 ObjectMeta 中注入 OwnerReference
		// 建立父子从属树，用于 EnqueueRequestForOwner 反向通知与 GC 级联回收
		if err := ctrl.SetControllerReference(redisCR, newDeploy, r.Scheme); err != nil {
			return ctrl.Result{}, fmt.Errorf("failed to set controller reference: %w", err)
		}

		logger.Info("阶段 5.2: 调谐差值收敛 - 创建关联 Deployment", "Deployment.Name", newDeploy.Name)
		// 【核心机制】：穿透 Cache，直接向 API Server 发起 POST 创建请求
		if err := r.Create(ctx, newDeploy); err != nil {
			return ctrl.Result{}, err
		}
		// 创建成功，返回 Requeue 重新入队，以便下一轮调谐时观察 Deployment 状态
		return ctrl.Result{Requeue: true}, nil
	} else if err != nil {
		return ctrl.Result{}, err
	}

	// --------------------------------------------------------------------------
	// 阶段 5.3：水平触发更新：计算 Spec 规格差异（如副本数）
	// --------------------------------------------------------------------------
	desiredReplicas := redisCR.Spec.Replicas
	if *foundDeployment.Spec.Replicas != desiredReplicas {
		logger.Info("阶段 5.3: 水平触发收敛 - 更新 Deployment 副本数", "From", *foundDeployment.Spec.Replicas, "To", desiredReplicas)
		foundDeployment.Spec.Replicas = &desiredReplicas
		// 直写更新 API Server
		if err := r.Update(ctx, foundDeployment); err != nil {
			return ctrl.Result{}, err
		}
	}

	// --------------------------------------------------------------------------
	// 阶段 5.4：回写实际运行状态 (Status 隔离更新)
	// --------------------------------------------------------------------------
	if redisCR.Status.ReadyReplicas != foundDeployment.Status.ReadyReplicas {
		redisCR.Status.ReadyReplicas = foundDeployment.Status.ReadyReplicas
		// 【核心机制】：通过专用的 Status() 子资源客户端更新，防止递增 metadata.generation 引发死循环调谐
		if err := r.Status().Update(ctx, redisCR); err != nil {
			return ctrl.Result{}, err
		}
	}

	// 调谐完结：Worker 协程退出后会将该 Key 移出 processing 集合
	// 设定定期同步兜底周期（如 30 分钟）应对不可感知的外部状态漂移
	return ctrl.Result{RequeueAfter: 30 * time.Minute}, nil
}

// 模拟清理外部持久化数据或云资源
func (r *MyRedisReconciler) cleanUpExternalResources(cr *dbv1alpha1.MyRedis) error {
	return nil
}

// 组装业务 Deployment
func (r *MyRedisReconciler) buildDeploymentForRedis(cr *dbv1alpha1.MyRedis) *appsv1.Deployment {
	labels := map[string]string{"app": cr.Name}
	return &appsv1.Deployment{
		ObjectMeta: metav1.ObjectMeta{
			Name:      cr.Name + "-backend",
			Namespace: cr.Namespace,
		},
		Spec: appsv1.DeploymentSpec{
			Replicas: &cr.Spec.Replicas,
			Selector: &metav1.LabelSelector{MatchLabels: labels},
			Template: corev1.PodTemplateSpec{
				ObjectMeta: metav1.ObjectMeta{Labels: labels},
				Spec: corev1.PodSpec{
					Containers: []corev1.Container{{
						Name:  "redis",
						Image: cr.Spec.Image,
					}},
				},
			},
		},
	}
}

```
