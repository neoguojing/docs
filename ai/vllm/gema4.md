下面整理为可直接保存为 `gemma4_vllm_analysis.md` 的 GitHub Markdown 文档。架构图、类图和流程图均使用 ASCII 文本，不依赖 Mermaid。

# vLLM Gemma4 模型源码分析

## 1. 整体架构

Gemma4 在 vLLM 中并不是简单地将 Hugging Face 模型结构搬过来，而是在保留模型原始计算逻辑的基础上，使用 vLLM 的并行 Linear、Attention、MoE、KV Cache 和执行优化机制完成推理。

整体结构：

```
                         Gemma4ForCausalLM
                                |
                 +--------------+--------------+
                 |                             |
             Gemma4Model                 LogitsProcessor
                 |                             |
        +--------+---------+                   |
        |        |         |                   |
    Embed    Decoder    Final Norm              |
             Layers         |                   |
        |        |           |                   |
        |        |           +-------------------+
        |        |                    |
        |        |                Logits
        |        |
        |   Gemma4DecoderLayer
        |        |
        |   +----+----------------------+
        |   |                           |
        | Attention                  MLP / MoE
        |   |                           |
        |   |                       Gemma4Router
        |   |                           |
        |   |                       FusedMoE
        |   |
        |   +---- PLE / Residual / Norm
        |
        +---- Per-Layer Embedding
```

### 1.1 核心设计

Gemma4 的主要特点：

- GQA：Grouped Query Attention。
- Q/K/V Norm：对 Attention 的 Q、K、V 做归一化。
- Sliding Window Attention：滑动窗口注意力。
- MoE：混合专家模型。
- PLE：Per-Layer Embedding，每层独立注入额外 Embedding 信息。
- YOCO：通过 KV Cache 共享减少部分层的 KV Cache 存储和计算。
- Fast Prefill：优化 Prefill 阶段的计算。
- Tensor Parallel / Pipeline Parallel：支持模型并行。

需要注意：这些是 Gemma4 的模型结构或模型级优化设计，不全部属于 vLLM 通用机制。

---

## 2. 核心类及职责

### 2.1 类关系图

```
Gemma4ForCausalLM
    |
    +-- Gemma4Model
    |      |
    |      +-- VocabParallelEmbedding
    |      |
    |      +-- Gemma4DecoderLayer[]
    |      |       |
    |      |       +-- Gemma4Attention
    |      |       |       |
    |      |       |       +-- QKVParallelLinear
    |      |       |       +-- RMSNorm
    |      |       |       +-- RotaryEmbedding
    |      |       |       +-- Attention
    |      |       |
    |      |       +-- Gemma4MLP
    |      |       |       |
    |      |       |       +-- MergedColumnParallelLinear
    |      |       |       +-- RowParallelLinear
    |      |       |
    |      |       +-- Gemma4MoE
    |      |       |       |
    |      |       |       +-- Gemma4Router
    |      |       |       +-- FusedMoE
    |      |       |
    |      |       +-- PLE
    |      |       +-- Norm
    |      |
    |      +-- Final Norm
    |
    +-- ParallelLMHead
    |
    +-- LogitsProcessor
```

### 2.2 核心类职责

| 类                             | 职责                                          | 重点                                            |
| ----------------------------- | ------------------------------------------- | --------------------------------------------- |
| `Gemma4ForCausalLM`           | 模型入口，负责组织模型和输出 Logits                       | `forward()`、`compute_logits()`                |
| `Gemma4Model`                 | 模型主体，负责 Embedding、Decoder Layers、Final Norm | `forward()`                                   |
| `Gemma4DecoderLayer`          | Transformer Decoder 层                       | Attention、MLP、MoE、PLE、Residual                |
| `Gemma4Attention`             | Attention 计算                                | QKV、Norm、RoPE、KV Cache                        |
| `Gemma4MLP`                   | Dense FFN                                   | Gate/Up Projection、Activation、Down Projection |
| `Gemma4Router`                | MoE 路由计算                                    | Router Logits、Top-K                           |
| `Gemma4MoE`                   | MoE 执行封装                                    | Routing、FusedMoE                              |
| `Gemma4ForCausalLM` 的 LM Head | 输出词表维度 Logits                               | Tensor Parallel                               |

