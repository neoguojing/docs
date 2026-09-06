# 视觉算法训练推理一体化平台｜面试指南

## 1. 项目一句话定位

> 在已有 VPS + Kestrel 基础设施之上，建立一套视觉算法标准化交付与自动适配能力，将 PyTorch 等训练平台产出的异构视觉模型统一封装为 Kestrel 算法包，再通过 Lua 模板自动生成可在 VPS 上调度执行的算法包，实现算法从训练、发布到视频推理运行的标准化流程。

---

## 2. 面试 2～3 分钟主讲稿

这个项目本质上是一个视觉算法训练到推理的交付平台。

之前训练平台基于 PyTorch 训练出检测、分类、实例分割、语义分割等模型，但不同算法在模型格式、输入输出、前后处理以及推理接口上存在差异，无法直接部署到现有的视频推理系统。

所以训练侧增加 Kestrel 适配，把训练模型转换成统一的 Kestrel 算法包。算法包主要包含模型 bin、依赖的 Kestrel 相关 so，以及算法类型、版本、输入输出等元数据。

算法包推送到推理平台后，平台负责算法注册、版本管理和对象存储。然后根据算法类型选择对应 Lua 模板，自动生成可以在 VPS 上运行的 Lua 算法包。

Lua 主要负责轻量级算法编排，例如“视频帧 → 人体检测 → 特征提取 → 属性识别”。这些节点之间的执行关系由 Lua 组织成 DAG。这样在增加算法或者调整 Pipeline 时，不需要修改 VPS Engine，只需要更新算法包和 Lua 配置。

用户创建任务时，只需要选择摄像头、算法类型和版本。任务下发到 VPS 后，由 VPS 原有的资源调度机制分配 Node，再由 Worker 启动 Lua 算法。Lua 获取视频帧后调用 Kestrel 完成模型推理，最后按照 VPS 标准协议输出结果，图片和结构化结果分别进入对应存储链路。

整个系统可以概括为：

> **训练侧负责模型标准化，推理平台负责算法管理和适配，VPS 负责任务调度和运行，Kestrel 负责模型推理。**

---

## 3. 整体架构

```text
                    ┌──────────────────────┐
                    │      训练平台         │
                    │   PyTorch / 自研框架 │
                    └──────────┬───────────┘
                               │
                        Kestrel 模型适配
                               │
                               ▼
                ┌──────────────────────────┐
                │      Kestrel 算法包       │
                │                          │
                │  model.bin               │
                │  libxxx.so               │
                │  manifest/config         │
                └────────────┬─────────────┘
                             │
                             ▼
                ┌──────────────────────────┐
                │       推理平台             │
                │                          │
                │  Algorithm Registry      │
                │  Object Storage          │
                │  Lua Template            │
                │  Package Generator       │
                │  Task Deployment         │
                └────────────┬─────────────┘
                             │
                       Lua Algorithm Package
                             │
                             ▼
                ┌──────────────────────────┐
                │          VPS             │
                │                          │
                │ Task Management          │
                │ Resource Scheduling      │
                │ Node / Worker             │
                │ Lua Runtime              │
                └────────────┬─────────────┘
                             │
                             ▼
                ┌──────────────────────────┐
                │         Kestrel          │
                │                          │
                │ Model Loading            │
                │ Inference Runtime        │
                │ Device Abstraction       │
                │ Plugin / Model Zoo       │
                └────────────┬─────────────┘
                             │
                             ▼
                       算法推理结果
                      ↙             ↘
                 图片/视频          结构化结果
                 对象存储           Kafka/DB/OSG
```

---

## 4. 核心模块：只需要记住这 6 个

| 模块 | 作用 | 面试重点 |
|---|---|---|
| 1. Kestrel 模型适配 | 将 PyTorch 等训练模型适配成统一 Kestrel 算法 | 模型格式、输入输出、前后处理、推理接口 |
| 2. Algorithm Registry | 管理算法、实例、版本和算法包 | algorithm_id + type + version |
| 3. Algorithm Package / Object Storage | 保存 bin、so 和元数据 | 可发布、可回滚、可追溯 |
| 4. Lua Template | 按算法类型生成标准 Lua | Detection / Classification / Segmentation |
| 5. Lua DAG / Package Generator | 自动生成 VPS 可执行算法包 | 多模型编排、无需修改 VPS Engine |
| 6. Task Deployment | 将用户选择的算法版本转换为 VPS Task | Task → Scheduler → Worker → Lua |

> **核心新增能力可以归纳为：Algorithm Registry + Lua Template + Task Deployment。**

