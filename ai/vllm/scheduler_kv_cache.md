下面整理成适合保存到 GitHub Markdown 的参数说明文档。按照 **整体配置 → Group 配置 → KV Cache Spec → Tensor 内存 → 计算公式** 分类，重点保留参数含义、实际数值和计算方式。

# **vLLM Scheduler KV Cache Config 参数说明**

## **1. KVCacheConfig：整体配置**

| 参数                                | 当前值     | 说明                               |
| --------------------------------- | ------- | -------------------------------- |
| `num_blocks`                      | 2454    | 每个 KV Cache Group 对应的逻辑 Block 数量 |
| `kv_cache_tensors`                | 6 个     | 实际 KV Cache Tensor 布局描述          |
| `kv_cache_groups`                 | 6 个     | KV Cache 管理分组                    |
| `prefix_cache_retention_interval` | 0       | Prefix Cache 保留策略相关的时间间隔配置       |
| `kv_cache_layout`                 | `LBNHC` | KV Cache 内存布局标识                  |
| `hisparse_host_num_blocks`        | None    | HiSparse Host 侧 Block 配置         |
| `hisparse_host_block_stride`      | None    | HiSparse Host Block 内存步长         |
| `hisparse_shared_host_pool`       | False   | 是否启用共享 Host KV Cache Pool        |

**核心理解：**

- `num_blocks`：决定每个 Group 可以管理多少个逻辑 Block。
- `kv_cache_tensors`：描述 KV Cache 实际内存区域。
- `kv_cache_groups`：描述哪些 Layer 使用相同的 KV Cache 规格。
- Tensor 和 Group 是通过 Layer 名称关联的。

## **2. KVCacheGroupSpec：分组配置**

当前模型共 60 层 Attention，分成 6 个 Group，每组 10 层。

| 参数                   | 当前值                                   | 说明                           |
| -------------------- | ------------------------------------- | ---------------------------- |
| `layer_names`        | 10 个 Layer                            | 当前 Group 管理的 Attention Layer |
| `kv_cache_spec`      | SlidingWindowSpec / FullAttentionSpec | KV Cache 规格                  |
| `is_eagle_group`     | False                                 | 是否属于 EAGLE 推测解码专用 Group      |
| `host_resident`      | False                                 | KV Cache 是否驻留在 Host          |
| `enable_kv_transfer` | True                                  | 是否允许 KV Cache Transfer       |
| `role`               | DEFAULT                               | KV Cache Group 的角色           |

### **2.1 Group 分布**

| Group | Layer 编号                              | 类型            |
| ----- | ------------------------------------- | ------------- |
| 0     | 0, 6, 12, 18, 24, 30, 36, 42, 48, 54  | SlidingWindow |
| 1     | 1, 7, 13, 19, 25, 31, 37, 43, 49, 55  | SlidingWindow |
| 2     | 2, 8, 14, 20, 26, 32, 38, 44, 50, 56  | SlidingWindow |
| 3     | 3, 9, 15, 21, 27, 33, 39, 45, 51, 57  | SlidingWindow |
| 4     | 4, 10, 16, 22, 28, 34, 40, 46, 52, 58 | SlidingWindow |
| 5     | 5, 11, 17, 23, 29, 35, 41, 47, 53, 59 | FullAttention |

分组规律：

```
Group 0: Layer % 6 == 0
Group 1: Layer % 6 == 1
Group 2: Layer % 6 == 2
Group 3: Layer % 6 == 3
Group 4: Layer % 6 == 4
Group 5: Layer % 6 == 5
```

分组依据是 KV Cache Spec 的兼容性，不代表模型的执行顺序发生变化。

## **3. SlidingWindowSpec 参数**

适用于 Group 0～4。