---

## 3. Gemma4ForCausalLM：模型入口

### 3.1 职责

`Gemma4ForCausalLM` 是 vLLM 加载和执行 Gemma4 模型的入口类。

主要负责：

1. 创建 `Gemma4Model`。
2. 创建 LM Head。
3. 组织模型前向计算。
4. 将模型输出转换为 Logits。
5. 处理权重加载和模型并行相关逻辑。

执行流程：

```
Input Token IDs
      |
      v
Gemma4ForCausalLM.forward()
      |
      v
Gemma4Model.forward()
      |
      v
Hidden States
      |
      v
LogitsProcessor
      |
      v
LM Head
      |
      v
Logits
```

### 3.2 关键概念

**Logits 和 Hidden States 的区别：**

- Hidden States：Transformer 最后一层输出的隐藏状态，维度通常为 `[num_tokens, hidden_size]`。
- Logits：词表每个 Token 对应的预测分数，维度通常为 `[num_tokens, vocab_size]`。

LM Head 将隐藏状态投影到词表空间。

---

## 4. Gemma4Model：模型主体

### 4.1 职责

`Gemma4Model` 负责整个 Transformer 主体的组织和执行。

主要包含：

- Token Embedding。
- Decoder Layers。
- Final Norm。
- PLE 相关数据。
- YOCO 相关执行逻辑。
- Pipeline Parallel 中间结果传递。
- Fast Prefill 优化逻辑。

### 4.2 前向流程

```
input_ids
    |
    v
Token Embedding
    |
    +----------------------+
    |                      |
    v                      v
Hidden States         PLE Inputs
    |                      |
    v                      |
Decoder Layer 0 <----------+
    |
    v
Decoder Layer 1
    |
    v
Decoder Layer 2
    |
    v
...
    |
    v
Final Norm
    |
    v
Output Hidden States
```

### 4.3 Pipeline Parallel

当模型被切分到多个 Pipeline Stage 时，每个 Stage 只执行自己负责的 Decoder Layers。

需要跨 Stage 传递的数据通过 `IntermediateTensors` 封装。

```
Pipeline Stage 0
    |
    | Embedding
    | Decoder Layers [0 ... N]
    |
    v
IntermediateTensors
    |
    | hidden_states
    | per_layer_inputs
    |
    v
Pipeline Stage 1
    |
    | Decoder Layers [N+1 ... M]
    |
    v
Final Norm
```

这里的 `IntermediateTensors` 不只是普通的 Hidden States 容器，也可能携带模型继续执行所需的额外状态。

---

## 5. Gemma4DecoderLayer：Decoder 层

### 5.1 结构

每个 Decoder Layer 负责执行 Attention、FFN/MoE、Norm 和残差连接等操作。

```
                  Input Hidden States
                          |
                          v
                    Input Norm
                          |
                          v
                      Attention
                          |
                          v
                    Residual Add
                          |
                          v
                      Post Norm
                          |
                          v
                       MLP/MoE
                          |
                          v
                    Residual Add
                          |
                          v
                    Output Hidden
```

实际代码中还包含 PLE、Layer Scalar 等模型特有逻辑，具体执行顺序以对应版本的 `forward()` 为准。

### 5.2 Dense 与 MoE

Decoder Layer 可以根据配置选择不同的 FFN 结构：

```
                 Gemma4DecoderLayer
                         |
                +--------+--------+
                |                 |
              Dense              MoE
                |                 |
            Gemma4MLP         Gemma4MoE
                |                 |
           Gate / Up          Router
                |                 |
            Activation         Top-K
                |                 |
             Down Proj       Expert MLP
                |                 |
                +--------+--------+
                         |
                      Output
```

### 5.3 Residual

Residual Connection 用于保留输入信息，缓解深层网络训练和信息传递问题。

典型形式：

[\
y = x + F(x)\
]

在源码分析时，需要关注残差连接发生的位置，以及是否存在 Layer Scalar 等额外缩放。

