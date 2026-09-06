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
