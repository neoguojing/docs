# trace
# 卡尔曼滤波 (Kalman Filter) 详解与数值推演

卡尔曼滤波是一种最优状态估计算法。在多目标跟踪（MOT）架构中，它扮演着**“预言家”**与**“纠偏者”**的双重角色。它通过结合目标的“历史运动惯性”和“当前检测结果”，在存在不可避免的检测噪声和画面遮挡时，推算处目标当前最准确的位置。

## 1. 核心状态与数据表达

在跟踪场景下，卡尔曼滤波器为每一个目标维护以下核心数据：

*   **状态向量 (State Vector)：** 描述目标的内部运动状态，通常包含位置坐标 $(x, y)$、边界框大小 $(w, h)$ 以及它们的变化速度（速度分量 $v_x, v_y$）。
*   **协方差矩阵 (Covariance Matrix)：** 描述当前状态的“不确定度”或“误差范围”。协方差越小，系统对当前位置的信心就越足。
*   **测量值 (Measurement)：** 目标检测模块（如 YOLO）在当前帧实际输出的边界框（BBox）。

## 2. 卡尔曼滤波的两步核心循环

算法在每一帧中通过“预测”和“更新”交替运行，形成状态机的闭环：

| 阶段 | 核心动作 | 物理意义 | 核心价值 |
| :--- | :--- | :--- | :--- |
| **预测 (Predict)** | 基于上一帧的最优状态，利用运动学模型（通常假设为匀速运动）向前推演。 | **“我猜它在哪里”** | 即使目标被完全遮挡（没有检测框），系统仍能依靠惯性预测出合理的虚拟位置，防止轨迹断裂。 |
| **更新 (Update)** | 获取当前帧的实际检测框，计算**卡尔曼增益**，将预测值与实际测量值进行加权融合。 | **“结合实际纠正预测”** | 消除检测框在帧与帧之间的轻微抖动，输出最终平滑的最优状态，并更新协方差供下一帧使用。 |

## 3. 卡尔曼增益 (Kalman Gain) 的动态博弈

卡尔曼增益是整个滤波器的灵魂，它本质上是一个**动态权重分配器**：
*   **当检测质量差（如画面模糊导致 BBox 剧烈抖动）：** 测量噪声偏大，卡尔曼增益会自动减小，系统会更信任“预测的平滑轨迹”。
*   **当目标突然急转弯（运动模型出现偏差）：** 预测误差变大，卡尔曼增益会自动增大，系统会更信任“检测器给出的最新实际位置”，迅速将轨迹拉回。

---

## 4. 状态转移矩阵与观测矩阵设计

在多目标跟踪（如经典的 SORT 和 DeepSORT 算法）中，通常假设目标在极短的两帧之间做**匀速直线运动（Constant Velocity Model）**。

### 定义核心状态向量（State Vector）
系统内部维护的 8 维状态向量 $\mathbf{x}$：

```math
\mathbf{x} = [x, y, a, h, v_x, v_y, v_a, v_h]^T
```

*   $x, y$：目标边界框（BBox）的中心坐标。
*   $a$：边界框的宽高比（Aspect Ratio，$w/h$）。
*   $h$：边界框的高度。
*   $v_x, v_y, v_a, v_h$：对应上述四个变量在帧与帧之间的变化速度。

### 状态转移矩阵（State Transition Matrix，F）
预测公式表达为 $\mathbf{x}_{k} = F\mathbf{x}_{k-1}$，矩阵 $F$ 被设计为 $8 \times 8$：

```math
F = \begin{bmatrix}
1 & 0 & 0 & 0 & 1 & 0 & 0 & 0 \\
0 & 1 & 0 & 0 & 0 & 1 & 0 & 0 \\
0 & 0 & 1 & 0 & 0 & 0 & 1 & 0 \\
0 & 0 & 0 & 1 & 0 & 0 & 0 & 1 \\
0 & 0 & 0 & 0 & 1 & 0 & 0 & 0 \\
0 & 0 & 0 & 0 & 0 & 1 & 0 & 0 \\
0 & 0 & 0 & 0 & 0 & 0 & 1 & 0 \\
0 & 0 & 0 & 0 & 0 & 0 & 0 & 1
\end{bmatrix}
```

> **物理机制：** 前 4 行将速度累加到位置和大小上；后 4 行假设系统不受外力影响，速度保持恒定。

### 观测矩阵（Measurement Matrix，H）
检测器的实际观测向量仅为 4 维：$\mathbf{z} = [x, y, a, h]^T$。观测矩阵 $H$ 为一个 $4 \times 8$ 的提取矩阵：