**源码注意点：** 如果 DecoderLayer 的 `forward()` 接收 `residual` 参数，但随后直接使用 `hidden_states` 覆盖或重新初始化它，需要结合实际调用路径确认该参数是否真正参与计算，不能只根据函数签名判断。

---

## 6. Gemma4Attention：注意力机制

### 6.1 结构

```
                  Hidden States
                        |
                        v
                 QKVParallelLinear
                        |
             +----------+----------+
             |          |          |
             v          v          v
             Q          K          V
             |          |          |
             v          v          v
          Q Norm      K Norm     V Norm
             |          |          |
             +----------+----------+
                        |
                        v
                      RoPE
                        |
                        v
                    Attention
                        |
                        v
                   KV Cache
                        |
                        v
                  Output Projection
                        |
                        v
                  Output Hidden
```

### 6.2 GQA

GQA 即 Grouped Query Attention。

其核心思想是多个 Query Head 共享一组 Key/Value Head。

设：

- (H_q)：Query Head 数量。
- (H\_{kv})：Key/Value Head 数量。
- (d_h)：每个 Head 的维度。

GQA 通常满足：

[\
H_q > H\_{kv}\
]

每组 Query Head 共享一个 KV Head。

相比 MHA，GQA 能减少 KV Cache 的存储量和带宽开销。

### 6.3 Q/K/V Norm

Gemma4 在 Attention 中引入 Q/K/V 归一化。

作用：

- 控制 Query、Key、Value 的数值分布。
- 提高注意力计算的稳定性。
- 适配模型自身的训练结构。

这属于模型架构要求，不是 vLLM 为所有模型统一添加的处理。

### 6.4 RoPE

RoPE（Rotary Position Embedding）通过旋转变换将位置信息注入 Query 和 Key。

它主要作用于 Q、K，而不是简单地向 Embedding 添加位置向量。

### 6.5 Sliding Window Attention

滑动窗口注意力限制当前 Token 能访问的历史 Token 范围。

如果窗口大小为 (W)，当前 Token 主要关注其附近的历史上下文：

[\
Attention(i) \rightarrow [i-W+1, i]\
]

它能够减少长序列 Attention 的计算和 KV Cache 访问开销。

### 6.6 vLLM Attention 的特殊之处

`Gemma4Attention` 并不是直接实现全部底层 CUDA Attention 运算，而是将计算委托给 vLLM 的通用 Attention 抽象和后端。

重点关注：

- `QKVParallelLinear`：负责 QKV 投影及 Tensor Parallel。
- `RotaryEmbedding`：位置编码。
- `Attention`：Attention 调用抽象。
- Attention Backend：选择具体计算实现。
- KV Cache：管理历史 Key/Value。
- KV Sharing：支持 YOCO 中的 KV 共享。

---

## 7. Gemma4MLP：Dense FFN

### 7.1 结构

典型的 Gated MLP：

```
             Hidden States
                   |
          +--------+--------+
          |                 |
          v                 v
       Gate Proj          Up Proj
          |                 |
          v                 |
      Activation            |
          |                 |
          +--------+--------+
                   |
                Element-wise
                Multiplication
                   |
                   v
               Down Proj
                   |
                   v
                Output
```

### 7.2 关键 Linear

| Linear    | vLLM 实现                      | 作用           |
| --------- | ---------------------------- | ------------ |
| Gate + Up | `MergedColumnParallelLinear` | 合并两个投影，支持 TP |
| Down      | `RowParallelLinear`          | 输出投影，支持 TP   |

### 7.3 为什么使用 Fused Linear

将 Gate 和 Up 两个投影合并，可以减少：

- Kernel Launch 次数。
- 中间 Tensor 管理开销。
- 部分内存访问开销。

这是 vLLM 对模型执行路径的优化。

---

## 8. Gemma4MoE：混合专家模型

### 8.1 MoE 结构

