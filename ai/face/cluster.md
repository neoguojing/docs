# 人脸档案

## 实时聚类

## 离线聚类
- 分类任务
- 聚类任务
- 标签任务
- 索引训练任务
### 分类任务
- 质量过滤
- 特征检索： 计算和类型的相似度；cstk search
- 选取相似度最高的类，作为类别
- 写磁盘

### 聚类任务
- 聚类特征导入
- gcn 相似度检索，输入所有特征，输出与其他特征的相似度
- - 利用图结构的邻接关系，将图中节点的信息通过卷积操作传播并聚合
  - 输入图结构和每个节点的特征
  - 返回每个节点的高维嵌入标识
- dbscan算法： 输入邻接表，输出：每个点的类标签
- - 借助图，查找联通分量，
#### 人脸人体聚类
- 将人体通过关联id与人脸特征关联
- 以人脸分量为组，组内计算特征相似度
- 未关联的人体，通过特征相似度检索
- 通过摄像头的时间和地点信息，计算可能的关联

# “一人一档”全息档案系统架构与图聚类算法完整设计文档

本方案旨在通过实时特征检索与离线图聚类双引擎架构，实现海量人脸、人体抓拍数据的精确归档，并附带核心聚类算法（GCN、DBSCAN）的推演与底层代码实现。

---

## 一、 系统整体数据流规划

| 任务类型 | 触发时机 | 核心算法与组件 | 目标与作用 |
| :--- | :--- | :--- | :--- |
| **分类任务** | 实时（视频流/抓拍接入） | 质量过滤、cstk / Milvus 向量检索、特征平滑更新 | 毫秒级归档，判定抓拍所属身份或新建档案。 |
| **聚类任务** | 离线（夜间/周期调度） | K-NN 图构建、GCN 特征增强、DBSCAN | 全局纠错，合并同人分档，拆分多档混一，剔除噪声。 |
| **标签任务** | 异步（档案生成后） | 属性识别模型（性别/年龄/衣着） | 提取结构化语义，建立业务搜索索引。 |
| **索引训练** | 周期（特征池扩容后） | Faiss / IVFPQ / HNSW 索引重建 | 维护底层向量库的高召回率与低延迟。 |

---

## 二、 实时流：特征检索与档案更新

实时流解决单张抓拍的即时归属问题。

1. **质量过滤：** 低质量（模糊、过曝）图像不提取人脸特征，仅保留人体特征与元数据。
2. **特征检索：** 输入高维特征向量，使用近似最近邻 (ANN) 召回 Top-K 档案。命中则追加，存疑则入 Pending 队列，未命中则新建档案。
3. **档案特征更新：**
    * **队列维护法 (FIFO)：** 档案维护近 N 次高质量特征列表，检索时计算平均/最大距离。
    * **指数滑动平均 (EMA)：** F_base = a * F_new + (1 - a) * F_base。极省内存，缓慢更新。
    * **质量加权融合：** 根据图像质量分数动态分配权重，计算特征质心。

---

## 三、 离线流：核心算法推演与代码实现

离线流通过挖掘时空上下文与全局拓扑结构，纠正实时流的误判。其核心是由 **GCN（纠偏特征）** 和 **DBSCAN（切割群体）** 组成。

### 3.1 GCN 图卷积网络：特征平滑与纠偏

GCN 的核心作用是让质量差（模糊）的特征吸收时空相连的高质量特征，在向量空间中“向正确的人靠拢”。其前向传播核心逻辑为：
Output = ReLU( D^(-1/2) * A_tilde * D^(-1/2) * X * W )

#### 数值推演演示
假设有 3 张抓拍图（节点 0, 1, 2）。节点 0 为正脸 X_0 = [10, 0]；节点 1 为模糊侧脸 X_1 = [0, 10]；节点 2 为清晰图 X_2 = [10, 10]。拓扑连通关系为 `0 --- 1 --- 2`。

1. **带自环的邻接矩阵 A_tilde (强制保留自身特征):**
   ```text
   A_tilde = 
   [1, 1, 0]
   [1, 1, 1]
   [0, 1, 1]
   ```

2. **度矩阵的负平方根 D^(-1/2):**
   节点 0 度为 2，节点 1 度为 3。1/sqrt(2) ≈ 0.707，1/sqrt(3) ≈ 0.577。

3. **对称归一化 A_hat = D^(-1/2) * A_tilde * D^(-1/2):**
   ```text
   A_hat = 
   [0.500, 0.408, 0    ]
   [0.408, 0.333, 0.408]
   [0    , 0.408, 0.500]
   ```

