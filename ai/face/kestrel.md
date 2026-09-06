# 底层软件架构 

- 视频帧编解码
- 视频帧表达
- 模型打包
- 设备抽象
- 输入输出抽象
- 插件系统
## 插件支持
- 目标和人脸检测
- 关键点检测
- 文字检测
- 图像矫正 
- 识别结果检测
- 文字特征抽取
- 图片特征提取
- 人脸属性提取
- 分类
- 人脸角度
- 双目活体
- 三目活体
- 人体特征
- 目标跟踪
- 人脸质量
- 分割
- 人脸模糊度
- 

# 软件架构设计与全景解析
> AI 视觉算法底层运行时和算法中间件。它位于业务 Pipeline 和底层异构硬件/推理 SDK 之间，通过统一的数据、模型、设备和算法插件抽象，屏蔽 NVIDIA GPU、国产 TPU/NPU 等硬件以及不同推理后端的差异。上层只需要通过统一 API 调用检测、分类、特征、分割、跟踪等算法插件，并通过 Pipeline/DAG 将多个算法组合起来。

# Kestrel 架构各模块功能与职责说明

本文档对 Kestrel 架构中涉及的核心模块及其职责进行了详细补充说明，以便于在面试中快速阐述架构设计的分层思想与解耦逻辑。

---

## 1. 顶层编排与统一入口

*   **Pipeline (算法流水线)**
    *   **定位**：业务上层调用方。
    *   **职责**：负责多个算法的业务串联、逻辑编排与调度。Pipeline 关心的是“业务流程怎么走”，而不关心“单个算法在什么硬件上怎么跑”。
*   **Kestrel (核心控制中心)**
    *   **定位**：中间件的全局统一入口。
    *   **职责**：负责统筹底层的三大管理器（Plugin, Model, Device），对外提供统一的 API 初始化接口、算法创建接口和资源释放接口，是连接上层业务与底层推理的桥梁。

---

## 2. 核心管理器 (Managers)
采用注册表 (Registry) 机制管理资源，实现按需加载和集中调度。

*   **PluginManager (插件管理器)**
    *   **职责**：负责算法插件的动态加载、注册、查询与卸载。它使得新增算法可以通过“热插拔”的方式集成，无需修改框架核心代码。
*   **ModelManager (模型管理器)**
    *   **职责**：统一管理网络模型文件及权重数据的生命周期。负责解析模型元数据（Metadata）、控制模型的加载至内存/显存以及卸载，避免模型的重复加载。
*   **DeviceManager (设备管理器)**
    *   **职责**：全局管理底层的异构计算资源（如 CPU、多个 GPU、TPU 等）。负责设备初始化、状态监控及分配，为算法运行提供指定的物理设备抽象。

---

## 3. 算法与业务逻辑层 (Algorithm Level)

*   **Plugin (算法插件接口)**
    *   **定位**：一类算法能力的抽象封装（如：人脸检测插件、OCR插件）。
    *   **职责**：对外暴露该算法支持的版本、基本信息，并作为工厂类负责创建具体的 `Algorithm` 实例。
*   **Algorithm (算法执行实例)**
    *   **定位**：单次/单路算法调用的核心业务控制实体。
    *   **职责**：负责串联整个算法的生命周期，包括数据预处理（Preprocess）、调用底层引擎进行推理（Inference）、以及数据后处理（Postprocess，如 NMS 非极大值抑制、坐标转换等），最终输出结构化结果。

---

## 4. 模型与执行层 (Model & Engine Level)

*   **Model (模型实体)**
    *   **定位**：深度学习网络模型的抽象。
    *   **职责**：维护模型的路径、版本号以及输入输出维度等元数据。它是静态的数据结构代表，不包含计算逻辑。
*   **Engine (执行引擎)**
    *   **定位**：针对特定 `Model` 在特定 `Runtime` 上的可执行实例。
    *   **职责**：负责将静态的 Model 转化为可执行的计算图或 TensorRT Engine。它将 Model、Device 与 InferenceRuntime 绑定在一起，直接接受预处理后的 Tensor 并触发推理计算。

---

## 5. 推理环境与底层适配层 (Runtime & Backend Level)
该层是 Kestrel 跨硬件平台的核心解耦点。

*   **InferenceRuntime (推理运行环境)**
    *   **定位**：推理框架级别的环境上下文（Context）。
    *   **职责**：作为 Engine 和 Backend 的中间层，它可以被多个 Engine 复用。主要负责维护推理所需的全局上下文，调用 Backend 执行实际计算。
*   **Backend (后端抽象接口)**
    *   **定位**：屏蔽底层异构硬件和第三方推理库差异的适配层。
    *   **职责**：将上层统一的推理指令，翻译为具体的 TensorRT、TPU SDK、OpenVINO 或 ONNXRuntime 的 API 调用。
*   **Device (物理设备)**
    *   **定位**：具体计算硬件的抽象描述。
    *   **职责**：提供与硬件直接打交道的基础能力，如显存/内存的 Allocate (分配)、Free (释放) 以及计算流的 Synchronize (同步)。

---

## 6. 数据结构与内存管理