```
                  Hidden States
                        |
                        v
                   Gemma4Router
                        |
                        v
                    Router Logits
                        |
                        v
                      Softmax
                        |
                        v
                      Top-K
                        |
                        v
                  Expert Selection
                        |
            +-----------+-----------+
            |           |           |
            v           v           v
         Expert 0    Expert 1    Expert N
            |           |           |
            +-----------+-----------+
                        |
                        v
                 Weighted Combine
                        |
                        v
                     Output
```

### 8.2 Gemma4Router

Router 的职责是根据 Token 的 Hidden States 计算各个 Expert 的选择分数。

主要流程：

1. 输入 Hidden States。
2. 执行归一化。
3. 计算 Router Logits。
4. 执行 Top-K 路由。
5. 生成 Expert ID 和对应权重。

代码中还包含模型特有的 Router Scaling 逻辑，例如：

- 不带可训练权重的 RMSNorm。
- Hidden Size 相关缩放。
- Learned Scale。
- Gate Linear。

这些缩放细节应以模型实现为准，不能简单等同于标准 Softmax Router。

### 8.3 Routing Function

Gemma4 提供自定义 Routing Function。

其 Torch 实现的核心逻辑：

```
Router Logits
     |
     v
Full Expert Softmax
     |
     v
Top-K Selection
     |
     v
Indicator Mask
     |
     v
Top-K Renormalization
     |
     v
Per-Expert Scale
     |
     v
Routing Weights + Expert IDs
```

数学上可以概括为：

首先计算所有 Expert 的概率：

[\
p_i = \operatorname{Softmax}(z)\_i\
]

选择 Top-K 专家：

[\
S = \operatorname{TopK}(p, K)\
]

只保留选中的专家：

[\
\tilde{p}\_i =\
\begin{cases}\
p_i, & i \in S \\\
0, & i \notin S\
\end{cases}\
]

对 Top-K 权重重新归一化：

[\
w_i = \frac{\tilde{p}*i}{\sum*{j \in S}p_j}\
]

最后乘以对应的 Expert Scale：