4. **特征聚合 A_hat * X:**
   ```text
   [0.500, 0.408, 0    ]   [10,  0]   [5.00, 4.08]
   [0.408, 0.333, 0.408] * [ 0, 10] = [8.16, 7.41]
   [0    , 0.408, 0.500]   [10, 10]   [5.00, 9.08]
   ```
   **结论：** 模糊的节点 1 原本特征为 `[0, 10]`，经过一次图聚合后变为 `[8.16, 7.41]`，成功吸收了节点 0 和 2 的第一维特征，向真实身份靠拢。

#### GCN PyTorch 核心实现

```python
import torch
import torch.nn as nn
import torch.nn.functional as F

def normalize_adjacency_matrix(adj: torch.Tensor) -> torch.Tensor:
    """计算对称归一化邻接矩阵"""
    num_nodes = adj.size(0)
    adj_tilde = adj + torch.eye(num_nodes, device=adj.device, dtype=adj.dtype)
    
    deg = torch.sum(adj_tilde, dim=1)
    deg_inv_sqrt = torch.pow(deg, -0.5)
    deg_inv_sqrt[torch.isinf(deg_inv_sqrt)] = 0.0 # 防除零
    
    deg_mat_inv_sqrt = torch.diag(deg_inv_sqrt)
    return torch.mm(torch.mm(deg_mat_inv_sqrt, adj_tilde), deg_mat_inv_sqrt)

class GCNConv(nn.Module):
    """单层图卷积层"""
    def __init__(self, in_features: int, out_features: int):
        super(GCNConv, self).__init__()
        self.weight = nn.Parameter(torch.FloatTensor(in_features, out_features))
        nn.init.xavier_uniform_(self.weight)

    def forward(self, x: torch.Tensor, norm_adj: torch.Tensor) -> torch.Tensor:
        support = torch.mm(x, self.weight) 
        out = torch.mm(norm_adj, support)  
        return out

class GCN(nn.Module):
    """用于聚类特征增强的双层 GCN"""
    def __init__(self, in_dim: int, hidden_dim: int, out_dim: int):
        super(GCN, self).__init__()
        self.gc1 = GCNConv(in_features=in_dim, out_features=hidden_dim)
        self.gc2 = GCNConv(in_features=hidden_dim, out_features=out_dim)

    def forward(self, x: torch.Tensor, norm_adj: torch.Tensor) -> torch.Tensor:
        x = F.relu(self.gc1(x, norm_adj))
        x = self.gc2(x, norm_adj)
        return F.normalize(x, p=2, dim=1) 
```

### 3.2 DBSCAN 基于密度的聚类

将 GCN 增强后的特征输入 DBSCAN，划分为具体的“人”。
* **核心参数：** Eps（相似度阈值），MinPts（成簇最小点数）。
* **点类型：** 核心点（邻域内点数 ≥ MinPts）、边界点（点数不足，但落在核心点邻域内）、噪声点（孤立废片）。

#### DBSCAN 数值演示
设定 Eps = 1.1，MinPts = 3。平面上有以下点：

| 节点 | 坐标 (X, Y) | 邻域内点集合 (距离 ≤ 1.1) | 数量 | 判定类型 |
|---|---|---|---|---|
| P1 | (1, 1) | P1, P2, P3 | 3 | **核心点** |
| P2 | (1, 2) | P2, P1, P4 | 3 | **核心点** |
| P3 | (2, 1) | P3, P1, P4, P5 | 4 | **核心点** |
| P4 | (2, 2) | P4, P2, P3 | 3 | **核心点** |
| P5 | (3, 1) | P5, P3 | 2 | **边界点** |
| P6 | (8, 8) | P6 | 1 | **噪声点 (-1)** |

**成簇过程：** 算法从 P1 开始创建“簇 1”，并向外感染其邻居 P2, P3。P2, P3 作为核心点继续向外感染 P4, P5。到达 P5 时，因其是边界点，感染停止。最终 {P1, P2, P3, P4, P5} 合并为一个身份档案，孤立点 P6 被丢弃。

---

## 四、 跨模态与时空约束（Graph 构建前提）

图的质量决定了 GCN 和 DBSCAN 的上限，构建 K-NN 图时必须叠加物理世界约束。

1. **时空冲突阻断 (Hard Constraint):** 若抓拍 A 和 B 视觉相似度极高，但时间差 1 分钟，空间距离相差 50 公里，则**直接切断图边缘**，禁止聚类。
2. **时空邻接增益 (Soft Constraint):** Edge_Weight = Cosine(F1, F2) * (1 + a * 时空重合概率)。时空轨迹重合度越高，边权重越大。
3. **人体特征降维融合：** 遇到背影无脸数据，通过关联的 BBox 以人脸连通分量为锚点拉入档案；无法关联的人体，执行人体特征子图聚类，并使用 `0.7 * FaceScore + 0.3 * BodyScore` 机制进行多模态判定。
