# VPS 视频处理系统源码解析

## 一、系统整体架构概述

VPS（Video Processing System）是一个分布式视频结构化处理平台，整体架构分为三层：

| 层级 | 组件 | 职责 |
|------|------|------|
| 资源调度层 | Manager（Master/Slave 主备模式） | 集群资源管理、任务分配、负载均衡 |
| 算法处理层 | Worker（C++ + Golang） | 视频流接入、解码、检测、追踪、选帧、特征提取 |
| 执行引擎层 | Algo-Worker（Go + Lua） | 灵活的插件化业务逻辑执行环境 |

**核心组件关系：**

- **VIS**：产生流地址的源头
- **Manager**：负责任务管理、资源分配。采用主备模式（Master/Slave），只有 Master 处理请求，Slave 仅转发，保证高可用
- **Zookeeper**：负责 Manager 的选举、任务信息的发布与订阅
- **Worker**：实际工作节点，负责取流、解码、跟踪、识别和存储
- **Kafka**：负责存储识别结果
- **OSG（Object Storage Gateway）**：负责存储场景大图和目标小图

---

## 二、资源调度与管理（Manager）

### 2.1 负载定义与状态维护

系统并非简单依赖 CPU/内存指标，而是采用**"容量占比"**模型来衡量负载：

- **基于分辨率的容量**：每个 Worker 节点的资源容量根据 GPU 型号预先配置，按视频流的分辨率划分。例如，一张显卡可能支持同时处理 20 路 1080P 视频，或 50 路 720P 视频。
- **负载计算公式**：

  > 总负载 = (已用 1080P 容量 / 总 1080P 容量) + (已用 720P 容量 / 总 720P 容量) + ...

- **状态同步**：Worker 节点将资源容量、已用资源等信息上报到 Zookeeper，Manager 通过监听 Zookeeper 节点变化维护全局负载视图。`updateNodeStatus` 函数负责在任务分配或释放后更新节点资源使用信息。

### 2.2 四大调度策略

| 策略 | 触发条件 | 核心逻辑 | 目的 |
|------|---------|---------|------|
| **Assign（分配）** | 新任务到来 | 在所有可用 Worker 中找出负载最低的节点分配任务 | 初始负载均衡 |
| **Rebalance（重平衡）** | 最高负载 > 0.8 且 最低 < 0.8 | 将高负载节点的部分任务迁移到低负载节点，持续直到最高负载降至 0.8 以下 | 防止热点，削峰填谷 |
| **ReChange（腾挪）** | 资源碎片化，无法容纳大任务 | 选中负载最低的节点，将其上的现有任务迁移到其他机器，腾出连续资源空间 | 解决碎片化，接纳大任务 |
| **Preemption（抢占）** | 高优先级（算法仓）任务资源不足 | 在集群中寻找运行低优先级任务的 Worker，强制终止这些任务释放资源，分配给高优先级任务 | 保障核心业务 SLA |

---

## 三、算法处理核心（C++ & Golang）

### 3.1 Worker 整体数据处理流水线（Pipeline）

Worker 节点的处理流程是串行的，分为四个主要阶段，数据在各个环节通过 Channel 进行流转：

| 阶段 | 组件/技术 | 功能 |
|------|----------|------|
| 阶段一：流接入 | joy4 开源组件 | 读取 RTSP 或本地流，Demux 解复用出视频帧。任何流读取错误触发指数后退重试（间隔公式：last_interval × (1 + random) × 1.5，最大不超过 3 分钟） |
| 阶段二：解码 | nvcuvid 硬解码 | 解码结果为 NV12 格式，需转为 BGR 格式的 GPU 帧。使用 Kestrel Frame Pool 管理内存 |
| 阶段三：跟踪 | 批处理 + 多目标追踪 | 帧进行拼 Batch 处理（触发条件：Batch 数量 > 当前流数量一半、等待 > 100ms、解码通道 Load Factor > 0.7）。Face SDK 检测→对齐→输出 Bounding Box 和关键点 |
| 阶段四：分析与输出 | 特征提取 + 结果输出 | 对抓拍图进行特征和属性提取，选择质量最好的目标进行 0.85 高分策略融合，结果通过 Kafka 发送，图片存储策略可配置 |