[\
w_i' = w_i \cdot s_i\
]

其中 (s_i) 表示对应 Expert 的缩放系数。

### 8.4 Triton Routing Kernel

代码同时提供 Triton Kernel 和 Torch Fallback。

| 实现     | 主要特点                                      |
| ------ | ----------------------------------------- |
| Torch  | 逻辑清晰，便于验证和调试                              |
| Triton | 融合 Routing 计算，减少中间 Tensor 和 Kernel Launch |

Triton 版本通过排序和 Top-K 选择专家。

需要注意：代码对浮点数 Bits 做了排序键变换，排序键的大小顺序与原始 Logits 的大小顺序相反。因此，转换后使用升序排序并取前 K 项，仍然可以得到原始 Logits 中最大的 K 项。

不能仅根据 `ascending=True` 就判断它取的是最小 Logits。

### 8.5 FusedMoE

`Gemma4MoE` 并不负责逐个 Expert 手工执行全部计算，而是通过 vLLM 的 `FusedMoEFactory` 和 `MoERunner` 等机制组织 Expert 执行。

```
Gemma4MoE
    |
    v
Gemma4Router
    |
    v
Custom Routing Function
    |
    v
FusedMoEFactory
    |
    v
MoERunner
    |
    v
Fused Expert Kernels
    |
    v
Combine Expert Outputs
```

重点理解：

- Router：决定 Token 去哪个 Expert。
- Routing Function：计算 Expert ID 和权重。
- FusedMoE：提供 MoE 执行抽象。
- FusedMoEFactory：根据配置创建相应执行实现。
- MoERunner：负责具体 Expert 执行路径。
- Triton/CUDA Kernel：完成底层计算。

---

## 9. PLE：Per-Layer Embedding

### 9.1 基本概念

PLE 是 Per-Layer Embedding，即每一层都可以获得额外的 Embedding 信息。

它不同于传统 Transformer 仅在输入端进行一次 Token Embedding。

```
                  Token IDs
                      |
                      v
                Token Embedding
                      |
                      +--------------------+
                      |                    |
                      v                    v
                 Main Hidden          PLE Embedding
                      |                    |
                      |                    v
                      |              Layer Projection
                      |                    |
                      |                    v
                      |              Per-Layer Inputs
                      |                    |
                      v                    v
                 Decoder Layer 0 <---------+
                      |
                      v
                 Decoder Layer 1 <---------+
                      |
                      v
                 Decoder Layer 2 <---------+
                      |
                     ...
```

### 9.2 作用

传统 Transformer：

```
Token Embedding
      |
      v
Decoder Layers
```

PLE：

```
Token Embedding
      |
      +----------------------+
      |                      |
      v                      v
Main Hidden             Per-Layer Inputs
      |                      |
      v                      |
Decoder Layer 0 <------------+
      |
      v
Decoder Layer 1 <------------+
      |
      v
Decoder Layer 2 <------------+
```

每层可以融合自己的额外输入信息。

### 9.3 源码关注点

重点关注：

- PLE Embedding 如何构建。
- Per-Layer Inputs 如何生成。
- 每层如何融合 PLE 信息。
- PLE 信息如何在 Pipeline Parallel 中传递。
- PLE 对 Fast Prefill 的影响。

---

## 10. YOCO：KV Cache 共享

### 10.1 基本概念

YOCO 是一种通过跨层 KV 共享减少缓存开销的设计。

在该模型实现中，Decoder Layers 可以分为两部分：

```
              Gemma4 Decoder Layers
                       |
             +---------+---------+
             |                   |
             v                   v
        Self Decoder        Cross Decoder
             |                   |
             v                   |
        Build KV Cache            |
             |                   |
             +-------------------+
                       |
                       v
              Reuse Shared KV Cache
```

前半部分执行 Self Attention 并产生 KV Cache。

后半部分通过 KV Sharing 机制复用前面对应层的 KV Cache，减少重复的 KV 计算和存储。

### 10.2 KV Sharing 的价值

传统结构中，每个 Attention Layer 都需要维护自己的 KV Cache。

如果多个层能够共享 KV Cache，则可以减少：

- KV Cache 显存占用。
- KV Cache 写入量。
- 部分重复的 Key/Value 计算。

但实际收益取决于模型结构、共享层映射以及 Attention 实现。

### 10.3 源码重点

重点查看：

- `kv_sharing_target_layer_name`。
- Attention 层如何声明 KV Sharing 目标。
- vLLM 如何查找目标层。
- KV Cache 如何映射到共享层。
- Attention Backend 如何处理共享 KV。

注意：KV Sharing 不等于简单地让两个 Attention 层使用相同的 Python Tensor。它涉及 vLLM 的 KV Cache 管理和执行元数据。

---

## 11. Fast Prefill

### 11.1 背景

Prefill 阶段需要处理输入 Prompt 中的所有 Token。

当模型存在 YOCO 或特殊 Attention 结构时，不同 Decoder 部分可能存在不同的计算需求。

Fast Prefill 的目的，是减少不必要的计算。

### 11.2 基本流程

```
                 Input Tokens
                      |
                      v
                Self Decoder
                      |
                      v
                Full Token States
                      |
                      v
             Select Required Logits
                      |
                      v
                Cross Decoder
                      |
                      v
               Required Outputs
```

核心思想：

1. Self Decoder 先处理完整输入 Token。
2. 根据实际需求筛选需要继续计算的 Token。
3. Cross Decoder 只处理必要的 Token。
4. 减少不必要的 Prefill 计算。

### 11.3 静态 Buffer 与 CUDA Graph

Fast Prefill 还需要适配 vLLM 的图执行机制。

相关代码涉及：

- 静态 Buffer。
- Batch Descriptor。
- Token 数量与索引管理。
- Piecewise CUDA Graph。

原因是 CUDA Graph 对执行期间的 Tensor 地址、形状和控制流有一定要求。

如果直接使用动态形状和动态内存分配，可能无法充分利用图捕获和重放。

### 11.4 源码注意点

如果代码修改了 `batch_descriptor`，随后再恢复原值，需要特别关注异常路径。

如果恢复操作没有放在 `try/finally` 中，异常发生时可能导致状态没有恢复，从而影响后续执行。

---

## 12. 权重加载机制

### 12.1 整体流程

```
             Hugging Face Checkpoint
                       |
                       v
                Weight Iterator
                       |
                       v
                 Name Mapping
                       |
                       v
             Tensor Split / Remapping
                       |
                       v
                AutoWeightsLoader
                       |
                       v
                 weight_loader
                       |
                       v
                 Model Parameters
```

vLLM 的模型权重加载不是简单地按照参数名称直接赋值。

由于模型中使用了融合 Linear、Tensor Parallel、MoE 等结构，通常需要对权重进行名称映射和 Tensor 转换。

### 12.2 常见权重转换

| 转换类型                 | 目的                                          |
| -------------------- | ------------------------------------------- |
| QKV Packing          | 将 Q、K、V 权重加载到融合参数                           |
| Gate/Up Packing      | 将 Gate 和 Up 权重加载到融合 Linear                  |
| MoE Expert Unpack    | 将 Checkpoint 中的 3D Expert 权重映射到独立 Expert 参数 |
| K/V Weight Copy      | 在 `k_eq_v` 等配置下复制 K 权重到 V                   |
| KV Scale Remap       | 将 Checkpoint 中的 KV Scale 映射到 vLLM 参数        |
| Multimodal Filtering | 过滤当前模型不需要加载的权重                              |

### 12.3 QKV Packing

Hugging Face 模型通常可能分别保存：

```
q_proj.weight
k_proj.weight
v_proj.weight
```

vLLM 可能使用融合参数：

```
qkv_proj.weight
```

因此需要通过对应的 `weight_loader` 将三个权重正确加载到融合参数的不同区域。

### 12.4 Gate/Up Packing

同理：

```
gate_proj.weight
up_proj.weight
```

可以映射到：

```
gate_up_proj.weight
```

这样可以在运行时减少独立算子调用。

### 12.5 MoE 权重映射

Checkpoint 中的 Expert 权重可能采用三维 Tensor：

```
[Num Experts, Output Dim, Input Dim]
```

而 vLLM 的执行结构可能需要按 Expert 拆分和加载。

因此需要将权重映射到对应的 Expert 参数。

### 12.6 AutoWeightsLoader

`AutoWeightsLoader` 负责根据模型参数和权重加载规则完成自动加载。

但模型仍然需要提供特殊的映射逻辑，例如：

- 参数名转换。
- 权重过滤。
- 融合权重加载。
- Expert 权重拆分。
- 特殊参数复制。

---

## 13. Gemma4 特有设计与 vLLM 通用机制

这是阅读模型源码时最需要区分的部分。

### 13.1 Gemma4 模型特有设计

| 设计             | 作用                  |
| -------------- | ------------------- |
| GQA            | 减少 KV Head 数量       |
| Q/K/V Norm     | 归一化 Attention 输入    |
| Sliding Window | 限制注意力窗口             |
| PLE            | 每层注入额外 Embedding    |
| YOCO           | 跨层共享 KV Cache       |
| Fast Prefill   | 减少 Prefill 阶段不必要的计算 |
| Router Scaling | 模型特有的 Router 缩放     |
| Expert Scale   | 对 Expert 权重进行额外缩放   |
| K=V            | 特定配置下共享 K/V 权重      |

### 13.2 vLLM 通用机制

| 机制                           | 作用                      |
| ---------------------------- | ----------------------- |
| `QKVParallelLinear`          | QKV 投影和 Tensor Parallel |
| `MergedColumnParallelLinear` | 融合多个 Column Parallel 投影 |
| `RowParallelLinear`          | Row Parallel 输出投影       |
| `Attention`                  | 通用 Attention 调用抽象       |
| KV Cache Manager             | KV Cache 管理             |
| Attention Backend            | 具体 Attention 执行后端       |
| `FusedMoE`                   | 通用 MoE 执行框架             |
| `MoERunner`                  | MoE 执行路径                |
| Triton Kernel                | 高性能 GPU 算子              |
| Torch Compile                | 模型图编译优化                 |
| CUDA Graph                   | 减少 CPU Launch 开销        |
| Pipeline Parallel            | 模型按层切分到不同 Stage         |
| Weight Loader                | 模型权重加载框架                |

### 13.3 两者的关系

```
                 Gemma4 Model Architecture
                           |
                           v
                Gemma4 Model Implementation
                           |
             +-------------+-------------+
             |                           |
             v                           v
        Model Specific              vLLM Generic
        Logic                        Operators
             |                           |
             |                           |
       PLE / YOCO                  Attention
       Router Scaling              Parallel Linear
       Fast Prefill                Fused MoE
       QKV Norm                    KV Cache
                                   Triton
                                   CUDA Graph
             |                           |
             +-------------+-------------+
                           |
                           v
                    GPU Execution
```

模型负责定义计算逻辑，vLLM 负责提供高性能的执行基础设施。

---

## 14. 如果要适配一个新的 Hugging Face 模型

阅读 Gemma4 源码，可以反向理解新模型接入 vLLM 需要完成哪些工作。

| 步骤 | 主要工作                 | 重点模块                     |
| -- | -------------------- | ------------------------ |
| 1  | 注册模型架构               | Model Registry           |
| 2  | 实现模型主体               | `nn.Module`、Model        |
| 3  | 实现 Decoder Layer     | Attention、MLP            |
| 4  | 适配 Attention         | QKV、RoPE、KV Cache        |
| 5  | 适配 Linear            | TP Linear                |
| 6  | 适配 MoE               | Router、FusedMoE          |
| 7  | 实现权重加载               | `load_weights()`         |
| 8  | 适配量化                 | Quantization             |
| 9  | 适配 Pipeline Parallel | IntermediateTensors      |
| 10 | 适配编译优化               | Torch Compile、CUDA Graph |
| 11 | 验证数值一致性              | HF vs vLLM               |
| 12 | 验证性能                 | Prefill、Decode、吞吐量       |

需要特别注意：并不是所有模型都需要自己实现 Attention Kernel、Linear Kernel 或 MoE Kernel。通常优先复用 vLLM 已有算子，只有现有抽象无法表达模型计算需求时才需要扩展。

---

## 15. 建议重点阅读的源码调用链

### 15.1 模型执行链

```
Gemma4ForCausalLM.forward()
          |
          v
Gemma4Model.forward()
          |
          v
Gemma4DecoderLayer.forward()
          |
          +--------------------+
          |                    |
          v                    v
   Gemma4Attention         Gemma4MLP/MoE
          |                    |
          +--------------------+
                     |
                     v
                  Norm
                     |
                     v
               Hidden States
                     |
                     v
            Gemma4ForCausalLM
                     |
                     v
              compute_logits()
```

### 15.2 Attention / KV Cache

```
Gemma4Attention
      |
      v
QKVParallelLinear
      |
      v
Q/K/V Norm
      |
      v
RoPE
      |
      v
Attention
      |
      +------> Attention Backend
      |
      +------> KV Cache
      |
      +------> KV Sharing
      |
      v
Output Projection
```

### 15.3 MoE 执行链

```
Gemma4Router
      |
      v
Router Logits
      |
      v
Custom Routing Function
      |
      v
Top-K
      |
      v
Triton / Torch
      |
      v
FusedMoEFactory
      |
      v
MoERunner
      |
      v
Expert Kernels
      |
      v
Combine Outputs
```

### 15.4 权重加载链

```
Checkpoint
    |
    v
_weight_iterator
    |
    v
Name Mapping
    |
    v
Tensor Remapping
    |
    v
AutoWeightsLoader
    |
    v
weight_loader
    |
    v
Model Parameters
```

### 15.5 编译与执行优化链

```
Model Implementation
        |
        v
support_torch_compile
        |
        v
Model Graph
        |
        v
Torch Compile
        |
        v
CUDA Graph / Piecewise Graph
        |
        v
Static Buffers
        |
        v
GPU Execution
```

---

## 16. 源码阅读时需要特别留意的问题

建议通过断点重点回答以下问题：

### Attention

1. `QKVParallelLinear` 如何处理 Tensor Parallel？
2. Q/K/V Norm 在什么位置执行？
3. RoPE 如何获取位置和相关参数？
4. Attention 如何选择 Backend？
5. `kv_sharing_target_layer_name` 如何找到共享目标层？
6. KV Cache 是在哪个阶段分配、写入和读取的？

### MoE

1. Router Logits 如何计算？
2. 自定义 Routing Function 与标准 Top-K Routing 有什么区别？
3. Triton Routing Kernel 如何完成 Top-K？
4. `FusedMoEFactory` 如何选择具体 Runner？
5. MoERunner 如何组织 Expert 权重和执行？

### Weight Loading

1. Checkpoint 参数名如何映射到模型参数？
2. QKV Packing 如何加载？
3. Gate/Up Packing 如何加载？
4. MoE 权重如何拆分？
5. Tensor Parallel 权重如何切分？

### Pipeline Parallel

1. Decoder Layers 如何分配到不同 Stage？
2. `IntermediateTensors` 包含哪些内容？
3. PLE 数据如何跨 Stage 传递？
4. 最后一个 Stage 如何执行 Final Norm 和 LM Head？

### Compile / CUDA Graph

1. `support_torch_compile` 如何控制模型编译？
2. `set_model_tag` 如何影响编译缓存或模型标识？
3. 哪些计算会进入 CUDA Graph？
4. Fast Prefill 如何适配 Piecewise CUDA Graph？
5. 静态 Buffer 如何避免动态内存分配？

### 代码细节

- Fast Prefill 修改 `batch_descriptor` 后，需要关注异常路径是否正确恢复状态。
- DecoderLayer 的 `residual` 参数需要结合实际执行路径判断是否参与计算，不能仅凭函数签名推断。
- Triton Routing Kernel 的排序键经过浮点 Bits 转换，升序排序不代表选择原始 Logits 的最小值。

---

## 17. 总结：如何理解这份源码

建议按照以下顺序建立完整认知：

```
第一层：模型结构
    |
    v
Gemma4Model
Gemma4DecoderLayer
Gemma4Attention
Gemma4MLP
Gemma4MoE
    |
    v
第二层：vLLM 算子抽象
    |
    v
Parallel Linear
Attention
FusedMoE
KV Cache
    |
    v
第三层：模型特有优化
    |
    v
PLE
YOCO
KV Sharing
Fast Prefill
    |
    v
第四层：运行时优化
    |
    v
Tensor Parallel
Pipeline Parallel
Torch Compile
CUDA Graph
Triton
    |
    v
第五层：GPU 执行
    |
    v
Kernel Launch
Memory Access
GPU Compute
```

### 推荐学习优先级

| 优先级 | 模块                         | 学习目标                         |
| --- | -------------------------- | ---------------------------- |
| P0  | `Gemma4DecoderLayer`       | 理解完整 Transformer 计算          |
| P0  | `Gemma4Attention`          | 理解 Attention、KV Cache、GQA    |
| P0  | `Gemma4MoE`                | 理解 Router、FusedMoE、Expert 执行 |
| P1  | `load_weights()`           | 理解 HF 权重到 vLLM 参数的映射         |
| P1  | PLE / YOCO                 | 理解模型特有结构                     |
| P1  | Fast Prefill               | 理解 Prefill 优化                |
| P2  | Pipeline Parallel          | 理解跨 Stage 状态传递               |
| P2  | Torch Compile / CUDA Graph | 理解执行优化                       |

**最终需要形成的核心认知：**

Gemma4 源码的价值不只是理解一个模型如何计算，更重要的是理解 vLLM 如何将不同模型的计算结构映射到统一的高性能推理基础设施中。

阅读模型实现时，应始终区分三个层次：

1. **模型层：** 模型本身要求执行什么计算。
2. **框架层：** vLLM 使用哪些通用抽象组织计算。
3. **执行层：** 最终如何通过并行、缓存、编译和 GPU Kernel 完成高性能推理。

掌握这三个层次，才能从“看懂模型代码”进一步走向“理解 vLLM 推理引擎的设计”。

补充文档目录与导航统一类名与职责表述修正执行链中的顺序