```math
H = \begin{bmatrix}
1 & 0 & 0 & 0 & 0 & 0 & 0 & 0 \\
0 & 1 & 0 & 0 & 0 & 0 & 0 & 0 \\
0 & 0 & 1 & 0 & 0 & 0 & 0 & 0 \\
0 & 0 & 0 & 1 & 0 & 0 & 0 & 0
\end{bmatrix}
```

> **物理机制：** 计算 $H\mathbf{x}_k$ 时，完美提取前 4 个位置与大小参数，舍弃后 4 个不可观测的速度参数。

---

## 5. 完整数值推演演示

设定场景：视频中的行人向右下方匀速行走。推演帧 $k-1$ 到帧 $k$ 的完整过程。

### 初始状态 (帧 k-1)
假设目标 BBox 中心点在 $(100, 200)$，宽度为 $50$，高度为 $100$（宽高比 $a = 0.5$）。
前几帧计算出 X 轴速度为 $+10$ 像素/帧，Y 轴为 $+5$ 像素/帧。

```math
\mathbf{x}_{k-1} = [100, 200, 0.5, 100, 10, 5, 0, 0]^T
```

### 步骤 1：状态预测 (Predict)
系统利用状态转移矩阵 $F$ 预测目标位置：

```math
\mathbf{x}_{k|k-1} = F\mathbf{x}_{k-1} = [110, 205, 0.5, 100, 10, 5, 0, 0]^T
```

> 根据惯性，预测中心点移动到了 $(110, 205)$。

### 步骤 2：获取实际观测值 (Measurement)
YOLO 检测器处理第 $k$ 帧画面，由于相机抖动，检测出 BBox 中心为 $(112, 203)$，高 $102$，宽 $51$（$a = 0.5$）。

```math
\mathbf{z}_k = [112, 203, 0.5, 102]^T
```

### 步骤 3：观测映射与计算残差 (Innovation)
映射预测值（降维提取）：

```math
H\mathbf{x}_{k|k-1} = [110, 205, 0.5, 100]^T
```

计算残差（观测值 - 预测映射值）：

```math
\mathbf{y} = \mathbf{z}_k - H\mathbf{x}_{k|k-1} = [2, -2, 0, 2]^T
```

### 步骤 4：状态更新 (Update)
假设系统计算出位置相关的卡尔曼增益权重为 $0.6$（60% 信任检测，40% 信任预测），速度修正权重为 $0.1$。

```math
\mathbf{x}_{k} = \mathbf{x}_{k|k-1} + K\mathbf{y}
```

**更新位置：**
*   $x_{最终} = 110 + 0.6 \times 2 = 111.2$
*   $y_{最终} = 205 + 0.6 \times (-2) = 203.8$
*   $h_{最终} = 100 + 0.6 \times 2 = 101.2$

**更新速度：**
*   $v_{x,最终} = 10 + 0.1 \times 2 = 10.2$
*   $v_{y,最终} = 5 + 0.1 \times (-2) = 4.8$

**最终输出的帧 $k$ 状态向量：**

```math
\mathbf{x}_{k} = [111.2, 203.8, 0.5, 101.2, 10.2, 4.8, 0, 0]^T
```

**结论：** 最终输出坐标为 $(111.2, 203.8)$，既没有完全听信抖动的 YOLO 结果 $(112, 203)$，也没有完全按照惯性走 $(110, 205)$，而是实现了完美平滑，且速度得到了自适应微调。

# GraphKM (KM 算法) 多目标匹配详解与推演

GraphKM 是基于图论中经典的 Kuhn-Munkres（匈牙利）算法设计的全局最优分配引擎。在多目标跟踪（MOT）架构中，它负责解决“哪条已知轨迹应该分配给画面中哪个新检测框”的问题，确保整个系统的匹配相似度总和最大，避免因局部贪心导致的目标丢失或 ID 频繁切换（ID Switch）。

## 1. 核心概念与定义

*   **二分图 (Bipartite Graph)：** 系统的顶点被严格划分为两个互斥的集合——集合 $U$（活跃的已知轨迹 Track）与集合 $V$（新检测框 Detection）。边只能在 $U$ 和 $V$ 之间建立，集合内部无连接。
*   **权重矩阵 (Weight Matrix)：** 连接 $U$ 与 $V$ 的边上附带的数值，代表两者的“相似度”（由特征余弦距离、BBox距离等综合计算）。
*   **顶标 (Vertex Labeling / 期望值)：** 算法为每个顶点分配的一个动态数值，记为 $L(u)$ 和 $L(v)$。核心约束条件为：对于任意一条连接 $u$ 和 $v$ 的边，其权重必须满足 $L(u) + L(v) \geq Weight(u, v)$。
*   **相等子图 (Equality Subgraph)：** 满足 $L(u) + L(v) = Weight(u, v)$ 的所有边构成的子图。匹配只能在相等子图中进行。
*   **交替路与增广路：** 当发生匹配冲突时，算法会尝试让已匹配的轨迹放弃当前选择，去寻找相等子图中的其他目标，从而形成一条交替更迭的路径。