### 3.2 C++ 核心处理链路

#### 3.2.1 目标检测（DetectTargets）

- 依次调用 `face_body_detector`（人脸人体）、`face_detector`（人脸）、`body_detector`（人体）模型
- **兜底逻辑**：如果 `face_body_detector` 未检测到人脸，则调用 `face_detector`；若未检测到人体，则调用 `body_detector`
- **结果过滤**：
  - 人脸：过滤掉置信度 < 0.3 的目标
  - 人体：根据置信度、重叠面积等计算分数，过滤掉分数 < 0.3 的目标

#### 3.2.2 多目标追踪（TrackTargets）

**核心组件：**

| 组件 | 职责 |
|------|------|
| `TargetBase` | 计算相似度（外形 + 特征）和特征融合（0.9 : 0.1） |
| `KalmanFilter` | 预测目标位置 |
| `GraphKM` | 负责二分图匹配 |

**处理步骤：**

1. **关联（Associate）**：将新检测目标与现有轨迹匹配
   - **预测**：使用卡尔曼滤波预测轨迹新位置
   - **分组**：将轨迹按置信度分为高置信度（> 0.5，`set_high`）和低置信度（`set_low`）两组
   - **匹配**：先用高置信度轨迹匹配，相似度阈值默认为 0.4；未匹配的目标再与低置信度轨迹匹配
2. **更新（Update）**：
   - 匹配成功则重新计算置信度；未匹配则衰减（`confidence_decay` 默认 0.85）
   - 置信度低于 0.5 转为低置信度，反之亦然。长时间未匹配的低置信度轨迹会被丢弃
   - 特征融合：匹配成功的轨迹，特征按 0.9（轨迹）: 0.1（目标）融合
3. **生成（Generate）**：为未匹配的目标创建新轨迹
4. **输出（Results）**：输出目标对应的轨迹 ID

**人-体匹配（MatchTargets）**：支持 `matcher` 插件、`cupid` 插件或基于 ROI 匹配。计算人脸与人体目标的匹配分数，构建二分图，使用最大流算法或二分图最大匹配算法找到最佳配对。

#### 3.2.3 选帧策略（SelectTargets）

选帧模块的核心任务是从持续不断的视频流中，为每个被追踪的目标（轨迹），挑选出质量最好、最具代表性的帧，用于后续特征提取和分析。

**轨迹缓存管理：**

- `max_tracklet_num`：每个视频流中最多同时追踪的目标数（默认 25），总量限制防止内存溢出
- `max_tracklet_item_size`：每条轨迹内部最多缓存的目标数（默认 1，即只保留质量最高的那一帧；若设为 N，则保留质量最高的 N 帧，后续特征提取时进行融合）

**选帧详细流程：**

**第一步：质量评估与初筛**

- 计算质量分：调用 `evaluator` 为每个人脸和人体目标计算综合质量分
  - 人脸分：综合模糊度（`face_quality`）、关键点清晰度（`face_landmark`）、头部姿态（`headpose`）、人脸大小
  - 人体分：主要由 `body_quality` 模型给出
- 阈值过滤：直接丢弃质量分低于 `quality_threshold` 的目标
- ROI 过滤：根据 `roi_filter_polygons` 过滤掉区域外的目标

**第二步：轨迹缓存管理**