| 参数                      | 当前值     | 说明                    |
| ----------------------- | ------- | --------------------- |
| `block_size`            | 16      | 一个 Block 存储的 Token 数量 |
| `num_kv_heads`          | 16      | KV Head 数量            |
| `head_size`             | 256     | Key Head Dimension    |
| `head_size_v`           | 256     | Value Head Dimension  |
| `dtype`                 | float16 | KV Cache 数据类型         |
| `kv_quant_mode`         | NONE    | 不启用 KV Cache 量化       |
| `page_size_padded`      | None    | Padding 后的 Page Size  |
| `num_head_slots`        | None    | Head Slot 数量配置        |
| `state_content_bytes`   | None    | State 内容字节数           |
| `tokens_per_state`      | 1       | 每个 State 对应的 Token 数量 |
| `sliding_window`        | 1024    | 滑动窗口长度                |
| `extra_retained_tokens` | 0       | 额外保留的 Token 数         |

### **计算公式**

单 Token 的 KV Cache 大小：

```
KV Bytes Per Token
= 2 × num_kv_heads × head_size × dtype_bytes

= 2 × 16 × 256 × 2
= 16,384 Bytes
```

其中：

- 2：K 和 V。
- 16：KV Heads。
- 256：Head Dimension。
- 2：FP16 每个元素占 2 Bytes。

单 Block 大小：

```
Block Bytes
= KV Bytes Per Token × block_size

= 16,384 × 16
= 262,144 Bytes
= 256 KiB
```

单 Layer 最大窗口对应的 Block 数量（按完整窗口计算）：

```
Window Blocks
= ceil(sliding_window / block_size)

= ceil(1024 / 16)
= 64 Blocks
```

## **4. FullAttentionSpec 参数**

适用于 Group 5。

| 参数                     | 当前值     | 说明                       |
| ---------------------- | ------- | ------------------------ |
| `block_size`           | 32      | 一个 Block 存储 32 个 Token   |
| `num_kv_heads`         | 4       | KV Head 数量               |
| `head_size`            | 512     | Key Head Dimension       |
| `head_size_v`          | 512     | Value Head Dimension     |
| `dtype`                | float16 | KV Cache 数据类型            |
| `kv_quant_mode`        | NONE    | 不启用 KV Cache 量化          |
| `page_size_padded`     | None    | Padding 后的 Page Size     |
| `num_head_slots`       | None    | Head Slot 数量配置           |
| `state_content_bytes`  | None    | State 内容字节数              |
| `tokens_per_state`     | 1       | 每个 State 对应的 Token 数量    |
| `sliding_window`       | None    | 不使用滑动窗口限制                |
| `attention_chunk_size` | None    | 未配置 Attention Chunk Size |
| `non_causal`           | False   | 使用因果 Attention           |

### **计算公式**

单 Token 的 KV Cache 大小：

```
KV Bytes Per Token
= 2 × num_kv_heads × head_size × dtype_bytes

= 2 × 4 × 512 × 2
= 8,192 Bytes
```

单 Block 大小：

```
Block Bytes
= 8,192 × 32
= 262,144 Bytes
= 256 KiB
```

**重要：两种 Attention 的 Block Size 不同，但最终每个 Block 的内存大小相同。**

| 对比项        | SlidingWindow | FullAttention |
| ---------- | ------------- | ------------- |
| Block Size | 16            | 32            |
| KV Heads   | 16            | 4             |
| Head Dim   | 256           | 512           |
| FP16       | 2 Bytes       | 2 Bytes       |
| 单 Token KV | 16 KiB        | 8 KiB         |
| 单 Block KV | 256 KiB       | 256 KiB       |

## **5. KVCacheTensor：内存布局参数**

每个 Group 对应一个 Tensor，共 6 个 Tensor。