*   **Input / Output (输入/输出封装)**
    *   **职责**：统一封装上层传入的数据（如视频帧 Frame、图像 Image、或者纯 Tensor），并携带相应的元数据（宽、高、通道数等），作为 Pipeline 与 Algorithm 交互的标准数据协议。
*   **Buffer (内存/显存缓冲)**
    *   **职责**：统一处理异构设备间的数据内存管理。封装了 Host (CPU) 到 Device (GPU/TPU) 的数据拷贝、内存生命周期管理，防止内存泄漏，并支持零拷贝（Zero-Copy）优化。

# Kestrel 架构面试速记

## 1. 定位

**Kestrel** 是面向视觉算法的 **AI Inference Runtime + Algorithm Middleware**。

**核心作用**：统一算法、模型、推理 Runtime、Backend 和硬件，为上层 Pipeline 提供标准算法执行能力。
*   **Pipeline** 属于上层，负责算法编排；
*   **Kestrel** 负责单个算法的运行。

---

## 2. 整体架构

```text
Pipeline
   ↓
Kestrel
   ↓
Plugin → Algorithm
            │
            ├── Model
            │
            └── Engine
                  │
                  │ runtime
                  ↓
          InferenceRuntime
                  │
                  │ backend
                  ↓
               Backend
                  ↓
        GPU / TPU / NPU / CPU
```

---

## 3. 核心类 UML

```text
┌─────────────────────────────────────────────┐
│ Kestrel                                     │
├─────────────────────────────────────────────┤
│ 统一入口；管理 Plugin / Model / Device      │
├─────────────────────────────────────────────┤
│ - pluginManager : PluginManager             │
│ - modelManager  : ModelManager              │
│ - deviceManager : DeviceManager             │
├─────────────────────────────────────────────┤
│ + initialize()                              │
│ + createAlgorithm(name) : Algorithm         │
│ + loadModel()                               │
│ + release()                                 │
└──────────────┬──────────────┬───────────────┘
               │              │
               ▼              ▼
┌──────────────────────┐ ┌──────────────────────┐
│ PluginManager        │ │ ModelManager         │
├──────────────────────┤ ├──────────────────────┤
│ 管理算法插件           │ │ 管理模型生命周期      │
├──────────────────────┤ ├──────────────────────┤
│ - plugins             │ │ - models             │
│ - registry            │ │ - registry           │
├──────────────────────┤ ├──────────────────────┤
│ + loadPlugin()       │ │ + loadModel()        │
│ + unloadPlugin()     │ │ + unloadModel()      │
│ + getPlugin()        │ │ + getModel()         │
└──────────┬───────────┘ └──────────┬───────────┘
           │                        │
           ▼                        ▼
┌──────────────────────┐    ┌──────────────────────┐
│ Plugin               │    │ Model                │
│ <<interface>>        │    ├──────────────────────┤
├──────────────────────┤    │ 具体模型及元数据       │
│ 算法能力封装           │    ├──────────────────────┤
├──────────────────────┤    │ - id                 │
│ - name               │    │ - version            │
│ - version            │    │ - path               │
│ - metadata           │    │ - metadata           │
├──────────────────────┤    ├──────────────────────┤
│ + init()             │    │ + load()             │
│ + destroy()          │    │ + unload()           │
│ + createAlgorithm()  │    │ + getMetadata()      │
└──────────┬───────────┘    └──────────┬───────────┘
           │                           │
           └─────────────┬─────────────┘
                         ▼
              ┌──────────────────────────┐
              │ Algorithm                │
              ├──────────────────────────┤
              │ 具体算法执行实例           │
              ├──────────────────────────┤
              │ - model : Model          │
              │ - engine : Engine        │
              │ - device : Device        │
              ├──────────────────────────┤
              │ + init()                 │
              │ + preprocess()           │
              │ + inference()            │
              │ + postprocess()          │
              │ + execute()              │
              │ + release()              │
              └────────────┬─────────────┘
                           │
                           ▼
              ┌──────────────────────────┐
              │ Engine                   │
              ├──────────────────────────┤
              │ 具体模型的执行引擎         │
              ├──────────────────────────┤
              │ - model : Model          │
              │ - runtime :              │
              │   InferenceRuntime       │
              │ - device : Device        │
              ├──────────────────────────┤
              │ + build()                │
              │ + loadEngine()           │
              │ + execute()              │
              │ + unload()               │
              └────────────┬─────────────┘
                           │ runtime
                           ▼
              ┌──────────────────────────┐
              │ InferenceRuntime         │
              ├──────────────────────────┤
              │ 推理框架运行环境           │
              ├──────────────────────────┤
              │ - backend : Backend      │
              ├──────────────────────────┤
              │ + loadEngine()           │
              │ + createContext()        │
              │ + execute()              │
              │ + synchronize()          │
              └────────────┬─────────────┘
                           │ backend
                           ▼
              ┌──────────────────────────┐
              │ Backend                  │
              │ <<interface>>            │
              ├──────────────────────────┤
              │ 屏蔽不同推理后端差异       │
              ├──────────────────────────┤
              │ - type                   │
              ├──────────────────────────┤
              │ + init()                 │
              │ + loadEngine()           │
              │ + execute()              │
              │ + synchronize()          │
              │ + release()              │
              └────────────┬─────────────┘
                           │
                 ┌─────────┼──────────┐
                 ▼         ▼          ▼
              TensorRT   TPU SDK     CPU
                 │         │
                 ▼         ▼
                GPU     TPU / NPU
```