- **新轨迹创建**：若目标轨迹 ID 是新的，检查当前视频总轨迹数是否达到 `max_tracklet_num` 上限。若已满则丢弃；若未满则创建新轨迹并加入缓存
- **已有轨迹更新**：
  - 缓存未满：直接将新目标加入
  - 缓存已满：将新目标质量分与缓存中质量分最低的目标比较，若更高则替换，若更低则丢弃
  - 场景大图更新：只要新目标质量分超过轨迹缓存中所有目标的最高分，就更新轨迹的 `scene_frame`

**第三步：触发筛选条件（UpdateStatus）**

系统遍历缓存中所有轨迹，满足以下**任一条件**即标记为 `will_be_selected`：

| 触发条件 | 说明 |
|---------|------|
| 超时筛选 | 轨迹存活时间超过 `max_track_time`（默认 25s），状态变为 `STATUS_TIMEOUT`，后续不再参与筛选 |
| 快速响应 | 轨迹首次被筛选，且存活时间达到 `quick_response_time` |
| 周期筛选 | 距离上次筛选时间超过 `time_interval` |
| 高质量触发 | 轨迹中首次出现质量分达到 `enough_ready_quality` 的目标 |
| 跟踪结束 | 追踪器判定该轨迹已经结束（目标丢失） |

**第四步：最终输出与清理**

1. 收集所有 `will_be_selected` 为 true 的轨迹，放入输出列表 `out_tracklets`
2. 对每条轨迹，将其缓存的目标按质量分从高到低排序
3. 状态重置：清空目标缓存数组、重置 `will_be_selected` 标记、更新"上次筛选时间"
4. 清理历史：移除已消失的轨迹，清理 `history_targets_` 中超过 60 秒的过期记录

**已知问题：**

- `FilterTracklets` 阶段存在索引错乱 Bug：先对身体目标编号，过滤后保留 ID，但最终筛选时对所有目标（含人脸）重新编号，导致过滤结果不符合预期
- `max_tracklet_num` 配置本应分别控制人脸和人体，但代码统计时混为一谈
- 距离计算只能计算同一帧中被提取特征的目标距离，由于选帧策略的异步性，多个目标同时被选中的概率较低

#### 3.2.4 特征提取（AnalyzeTracklets）

- 提取人脸/人体特征
- 特征融合：将轨迹中所有目标的特征取平均值并标准化
- 属性提取：对质量分最高的目标提取属性
- 策略：选择质量最好的目标，进行 0.85 高分策略融合，特征重新归一化（Normalize，支持点积运算计算余弦相似性）

### 3.3 Golang 接入与后处理

**数据输入约束（Golang 读取批次帧交给 C++ 处理时）：**

1. 一个视频的帧在一个批次中只能有一个
2. 帧序号必须连续（n → n+1）
3. 除非视频结束，否则必须包含下一帧

**后处理流程：**

1. 生成全局唯一的 `ObjectID`（基于轨迹 ID、时间、区域等）
2. 多重过滤：Margin（边缘过滤）、SelectInterval（帧间隔过滤）、ROI（区域过滤）、DisableSelect（开关控制）、Min/MaxPixel（目标宽度过滤）
3. 结果输出：裁剪目标图、转换坐标、计算目标间距离、写入通道

### 3.4 关键配置参数

