# **vLLM&#x20

先看结论：这份配置最重要的三个数字是：

- `num_blocks = 2454`：KV Cache 管理层可管理的物理 Block 数量。
- `block_stride = 262144`：一个 Attention Layer 中，一个物理 Block 占用的字节数，即 **256 KiB**。
- `layer_stride = 643301376`：一个 Attention Layer 的全部 KV Cache Block 占用的字节数，即 **614 MiB**。

这份配置有 60 个 Attention Layer，每层 2454 个物理 Block。不是 60 层一共只有 2454 个 Block，而是每层都有对应的 2454 个 Block 存储位置。

## **1. 先整理配置结构**

### **1.1 KVCacheConfig 总览**

| 字段                   | 值               | 含义                                |
| -------------------- | --------------- | --------------------------------- |
| `num_blocks`         | 2454            | 物理 Block 数量                       |
| `kv_cache_tensors`   | 6 个             | 分配的 KV Cache Tensor 描述            |
| 每个 Tensor 的 `size`   | 6,433,013,760 B | 约 6 GiB                           |
| 每个 Tensor 的 `layers` | 10 层            | 每个 Tensor 覆盖 10 个 Attention Layer |
| `layer_stride`       | 643,301,376 B   | 每层 KV Cache 占用                    |
| `block_stride`       | 262,144 B       | 每层每个 Block 占用                     |
| `offset`             | 0               | Tensor 内偏移                        |
| `kv_cache_layout`    | `LBNHC`         | KV Cache 的布局标识                    |

六个 Tensor 分别包含不同层：

```
Tensor 0: Layer 0,  6, 12, 18, ... 54
Tensor 1: Layer 1,  7, 13, 19, ... 55
Tensor 2: Layer 2,  8, 14, 20, ... 56
Tensor 3: Layer 3,  9, 15, 21, ... 57
Tensor 4: Layer 4, 10, 16, 22, ... 58
Tensor 5: Layer 5, 11, 17, 23, ... 59
```

合计：

```
6 个 Tensor × 10 层 = 60 个 Attention Layer
```

### **1.2 KV Cache Group 配置**

你给出的 `kv_cache_groups` 中包含两种 Spec：

| 类型                  | Block Size | KV Heads | Head Size | dtype | Sliding Window |
| ------------------- | ---------- | -------- | --------- | ----- | -------------- |
| `SlidingWindowSpec` | 16         | 16       | 256       | FP16  | 1024           |
| `FullAttentionSpec` | 32         | 4        | 512       | FP16  | 无              |

这两种 Attention 的 KV 形状不同，但它们每个 Block 的实际存储大小恰好相同，下面计算。

注意：你贴出的配置文本中有 7 个 Group Spec，最后两个 Group 的 `layer_names` 都是 Layer 5、11、17……59，但 Spec 一个是 Sliding Window，另一个是 Full Attention。这存在层名重复现象，与前面 6 个 Tensor 的层分组也需要进一步核对。下面的 60 层存储计算依据的是 `kv_cache_tensors` 中明确列出的层，不把重复的 Group Spec 当成额外的 10 层。

## **2. 为什么&#x20;**`block_stride = 262144`**？**

可以直接从 Attention Spec 计算。

一个 KV Block 同时保存 K 和 V：

\\[ BlockBytes = 2 \times BlockSize \times NumKVHeads \times HeadSize \times DtypeBytes \\]

这里 FP16 每个元素占 2 Bytes。

**SlidingWindowSpec**

```
Block Size = 16
KV Heads   = 16
Head Size  = 256
dtype      = FP16 (2 Bytes)

BlockBytes
= 2 × 16 × 16 × 256 × 2
= 262144 Bytes
= 256 KiB
```

**FullAttentionSpec**

```
Block Size = 32
KV Heads   = 4
Head Size  = 512
dtype      = FP16 (2 Bytes)

BlockBytes
= 2 × 32 × 4 × 512 × 2
= 262144 Bytes
= 256 KiB
```

关键发现

虽然两种 Attention 的 Block Size 不同，但由于 KV Heads 和 Head Size 不同，最终一个 Block 的存储大小相同，都是 256 KiB。

因此它们都可以对应配置中的：

```
block_stride = 262144
```

## **3. 为什么&#x20;**`layer_stride = 643301376`**？**

每个 Layer 有 2454 个 Block，每个 Block 占 262144 Bytes。

所以：

\\[ LayerStride = num\\\_blocks \times block\\\_stride \\]

代入：

```
2454 × 262144
= 643301376 Bytes
= 613.5 MiB
```

也就是：