## 2. 算法执行步骤

1.  **顶标初始化：** 将所有轨迹顶点（$U$）的期望值 $L(u)$ 设为与之相连边的最大权重。将所有检测框顶点（$V$）的期望值 $L(v)$ 设为 0。
2.  **寻找增广路：** 遍历轨迹集，在相等子图中尝试配对。如果目标未被占用，则直接匹配。
3.  **处理冲突：** 如果目标已被占用，要求占用该目标的轨迹去寻找其他备选对象（触发交替路搜索）。
4.  **顶标更新 (计算松弛变量 Slack)：** 如果在当前相等子图中走投无路，说明期望过高。计算已访问轨迹与未访问检测框之间的最小差值 $d = \min(L(u) + L(v) - Weight(u, v))$。将已访问轨迹的顶标减去 $d$，已访问检测框的顶标加上 $d$。
5.  **迭代扩展：** 顶标更新后，会有新的边满足条件加入相等子图。重复上述过程，直到所有顶点都找到匹配。

---

## 3. 完整数值推演演示

设定一个 $2 \times 2$ 的权重矩阵，代表两条已知轨迹（T1, T2）与两个新检测框（D1, D2）的相似度：

| 轨迹 / 检测框 | D1 (目标 1) | D2 (目标 2) |
| :--- | :--- | :--- |
| **T1** | 9 | 8 |
| **T2** | 7 | 1 |

### 步骤 1：顶标初始化
为轨迹顶点分配最大权重作为初始期望值，检测框顶点期望值设为 0。
```math
L(T1) = \max(9, 8) = 9
```
```math
L(T2) = \max(7, 1) = 7
```
```math
L(D1) = 0, \quad L(D2) = 0
```

### 步骤 2：为 T1 寻找匹配
在相等子图中尝试连线：
```math
L(T1) + L(D1) = 9 + 0 = 9 \quad (= Weight)
```
满足条件，且 D1 未被占用，直接配对成功：**T1 ↔ D1**。

### 步骤 3：为 T2 寻找匹配（触发冲突）
检查 T2-D1：
```math
L(T2) + L(D1) = 7 + 0 = 7 \quad (= Weight)
```
满足条件，但 **发生冲突：** D1 已被 T1 占用。
寻找交替路：要求 T1 尝试让出 D1。T1 检查其另一条边 T1-D2：
```math
L(T1) + L(D2) = 9 + 0 = 9 \quad (\neq Weight 8)
```
边 T1-D2 不在相等子图中，T1 无法移动。
此时，已访问顶点为：$\{T1, T2\}$，$\{D1\}$。未访问检测框为：$\{D2\}$。

### 步骤 4：计算松弛变量 (Slack) 与顶标更新
计算已访问轨迹与未访问检测框之间的差值：
*   T1 到 D2：$9 + 0 - 8 = 1$
*   T2 到 D2：$7 + 0 - 1 = 6$
取最小差值为松弛变量 $d = 1$。

**更新期望值：** 访问轨迹减 $d$，已访问检测框加 $d$。
```math
L(T1) = 9 - 1 = 8
```
```math
L(T2) = 7 - 1 = 6
```
```math
L(D1) = 0 + 1 = 1
```
$L(D2)$ 未被访问，保持为 0。

### 步骤 5：扩展子图与重新匹配
利用更新后的期望值，重新评估边：
*   验证 T1-D2：$L(T1) + L(D2) = 8 + 0 = 8$ (= Weight)。成功加入相等子图。
*   验证 T2-D1：$L(T2) + L(D1) = 6 + 1 = 7$ (= Weight)。保留在子图中。

系统再次为 T2 匹配：T2 请求 D1。D1 被 T1 占用。T1 尝试移动，发现 D2 现在处于相等子图中且未被占用。
**交替成功：** T1 释放 D1，改为匹配 D2 (**T1 ↔ D2**)。空出的 D1 分配给 T2 (**T2 ↔ D1**)。