| 参数              | 当前值                 | 说明                  |
| --------------- | ------------------- | ------------------- |
| `size`          | 6,433,013,760 Bytes | 整个 Tensor 的内存大小     |
| `layers`        | 10 个 Layer          | Tensor 管理的 Layer 名称 |
| `layer_stride`  | 643,301,376 Bytes   | 相邻 Layer 的内存步长      |
| `block_stride`  | 262,144 Bytes       | 相邻 Block 的内存步长      |
| `offset`        | 0                   | Tensor 起始偏移量        |
| `host_resident` | False               | Tensor 是否驻留 Host    |

### **5.1 Block 内存大小**

```
Block Stride
= block_size × 2 × num_kv_heads × head_size × dtype_bytes

SlidingWindow:
= 16 × 2 × 16 × 256 × 2
= 262,144 Bytes

FullAttention:
= 32 × 2 × 4 × 512 × 2
= 262,144 Bytes
```

### **5.2 单 Layer 内存大小**

```
Layer Stride
= num_blocks × block_stride

= 2454 × 262,144
= 643,301,376 Bytes

= 613.5 MiB
```

### **5.3 单 Tensor 内存大小**

```
Tensor Size
= num_layers_per_group × layer_stride

= 10 × 643,301,376
= 6,433,013,760 Bytes

= 6 GiB
```

### **5.4 全部 Tensor 内存大小**

```
Total KV Cache Size
= num_groups × tensor_size

= 6 × 6 GiB
= 36 GiB
```

## **6. 内存布局与地址计算**

当前配置：

```
kv_cache_layout = "LBNHC"
```

关键参数：

```
L = Layer
B = Block
N = KV Head
H = Head Dimension
C = K/V 存储维度
```

按照当前 Tensor 的 Layer 和 Block 两级布局，可以计算逻辑内存偏移：

```
Offset
= layer_index × layer_stride
  + block_index × block_stride
```

例如：

```
Layer Index = 2
Block Index = 10

Offset
= 2 × 643301376
  + 10 × 262144

= 1,289,224,192 Bytes
```

注意：这只是 Layer/Block 级别的逻辑偏移。实际访问 K/V 数据还需要结合后端的物理 Tensor Shape、stride 和索引方式。

## **7. 参数计算汇总**

| 计算目标                   | 计算公式               | 结果        |
| ---------------------- | ------------------ | --------- |
| SlidingWindow 单 Token  | `2 × 16 × 256 × 2` | 16 KiB    |
| FullAttention 单 Token  | `2 × 4 × 512 × 2`  | 8 KiB     |
| SlidingWindow 单 Block  | `16 × 16 KiB`      | 256 KiB   |
| FullAttention 单 Block  | `32 × 8 KiB`       | 256 KiB   |
| 单 Layer 内存             | `2454 × 256 KiB`   | 613.5 MiB |
| 单 Group 内存             | `10 × 613.5 MiB`   | 6 GiB     |
| 全部 KV Cache            | `6 × 6 GiB`        | 36 GiB    |
| Sliding Window Block 数 | `ceil(1024 / 16)`  | 64        |

## **8. 核心概念总结**

建议记住下面五个关系：

```
KVCacheConfig
    └── num_blocks = 2454
         每个 Group 的逻辑 Block 数量

KVCacheGroupSpec
    └── Layer 分组 + KVCacheSpec
         决定哪些 Layer 共享同一类 KV Cache 管理规格

KVCacheSpec
    └── block_size + num_kv_heads + head_size + dtype
         决定单 Token 和单 Block 的 KV Cache 大小

KVCacheTensor
    └── size + layer_stride + block_stride
         描述实际 KV Cache 内存布局

KVCacheManager
    └── Request + Block 分配与回收
         管理请求运行时的 KV Cache 使用
```

**一个需要特别注意的区别：** `num_blocks=2454` 是配置中的逻辑 Block 容量，不代表每个请求可以使用 2454 个 Block，也不代表 Sliding Window 请求一定会占用全部 2454 个 Block。实际分配和回收还受到 Scheduler、KVCacheManager、BlockPool 以及 Prefix Cache 等机制影响。