---

## 5. 模型适配：Kestrel 到底解决什么

训练框架很多，但推理平台希望使用统一接口。

```text
PyTorch / 自研训练框架
          ↓
      模型转换
          ↓
   Kestrel 标准适配
          ↓
    model.bin + .so
          ↓
      Kestrel Runtime
```

### 适配关注点

- **模型格式**：训练模型转换成 Kestrel 可加载的格式。
- **输入适配**：尺寸、CHW/NCHW、dtype、归一化、预处理等统一。
- **输出适配**：检测框、类别、分数、Mask、分类结果等转换成统一输出。
- **算法动态库**：通过 Kestrel 相关 `.so` 完成算法侧适配、前后处理或者插件接口对接。

### 面试时的边界

> Kestrel 已经负责模型加载、推理运行、硬件抽象等能力，项目不是重新开发推理引擎，而是在其之上建立标准算法交付能力。

---

## 6. Algorithm Registry：为什么不能只有“算法类型”

同一种算法类型可能存在多个算法实例和多个版本。

```text
Algorithm
├── algorithm_id
├── name
├── type
├── instance
└── versions
    ├── v1.0
    ├── v1.1
    └── v2.0
```

例如：

```yaml
algorithm_id: vehicle_detection
type: detection
version: 2.0.0
package_uri: oss://.../vehicle_detection/2.0.0/package.tar
```

主要解决：

- 算法注册
- 版本管理
- 包地址管理
- 发布/下线
- 回滚
- 任务引用固定版本

---

## 7. Lua：项目里最值得讲的设计点

### Lua 不负责什么

Lua 不负责模型计算，也不是新的推理 Runtime。

### Lua 负责什么

> **Lua 是轻量级算法编排层。**

例如：

```text
视频帧
  ↓
人体检测
  ↓
特征提取
  ↓
属性识别
  ↓
结果融合
  ↓
输出
```

也可以形成分支：

```text
             ┌→ Feature
Detection ───┤
             └→ Attribute
```

Lua 将这些节点组织成 DAG，从而实现轻量 Pipeline 定制。

### 为什么采用 Lua

因为 VPS 已经内置 Lua Runtime，因此可以：

- 不修改 VPS Engine
- 不修改 Worker 核心代码
- 快速增加新的算法 Pipeline
- 将算法编排逻辑从 Engine 中解耦出来

---

## 8. Lua 标准化要求

为了让不同算法都可以统一运行，Lua 接口需要标准化：

### 输入

```text
Frame
Image
Timestamp
Camera ID
Frame ID
```

### 输出

```text
标准化结构化结果
图片/视频结果
```

### 算法包路径

```text
/data/algorithm/<algorithm>/<version>/
```

### 配置

```text
算法类型
模型路径
输入配置
输出配置
Pipeline 配置
运行参数
```

---

## 9. Lua 自动打包流程

```text
Kestrel Algorithm Package
          ↓
     读取 Manifest
          ↓
     判断算法类型
          ↓
  选择 Lua Template
          ↓
  注入模型/配置/路径
          ↓
    生成 Lua Package
          ↓
     发布到算法仓库
```

例如：

```text
vehicle_detection/
├── main.lua
├── config.lua
├── model/
│   ├── model.bin
│   └── libxxx.so
└── manifest.json
```

---

## 10. 用户创建任务后的完整运行流程

```text
用户
 ↓
选择 Camera + Algorithm + Version
 ↓
Task Service
 ↓
查询 Algorithm Registry
 ↓
获取算法包 / Lua 包
 ↓
提交 VPS Task
 ↓
VPS Resource Scheduler
 ↓
分配 Node
 ↓
Worker 启动算法
 ↓
Lua Runtime
 ↓
接收视频帧
 ↓
执行 Lua DAG
 ↓
调用 Kestrel
 ↓
模型推理
 ↓
Lua 结果适配
 ↓
VPS 标准输出
 ↓
┌───────────────┬────────────────┐
│               │                │
▼               ▼                │
图片/视频     结构化结果         │
│               │                │
▼               ▼                │
Object Storage  Kafka/DB/OSG    │
```

---

## 11. 最可能被问的面试问题

### ⭐⭐⭐⭐⭐ 1. 为什么需要这个平台？

**回答：**

不同训练算法存在模型格式、输入输出和推理接口差异，无法直接进入统一视频推理体系。平台通过 Kestrel 标准化 + Lua 自动适配，把训练产物变成 VPS 可执行任务。

---

