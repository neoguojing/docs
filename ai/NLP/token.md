下面这版我按**面试 + LLM 实际工程**重新压缩，去掉 BPE、Embedding、计费中重复解释的部分，只保留 Token 必须掌握的概念、公式和一个完整工程链路。

# Token 与 Tokenizer

## 1. Token 是什么？

Token 是 LLM 处理文本的基本离散单位，**不等于字符、单词**。

```text
Text
 ↓ Tokenizer
Token
 ↓ Vocabulary
Token ID
 ↓ Embedding
Vector
 ↓
Transformer
```

例如：

```text
"Hello world"
→ ["Hello", " world"]
→ [15496, 995]
```

* **Token**：文本切分后的单位
* **Token ID**：Token 在词表中的整数索引
* **Embedding**：Token ID 对应的向量

> LLM 实际接收的是 **Token ID 序列**，而不是原始字符串。

---

## 2. Vocabulary 与 Embedding

Vocabulary：

```math
V=\{token_0,token_1,...,token_{V-1}\}
```

其中 `V = vocab_size`。

Embedding 矩阵：

```math
E\in R^{V\times d}
```

Token ID 为 `i` 时：

```math
\boxed{x_i=E[i]}
```

例如：

```text
vocab_size = 50,000
d_model = 4,096

E ∈ R^(50000×4096)
```

输入 3 个 Token：

```text
[100, 2050, 3782]
```

得到：

```math
X\in R^{3\times4096}
```

---

## 3. Tokenizer

Tokenizer 负责：

```text
Text → Token IDs
```

常见方法：

| 方法         | 特点            |
| ---------- | ------------- |
| Word       | 词表大，存在 OOV    |
| Character  | 词表小，但序列长      |
| Subword    | 兼顾词表与序列长度     |
| Byte-level | 基于 Byte，覆盖能力强 |

现代 LLM 主要采用 **Subword / Byte-level** 方案，常见算法包括：

```text
BPE
WordPiece
Unigram / SentencePiece
```

不同模型使用的 Tokenizer 不同，因此：

```text
同一文本
→ Qwen Token 数
≠ Llama Token 数
≠ GPT Token 数
```

**Token 数必须以具体模型的 Tokenizer 为准。**

---

## 4. BPE 核心原理

BPE（Byte Pair Encoding）核心思想：

> 从较小的 Token 开始，不断合并训练语料中高频出现的相邻 Token Pair。

例如：

```text
low
lower
lowest
```

初始：

```text
l o w
l o w e r
l o w e s t
```

假设：

```text
l + o
```

频率最高：

```text
l o → lo
```

继续：

```text
lo + w → low
```

最终训练得到：

```text
Vocabulary + Merge Rules
```

在线阶段只执行这些规则：

```text
Text → Token IDs
```

> **Tokenizer 训练是离线过程；API 每次请求不会重新训练。**

---

## 5. 中文与 Token

不能简单认为：

```text
1 汉字 = 1 Token
1 英文单词 = 1 Token
```

例如：

```text
playing
→ play + ing
```

中文经过 UTF-8 / Byte-level Tokenizer 后，一个汉字通常包含多个 Byte，但最终可能对应一个或多个 Token。

因此：

```text
字符数 ≠ Token 数
```

实际 Token 数取决于：

```text
文本
 ↓
具体模型 Tokenizer
 ↓
Token 数
```

---

## 6. Token 与 Context Window

LLM 的上下文窗口以 **Token** 计算。

例如：

```text
128K Context
≈ 131072 Tokens
```

不是：

```text
128K 个汉字
128K 个英文单词
```

一次请求的上下文可能包括：

```text
System Prompt
+ Conversation History
+ User Prompt
+ Tool Definition
+ Tool Result
+ Model Output
```

因此：

```math
\boxed{
Context\ Tokens
=
Input\ Tokens + Output\ Tokens
}
```

实际可用长度还受到模型 Context Window 限制。

---

## 7. Token 对推理的影响

Self-Attention：

```math
Attention(Q,K,V)
=
softmax(\frac{QK^T}{\sqrt{d_k}})V
```

对于：

```math
Q,K,V\in R^{L\times d}
```

其中 `L` 是 Token 数。

因为：

```math
QK^T\in R^{L\times L}
```

Attention 计算复杂度近似：

```math
\boxed{O(L^2d)}
```

因此 Token 数增加会带来：

```text
Token ↑
  ↓
Attention 计算 ↑
  ↓
KV Cache ↑
  ↓
显存 / 延迟 / 成本 ↑
```

现代 LLM Serving 会通过 **KV Cache、GQA/MQA、Sliding Window、稀疏 Attention 等技术**降低长上下文的实际成本。

---

# 8. API 如何统计 Token？

生产 API 通常分成两个阶段：