```
一个 Attention Layer 的 KV Cache：

+---------+---------+---------+       +---------+
| Block 0 | Block 1 | Block 2 | ...   | Block2453|
+---------+---------+---------+       +---------+
    256KiB    256KiB    256KiB           256KiB

总大小 = 613.5 MiB
```

## **4. 为什么一个 Tensor 的 size 是 6433013760？**

每个 Tensor 包含 10 个 Layer。

```
layer_stride = 643301376 Bytes

Tensor size
= 10 × layer_stride
= 6433013760 Bytes
≈ 5.99 GiB
```

六个 Tensor：

```
Tensor 0: 10 Layers  ≈ 5.99 GiB
Tensor 1: 10 Layers  ≈ 5.99 GiB
Tensor 2: 10 Layers  ≈ 5.99 GiB
Tensor 3: 10 Layers  ≈ 5.99 GiB
Tensor 4: 10 Layers  ≈ 5.99 GiB
Tensor 5: 10 Layers  ≈ 5.99 GiB
--------------------------------
Total:    60 Layers  ≈ 35.95 GiB
```

所以这份配置描述的 KV Cache Tensor 总大小约为 **35.95 GiB**。

## **5. 为什么是 2454 个物理 Block？**

这是最值得理解的地方。

`num_blocks` 表示每个 Layer 的 KV Cache 都有 2454 个 Block 槽位。

它不是通过 Block Size 16 或 32 直接算出来的，而是由 GPU 可用于 KV Cache 的显存预算决定。

基本计算逻辑是：

\\[ num\\\_blocks = \left\lfloor \frac{KVCache可用显存} {每增加一个Block所需的总显存} \right\rfloor \\]

在这份配置中：

一个 Block ID 对应所有 60 个 Layer 中相同编号的 Block 槽位。

因此，增加一个全局 Block ID，需要的显存为：

```
每层 Block 大小 = 256 KiB

60 层 × 256 KiB
= 15 MiB
```

总显存：

```
2454 × 15 MiB
= 35.947 GiB
```

这与上面从六个 Tensor 的 `size` 累加得到的结果一致。

**但需要注意：仅凭这份最终配置，无法反推出为什么显存预算最终取到了 2454。** 还需要看前面的 GPU 显存测算、可用 KV Cache 内存以及 `num_blocks` 计算代码。这里能确认的是 2454 对应的存储容量。

## **6. 逻辑 Block 和物理 Block 如何分布？**

这是把前面的概念与当前配置结合起来的重点。

### **6.1 物理 Block 的分布**

GPU KV Cache 可以直观理解为：

```
                   GPU KV Cache

Layer 0:  [P0][P1][P2][P3] ... [P2453]
Layer 1:  [P0][P1][P2][P3] ... [P2453]
Layer 2:  [P0][P1][P2][P3] ... [P2453]
  ...
Layer 59: [P0][P1][P2][P3] ... [P2453]

           每个 Layer 有 2454 个物理 Block
           每个 Block 在该 Layer 中占 256 KiB
```

这里的 `P7` 表示某个 Layer 的第 7 个物理 Block 槽位。

需要注意：

- Block ID 7 并不是整个模型只保存一份 256 KiB 数据。
- 每个 Attention Layer 都有自己的 KV Cache。
- 同一个物理 Block ID 对应各 Layer KV Cache 中相同编号的存储位置。

因此，跨 60 层看，一个全局 Block 编号对应的存储总量是 15 MiB。

### **6.2 不同 Group 的逻辑 Block**

你的配置中有两种 Block Size：

```
SlidingWindowSpec:
    block_size = 16

FullAttentionSpec:
    block_size = 32
```

假设一个 Request 有 1024 个 Token：

**Sliding Window Group**

```
1024 / 16 = 64 个逻辑 Block

Request A:
L0 L1 L2 L3 ... L63
```

**Full Attention Group**

```
1024 / 32 = 32 个逻辑 Block

Request A:
L0 L1 L2 L3 ... L31
```

所以，同一个 Request 在不同 KV Cache Group 中，逻辑 Block 数量可能不同。

### **6.3 逻辑 Block 到物理 Block 的映射**

假设某个 Group 为 Request A 分配了以下物理 Block ID（仅作示意）：

```
Sliding Window Group (Block Size = 16)

Logical Block       Physical Block
    L0       ------>      P7
    L1       ------>      P3
    L2       ------>      P9
    L3       ------>      P2

Block Table = [7, 3, 9, 2, ...]
```

Full Attention Group 则有自己的逻辑分块和映射：

```
Full Attention Group (Block Size = 32)

Logical Block       Physical Block
    L0       ------>      P5
    L1       ------>      P8
    L2       ------>      P1

Block Table = [5, 8, 1, ...]
```