### 辅助管理类与数据结构

```text
┌──────────────────────────┐
│ DeviceManager            │
├──────────────────────────┤
│ 管理计算设备              │
├──────────────────────────┤
│ - devices                │
│ - registry               │
├──────────────────────────┤
│ + initialize()           │
│ + getDevice()            │
│ + release()              │
└────────────┬─────────────┘
             ▼
┌──────────────────────────┐
│ Device                   │
│ <<interface>>            │
├──────────────────────────┤
│ GPU / TPU / NPU / CPU    │
├──────────────────────────┤
│ - id                     │
│ - type                   │
│ - memory                 │
├──────────────────────────┤
│ + allocate()             │
│ + free()                 │
│ + synchronize()          │
└──────────────────────────┘


┌──────────────────────────┐
│ Input                    │
├──────────────────────────┤
│ 算法输入                  │
│ - type                   │
│ - data                   │
│ - metadata               │
├──────────────────────────┤
│ + getData()              │
└────────────┬─────────────┘
             │
       ┌─────┼──────┐
       ▼     ▼      ▼
     Frame Image  Tensor


┌──────────────────────────┐
│ Buffer                   │
├──────────────────────────┤
│ 数据内存管理              │
│ - address                │
│ - size                   │
│ - memoryType             │
├──────────────────────────┤
│ + allocate()             │
│ + free()                 │
│ + copy()                 │
└──────────────────────────┘


┌──────────────────────────┐
│ Output                   │
├──────────────────────────┤
│ 算法输出                  │
│ - type                   │
│ - data                   │
│ - metadata               │
├──────────────────────────┤
│ + getData()              │
└──────────────────────────┘
```

---

## 4. 初始化流程

```text
Pipeline
   │
   │ createAlgorithm("face_detection")
   ▼
Kestrel
   │
   │ PluginManager.getPlugin()
   ▼
FaceDetectionPlugin
   │
   │ createAlgorithm()
   ▼
FaceDetectionAlgorithm
   │
   ├── ModelManager.getModel()
   │        ↓
   │      Model
   │
   ├── DeviceManager.getDevice()
   │        ↓
   │      Device
   │
   └── 创建 Engine
           │
           │ build/loadEngine()
           ▼
    InferenceRuntime
           │
           │ loadEngine()
           │ createContext()
           ▼
        Backend
           │
           ▼
      GPU / TPU / NPU
```

**核心初始化调用链：**
`createAlgorithm()` → `Algorithm.init()` → `Model.load()` → `Engine.build()` → `Engine → InferenceRuntime` → `Runtime → Backend` → `createContext()`

---

## 5. 一次完整算法调用流程

以人脸检测为例：

```text
Pipeline
   │
   │ execute(input)
   ▼
Algorithm
   │
   │ preprocess(input)
   ▼
Input / Image / Tensor
   │
   │ 数据转换、Resize、Normalize
   ▼
Buffer / Tensor
   │
   │
   ▼
Engine
   │
   │ execute(inputTensor)
   ▼
InferenceRuntime
   │
   │ execute(context, tensor)
   ▼
Backend
   │
   │ execute()
   ▼
GPU / TPU / NPU
   │
   │ 模型推理
   ▼
Output Tensor
   │
   │
   ▼
Algorithm.postprocess()
   │
   │ Decode / NMS / Result Transform
   ▼
DetectionResult
   │
   ▼
Pipeline
```

**完整调用链：**
`execute(input)` → `preprocess()` → `Engine.execute()` → `Runtime.execute()` → `Backend.execute()` → `Device / GPU` → `Output Tensor` → `postprocess()` → `DetectionResult`

---

## 6. Engine 与 Runtime

```text
Engine
  ↓
InferenceRuntime
  ↓
Backend
```

**一句话：**
Engine 是具体模型的执行实例，Runtime 是推理框架的运行环境。

**例如：**
```text
TensorRT Runtime
      │
      ├── Face Detection Engine
      ├── Face Recognition Engine
      └── OCR Engine
```
**设计目的：** 这样设计主要为了模型执行与推理框架解耦，并最大化复用 Runtime / 底层计算资源。

---

## 7. 面试最终总结

**Kestrel** 是视觉算法的底层推理 Runtime 和算法中间层。
*   通过 **Plugin** 管理算法。
*   通过 **Model** 管理模型。
*   **Algorithm** 负责整体算法执行逻辑（前后处理等）。
*   **Engine** 负责具体模型执行，引用 **InferenceRuntime**。
*   **Runtime** 再通过 **Backend** 适配 TensorRT、TPU SDK 等不同推理后端。
*   最终运行在 **GPU、TPU、NPU** 等底层硬件上。

**核心设计思想：分层解耦、统一接口和硬件适配。**