| 参数名 | 所属模块 | 默认值 | 说明 |
|--------|---------|--------|------|
| `MaxChannelNum` | Worker | - | 控制 Worker 最大支持的任务数 |
| `Decode.Num` | 解码 | 1 | 解码的并发度 |
| `TrackerNum` | 跟踪 | - | 跟踪并发度 |
| `FrameChanLen` | 解码 | 32 | 接收解码数据的队列缓冲区长度 |
| `UseVPSAnalyser` | 分析 | False | 是否使用 VPS 通用解析 |
| `SelectInterval` | 选帧 | - | 选帧的时间间隔 |
| `AnalyzeConfig.BatchSize` | 分析 | - | 分析阶段的批处理大小 |
| `OutputConcurrency` | 输出 | - | 输出阶段的并发控制 |
| `ObjectIDGen` | Tracker | share | Tracker 阶段是否生成唯一 ID |
| `max_tracklet_num` | 选帧 | 25 | 每个视频每类目标最大轨迹数 |
| `max_tracklet_item_size` | 选帧 | 1 | 每条轨迹缓存的目标数 |
| `quality_threshold` | 选帧 | - | 进入缓存的质量阈值 |
| `quick_response_time` | 选帧 | -1（关闭） | 快速响应时间 |
| `time_interval` | 选帧 | -1 | 选帧周期 |
| `max_track_time` | 选帧 | 25s | 轨迹最大追踪时间 |
| `max_tracking_retention_frame_num` | 追踪 | +5（硬编码） | 低置信度轨迹保留帧数 |
| `max_tracking_distance` | 追踪 | - | 匹配最大距离（像素） |
| `match_threshold` | 追踪 | 0.4 | 二分图匹配相似度阈值 |

---

## 四、Lua 执行引擎（Algo-Worker）

### 4.1 配置体系

| 配置文件 | 职责 |
|---------|------|
| `algo_config.json` | 入口配置，定义 GPU ID（默认 0）、ObjectType（固定 OBJECT_ALGO）、路径配置、并发配置（channels_per_replica） |
| `algo_pipeline.json` | 定义解码参数、输出类型（特征加密、跟踪存储、事件输出等）、Stages 插件配置、选帧缓存控制 |
| `pipeline.json` | 定义具体的处理阶段（Stages）和插件 |
| `render_config.json` | 运行时覆盖配置，定义支持的特征版本和模型。若 ObjectType 为 OBJECT_ALGO，优先使用 renderConfig.Features 替代 config.App 中的相关配置 |

### 4.2 初始化流程

1. **加载配置**：加载 `render_config.json` 和 `algo_pipeline.json`
2. **构建 Worker**：创建 OSG 和 Kafka 连接对象，构建 `SDKPipelineConfig`
3. **构建 Pipeline（SDKPipelineBuilder）**：
   - 加载 `algo_pipeline.json` 和 `pipeline.json`
   - 使用 `algo_pipeline.json` 的 output 配置覆盖 `pipeline.json` 的 output
   - 下载模型，加载插件（`kestrel.LoadPlugin`）
4. **构建执行器（Executors）**：
   - **OutputExecutor**：初始化特征加密、图片编码、存储对象，构建输出 Channel
   - **DecodeExecutor**：构建图片解码器
   - **AppExecutor**：启动 Lua VM，执行 `main.lua`，构建 DAG（有向无环图）执行图，为每个 Stage 创建 `StageExecutor`（内部包含独立的 Lua VM 和 Handler）

### 4.3 运行时数据流

系统采用**生产者-消费者模型**，通过 Channel 进行通信：

**生产者（AppExecutor）：**
1. 从 `frameCh` 中获取解码后的帧（`DecodedFrame`）
2. 调用 `writeFrame`，将帧转换为 Lua 对象（`videoprocess.Packet`）
3. 写入 `DirectedAcyclicGraphAppV1RunnerContext.chan`，该 Channel 实际上是 `StageExecutor.input`

**消费者（StageExecutor & DAG）：**
1. `DirectedAcyclicGraphAppV1Runner` 遍历所有 Stages，启动 `StageExecutor` 协程
2. `StageExecutor` 循环检测输入 Channel，执行 `processPacket`
3. **Lua 交互**：
   - **Marshal**：将 Packet 转换为 Lua Table
   - **执行**：调用 Lua 的 `process` 函数
   - **输出**：根据 PacketType 调用 `writeOutput`（写入结果）或 `writeNextStage`（写入下一级）

**输出（OutputExecutor）：**
1. **对象处理**：从 `AnalyzeResult` Channel 读取数据，进行图片编码、压缩、绘制矩形框（`DrawRect`），存储大图/小图并推送 Kafka
2. **事件处理**：从 `AlertEvent` Channel 读取数据，压缩图片，转换为 `ObjectInfo`，设置时间戳后推送 Kafka
3. **注意**：`EventOutputConcurrency` 为 0 时无法输出事件