这里的物理 Block ID 只是示意，实际分配结果由 BlockPool 和 KV Cache 管理器决定。

### **6.4 整体关系图**

**Request A：1024 Tokens**

Sliding Window Group · Block Size 16

64 个逻辑 Block

**L0**

P7

**L1**

P3

**L2**

P9

**L3**

P2

**L4**

P12

**L5**

P4

**L6**

P8

**L7**

P1

…

Full Attention Group · Block Size 32

32 个逻辑 Block

**L0**

P5

**L1**

P8

**L2**

P1

**L3**

P11

**L4**

P6

**L5**

P13

**L6**

P4

**L7**

P10

…

**物理存储空间**

每个 Layer 有 2454 个物理 Block 槽位，Block ID 用于定位对应槽位。

P0

P1

P2

P3

P4

P5

P6

P7

P8

P9

P10

P11

… P2453

图中的两组物理 ID 只是为了说明各自的映射关系，不代表实际运行时的分配结果，也不代表每个 Group 一定独占一套物理 Block。

## **7. 这些数字最终应该怎么记？**

| 数字                 | 值         | 代表什么                     |
| ------------------ | --------- | ------------------------ |
| `num_blocks`       | 2454      | 每层可用的物理 Block 数量         |
| `block_stride`     | 256 KiB   | 单层、单个物理 Block 的存储大小      |
| `layer_stride`     | 613.5 MiB | 单个 Layer 的 KV Cache 大小   |
| Tensor `size`      | 5.99 GiB  | 10 个 Layer 的 KV Cache 大小 |
| Tensor 数量          | 6         | 六组 Tensor 存储             |
| Layer 数量           | 60        | Attention Layer 总数       |
| 总 KV Cache         | 35.95 GiB | 六个 Tensor 的总大小           |
| Sliding Block Size | 16        | Sliding Window 的逻辑分块粒度   |
| Full Block Size    | 32        | Full Attention 的逻辑分块粒度   |

**最后用三句话建立直觉：**

1. `2454` 决定每个 Layer 有多少个物理 Block 槽位，决定可容纳多少 KV Cache。
2. `block_stride` 决定一个物理 Block 在单个 Layer 中占多少显存。
3. `block_size=16/32` 决定 Request 的 Token 序列如何划分逻辑 Block，不直接决定物理 Block 总数。

下一步最值得追踪的是 `num_blocks` 的计算函数：把显存预算与这里的 `15 MiB / global block` 对上，就能完整解释 2454 是如何产生的。

是的。**在你这份配置中，一个 Layer 的一个物理 Block（**`block_stride = 256 KiB`**），恰好可以存储 16 或 32 个 Token 的 K/V 数据。**

但要注意：16 和 32 对应不同的 Attention Spec，不是同一个 Block 同时存储两种数量的 Token。

### **1. 两种 Attention 的计算**

一个 Token 的 K/V 存储大小：

\\[ 2 \times NumKVHeads \times HeadSize \times 2Bytes \\]

| 类型             | KV Heads | Head Size | 单 Token K+V |
| -------------- | -------- | --------- | ----------- |
| Sliding Window | 16       | 256       | 16 KiB      |
| Full Attention | 4        | 512       | 16 KiB      |

所以：

**Sliding Window**

```
16 Tokens × 16 KiB = 256 KiB
```

**Full Attention**

```
32 Tokens × 8 KiB = 256 KiB
```

修正一下上表：Full Attention 的单 Token K+V 是：

```
2 × 4 × 512 × 2 Bytes = 8192 Bytes = 8 KiB
```

### **2. 直观对比**

```
一个物理 Block：256 KiB

Sliding Window:
+------+------+ ... +------+
| T0   | T1   |     | T15  |
+------+------+ ... +------+
      16 Tokens

Full Attention:
+------+------+ ... +------+
| T0   | T1   |     | T31  |
+------+------+ ... +------+
      32 Tokens
```

### **3. 为什么 Block Size 不同？**

因为两种 Attention 的 KV Head 配置不同：

- Sliding Window：16 个 KV Heads，每个 Head 256 维。
- Full Attention：4 个 KV Heads，每个 Head 512 维。

Full Attention 每个 Token 的 K/V 数据量只有 Sliding Window 的一半，因此一个相同大小的物理 Block 可以存放两倍 Token。

**记住：**

`block_stride = 256 KiB` 描述的是物理存储容量；`block_size = 16/32` 描述的是这个物理 Block 对应多少个 Token。两者通过 KV Head 数量、Head Size 和数据类型联系起来。