### 最终输出
*   T1 ↔ D2 (权重 8)
*   T2 ↔ D1 (权重 7)
**全局最大总收益：15**。算法完美避开了局部贪心可能导致的“T1强占D1导致T2只能匹配极差的D2（总收益仅10）”的灾难性分配。

# 级联多目标跟踪 (MOT) 系统执行流程与伪代码

级联多目标跟踪（MOT）系统通过目标检测、卡尔曼滤波（状态估计）与匈牙利算法（最大权匹配）的深度结合，解决目标相互遮挡和特征形变导致的光流断裂问题。

## 1. 核心执行流程

*   **检测 (Detection)：** 通过视觉模型提取当前帧画面的所有目标边界框（BBox）及其对应的特征向量（Align Feature）。
*   **预测 (Prediction)：** 利用卡尔曼滤波器，将上一帧留存的所有活跃轨迹基于运动惯性向前推演，计算出目标在当前帧的预测坐标。
*   **分组与匹配 (Association)：** 将已知轨迹按置信度划分为高置信度（`set_high`）与低置信度（`set_low`）两组。优先计算 `set_high` 与新检测框的相似度矩阵，通过 GraphKM 进行二分图最优分配；未匹配的目标进入第二轮，与 `set_low` 进行挽救匹配。
*   **更新与融合 (Update)：** 对匹配成功的轨迹，利用实际检测框修正卡尔曼预测状态，并将新老特征按 0.9 : 0.1 的动量比例进行融合平滑。
*   **生命周期 (Lifecycle)：** 连续未匹配的轨迹置信度按 0.85 衰减，低于阈值则彻底销毁；未被任何现有轨迹认领的新检测框，则被赋予新 ID 生成独立轨迹。

---

## 2. 完整数值推演

设定场景：
*   **帧 $T_1$**：存在轨迹 T1（中心 $x=100$，速度 $v=10$）和 T2（中心 $x=200$，速度 $v=5$）。
*   **帧 $T_2$**：视觉检测器输出目标 D1（$x=112$）和 D2（$x=206$）。

**阶段 A：卡尔曼预测**
*   T1 预测位置：$100 + 10 = 110$
*   T2 预测位置：$200 + 5 = 205$

**阶段 B：构建相似度矩阵**
系统综合距离与特征生成代价矩阵（分值越高越匹配）：

| 轨迹 / 检测 | D1 ($x=112$) | D2 ($x=206$) |
| :--- | :--- | :--- |
| **T1 (预=$110$)** | 0.9 (距离 2) | 0.1 (距离 96) |
| **T2 (预=$205$)** | 0.2 (距离 93) | 0.85 (距离 1) |

**阶段 C：GraphKM 最优分配**
算法锁定全局最大权重和为 1.75（$0.9 + 0.85$）。输出正确配对：**T1 ↔ D1**，**T2 ↔ D2**。

**阶段 D：卡尔曼状态更新**
设卡尔曼增益 $K=0.6$。T1 的最终修正位置为 $110 + 0.6 \times (112 - 110) = 111.2$。完成位置纠偏与特征更新。

---

## 3. 完整级联匹配伪代码