### 4.4 核心数据结构

| 数据结构 | 说明 |
|---------|------|
| `AlertEvent` | 事件对象，包含事件基本信息、全景图、小图、任务信息及对象标注 |
| `TriggeredEvent`（Interface） | 定义事件标准行为：获取事件名、Track ID、边界框（Rect）、持续时间等 |
| `Rule`（Interface） | 定义触发规则：ROI 区域、持续时间阈值、运动方向等 |
| `OutputPacket` | Lua 输出的顶层包，包含流 ID、全景图、对象列表 |
| `OutputObject` | 对应具体检测对象，包含用户自定义类型、版本、是否为垃圾数据、特征（Feature）、小图以及 Lua 格式的标注信息（`objectAnnotation`） |
| `AnyPacket` | 通用的 Lua 数据包，包含流 ID 和任意 Lua 数据 |

---

## 五、Worker 端并发模型

Worker 内部通过 Go 语言的 Goroutine 和 Channel 实现高并发处理：

| 执行器 | 职责 | 并发方式 |
|--------|------|---------|
| `DecodeExecutor` | 负责解码 | 支持多协程 |
| `TrackExecutor` | 负责跟踪 | 支持多协程 |
| `AnalyzeExecutor` | 负责分析 | 支持多协程 |
| `OutputExecutor` | 负责输出 | 支持多协程 |

每个 Executor 维护自己的 Input/Output Channel，形成流水线作业。

---

## 六、面试问答速查

### Q1：请简要介绍 VPS 系统的整体架构。

**A1：** VPS 是一个分布式视频结构化处理平台，整体架构分为三层：

1. **资源调度层（Manager）**：负责集群资源管理。采用基于 GPU 显存容量的"容量占比"模型来衡量负载，通过 Assign、Rebalance、ReChange、Preemption 四种策略实现任务的动态调度与资源优化。
2. **算法处理层（Worker）**：负责具体的视频分析任务。采用 C++ 和 Golang 混合编程，C++ 负责高性能的视觉算法（检测、追踪、选帧），Golang 负责数据接入、后处理和与执行引擎的对接。
3. **执行引擎层（Algo-Worker）**：基于 Go + Lua 构建，提供灵活的插件化执行环境。通过 Lua 虚拟机执行具体的业务逻辑，实现算法与业务代码的解耦。

### Q2：Manager 是如何实现负载均衡的？有哪些调度策略？

**A2：** Manager 通过四种核心策略实现智能调度：

- **Assign（分配）**：新任务到来时，选择当前负载最低的节点进行分配。
- **Rebalance（重平衡）**：当集群出现热点（最高负载 > 0.8 且最低 < 0.8）时，将高负载节点的任务迁移到低负载节点，削峰填谷。
- **ReChange（腾挪）**：当资源出现碎片化，无法容纳新的大任务时，选中负载最低的节点，将其上的小任务移走，腾出连续的大块资源。
- **Preemption（抢占）**：当高优先级任务（如算法仓任务）资源不足时，强制终止低优先级任务，保障核心业务的 SLA。

### Q3：C++ 部分的追踪流程是怎样的？如何解决目标匹配问题？

**A3：** 追踪流程是一个串行的处理过程，核心是多目标追踪（Multi-Target Tracking）：

1. **目标检测**：依次调用 face_body、face、body 模型进行检测，并过滤掉低置信度（< 0.3）的目标。
2. **轨迹匹配**：
   - **预测**：使用卡尔曼滤波（Kalman Filter）预测现有轨迹在当前帧的位置
   - **特征融合**：结合目标的外观特征和轨迹的运动信息进行相似度计算
   - **关联**：将轨迹按置信度（阈值 0.5）分为高低两组，优先匹配高置信度轨迹。最终使用二分图最大匹配算法找到全局最优的匹配方案