### ⭐⭐⭐⭐⭐ 2. Kestrel 适配具体做什么？

**回答：**

主要解决模型格式、输入输出、预处理、后处理以及 Kestrel Plugin/Runtime 接口适配，使 PyTorch 训练模型能够以统一方式加载和运行。

---

### ⭐⭐⭐⭐⭐ 3. 为什么要用 Lua？

**回答：**

Lua 主要承担轻量 Pipeline 编排，不负责真正的模型计算。利用 VPS 已有 Lua Runtime，可以在不修改 Engine 的情况下动态定义算法 DAG。

---

### ⭐⭐⭐⭐⭐ 4. Lua 怎么实现 DAG？

**回答：**

把 Detection、Feature、Attribute 等能力抽象为 Node，通过依赖关系组织执行顺序和分支，Lua 负责组装和驱动这些节点。

---

### ⭐⭐⭐⭐ 5. 一个算法包里面有什么？

**回答：**

核心是 `model.bin + Kestrel 相关 so + manifest/config`，Manifest 记录算法类型、版本、输入输出、依赖和运行信息。

---

### ⭐⭐⭐⭐ 6. 如何管理算法版本？

**回答：**

通过 `algorithm_id + version` 唯一定位算法包，Registry 管理元数据和对象存储地址，任务固定引用具体版本，从而实现发布、回滚和追溯。

---

### ⭐⭐⭐⭐ 7. 新增一个算法类型怎么办？

**回答：**

训练侧新增对应 Kestrel Adapter，推理侧增加对应算法类型模板。只要遵守标准包和接口，一般不需要修改 VPS Engine、Scheduler 和 Worker。

---

### ⭐⭐⭐⭐ 8. 为什么不直接让 VPS 调 Kestrel？

**回答：**

VPS 是通用的视频任务运行引擎，不应该耦合具体业务算法。Lua 作为中间的编排层，可以隔离 VPS 和算法逻辑。

---

### ⭐⭐⭐ 9. Lua 性能会不会成为瓶颈？

**回答：**

Lua 主要承担任务编排、参数传递和结果组织，模型计算在 Kestrel/C++/硬件推理侧完成，所以 Lua 本身不是主要计算瓶颈。

---

### ⭐⭐⭐ 10. Python 为什么不合适？

**回答：**

这里并不是做复杂业务逻辑，而是做轻量级实时 Pipeline 编排；同时 VPS 已经提供 Lua Runtime，因此 Lua 更符合现有基础设施和运行模型。

---

## 12. 面试官继续深挖时，优先准备这几个方向

### 方向 A：Kestrel

必须能讲清楚：

```text
PyTorch
 → 模型转换
 → Input Adapter
 → Kestrel Plugin
 → Runtime
 → Output Adapter
```

### 方向 B：算法包

必须能讲清楚：

```text
Algorithm ID
Version
Type
Package
Manifest
Dependency
```

### 方向 C：Lua DAG

必须能举一个真实例子：

```text
Frame
 ↓
Detection
 ↓
Feature
 ↓
Attribute
 ↓
Result
```

### 方向 D：任务运行

必须能完整说出：

```text
Task
 → Scheduler
 → Node
 → Worker
 → Lua
 → Kestrel
 → Result
```

---

## 13. 面试时不要讲错的几个边界

### 不要说

> “我们自己开发了推理引擎。”

### 应该说

> “基于现有 Kestrel 推理基础设施，增加算法标准化和交付能力。”

---

### 不要说

> “Lua 就是运行模型的。”

### 应该说

> “Lua 负责 Pipeline 编排，Kestrel 负责模型推理。”

---

### 不要说

> “为了每个算法修改 VPS。”

### 应该说

> “通过标准 Lua Template 和算法包适配，尽量避免修改 VPS Engine。”

---

## 14. 最后一句话总结项目

> **我做的核心不是重新开发一个推理引擎，而是在已有 VPS + Kestrel 之上建立一套算法标准化、版本管理、Lua 自动适配和任务部署能力，把 PyTorch 训练模型最终变成可以被 VPS 统一调度和运行的视频算法任务。**

---

# 面试前必背

```text
项目定位：算法训练 → 标准化交付 → 视频推理

训练侧：PyTorch → Kestrel Adapter → bin + so

推理平台：Registry → Object Storage → Lua Template → Package

运行侧：Task → Scheduler → Worker → Lua → Kestrel

核心设计：Lua = 轻量 DAG 编排，不修改 VPS Engine

核心价值：异构算法标准化、自动适配、统一部署运行
```