```python
class Track:
    def __init__(self, id, bbox, feature, confidence):
        self.id = id
        self.state = bbox
        self.feature = feature
        self.confidence = confidence
        self.kf = KalmanFilter()
        self.missed_frames = 0
        
    def is_high_confidence(self):
        return self.confidence > 0.5

class CascadeTracker:
    def __init__(self):
        self.tracks = []
        self.next_id = 1
        self.decay_rate = 0.85
        self.match_thresh = 0.4
        self.feat_momentum = 0.9

    def step(self, frame_detections):
        '''
        处理单帧输入，返回当前活跃的目标轨迹
        '''
        # 1. 预测阶段 (Predict)
        for track in self.tracks:
            track.state = track.kf.predict(track.state)
            track.missed_frames += 1

        # 2. 轨迹分组
        set_high = [t for t in self.tracks if t.is_high_confidence()]
        set_low  = [t for t in self.tracks if not t.is_high_confidence()]

        # 3. 第一次级联匹配 (高置信度组)
        cost_matrix_high = TargetBase.compute_similarity(set_high, frame_detections)
        matches_a, uninit_trk_a, uninit_det_a = GraphKM.solve(cost_matrix_high, self.match_thresh)

        # 提取第一次未匹配的检测框
        rem_detections = [frame_detections[i] for i in uninit_det_a]

        # 4. 第二次级联匹配 (低置信度组挽救)
        cost_matrix_low = TargetBase.compute_similarity(set_low, rem_detections)
        matches_b, uninit_trk_b, uninit_det_b = GraphKM.solve(cost_matrix_low, self.match_thresh)

        # 5. 状态更新与特征融合 (Update)
        matched_track_ids = set()

        # 更新第一轮匹配结果
        for trk_idx, det_idx in matches_a:
            self._update_track(set_high[trk_idx], frame_detections[det_idx])
            matched_track_ids.add(set_high[trk_idx].id)

        # 更新第二轮匹配结果
        for trk_idx, rem_det_idx in matches_b:
            self._update_track(set_low[trk_idx], rem_detections[rem_det_idx])
            matched_track_ids.add(set_low[trk_idx].id)

        # 6. 未匹配轨迹衰减与清理 (Decay & Delete)
        surviving_tracks = []
        for track in self.tracks:
            if track.id not in matched_track_ids:
                track.confidence *= self.decay_rate  # 置信度衰减
                if track.confidence >= 0.1:          # 未跌破死亡阈值则保留
                    surviving_tracks.append(track)
            else:
                surviving_tracks.append(track)
        
        self.tracks = surviving_tracks

        # 7. 生成新轨迹 (Generate)
        for rem_det_idx in uninit_det_b:
            det = rem_detections[rem_det_idx]
            new_track = Track(self.next_id, det.bbox, det.feature, det.confidence)
            self.tracks.append(new_track)
            self.next_id += 1

        # 返回当前帧仍符合输出条件的轨迹
        return [t for t in self.tracks if t.is_high_confidence()]

    def _update_track(self, track, det):
        '''内部辅助函数：更新匹配成功的轨迹状态'''
        # 卡尔曼滤波修正位置
        track.state = track.kf.update(track.state, det.bbox)
        # 特征 EMA 平滑融合
        track.feature = self.feat_momentum * track.feature + (1 - self.feat_momentum) * det.feature
        # 置信度重置/提升
        track.confidence = max(track.confidence, det.confidence)
        track.missed_frames = 0
```


## 概念
- 卡尔曼滤波器（Kalman Filter, KF）**是一种用于线性动态系统状态估计的递归算法，特别适合噪声和不确定性环境下的状态预测和估计
### 追踪
- 目标轨迹：一般需要包含，位置信息、速度信息、时间和置信度等信息
- 输入:
- - 多帧连续的视频帧。
- - 每帧检测出的目标的边界框（bounding box）、特征（align feature）。
- 输出:
- 每个目标在多帧之间的轨迹。一般有个跟踪id
- 核心组件:
- - 目标检测模块：检测目标并输出其边界框和特征。
- - 轨迹预测模块：基于卡尔曼滤波器预测目标位置。
- - 相似度计算模块：通过特征相似性、边界框距离、大小变化计算匹配分数。
- - 目标分配模块：采用匈牙利算法（KM算法）完成目标与轨迹的关联。
- - 轨迹管理模块：处理轨迹的创建、更新、删除。
- 卡夫慢滤波器：
- - 平滑目标的运动轨迹并降低噪声影响
- - 对目标在当前帧的位置进行预测，以便在下一帧中快速和检测结果进行匹配
  - 在目标遮挡等情况下可以正确预测合理位置
  - 输入： 状态向量：坐标、速度、大小，测量值，协方差矩阵
  - 输出： 预测状态，修正状态和误差协方差矩阵
- 匈牙利算法（最大二分图）：
- - 解决轨迹和目标的匹配问题
  - 输入： 轨迹的集合，目标集合，计算一个代价矩阵
  - 输出： 最优匹配结果，矩阵中，是的代价最小
### 选帧
- 选帧的目的：基本假设：目标出现5s，视频帧每秒25帧，图片相似度高，浪费算力；一般选帧5帧
- 如何选帧？ 角度、模糊度
- 问题1：不能按质量分排序，因为相邻的图片相似度高
- 问题2: 抽样不同位置的帧
- 选帧时机：目标离开之后，从全局选帧；设置返回时间，超时则立即返回
- 缓存：缓存目标信息，目标小图和场景大图；可以设计GPU缓存池，降低图像分辨率
- 选帧逻辑：动态选帧（每一帧都更新缓存和选帧），强制选帧：跟踪结束或者超时
- 选帧策略：排序选帧topk；分段选帧，基于分段再选帧
- 必要参数：最大跟踪目标数、各个跟踪目标的数量、进入画面到选帧的时间（用于快速选帧）、分段选帧的触发时间
- 架构：每个流绑定一个选帧器，每个选帧器需要缓存不同的目标的抓拍