3. **状态更新**：匹配成功的轨迹会融合新目标的特征（轨迹:目标 = 0.9:0.1），未匹配的轨迹置信度会衰减，过低则被丢弃。

### Q4：选帧模块的核心逻辑是什么？如何保证选出高质量的帧？

**A4：** 选帧模块的目标是为每个追踪轨迹挑选出最具代表性的帧。其核心是一个基于轨迹缓存的质量筛选机制：

1. **质量评估**：对每个检测到的目标，综合计算其质量分（人脸考虑模糊度、姿态、大小；人体由模型打分）。
2. **轨迹缓存**：每条轨迹维护一个缓存（`max_tracklet_item_size`，默认为 1），始终保留质量分最高的那一帧。
3. **触发筛选**：当满足以下任一条件时，轨迹会被标记为"待输出"：
   - **超时**：轨迹存活时间超过 `max_track_time`（默认 25s）
   - **高质量**：轨迹中首次出现质量分达到 `enough_ready_quality` 的目标
   - **周期/结束**：达到筛选周期或目标跟踪结束
4. **输出**：将标记的轨迹中缓存的最佳帧输出，用于后续特征提取。

### Q5：Golang 在算法处理链路中扮演什么角色？

**A5：** Golang 主要扮演"胶水层"和"后处理"的角色：

- **数据接入**：从上游读取视频帧，并按严格的顺序（帧号连续）传递给 C++ 库处理
- **后处理**：接收 C++ 返回的分析结果，进行二次过滤（如 ROI、目标大小过滤），生成全局唯一的 ObjectID
- **数据输出**：负责裁剪目标图片、转换坐标，并将最终结果写入 Kafka 或存储系统

### Q6：Algo-Worker 的执行引擎是如何设计的？为什么使用 Lua？

**A6：** Algo-Worker 采用 Go + Lua 的混合架构，核心是一个基于生产者-消费者模型的执行引擎。

- **设计模式**：
  - **生产者（AppExecutor）**：负责将解码后的视频帧封装成 Packet，推入 Channel
  - **消费者（StageExecutor）**：从 Channel 中取出 Packet，在独立的 Lua 虚拟机中执行 process 函数
- **使用 Lua 的原因**：主要是为了灵活性和解耦。业务逻辑（如事件规则、数据处理流程）可以通过 Lua 脚本动态下发和更新，无需重新编译和部署 Go 程序，实现了算法能力与业务逻辑的分离。

### Q7：请描述 Algo-Worker 的初始化和运行流程。

**A7：**

1. **初始化**：
   - 加载 algo_config.json、algo_pipeline.json 等配置文件
   - 下载模型，加载 C++ 插件
   - 启动 Lua VM，并根据 pipeline.json 构建一个有向无环图（DAG）的执行计划
2. **运行**：
   - AppExecutor 作为生产者，不断从帧队列中读取数据，转换为 Lua 对象后写入 Stage 的输入 Channel
   - StageExecutor 作为消费者，启动协程监听 Channel，拿到数据后调用 Lua 脚本进行处理
   - 处理结果通过 OutputExecutor 进行编码、存储或推送到 Kafka

### Q8：在 Algo-Worker 中，Go 和 Lua 是如何进行数据交互的？

**A8：** 数据交互的核心是 Packet 和 Marshal 过程。

- Go 端定义了一系列数据结构，如 OutputPacket、OutputObject 等，用于封装图像、特征和标注信息
- 当数据需要从 Go 传递给 Lua 时，StageExecutor 会调用 MarshalPacket 函数，将 Go 的结构体转换为 Lua 能够识别的 table 格式
- 反之，Lua 脚本处理完的结果（LTable）也会被 Go 端接收并转换回内部结构体，以便进行后续的存储或网络传输