```text
              请求阶段
                 │
                 ▼
           Chat Template
                 │
                 ▼
             Tokenizer
                 │
                 ▼
          Input Token Count
                 │
                 ▼
        Context / Quota Check
                 │
                 ▼
             LLM Serving
                 │
        ┌────────┴────────┐
        ▼                 ▼
      Prefill           Decode
        │                 │
        │            Generated Tokens
        ▼                 ▼
  Input Tokens       Output Tokens
```

### 预检查

API Gateway / Serving 层使用**与模型匹配的 Tokenizer**：

```python
tokens = tokenizer.encode(prompt)
input_tokens = len(tokens)
```

用于：

```text
Context Limit
Quota
Rate Limit
Cost Estimate
```

> 准确统计应该在 **Chat Template 处理之后**进行，因为 System Prompt、Tool Schema、Special Tokens 等都会影响 Token 数。

---

## 9. Token 统计与计费

模型生成过程中：

```text
Input Tokens
Output Tokens
Cached Tokens
```

最终形成 Usage：

```json
{
  "input_tokens": 8500,
  "output_tokens": 1200,
  "cached_tokens": 3000,
  "total_tokens": 9700
}
```

基本计费：

```math
Cost
=
InputTokens\times P_{input}
+
OutputTokens\times P_{output}
```

如果存在 Prompt Cache：

```math
Cost
=
CachedTokens\times P_{cached}
+
NonCachedInputTokens\times P_{input}
+
OutputTokens\times P_{output}
```

Streaming 并不会影响 Token 统计：

```text
Decode
 ↓
每生成一个 Token
 ↓
generated_tokens++
 ↓
请求结束
 ↓
Usage
```

因此：

> **预检查 Token ≠ 最终计费 Token。**

预检查用于提前判断：

```text
能不能执行
```

最终 Usage 用于确认：

```text
实际用了多少
```

---

# 10. Agent 中的 Token 统计

Agent 往往存在多次 LLM 调用：

```text
User
 ↓
LLM #1
 ↓
Tool
 ↓
LLM #2
 ↓
Tool
 ↓
LLM #3
 ↓
Final Answer
```

每次调用独立统计：

```text
LLM #1 → input / output
LLM #2 → input / output
LLM #3 → input / output
```

最终聚合：

```math
TotalInput=\sum Input_i
```

```math
TotalOutput=\sum Output_i
```

形成完整 Trace：

```text
Agent Trace
├── LLM Call #1
│   ├── input_tokens
│   └── output_tokens
├── Tool Call
├── LLM Call #2
│   ├── input_tokens
│   └── output_tokens
└── LLM Call #3
    ├── input_tokens
    └── output_tokens
```

这也是 LLM Agent 平台实现 **Observability + Usage Metering + Billing** 的基础。

---

# 11. 常见开源 Tokenizer

| 方案                             | 用途                          |
| ------------------------------ | --------------------------- |
| **Hugging Face Tokenizers**    | 通用、高性能 Tokenizer            |
| **tiktoken**                   | OpenAI 系列 Tokenizer         |
| **SentencePiece**              | LLaMA、Qwen 等模型生态常见          |
| **Transformers AutoTokenizer** | 根据模型自动加载 Tokenizer          |
| **vLLM Tokenizer**             | LLM Serving 中的 Tokenization |

核心思想都一样：

```text
加载模型对应的 Tokenizer
        ↓
Encode
        ↓
Token IDs
        ↓
len(Token IDs)
```

---

# 12. 面试核心知识链

```text
                    ┌── Vocabulary
                    │
Text
 ↓
Tokenizer ──────────┤
 ↓                  │
Token IDs            └── Merge Rules
 ↓
Embedding
 ↓
Transformer
 ↓
Prefill / Decode
 ↓
KV Cache
 ↓
Output Token IDs
 ↓
Decode
 ↓
Text
```

API 侧：

```text
Request
 ↓
Chat Template
 ↓
Tokenizer
 ↓
Input Token Count
 ↓
Context / Quota Check
 ↓
LLM Serving
 ↓
Output Token Count
 ↓
Usage
 ↓
Billing
```

### 必记公式

```math
\boxed{x_i=E[token\_id_i]}
```

```math
\boxed{E\in R^{V\times d}}
```

```math
\boxed{Attention\approx O(L^2d)}
```

```math
\boxed{
Cost=
InputTokens\times P_{in}
+
OutputTokens\times P_{out}
}
```

### 一句话总结

> **Tokenizer 将文本转换成 Token ID；Embedding 将 Token ID 转成向量；Transformer 基于 Token 序列进行计算；Context Window 按 Token 计算；Serving Runtime 统计实际 Input/Output Token；Usage 系统汇总 Token，Billing 根据不同 Token 类型计费。**

这版已经可以直接作为你 **LLM 基础 → Embedding → Transformer → LLM Serving** 文档中的 Token 章节，后面重点接 **Embedding** 会比较自然。
