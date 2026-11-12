# vLLM 配置参数深度解析：原理、使用场景与实践

以下按照你列出的参数逐项展开。重点不只是解释参数，而是把它们放回 vLLM 的推理链路中，说明它们解决的问题。

先明确一个原则：配置项的可选值、默认值和是否支持，必须以指定分支为准。 有些配置是嵌套对象的字段，并不是独立 CLI 参数；有些参数则会随模型架构和 GPU 后端变化。下面会区分通用原理、配置示例和需要核对源码的部分。

## 一、ModelConfig：模型与 Tokenizer

### 1. `--chat-template`

是什么？

Chat Template 是将 OpenAI 风格的消息结构转换成模型训练时使用的 Token 序列格式的模板。

OpenAI API 使用统一的消息格式：

```
{
  "messages": [
    {"role": "system", "content": "You are an assistant."},
    {"role": "user", "content": "Hello"},
    {"role": "assistant", "content": "Hi"}
  ]
}
```

但不同模型的训练格式不同。

例如，某些模型使用：

```
<|im_start|>system
You are an assistant.<|im_end|>
<|im_start|>user
Hello<|im_end|>
<|im_start|>assistant
```

另一些模型可能使用：

```
<|system|>
You are an assistant.
<|user|>
Hello
<|assistant|>
```

Chat Template 就负责完成这个转换。

处理流程：

OpenAI Messages

system / user / assistant / tool

Chat Template（Jinja2）

角色映射、消息拼接、控制标记

模型专用文本格式

ChatML / Llama 格式 / 其他模板

Tokenizer

文本转换成 Token IDs

LLM Prefill

配置方式一：使用模型自带模板

```
vllm serve Qwen/Qwen3-8B
```

如果模型仓库包含 `tokenizer_config.json` 中的 `chat_template`，vLLM 通常可以自动读取。

这是最推荐的方式。

配置方式二：指定自定义模板

```
vllm serve Qwen/Qwen3-8B \
  --chat-template ./chat_template.jinja
```

模板文件示例：

```
{% for message in messages %}
{{ '<|im_start|>' + message['role'] + '\n' }}
{{ message['content'] }}
{{ '<|im_end|>\n' }}
{% endfor %}
{% if add_generation_prompt %}
{{ '<|im_start|>assistant\n' }}
{% endif %}
```

这里的 `add_generation_prompt` 很重要。

用户消息结束后，模型需要知道接下来应该生成 Assistant 内容。模板通过追加 Assistant 起始标记来实现。

什么时候需要自定义？

| 场景                   | 是否需要  |
| -------------------- | ----- |
| 使用官方 Hugging Face 模型 | 通常不需要 |
| 模型没有提供 Chat Template | 需要    |
| 模型使用特殊对话格式           | 需要    |
| 模型需要特殊 Tool Call 标记  | 可能需要  |
| 微调模型修改了对话格式          | 需要    |
| 使用 OpenAI API 接入不同模型 | 可能需要  |

特别注意：Chat Template 不只是消息拼接。错误的模板会导致模型无法正确识别角色、工具调用边界、结束标记，甚至影响生成质量。

### 2. `--chat-template-content-format`

是什么？

控制传入 Chat Template 的 `content` 数据结构。

它解决的问题是：OpenAI 消息中的 `content` 不一定是字符串，也可能是多模态内容块。

主要模式：

| 模式       | 含义                 | 使用场景        |
| -------- | ------------------ | ----------- |
| `auto`   | 自动检测模板期望的内容格式      | 一般推荐        |
| `string` | 将内容作为普通字符串处理       | 纯文本模型、兼容旧模板 |
| `openai` | 保留 OpenAI 多模态内容块结构 | 视觉、音频等多模态模板 |

#### `string` 模式

输入：

```
{
  "role": "user",
  "content": "Hello"
}
```

模板获得的是字符串：

```
message["content"] == "Hello"
```

适合传统纯文本 Chat Template。

#### `openai` 模式

多模态输入：

```
{
  "role": "user",
  "content": [
    {"type": "text", "text": "What is in this image?"},
    {
      "type": "image_url",
      "image_url": {"url": "https://example.com/image.jpg"}
    }
  ]
}
```

模板可以访问结构化的内容块，而不是简单地把它们转换成字符串。

#### `auto` 模式

由 vLLM 判断模板的内容格式需求，尽量选择匹配的方式。

配置示例：

```
# 自动模式
vllm serve Qwen/Qwen3-8B \
  --chat-template-content-format auto

# 字符串模式
vllm serve Qwen/Qwen3-8B \
  --chat-template-content-format string

# OpenAI 结构化模式
vllm serve Qwen/Qwen3-VL-4B-Instruct \
  --chat-template-content-format openai
```

实际建议：

- 纯文本模型：优先 `auto`。
- 自己编写的旧式 Jinja 模板：如果明确期望字符串，使用 `string`。
- 多模态模型：优先 `auto`，模板明确要求 OpenAI 内容块时使用 `openai`。

两者的关键区别：

`chat-template` 决定如何组织消息；`chat-template-content-format` 决定模板接收到什么形态的消息内容。

### 3. `--tokenizer-mode`

是什么？

指定 Tokenizer 的实现方式。

Tokenizer 负责将文本转换为 Token ID，反向则负责将 Token ID 解码为文本。

\\[ Text \xrightarrow{Tokenizer} TokenIDs \\]

例如：

```
"Hello world"
    ↓
[9906, 1917]
```

Tokenizer 不同，最终 Token ID 也可能不同。必须和模型训练时使用的词表及分词规则匹配。

常见模式

| 模式        | 含义                            | 使用场景              |
| --------- | ----------------------------- | ----------------- |
| `auto`    | 自动选择                          | 默认推荐              |
| `slow`    | Hugging Face Python Tokenizer | 兼容需要 Python 实现的模型 |
| `mistral` | Mistral 专用 Tokenizer          | Mistral 系列模型      |
| `custom`  | 特定自定义实现（如果分支支持）               | 特定模型适配            |

其中 `auto` 通常会根据模型的 Tokenizer 配置选择 Fast 或兼容实现，并不意味着所有模型都使用同一种 Tokenizer。

示例：

```
# 自动选择
vllm serve Qwen/Qwen3-8B \
  --tokenizer-mode auto

# 使用 Hugging Face Python 实现
vllm serve Qwen/Qwen3-8B \
  --tokenizer-mode slow

# Mistral 专用实现
vllm serve mistralai/Mistral-7B-Instruct-v0.3 \
  --tokenizer-mode mistral
```

什么时候使用 slow？

- Fast Tokenizer 加载失败。
- 自定义 Tokenizer 只提供 Python 实现。
- 某些模型的特殊 Token 处理与 Fast 实现不兼容。

缺点是 Python Tokenizer 通常速度较慢。

注意：上面的模式属于常见 vLLM Tokenizer 设计，具体枚举必须与当前分支的 `arg_utils.py` 和 Tokenizer 初始化代码核对，不应把未注册的模式直接传入。

### 4. `--trust-remote-code`

是什么？

允许 Hugging Face 加载模型时执行模型仓库提供的 Python 代码。

普通模型一般包含：

```
model/
├── config.json
├── tokenizer_config.json
├── model.safetensors
└── generation_config.json
```

这些文件主要提供模型结构、词表、权重和生成配置。

但是一些模型有自定义架构，例如：

```
model/
├── configuration_custom.py
├── modeling_custom.py
├── tokenization_custom.py
├── config.json
└── model.safetensors
```

其中：

- `configuration_custom.py`：定义模型配置类。
- `modeling_custom.py`：定义模型计算结构。
- `tokenization_custom.py`：定义自定义 Tokenizer。

这些 Python 文件不能仅靠解析 JSON 使用，需要动态加载执行。

配置示例：

```
vllm serve organization/custom-model \
  --trust-remote-code
```

指定 revision：

```
vllm serve organization/custom-model \
  --trust-remote-code \
  --revision main \
  --code-revision <trusted-commit>
```

为什么需要它？

Hugging Face Transformers 允许模型仓库通过 `auto_map` 指定自定义 Python 类。

例如：

```
{
  "auto_map": {
    "AutoConfig": "configuration_custom.CustomConfig",
    "AutoModel": "modeling_custom.CustomModel"
  }
}
```

启用后，加载器可能动态导入这些代码。

安全风险：

这相当于允许模型仓库代码在你的 Python 进程中执行。

因此：

- 不要随意信任未知模型仓库。
- 尽可能固定 revision 或 commit。
- 检查自定义 Python 文件。
- 生产环境优先使用经过审计的模型。

它不是模型权重格式选项，而是模型代码加载权限选项。

### 5. `--seed`

是什么？

随机数种子，用于控制推理过程中随机数生成器的初始状态。

LLM 生成过程中，模型输出的是每个 Token 的概率分布：

\\[ P(x_t \mid x_1,\ldots,x\_{t-1}) \\]

例如：

| Token | 概率  |
| ----- | --- |
| A     | 0.5 |
| B     | 0.3 |
| C     | 0.2 |

如果使用随机采样，模型可能选择 A、B 或 C。

随机种子用于初始化随机数生成器，从而让随机采样在相同条件下尽可能可复现。

配置示例：

```
vllm serve Qwen/Qwen3-8B \
  --seed 42
```

Python：

```
from vllm import LLM, SamplingParamsllm = LLM(    model="Qwen/Qwen3-8B",    seed=42,)params = SamplingParams(    temperature=0.8,    top_p=0.9,    max_tokens=100,)outputs = llm.generate(    ["Explain KV Cache"],    params,)
```

Seed 和 Temperature 的区别：

| 参数          | 作用            |
| ----------- | ------------- |
| Seed        | 控制随机数序列       |
| Temperature | 调整概率分布的平滑程度   |
| Top-p       | 限制累计概率范围      |
| Top-k       | 限制候选 Token 数量 |

Temperature 越低，概率越集中；越高，概率越分散。

需要注意：Seed 不等于完全确定性。不同 GPU、不同 Kernel、不同批处理顺序或并行调度可能引入非确定性。

### 6. `--hf-config-path`

是什么？

指定 Hugging Face 模型配置文件的位置或替代配置来源。

一般模型目录中存在：

```
model/
├── config.json
├── model.safetensors
└── tokenizer_config.json
```

`config.json` 定义模型结构，例如：

```
{
  "model_type": "llama",
  "hidden_size": 4096,
  "num_hidden_layers": 32,
  "num_attention_heads": 32,
  "num_key_value_heads": 8,
  "intermediate_size": 11008
}
```

vLLM 需要读取这些参数，才能构建模型结构。

为什么需要单独指定？

某些场景中，模型权重目录和模型配置并不在一起：

```
/model/weights/
    model-00001.safetensors

/model/config/
    config.json
```

此时可以通过指定配置来源，让 vLLM 使用正确的模型结构定义。

示例：

```
vllm serve /model/weights \
  --hf-config-path /model/config
```

典型使用场景：

- 模型权重和配置文件分开存放。
- 使用修改过的模型 Config。
- 模型仓库配置不完整。
- 同一份权重需要适配不同的配置来源。

注意：它不是用于修改权重本身的参数。配置文件中的层数、隐藏维度、Attention 头数等必须与权重实际结构兼容，否则可能在模型构建或权重加载时失败。

### 7. `--runner`

是什么？

指定模型运行时采用哪一种任务执行模式。

不能把所有模型都理解为文本生成模型。

vLLM 除了自回归生成，还支持 Embedding、分类、评分等任务。

常见 Runner

| Runner     | 任务     | 输入           | 输出          |
| ---------- | ------ | ------------ | ----------- |
| `generate` | 自回归生成  | Prompt Token | 新生成 Token   |
| `pooling`  | 池化类任务  | Token 序列     | 固定维度向量或任务结果 |
| `draft`    | 草稿模型执行 | 输入 Token     | Draft Token |

其中，`draft` 的具体可用性与推测解码配置有关。

Generate 模式

例如：

```
Input:
What is AI?

Output:
AI is ...
```

执行流程：

```
Prompt
  ↓
Prefill
  ↓
KV Cache
  ↓
Decode
  ↓
Generated Tokens
```

Pooling 模式

例如文本 Embedding：

```
Input:
"How to use vLLM?"

Output:
[0.12, -0.31, 0.52, ...]
```

执行流程：

```
Input Text
    ↓
Tokenizer
    ↓
Transformer
    ↓
Hidden States
    ↓
Pooling
    ↓
Embedding
```

常见 Pooling 方式：

- CLS Pooling：取指定 CLS Token 对应的向量。
- Mean Pooling：对有效 Token 向量求平均。
- Last Token Pooling：取最后一个有效 Token 的向量。

公式：

\\[ e=\frac{1}{N}\sum\_{i=1}^{N}h_i \\]

其中 \\(h_i\\) 为第 i 个 Token 的隐藏状态。

配置示例：

```
# 文本生成
vllm serve Qwen/Qwen3-8B \
  --runner generate

# Embedding 模型
vllm serve Qwen/Qwen3-Embedding-0.6B \
  --runner pooling
```

具体模型必须支持相应任务。设置 Runner 不会自动把一个普通生成模型变成训练好的 Embedding 模型。

### 8. `--convert`

是什么？

控制模型加载或执行时采用的模型转换适配方式。

这里需要特别区分两个概念：

1. 模型转换：适配模型结构或任务输出形式。
2. 权重量化：改变权重存储和计算精度。

`--convert` 不是通用的权重量化开关。

例如，模型原始结构可能输出 Token Hidden States，但某些任务需要：

- 对 Token 表示做 Pooling。
- 在模型主体后接分类或评分 Head。
- 将模型适配为特定的 Embedding 或 Classification 任务。

模型转换机制可以负责选择对应的适配实现。

什么时候使用？

| 场景           | 作用            |
| ------------ | ------------- |
| Embedding 模型 | 适配 Pooling 输出 |
| 分类模型         | 适配分类任务        |
| Reward Model | 适配评分输出        |
| 特殊模型结构       | 使用对应转换适配器     |

示例（仅在当前分支注册了相应转换器时使用）：

```
vllm serve /path/to/model \
  --runner pooling \
  --convert <registered-converter>
```

不要把 `--convert` 与 `--quantization` 混用。前者关注模型执行结构或任务适配，后者关注权重和计算精度。

### 9. `--skip-tokenizer-init`

是什么？

跳过 Engine 初始化阶段的 Tokenizer 加载。

通常启动 vLLM 时，会同时准备：

```
Model
  ├── Model Weights
  └── Tokenizer
```

Tokenizer 主要负责：

- 文本转 Token ID。
- Token ID 转文本。
- 特殊 Token 识别。

但有些调用方本身已经拥有 Token ID，不需要 vLLM 再加载 Tokenizer。

配置示例：

```
from vllm import LLMllm = LLM(    model="Qwen/Qwen3-8B",    skip_tokenizer_init=True,)outputs = llm.generate(    prompt_token_ids=[[151644, 872, 9906]],)
```

这里直接传入 Token ID。

适用场景：

- 上游服务已经完成 Tokenization。
- 使用自定义 Tokenizer 服务。
- 希望减少启动时的 Tokenizer 初始化。
- 通过 Token ID 进行离线推理。

限制：

如果跳过 Tokenizer 初始化，就不能再假设 vLLM 可以自动完成普通文本输入的 Tokenize，以及所有需要 Tokenizer 的输出解码功能。

这适合 Token ID 驱动的推理接口，不适合普通文本用户直接调用的服务。

### 10. `--disable-sliding-window`

是什么？

关闭模型的 Sliding Window Attention（滑动窗口注意力）机制。

先理解普通 Causal Attention：

\\[ Attention(Q,K,V)= Softmax\left(\frac{QK^T}{\sqrt{d_k}}+Mask\right)V \\]

标准 Causal Mask 允许当前 Token 关注它之前的所有 Token。

但对于 Sliding Window Attention，每个 Token 只允许关注最近的 \\(W\\) 个 Token。

例如：

标准 Causal Attention

当前 Token 可以关注所有历史 Token

T1

T2

T3

T4

T5

T6

Sliding Window Attention（W=3）

当前 Token 只关注最近 3 个 Token

T1

T2

T3

T4

T5

T6

如果模型的窗口大小是 4096，那么在某些 Attention 层中，当前 Token 只关注最近 4096 个 Token。

为什么要使用滑动窗口？

1. 限制 Attention 计算量。
2. 降低长上下文场景的计算开销。
3. 降低部分模型的 KV Cache 需求。
4. 提高长序列推理效率。

为什么可以关闭？

某些模型架构允许通过配置关闭滑动窗口限制，使 Attention 使用更完整的历史上下文。

使用场景：

- 模型支持完整上下文 Attention。
- 需要验证 Sliding Window 对输出质量的影响。
- 特定任务需要访问更早的历史信息。

示例：

```
vllm serve <model> \
  --disable-sliding-window
```

注意：

关闭滑动窗口不代表模型一定获得更好的效果。

它可能增加 KV Cache 占用和 Attention 计算量，也可能改变模型原本训练时的注意力行为。对于依赖局部注意力架构训练的模型，强行关闭可能影响模型质量，甚至不受模型实现支持。

## 二、LoadConfig：权重加载

### 11. `--load-format`

是什么？

指定模型权重的加载格式，以及对应的加载器实现。

模型权重可以存储为不同文件格式。格式不同，权重的序列化方式、读取路径和加载效率也不同。

常见模型权重格式

| 格式           | 文件                  | 特点                |
| ------------ | ------------------- | ----------------- |
| Safetensors  | `.safetensors`      | 安全、支持高效读取和映射      |
| PyTorch      | `.bin`、`.pt`、`.pth` | PyTorch 序列化格式     |
| GGUF         | `.gguf`             | 常见于 llama.cpp 生态  |
| ModelOpt     | 相关量化权重              | NVIDIA 模型优化生态     |
| BitsAndBytes | 特定量化权重              | 8-bit、4-bit 等量化加载 |
| FP8          | `.safetensors` 等    | 低精度权重存储           |

注意：这些是权重存储格式或权重表示方式，不一定都是 `--load-format` 可以直接接受的值。是否支持取决于具体 Loader 和模型架构。

常见 vLLM Load Format

| 格式标识             | 含义                      |
| ---------------- | ----------------------- |
| `auto`           | 自动选择加载方式                |
| `safetensors`    | Safetensors 加载          |
| `pt`             | PyTorch 权重加载            |
| `npcache`        | NumPy Cache 相关加载        |
| `dummy`          | 创建虚拟权重用于测试              |
| `tensorizer`     | Tensorizer 加载           |
| `runai_streamer` | Run:ai Streaming Loader |
| `bitsandbytes`   | BitsAndBytes 加载         |
| `sharded_state`  | 分片状态字典加载                |
| `gguf`           | GGUF 加载（支持条件取决于分支）      |

示例：

```
# 自动加载
vllm serve /models/Qwen \
  --load-format auto

# Safetensors
vllm serve /models/Qwen \
  --load-format safetensors

# PyTorch
vllm serve /models/Qwen \
  --load-format pt

# 测试模型启动流程
vllm serve /models/Qwen \
  --load-format dummy
```

为什么需要不同加载器？

不同文件格式的数据组织方式不同：

```
Checkpoint
    │
    ▼
Load Format
    │
    ├── Safetensors Loader
    ├── PyTorch Loader
    ├── Tensorizer Loader
    ├── BitsAndBytes Loader
    └── Dummy Loader
    │
    ▼
State Dict / Weight Mapping
    │
    ▼
Model Parameters
    │
    ▼
GPU
```

其中 Dummy Loader 特别适合性能测试，因为它可以减少真实权重加载对测试的干扰，但不能用于正常模型推理。

### 12. `--safetensors-load-strategy`

是什么？

控制 Safetensors 文件加载时，如何从磁盘将权重数据读取到内存。

Safetensors 的优势是：

- 文件有明确的 Header 和 Tensor 元数据。
- Tensor 数据按偏移组织。
- 可以通过 Memory Mapping 等方式读取。
- 不需要像普通 Pickle Checkpoint 那样执行任意 Python 反序列化代码。

常见策略

| 策略                 | 原理          | 使用场景      |
| ------------------ | ----------- | --------- |
| `lazy`             | 延迟读取，需要时再访问 | 减少一次性读取   |
| `eager`            | 尽可能提前读取数据   | 适合顺序加载    |
| `prefetch`         | 提前读取后续数据    | 降低 I/O 等待 |
| `torchao` / 特殊加载策略 | 取决于分支扩展     | 特定加载路径    |

这里需要特别说明：上表是加载策略的概念分类，不是对你这个分支实际枚举值的确认。不能直接把所有名称都作为 CLI 取值。必须查看 `safetensors_load_strategy` 的 Enum 或 Literal 定义。

为什么需要加载策略？

假设一个 70B 模型有数十个 Safetensors 文件。

普通读取方式：

```
读取文件1
    ↓
等待磁盘读取
    ↓
加载文件1
    ↓
读取文件2
    ↓
等待磁盘读取
    ↓
加载文件2
```

预读取方式：

```
CPU 读取文件1 ────────► GPU 加载文件1
       │
       └── 预读取文件2
                 │
                 ▼
           GPU 加载文件2
                 │
                 └── 预读取文件3
```

通过 I/O 和权重加载工作重叠，减少等待时间。

### 13. `--safetensors-prefetch-num-threads`

是什么？

控制 Safetensors 预读取时使用的线程数量。

它的主要作用是提前将后续可能读取的数据放入 OS Page Cache。

例如：

```
vllm serve /models/Qwen \
  --safetensors-prefetch-num-threads 4
```

假设有 8 个权重文件：

```
Thread 1 → shard-1
Thread 2 → shard-2
Thread 3 → shard-3
Thread 4 → shard-4
```

多个线程可以并发发起文件读取或预取操作。

为什么需要？

模型加载通常受到磁盘 I/O 限制。

如果 GPU/CPU 已经处理完当前权重，但下一个权重还没读取，就会出现等待。

增加预取线程可以提高 I/O 并发，但不是越多越好：

- HDD：大量随机并发可能降低性能。
- SSD：适度增加并发可能提升吞吐。
- NVMe：需要根据设备和队列深度调优。
- 网络文件系统：需要结合带宽、延迟和服务端并发能力。

### 14. `--safetensors-prefetch-block-size`

是什么？

控制预取操作的块大小，即每次预读取多少数据。

示例：

```
vllm serve /models/Qwen \
  --safetensors-prefetch-block-size 1048576
```

这里用 1 MiB 作为示例，实际单位和合法取值要看当前分支实现。

什么是预读取？

预读取（Prefetch）就是在真正需要数据之前，提前把可能马上使用的数据读入操作系统缓存。

CPU 访问文件时，通常并不是直接每次都从物理磁盘取一个字节，而是借助操作系统的 Page Cache。

```
磁盘文件
   │
   ▼
OS Read / Readahead
   │
   ▼
Page Cache（内存）
   │
   ▼
CPU 读取权重数据
   │
   ▼
Tensor / GPU
```

Linux 中可以使用 `readahead()` 等机制提示内核提前读取文件内容。

块大小的影响：

| 块大小 | 特点                 |
| --- | ------------------ |
| 太小  | 系统调用和调度开销增加        |
| 适中  | 有利于连续 I/O          |
| 太大  | 浪费内存带宽和 Page Cache |

它与线程数的关系：

- 线程数控制并发读取任务数量。
- Block Size 控制单次读取数据量。

两者共同决定预取并发量。

### 15. `--ignore-patterns`

是什么？

在加载模型目录时，根据文件名模式跳过某些文件。

例如：

```
vllm serve /models/Qwen \
  --ignore-patterns "*.md" "*.txt"
```

这表示加载过程中忽略匹配的文件。

为什么需要？

Hugging Face 模型仓库除了模型权重，还可能包含：

```
model/
├── config.json
├── tokenizer.json
├── model.safetensors
├── README.md
├── training_args.bin
├── optimizer.pt
├── scheduler.pt
└── events.out.tfevents
```

某些文件并不参与推理，例如：

- README。
- 训练日志。
- 优化器状态。
- 训练过程的临时文件。
- 不需要的额外权重文件。

使用忽略模式可以避免加载器扫描或尝试加载不需要的文件。

注意：

不要忽略模型结构所依赖的权重分片、Tokenizer、Config 或量化元数据，否则会导致加载失败。

另外，具体模式匹配规则应以当前加载器实现为准，不能假设所有路径匹配行为都完全等同于 Shell Glob。

### 16. `--pt-load-map-location`

是什么？

控制 PyTorch Checkpoint 加载时，如何映射原始设备到目标设备。

例如，一个 PyTorch Checkpoint 可能是在 GPU 0 上保存的：

```
torch.save(    model.state_dict(),    "model.pt")
```

加载时，如果直接恢复原始设备映射，可能产生设备不存在的问题。

PyTorch 提供：

```
torch.load(    "model.pt",    map_location="cpu",)
```

将原始存储位置映射到 CPU。

vLLM 配置示例：

```
vllm serve /models/pt-model \
  --load-format pt \
  --pt-load-map-location cpu
```

常见映射方式

| 值        | 作用                            |
| -------- | ----------------------------- |
| `cpu`    | 所有 Tensor 映射到 CPU             |
| `cuda:0` | 映射到指定 GPU                     |
| `cuda`   | 映射到当前 CUDA 设备（具体解释依赖 PyTorch） |
| 字典映射     | 按原始设备映射到目标设备                  |

为什么通常使用 CPU？

模型加载时，先在 CPU 内存中读取权重，再根据分布式并行策略将权重分配到 GPU。

这能避免直接恢复原始 Checkpoint 的 GPU 设备位置。

注意：对于不同版本的 PyTorch，`torch.load` 还涉及 `weights_only` 等安全行为。不要把 `map_location` 理解为加载器的全部安全配置。

### 17. `--shuffle-safetensors-files`

是什么？

打乱 Safetensors 权重文件的加载顺序。

假设一个模型有：

```
model-00001-of-00008.safetensors
...
model-00008-of-00008.safetensors
```

默认可能按照固定顺序读取。

启用随机打乱后，不同 Worker 可能获得不同的文件读取顺序。

为什么分布式加载需要它？

假设多个 GPU Worker 同时加载：

```
Worker 0 → shard 1 → shard 2 → shard 3
Worker 1 → shard 1 → shard 2 → shard 3
Worker 2 → shard 1 → shard 2 → shard 3
```

如果所有 Worker 在相同时间读取相同文件，可能造成：

- 存储 I/O 热点。
- 文件系统竞争。
- 网络存储突发流量。
- 多个 Worker 同时等待同一组 I/O。

随机化后：

```
Worker 0 → shard 1 → shard 4 → shard 7
Worker 1 → shard 3 → shard 6 → shard 2
Worker 2 → shard 5 → shard 8 → shard 1
```

在合适的加载实现中，可以改善读取请求的分布。

什么时候使用？

- 多 GPU 分布式加载。
- 大量权重分片。
- 网络存储或共享文件系统。
- 存储设备出现 I/O 热点。

注意：它不是改变权重分片或模型参数顺序，而是改变文件读取顺序。具体是否改善性能，需要结合实际加载器是否按分片顺序进行调度判断。

## 三、AttentionConfig：Attention 计算配置

### 18. `--attention-backend`

是什么？

指定 Attention 算子使用哪一种底层计算实现。

虽然 Attention 数学公式固定，但不同 Kernel 的内存访问方式、线程组织、计算精度和硬件适配不同。

标准 Attention：

\\[ Attention(Q,K,V)=Softmax\left(\frac{QK^T}{\sqrt{d_k}}\right)V \\]

实际计算包含：

1. Q 和 K 矩阵乘法。
2. Softmax。
3. Softmax 结果与 V 矩阵乘法。

如果直接计算完整的 \\(N \times N\\) Attention 矩阵，会产生很大的中间显存开销。

FlashAttention 等实现通过分块计算和在线 Softmax，减少 HBM 与 GPU SRAM 之间的数据搬运。

常见 Backend

| Backend        | 技术特点                   | 典型用途                |
| -------------- | ---------------------- | ------------------- |
| FlashAttention | 分块 Attention、降低 HBM 访问 | Prefill、Decode      |
| FlashInfer     | 面向推理的 Attention Kernel | 高性能 Decode、Paged KV |
| Triton         | 自定义 GPU Kernel         | 灵活的算子实现             |
| Triton MLA     | MLA 专用实现               | DeepSeek 等 MLA 架构   |
| CUTLASS        | NVIDIA CUDA 模板化 Kernel | 高性能矩阵计算             |
| FlexAttention  | PyTorch 灵活 Attention   | 特定 Attention 模式     |
| TRT-LLM        | TensorRT-LLM 实现        | NVIDIA GPU 推理       |

这些是技术生态中的实现类别，并非保证全部都能在当前分支中通过同一个 CLI 枚举启用。

配置示例：

```
vllm serve Qwen/Qwen3-8B \
  --attention-backend FLASH_ATTN
```

如果当前分支支持自动选择：

```
vllm serve Qwen/Qwen3-8B \
  --attention-backend AUTO
```

需要结合 GPU 架构、CUDA、模型结构、数据类型和 KV Cache 格式判断兼容性。

如何选择？

| 场景                | 优先考虑                 |
| ----------------- | -------------------- |
| 常规 NVIDIA GPU 推理  | 自动选择或 FlashAttention |
| 高并发 Decode        | FlashInfer 等推理优化后端   |
| MLA 模型            | MLA 专用后端             |
| 特殊 Attention Mask | 支持相应 Mask 的后端        |
| 调试自定义 Attention   | Triton               |

不要简单认为某个后端永远最快。Prefill 和 Decode 的瓶颈不同，同一个后端在不同 Batch Size 下表现也可能不同。

### 19. `--attention-config`

是什么？

用于传递 AttentionConfig 的高级配置对象。

它不是单独指定 Attention Backend，而是对 Attention 执行策略进行更细粒度的控制。

例如：

```
vllm serve <model> \
  --attention-config '{"backend":"FLASH_ATTN"}'
```

上面是展示嵌套配置的写法示意，字段名称必须以当前分支 `AttentionConfig` 的定义为准。

需要理解的几个核心配置概念

#### 19.1 Prefill 和 Decode 后端分离

Prefill：

- 一次处理大量 Prompt Token。
- 主要是大规模矩阵运算。
- 通常具有较高计算并行度。

Decode：

- 每次通常生成一个或少量 Token。
- 需要读取历史 KV Cache。
- 经常受到显存带宽和访存效率限制。

因此，不同阶段可能适合不同 Kernel。

#### 19.2 MLA Prefill Backend

MLA 是 Multi-head Latent Attention。

传统 MHA 会保存每个 Attention Head 对应的 K/V，而 MLA 通过低维潜在表示压缩 KV 状态，降低缓存占用。

MLA Prefill 后端负责执行对应的 Prefill 计算。

#### 19.3 Backend Per Kind

不同 Attention 类型可以选择不同后端。

例如：

```
Decoder Self Attention → Backend A
Encoder Attention      → Backend B
Cross Attention        → Backend C
```

这样可以避免强制所有 Attention 使用同一种实现。

#### 19.4 FlashAttention 版本

某些 GPU 架构和 CUDA 环境适合不同版本的 FlashAttention。版本选择可能影响支持的数据类型、Head Dimension 和 Attention 模式。

配置原则：

如果没有明确的性能问题，建议使用自动选择。只有在性能测试、模型兼容性排查或明确需要特定 Kernel 时，再覆盖默认选择。

### 20. `use_prefill_query_quantization`

是什么？

控制 Prefill 阶段 Query 是否使用量化表示进行计算。

先区分 Prefill 和 Decode：

| 阶段      | 计算特点                         |
| ------- | ---------------------------- |
| Prefill | 对输入的全部 Token 进行并行计算          |
| Decode  | 每轮计算新 Token 的 Query，并读取历史 KV |

Prefill 中，Query 的维度通常为：

\\[ Q \in \mathbb{R}^{B \times H \times S \times D} \\]

其中：

- \\(B\\)：Batch Size。
- \\(H\\)：Attention Head 数。
- \\(S\\)：Prompt 长度。
- \\(D\\)：Head Dimension。

当 \\(S\\) 很大时，Query 张量的计算和访存开销也会增加。

量化原理

例如将 FP16 Query 转换为 FP8：

\\[ Q\_{FP8}=Quantize(Q\_{FP16}) \\]

一种常见的线性量化形式：

\\[ Q\_{quant}=\operatorname{round}(Q/s) \\]

其中 \\(s\\) 为量化 Scale。

计算完成后，根据需要进行反量化或使用对应的低精度 Kernel。

为什么只强调 Prefill Query？

Prefill 和 Decode 的计算特征不同。Prefill 更容易形成大规模矩阵运算，能够利用低精度 Tensor Core 提高计算吞吐。

Decode 的 Query 通常很小，整体瓶颈也可能主要来自 KV Cache 读取，因此收益并不相同。

什么时候使用？

- 长 Prompt Prefill。
- 支持 FP8 或相关低精度计算的 GPU。
- Attention 后端明确支持 Query 量化。
- 经过精度验证后希望提升 Prefill 吞吐。

注意：这是后端相关的高级配置，不等于把模型权重转换成 FP8，也不等于把 KV Cache 转换成 FP8。

### 21. `indexer_kv_dtype`

是什么？

控制 Indexer 使用的 KV 数据类型。

这里的 Indexer 需要结合 MLA / 稀疏 Attention 架构理解。

在普通 MHA 中，Attention 通常直接计算：

\\[ QK^T \\]

但在一些稀疏 Attention 架构中，模型会先通过一个索引器（Indexer）判断哪些历史 Token 值得参与后续 Attention 计算。

可以理解为：

```
历史 Token
    │
    ▼
Indexer
    │
    ▼
计算相关性 / 重要性
    │
    ▼
选择 Top-K 历史 Token
    │
    ▼
Sparse Attention
```

Indexer 需要维护相关的 KV 表示。

`indexer_kv_dtype` 控制这些 KV 数据的精度和存储格式。

为什么需要单独设置？

Indexer 的 KV 与主 Attention 使用的 KV 不一定需要完全相同的精度。

降低 Indexer KV 精度可以减少存储和访存开销，但可能影响相关性排序和 Token 选择结果。

因此，需要在精度与性能之间权衡。

具体可用数据类型要看当前模型和后端支持，不能直接假定所有 `kv-cache-dtype` 的值都能用于 Indexer。

### 22. `use_non_causal`

是什么？

控制 Attention 是否采用非因果模式。

标准自回归语言模型采用 Causal Attention。

当前 Token 只能看到自身以及之前的 Token：

\\[ AttentionMask(i,j)= \begin{cases} 0 & j\leq i\\\ -\infty & j>i \end{cases} \\]

其中 \\(i\\) 是 Query 位置，\\(j\\) 是 Key 位置。

非因果 Attention 则允许 Query 访问未来位置的 Key：

\\[ Attention(Q,K,V)=Softmax(QK^T/\sqrt{d_k})V \\]

不再使用标准的上三角因果限制。

示意：

Causal Attention

每行表示一个 Query，只能访问当前位置之前的 Key

✓

—

—

—

✓

✓

—

—

✓

✓

✓

—

✓

✓

✓

✓

Non-Causal Attention

每个 Query 可以访问整个序列

✓

✓

✓

✓

✓

✓

✓

✓

✓

✓

✓

✓

✓

✓

✓

✓

什么时候使用？

- Encoder 模型。
- BERT 类双向编码任务。
- 某些非自回归任务。
- 特殊的 Prefix / Bidirectional Attention 计算。

注意：

不要在普通自回归生成模型上随意开启。非因果 Attention 会破坏标准自回归生成所依赖的因果约束，可能让模型在训练和推理时的 Attention 行为不一致。

## 四、MambaConfig：状态空间模型

### 23. Mamba 是什么？

Mamba 是一种基于状态空间模型（State Space Model，SSM）的序列建模架构。

Transformer 使用 Attention 建立 Token 之间的关联，而 Mamba 使用状态空间更新来传播历史信息。

Transformer 的计算特点

每个新 Token 都要读取历史 Token 的 K/V：

\\[ Attention(Q,K,V) \\]

历史 KV Cache 随序列长度增长。

Mamba 的计算特点

通过状态 \\(h_t\\) 保存历史信息：

\\[ h_t=A h\_{t-1}+B x_t \\]

\\[ y_t=C h_t+D x_t \\]

其中：

- \\(x_t\\)：当前输入。
- \\(h_t\\)：历史状态。
- \\(A,B,C,D\\)：状态空间参数。
- \\(y_t\\)：当前输出。

Mamba 的核心思想是通过选择性状态更新（Selective State Space）控制历史信息如何进入状态。

对比：

| 特征     | Transformer      | Mamba       |
| ------ | ---------------- | ----------- |
| 核心机制   | Attention        | State Space |
| 历史信息   | KV Cache         | 状态缓存        |
| Decode | 读取历史 KV          | 更新状态        |
| 序列长度   | KV 存储随长度增长       | 每层状态规模通常固定  |
| 长序列计算  | Attention 计算开销较大 | 适合高效序列建模    |

注意，Mamba 的状态并不意味着完全没有缓存，它仍然需要维护每层的状态。

### 23.1 Mamba 的主要计算流程

```
Input Token
    │
    ▼
Projection
    │
    ▼
Selective Parameter Generation
    │
    ├── Δ（时间步）
    ├── B（输入映射）
    └── C（输出映射）
    │
    ▼
Selective Scan
    │
    ▼
State Update
    │
    ▼
Output
```

Selective Scan 是其核心计算过程。

它需要处理递归状态更新：

\\[ h_t=\bar A_t h\_{t-1}+\bar B_t x_t \\]

其中 \\(\bar A_t,\bar B_t\\) 可以随输入变化。

### 23.2 MambaConfig 相关参数

| 参数                                       | 作用                           |
| ---------------------------------------- | ---------------------------- |
| `mamba_backend`                          | 选择 Mamba 计算后端                |
| `mamba_ssu_algorithm`                    | 选择 Selective State Update 算法 |
| `mamba_prefill_backend`                  | Prefill 阶段 Scan 后端           |
| `enable_mamba_cache_stochastic_rounding` | 启用状态缓存随机舍入                   |
| `mamba_cache_philox_rounds`              | Philox 随机数生成相关配置             |

Prefill 和 Decode 的区别

Prefill：

- 一次输入多个 Token。
- 可以通过并行 Scan 算法处理序列。
- 重点在于并行计算。

Decode：

- 每次输入一个新 Token。
- 使用上一次的状态更新当前状态。
- 重点在于低延迟状态更新。

使用场景：

- 使用 Mamba 模型。
- 使用 Mamba 与 Transformer 混合架构。
- 需要优化 SSM 状态缓存。
- 需要针对不同 GPU 选择 Scan 后端。

普通 Qwen Transformer 模型不需要专门配置 Mamba 后端。

## 五、StructuredOutputsConfig：结构化输出

### 24. 是什么？

结构化输出（Structured Outputs）用于限制 LLM 的输出，使其符合预先定义的格式。

普通 LLM 输出：

```
The weather is sunny and temperature is 25 degrees.
```

结构化输出：

```
{
  "weather": "sunny",
  "temperature": 25
}
```

为什么需要？

Agent 系统通常需要解析模型输出并执行后续动作。

例如：

```
LLM
 │
 ▼
JSON
 │
 ▼
Parser
 │
 ▼
Business Logic
```

如果模型输出不符合 JSON 格式，就可能导致解析失败。

结构化输出通过 Grammar 或其他约束解码方式限制模型可生成的 Token。

### 24.1 约束解码原理

假设模型当前的候选 Token 概率分布为：

\\[ P(token|context) \\]

Grammar 根据当前生成状态计算允许的 Token 集合：

\\[ A=\\{token \mid token \text{ 符合当前语法状态}\\} \\]

然后屏蔽不允许的 Token：

\\[ P'(token)= \begin{cases} P(token),&token\in A\\\ 0,&token\notin A \end{cases} \\]

再进行归一化采样。

这样可以在生成阶段保证输出符合约束，而不是完全依赖生成后校验。

### 24.2 配置示例

CLI：

```
vllm serve Qwen/Qwen3-8B \
  --structured-outputs-config \
  '{"backend":"xgrammar"}'
```

注意：具体 Backend 及其可选参数以当前分支为准。

API 请求示例：

```
from openai import OpenAIclient = OpenAI(    base_url="http://localhost:8000/v1",    api_key="EMPTY",)response = client.chat.completions.create(    model="qwen",    messages=[        {            "role": "user",            "content": "Generate a person profile."        }    ],    response_format={        "type": "json_schema",        "json_schema": {            "name": "person",            "schema": {                "type": "object",                "properties": {                    "name": {"type": "string"},                    "age": {"type": "integer"}                },                "required": ["name", "age"],                "additionalProperties": False            }        }    })print(response.choices[0].message.content)
```

### 24.3 主要配置概念

| 配置                              | 含义             |
| ------------------------------- | -------------- |
| `backend`                       | 约束解码后端         |
| `disable_fallback`              | 是否禁止不支持时回退     |
| `disable_any_whitespace`        | 是否限制空白字符       |
| `disable_additional_properties` | 是否禁用额外 JSON 字段 |
| `whitespace_pattern`            | 空白字符规则         |

这些字段是结构化输出配置中需要关注的概念；最终字段名、类型和默认值应以当前分支的 `StructuredOutputsConfig` 定义为准。

常见后端包括：

- XGrammar：面向结构化输出的高性能 Grammar 引擎。
- Guidance：提供约束生成能力的框架。
- Outlines：结构化生成和有限状态约束生态。

是否支持这些后端，取决于当前 vLLM 版本。

## 六、ParallelConfig：分布式并行配置

### 25. 分布式并行的基本概念

先理解为什么需要并行。

假设模型有 70B 参数，BF16 权重理论上需要：

\\[ 70B \times 2 Bytes = 140GB \\]

单张 GPU 无法容纳时，需要把模型分布到多张 GPU。

vLLM 主要涉及以下并行策略：

Tensor Parallel（TP）

单层模型计算拆分

GPU 0

W 分片 0

GPU 1

W 分片 1

Pipeline Parallel（PP）

GPU 0

Layer 0–15

GPU 1

Layer 16–31

Data Parallel（DP）

Replica 0

Model Copy

Replica 1

Model Copy

Expert Parallel（EP）

GPU 0

Experts

GPU 1

Experts

GPU 2

Experts

GPU 3

Experts

### 25.1 Tensor Parallel：`--tensor-parallel-size`

是什么？

把 Transformer 单层中的矩阵运算拆分到多个 GPU。

假设：

\\[ Y=XW \\]

如果：

\\[ W=[W_1,W_2] \\]

那么：

\\[ Y=[XW_1,XW_2] \\]

不同 GPU 可以分别计算矩阵分片，再根据计算结构进行必要的通信。

示例：

```
vllm serve Qwen/Qwen3-32B \
  --tensor-parallel-size 2
```

表示使用两张 GPU 进行 Tensor Parallel。

适用：

- 模型权重无法放入单卡。
- 单层计算需要拆分。
- 多 GPU 之间具备较高通信带宽。

TP 会产生 GPU 间通信，例如 All-Reduce、All-Gather 等。

### 25.2 Pipeline Parallel：`--pipeline-parallel-size`

是什么？

将 Transformer 的不同 Layer 分配给不同 GPU。

例如 32 层模型：

```
GPU 0：Layer 0～15
GPU 1：Layer 16～31
```

示例：

```
vllm serve Qwen/Qwen3-32B \
  --pipeline-parallel-size 2
```

使用场景：

- GPU 数量较多。
- 模型层数较多。
- 希望降低单卡权重占用。
- TP 通信成本较高。

PP 的通信主要发生在 Pipeline Stage 之间，传递中间 Hidden States。

### 25.3 Data Parallel：`--data-parallel-size`

是什么？

创建多个模型副本，每个副本处理不同请求。

示例：

```
vllm serve Qwen/Qwen3-8B \
  --data-parallel-size 2
```

逻辑：

```
                API Server
                    │
             Request Router
                /       \
               /         \
          Replica 0    Replica 1
          GPU 0        GPU 1
```

DP 的主要目的不是拆分模型权重，而是提高整体请求处理能力。

适合：

- 模型可以单卡容纳。
- 并发请求很多。
- 希望通过多个副本提高吞吐。

### 25.4 Expert Parallel：`--enable-expert-parallel`

是什么？

MoE 模型包含多个 Expert，不同 Token 会通过 Router 选择部分 Expert。

例如：

```
Token
  │
  ▼
Router
  │
  ├── Expert 0
  ├── Expert 1
  ├── Expert 2
  └── Expert 3
```

EP 将不同 Expert 分配到不同 GPU。

示例：

```
vllm serve <moe-model> \
  --tensor-parallel-size 2 \
  --enable-expert-parallel
```

具体 GPU 数量和 Expert 分片方式还取决于 TP、EP Group 等配置。

### 25.5 Context Parallel

是什么？

把输入序列的上下文维度分配到多个设备。

假设：

\\[ S=8192 \\]

可以将序列拆分：

```
GPU 0：Token 0～4095
GPU 1：Token 4096～8191
```

它主要解决超长上下文计算和显存压力问题。

与 TP 的区别：

- TP：拆分模型矩阵维度。
- CP：拆分序列维度。

### 26. `--distributed-executor-backend`

是什么？

指定 vLLM 如何创建、管理和调度分布式 Worker。

它解决的是执行进程管理问题，不是 Attention 算法或通信算法本身。

常见 Backend

| Backend | 原理                     | 使用场景        |
| ------- | ---------------------- | ----------- |
| `mp`    | Python Multiprocessing | 单机多 GPU     |
| `ray`   | Ray 分布式任务执行框架          | 多机、多 GPU 集群 |
| `uni`   | 单 Worker / 单进程执行       | 单 GPU 或简单场景 |

具体可选值和自动选择逻辑以当前分支为准。

#### Multiprocessing

```
vllm serve Qwen/Qwen3-8B \
  --tensor-parallel-size 2 \
  --distributed-executor-backend mp
```

流程：

```
Main Process
    │
    ├── Worker Process 0 → GPU 0
    │
    └── Worker Process 1 → GPU 1
```

适合单机多 GPU，部署简单，不需要独立 Ray 集群。

#### Ray

```
ray start --head

vllm serve Qwen/Qwen3-32B \
  --tensor-parallel-size 4 \
  --distributed-executor-backend ray
```

Ray 提供：

- 分布式进程调度。
- Worker 生命周期管理。
- 节点资源管理。
- 任务分发。
- 多节点执行。

注意：Ray 是执行框架，不是 NCCL 的替代品。GPU 间底层集合通信仍可能由 NCCL 等通信库承担。

### 27. `--node-rank`

是什么？

表示当前机器在分布式集群中的节点编号。

例如有两个节点：

```
Node 0
Node 1
```

Node 0 是主节点，Node 1 是工作节点。

配置示例：

主节点：

```
vllm serve Qwen/Qwen3-32B \
  --nnodes 2 \
  --node-rank 0 \
  --master-addr 192.168.1.100 \
  --master-port 29500 \
  --tensor-parallel-size 8
```

从节点：

```
vllm serve Qwen/Qwen3-32B \
  --nnodes 2 \
  --node-rank 1 \
  --master-addr 192.168.1.100 \
  --master-port 29500 \
  --tensor-parallel-size 8
```

Node Rank 与 GPU Rank 的区别

| 参数           | 表示                   |
| ------------ | -------------------- |
| `node-rank`  | 当前机器在集群中的编号          |
| `LOCAL_RANK` | 当前机器内的 GPU Worker 编号 |
| `RANK`       | 分布式进程全局编号            |

例如：

```
Node 0:
  GPU 0 → Global Rank 0
  GPU 1 → Global Rank 1

Node 1:
  GPU 0 → Global Rank 2
  GPU 1 → Global Rank 3
```

### 28. `--master-addr` 和 `--master-port`

是什么？

用于分布式进程之间建立初始化通信的地址和端口。

例如：

```
Node 0: 192.168.1.100
Node 1: 192.168.1.101
```

指定：

```
--master-addr 192.168.1.100
--master-port 29500
```

所有节点通过这个地址和端口连接到分布式初始化协调端。

通信什么？

主要包括：

1. 进程组初始化。
2. Rank 和 World Size 等信息协调。
3. 通信组建立。
4. 集合通信同步。
5. 模型权重分片加载后的运行时通信。

例如 Tensor Parallel 中：

\\[ Y_0=XW_0 \\]

\\[ Y_1=XW_1 \\]

两个 Worker 需要通过通信操作交换或聚合结果。

注意：Master Address/Port 通常用于分布式初始化，不代表所有 GPU 通信都通过这个 TCP 端口完成。实际数据传输可能通过 NCCL 使用 NVLink、PCIe、InfiniBand 或 TCP 等链路。

### 29. `--distributed-timeout-seconds`

是什么？

设置分布式初始化或通信操作的超时时间。

例如：

```
vllm serve Qwen/Qwen3-32B \
  --distributed-timeout-seconds 1800
```

这里 1800 秒只是示例。

为什么需要？

分布式推理需要多个 Worker 协同工作。

例如 TP=4：

```
Worker 0 ─┐
Worker 1 ─┤
Worker 2 ─┼── Collective Communication
Worker 3 ─┘
```

如果 Worker 2 没有启动，其他 Worker 可能一直等待。

常见原因：

- 某个节点网络不通。
- Worker 进程崩溃。
- GPU 初始化失败。
- NCCL 初始化失败。
- 某个 Worker 执行时间过长。
- 节点间环境或模型文件不一致。

超时后可以让程序报错退出，而不是无限等待。

注意：

增大 Timeout 只能解决合法但耗时较长的初始化问题，不能修复网络错误、Rank 不匹配或 Worker 崩溃。

### 30. `--all2all-backend`

是什么？

选择 MoE Expert Parallel 中 All-to-All 通信的后端实现。

先理解 MoE：

```
Tokens
   │
   ▼
Router
   │
   ▼
Expert Selection
   │
   ▼
Dispatch Tokens
   │
   ▼
Experts on Different GPUs
   │
   ▼
Return Expert Outputs
```

假设 Token 0 被分配到 GPU 1 上的 Expert 2，而 Token 1 被分配到 GPU 0 上的 Expert 3。

就需要将 Token 分发到对应 GPU。

这就涉及 All-to-All 通信。

All-to-All 是什么？

每个进程向其他进程发送不同的数据，同时从其他进程接收数据。

例如：

GPU 0

Tokens / Experts

GPU 1

Tokens / Experts

GPU 2

Tokens / Experts

GPU 3

Tokens / Experts

All-to-All Exchange

每个 GPU 都可能与其他 GPU 交换 Token 数据

常见通信后端

| Backend                   | 技术特点                           |
| ------------------------- | ------------------------------ |
| `naive`                   | 简单通用通信实现                       |
| `allgather_reducescatter` | All-Gather + Reduce-Scatter 组合 |
| `pplx`                    | DeepEP 等生态相关实现，需核对具体注册项        |
| `deepep_high_throughput`  | 面向高吞吐的 Expert 通信               |
| `deepep_low_latency`      | 面向低延迟的 Expert 通信               |
| `mori`                    | 特定分布式通信实现                      |
| `flashinfer_all2allv`     | FlashInfer 相关变长 All-to-All 实现  |

具体取值随分支变化。

为什么要选择不同后端？

MoE 的通信有两种不同优化目标：

- High Throughput：一次传输更多数据，提高总体带宽利用率。
- Low Latency：减少单次通信等待时间，降低 Token 响应延迟。

适用场景：

| 场景             | 优先目标             |
| -------------- | ---------------- |
| 大 Batch MoE 推理 | 高吞吐              |
| 小 Batch、实时请求   | 低延迟              |
| 多节点 EP         | 网络通信优化           |
| 单机多 GPU EP     | NVLink/PCIe 通信优化 |

### 31. `--ubatch-size`

是什么？

Micro Batch Size，微批次大小。

它与普通 Batch Size 的区别在于：一个大 Batch 可以拆成多个更小的微批次，分别执行。

例如：

```
Batch Size = 16

拆分为：

Ubatch 0：4
Ubatch 1：4
Ubatch 2：4
Ubatch 3：4
```

为什么需要微批次？

假设 GPU 一次执行 16 个请求：

- 显存压力较大。
- 某些并行阶段可能出现资源竞争。
- Prefill 和 Decode 可能相互影响。

拆成更小微批次，可以控制单次执行规模。

在 vLLM 中还需要结合 Dual Batch Overlap（DBO）理解。

### 31.1 Dual Batch Overlap

DBO 的核心思想是让不同微批次的计算阶段尽可能重叠执行。

例如：

```
时间 →
Ubatch 0: Prefill ───── Decode
Ubatch 1:       Prefill ───── Decode
```

如果硬件和执行路径支持，可以减少 GPU 空闲时间。

配置示例：

```
vllm serve <model> \
  --enable-dbo \
  --ubatch-size 4
```

注意：这只是说明参数之间的关系。实际 DBO 还有触发阈值、并行调度和模型支持条件，不应仅凭设置 `ubatch-size` 就认为一定启用了 DBO。

### 32. `--enable-elastic-ep`

是什么？

Elastic Expert Parallel，弹性专家并行。

普通 EP 中，Expert 到 GPU 的映射通常在启动时确定。

例如：

```
GPU 0 → Expert 0,1
GPU 1 → Expert 2,3
GPU 2 → Expert 4,5
GPU 3 → Expert 6,7
```

Elastic EP 的目标是使 Expert Parallel 的部署和资源分配具备弹性。

为什么需要？

MoE 模型中，各个 Expert 的负载不一定均匀。

例如：

```
Expert 0：1000 tokens
Expert 1：100 tokens
Expert 2：800 tokens
Expert 3：50 tokens
```

可能出现某些 GPU 负载很高，另一些 GPU 资源闲置。

弹性 EP 的设计目标是提高资源利用率，并适应专家负载或资源配置的变化。

与普通 EP 的区别

| 普通 EP          | Elastic EP |
| -------------- | ---------- |
| 相对固定的专家布局      | 支持弹性资源布局   |
| 负载变化时可能出现热点    | 目标是改善负载分布  |
| 主要关注 Expert 分片 | 还涉及专家资源调度  |

配置：

```
vllm serve <moe-model> \
  --enable-expert-parallel \
  --enable-elastic-ep
```

这里要注意：Elastic EP 不是一个简单的动态负载均衡开关。它需要对应的分布式运行时、专家迁移或资源协调机制支持。具体当前分支实现应查看 Elastic EP 的初始化和 Worker 代码。

### 33. `--max-parallel-loading-workers`

是什么？

控制模型加载阶段，最多允许多少个 Worker 并行加载模型权重。

假设 TP=8：

```
GPU Worker 0
GPU Worker 1
...
GPU Worker 7
```

如果所有 Worker 同时读取权重，可能造成存储 I/O 和 CPU 内存压力。

通过该参数可以限制并发加载 Worker 数量。

示例：

```
vllm serve Qwen/Qwen3-32B \
  --tensor-parallel-size 8 \
  --max-parallel-loading-workers 4
```

意味着控制最多 4 个 Worker 同时参与对应的并行加载过程。

为什么需要？

| 并发加载数量 | 特点               |
| ------ | ---------------- |
| 太小     | 模型加载时间可能较长       |
| 适中     | 能充分利用存储带宽        |
| 太大     | 可能产生 I/O 竞争、内存峰值 |

特别是在网络文件系统、共享存储和多节点部署中，控制加载并发很有价值。

它影响的是启动阶段，不是 Decode 阶段的请求并发数。

## 七、CacheConfig：KV Cache 和内存管理

### 34. `--block-size`

是什么？

KV Cache 的 Block 大小，即一个 KV Cache Block 能保存多少个 Token 的 K/V 数据。

vLLM 使用 PagedAttention 管理 KV Cache，不要求每条请求都占据一段连续的 GPU 显存。

为什么需要 Block？

假设请求 A 有 10 个 Token，请求 B 有 15 个 Token。

如果每个请求都预分配连续的最大上下文空间：

```
Request A：预分配 8192 Token
Request B：预分配 8192 Token
```

会浪费大量显存。

PagedAttention 把 KV Cache 分成固定大小的 Block：

假设：

\\[ BlockSize=16 \\]

那么：

```
Request A：使用 Block 0
Request B：使用 Block 1
```

随着 Token 增长，再分配新的 Block。

Paged KV Cache 示意（Block Size = 4）

不同请求的 KV Block 可以分散存储，不要求物理连续

B0

A

B1

B

B2

Free

B3

A

B4

Free

B5

B

B6

Free

B7

Free

Request A

Request B

Free

实际实现中还会维护 Block Table，记录逻辑 Token Block 到物理 KV Block 的映射。

Block 的 KV Cache 显存大小

对于标准 Transformer，可以近似计算：

\\[ M\_{block}= 2\times L\times B\times H\_{kv}\times D_h\times S\times Bytes \\]

其中：

- 2：K 和 V。
- \\(L\\)：层数。
- \\(B\\)：Block Size。
- \\(H\_{kv}\\)：KV Head 数量。
- \\(D_h\\)：Head Dimension。
- \\(S\\)：一个 Block 中的 Token 数。
- Bytes：每个元素占用字节数。

注意：上式中的 \\(B\\) 和 \\(S\\) 不应重复计算。更清晰的写法是：

\\[ M\_{block}=2L \times H\_{kv}\times D_h\times BlockSize\times Bytes \\]

默认值是什么？

在不同 vLLM 版本中，默认 Block Size 可能不同，且会受到平台和 Attention 后端限制。常见配置为 16，但不能直接将其视为你指定分支的默认值。

建议通过：

```
vllm serve <model> --help
```

以及 `CacheConfig` 中的 `block_size` 默认值确认。

配置示例：

```
vllm serve Qwen/Qwen3-8B \
  --block-size 16
```

Block Size 越大，不代表性能一定越好。它还会影响内部碎片、调度和部分后端的兼容性。

### 35. `--prefix-caching-hash-algo`

是什么？

控制 Prefix Cache 中 Block Hash 的计算算法。

先理解 Prefix Caching。

如果多个请求共享相同的 Prompt 前缀，那么这部分 Token 的 KV Cache 可以复用。

例如：

```
Request A:
System Prompt + User A

Request B:
System Prompt + User B
```

两个请求的 System Prompt 相同。

可以复用前缀部分的 KV Cache，避免重复 Prefill。

为什么需要 Hash？

vLLM 需要识别一个 Block 是否与历史 Block 内容相同。

可以将 Block 的 Token 内容和相关上下文转换为 Hash：

\\[ H_i=Hash(H\_{i-1},Tokens_i) \\]

其中：

- \\(H_i\\)：当前 Block 的 Hash。
- \\(H\_{i-1}\\)：前一个 Block 的 Hash。
- \\(Tokens_i\\)：当前 Block 的 Token 内容。

这种链式结构能够让相同 Token Block 在相同前缀上下文下得到相同标识。

常见算法概念

| 算法          | 特点             |
| ----------- | -------------- |
| SHA-256     | 密码学哈希，碰撞概率极低   |
| xxHash      | 高性能非密码学哈希      |
| SHA-1       | 传统哈希算法         |
| Python Hash | 进程相关哈希行为，取决于实现 |

这些是常见哈希算法概念，不代表全部都被当前分支注册。

示例（以 SHA-256 为例，具体取值须确认）：

```
vllm serve Qwen/Qwen3-8B \
  --enable-prefix-caching \
  --prefix-caching-hash-algo sha256
```

为什么要区分算法？

- 速度：Hash 计算也是 CPU 工作。
- 碰撞风险：错误复用 Cache 会导致输出错误。
- 可复现性：跨进程、跨机器需要一致的 Hash 行为。
- 安全性：避免恶意构造碰撞造成错误缓存命中。

需要注意：Hash 只用于识别 Cache Block，并不是把 KV Cache 本身压缩或加密。

### 36. `--swap-space`

是什么？

为每个 GPU Worker 配置 CPU Swap 空间，用于 KV Cache 等相关资源的交换管理。

它不是 Linux 的磁盘 Swap 分区。

这是一个容易混淆的概念：

| 概念              | 含义                  |
| --------------- | ------------------- |
| Linux Swap      | 内存页与磁盘之间的交换         |
| vLLM Swap Space | vLLM 使用的 CPU 内存交换预算 |
| KV Cache        | GPU 上保存历史 K/V       |
| CPU Offload     | 将部分数据保留或迁移到 CPU 内存  |

例如：

```
vllm serve Qwen/Qwen3-8B \
  --swap-space 8
```

这里通常以 GiB 为单位，表示每个 GPU Worker 的交换空间预算，具体单位以当前分支实现为准。

为什么需要？

当 GPU 显存不足时，部分缓存可能需要转移到 CPU 内存。

但需要注意，Swap 并不意味着可以无成本地扩大 GPU 推理容量：

```
GPU KV Cache
     │
     ▼
CPU Memory
     │
     ▼
需要时重新传回 GPU
```

GPU 与 CPU 之间的数据传输会增加延迟。

因此，它更适合处理特定缓存交换场景，而不是替代充足的 GPU 显存。

### 37. `--cpu-offload-gb`

是什么？

控制每个 GPU Worker 可使用的 CPU Offload 容量。

例如：

```
vllm serve Qwen/Qwen3-32B \
  --cpu-offload-gb 8
```

表示配置 CPU Offload 相关容量，通常是每 GPU Worker 的预算。

为什么需要 Offload？

假设模型权重需要 48GB，而 GPU 只有 24GB。

可以考虑将一部分权重放到 CPU 内存。

```
             Model Weights
                  │
           ┌──────┴──────┐
           │             │
       GPU Memory    CPU Memory
       24 GB         Offloaded Weights
           │             │
           └──────┬──────┘
                  │
             GPU Compute
```

执行时，需要将部分数据从 CPU 传输到 GPU。

Offload 与 Quantization 的区别

| 方式                | 原理          | 代价        |
| ----------------- | ----------- | --------- |
| Quantization      | 降低权重精度      | 精度损失、量化开销 |
| CPU Offload       | 将部分数据放到 CPU | PCIe 数据传输 |
| Tensor Parallel   | 分散到多 GPU    | GPU 通信    |
| Pipeline Parallel | 分层放到多 GPU   | Stage 间通信 |

CPU Offload 的使用场景：

- GPU 显存不足。
- 模型略大于 GPU 可用容量。
- 推理吞吐要求不是特别高。
- CPU 内存充足。
- GPU 与 CPU 之间具有较高带宽。

为什么不建议无限增加？

假设 GPU 计算需要 1ms，而 CPU-GPU 数据传输需要 10ms，那么计算单元会长时间等待数据。

所以 Offload 能解决容量问题，但可能牺牲吞吐和延迟。

## 八、LoRAConfig：动态模型适配器

### 38. LoRA 是什么？

LoRA（Low-Rank Adaptation）是一种参数高效微调技术。

它不需要完整更新模型原始权重，而是增加低秩矩阵来表达微调变化。

原始权重：

\\[ W\in\mathbb{R}^{d\times k} \\]

微调时：

\\[ W'=W+\Delta W \\]

LoRA 将变化矩阵近似分解为：

\\[ \Delta W=BA \\]

其中：

\\[ A\in\mathbb{R}^{r\times k} \\]

\\[ B\in\mathbb{R}^{d\times r} \\]

并且：

\\[ r\ll \min(d,k) \\]

因此需要训练和保存的参数大幅减少。

推理时：

\\[ Y=XW+XBA \\]

原始模型权重 W 保持不变，BA 表示 LoRA 适配器带来的变化。

### 38.1 vLLM 如何使用 LoRA？

```
                  Base Model
                      │
          ┌───────────┼───────────┐
          │           │           │
       Request A   Request B   Request C
          │           │           │
       LoRA A      LoRA B      LoRA C
          │           │           │
          └───────────┼───────────┘
                      │
                  GPU Compute
```

不同请求可以使用同一个基础模型，但指定不同的 LoRA Adapter。

### 38.2 配置示例

```
vllm serve Qwen/Qwen3-8B \
  --enable-lora \
  --max-loras 4 \
  --max-lora-rank 64 \
  --max-cpu-loras 8 \
  --lora-modules \
    customer-a=/models/lora/customer-a \
    customer-b=/models/lora/customer-b
```

这里的 LoRA 路径和 Rank 只是示例，需要与实际适配器一致。

### 38.3 关键参数

| 参数                      | 含义                      |
| ----------------------- | ----------------------- |
| `enable_lora`           | 是否开启 LoRA               |
| `max_loras`             | 单个 Batch 可使用的最大 LoRA 数量 |
| `max_lora_rank`         | 支持的最大 LoRA Rank         |
| `max_cpu_loras`         | CPU 侧可管理的 LoRA 数量       |
| `lora_modules`          | 启动时注册的 LoRA 模块          |
| `lora_extra_vocab_size` | 额外词表空间                  |
| `fully_sharded_loras`   | LoRA 参数分片执行策略           |
| `max_lora_chunk_size`   | LoRA 分块大小               |

### 38.4 使用场景

- 多租户模型服务。
- 企业客户定制模型。
- 多个业务领域共享一个 Base Model。
- 动态切换模型适配器。
- 减少重复加载完整模型的显存开销。

注意：LoRA 不代表完全没有额外显存开销。Adapter 权重、运行时缓冲区、Rank 和并发 LoRA 数量都会影响资源占用。

## 九、SchedulerConfig：调度策略

### 39. `--scheduling-policy`

是什么？

控制 Scheduler 在多个待处理请求之间如何选择下一个执行对象。

vLLM 的 Scheduler 不仅决定谁先执行，还需要考虑：

- Prefill 和 Decode 的竞争。
- 每轮 Token 预算。
- GPU KV Cache Block 是否足够。
- 请求是否需要抢占。
- 请求优先级和等待时间。

常见策略

| 策略         | 原理                            | 使用场景         |
| ---------- | ----------------------------- | ------------ |
| `fcfs`     | First Come First Served，先到先服务 | 公平、简单        |
| `priority` | 按请求优先级调度                      | 多租户服务、VIP 请求 |

具体枚举以目标分支为准。

#### FCFS

例如：

```
Request A：10:00:01
Request B：10:00:02
Request C：10:00:03
```

执行顺序：

```
A → B → C
```

优点：

- 简单。
- 容易理解。
- 不需要复杂优先级管理。

缺点：

- 后续高优先级请求不能轻易超越前面请求。
- 长请求可能增加其他请求的等待时间。

#### Priority

例如：

```
Request A：Priority 1
Request B：Priority 10
Request C：Priority 5
```

执行顺序可能是：

```
B → C → A
```

优点：

- 支持服务等级区分。
- 重要请求可以优先执行。

缺点：

- 低优先级请求可能长期等待。
- 需要合理设置优先级。
- 具体是否支持抢占，取决于调度实现。

配置示例：

```
vllm serve Qwen/Qwen3-8B \
  --scheduling-policy fcfs
```

或者：

```
vllm serve Qwen/Qwen3-8B \
  --scheduling-policy priority
```

### 40. `--preemption-mode`

是什么？

控制当 GPU KV Cache Block 不足、无法继续调度请求时，vLLM 如何处理已经运行的请求。

假设：

```
GPU KV Cache 总容量：100 Blocks

Request A：50 Blocks
Request B：40 Blocks
Request C：20 Blocks
```

当前只剩 10 Blocks。

Request C 需要 20 Blocks，无法直接继续运行。

这时就可能触发 Preemption。

常见抢占方式

| 方式          | 原理                   | 影响              |
| ----------- | -------------------- | --------------- |
| `recompute` | 释放请求 KV Cache，之后重新计算 | 消耗额外 Prefill 计算 |
| `swap`      | 将 KV Cache 转移到 CPU   | 消耗 CPU 内存和传输带宽  |

#### Recompute

```
Request A
   │
   ▼
释放 KV Cache
   │
   ▼
其他请求获得 Block
   │
   ▼
Request A 再次调度
   │
   ▼
重新计算 Prompt
```

优点：

- 不需要大量 CPU 内存。
- 不需要在 CPU 和 GPU 间搬运完整 KV Cache。

缺点：

- 重新执行 Prefill。
- 浪费 GPU 计算。
- 长 Prompt 代价较高。

#### Swap

```
GPU KV Cache
      │
      ▼
CPU Memory
      │
      ▼
等待资源
      │
      ▼
重新传回 GPU
```

优点：

- 保留已有 KV Cache。
- 避免完整重新计算。

缺点：

- CPU 内存占用。
- PCIe/NVLink 传输开销。
- Swap 和恢复操作可能增加延迟。

配置示例：

```
vllm serve Qwen/Qwen3-8B \
  --preemption-mode recompute
```

具体支持的模式和默认值应检查当前分支。

什么时候使用？

- KV Cache 经常耗尽。
- 长上下文请求很多。
- 并发数超过 GPU Cache 容量。
- 希望在显存不足时仍然维持服务。

实际上，优先应该调整：

1. `max-model-len`。
2. `max-num-seqs`。
3. `max-num-batched-tokens`。
4. `gpu-memory-utilization`。
5. `kv-cache-memory-bytes`。

而不是单纯依赖抢占。

## 十、CompilationConfig：模型编译与 CUDA Graph

### 41. CompilationConfig 是什么？

CompilationConfig 控制 vLLM 如何将模型计算图转换为更高效的执行形式。

PyTorch 默认的 Eager Mode：

```
y = model(x)
```

执行时，Python/PyTorch 会逐步调度算子：

```
Python
  │
  ├── Linear
  ├── RMSNorm
  ├── Attention
  ├── MLP
  └── Linear
```

每次执行都可能涉及 Python 调度、算子 Dispatch、CUDA Kernel Launch。

当模型推理请求很多时，CPU Launch Overhead 也可能成为瓶颈。

CompilationConfig 的目的就是减少这些开销，并通过算子融合、图优化和 CUDA Graph 提升 GPU 执行效率。

### 41.1 PyTorch 编译的基本流程

PyTorch Model

TorchDynamo

捕获 Python 模型执行过程

FX Graph

生成计算图表示

TorchInductor

图优化、算子融合、代码生成

Triton / CUDA Kernel

GPU Execution

需要注意，vLLM 并不是所有算子都必须经过 TorchInductor。它也有大量自定义 CUDA/Triton Kernel，以及模型特定的执行路径。

### 41.2 `mode`

编译模式决定编译优化的程度。

不同分支中的模式定义可能变化。常见 vLLM 设计包含：

| 模式  | 含义        |
| --- | --------- |
| `0` | 不编译       |
| `1` | 部分编译      |
| `2` | 更完整的编译    |
| `3` | 更激进的编译和优化 |

这里的数字只是常见模式设计示意，不能代替当前分支枚举定义。应查看 `CompilationMode` 或相应配置类。

配置示例：

```
vllm serve Qwen/Qwen3-8B \
  --compilation-config '{"mode":3}'
```

### 41.3 `backend`

指定编译后端。

常见概念：

- Eager：直接执行。
- TorchInductor：PyTorch 编译后端。
- Triton：生成 GPU Kernel。
- 自定义编译后端：vLLM 的专用优化实现。

编译后端决定如何把计算图转化为可执行代码。

### 41.4 `custom_ops`

控制自定义算子的启用或禁用。

例如：

```
RMSNorm
  │
  ├── PyTorch 原生实现
  │
  └── vLLM Custom CUDA Kernel
```

自定义 Kernel 通常会针对 GPU 架构优化：

- 内存访问。
- 线程组织。
- Warp 调度。
- Shared Memory 使用。
- 向量化访问。

自定义算子可能比通用 PyTorch 实现更快，但也依赖硬件和数据类型支持。

### 41.5 `splitting_ops`

控制计算图在哪些算子边界进行拆分。

为什么需要拆分？

并不是所有算子都适合一起编译。

例如：

```
RMSNorm
  ↓
Attention
  ↓
All Reduce
  ↓
MLP
```

其中 Attention 可能使用专门的 CUDA Kernel，All Reduce 需要调用通信库。

如果强行将所有算子合并为一个计算图，可能增加编译复杂度或遇到不支持的操作。

因此需要在适当位置拆分计算图。

### 41.6 CUDA Graph

CUDA Graph 是什么？

CUDA Graph 是 NVIDIA CUDA 提供的一种执行机制，可以将一系列 GPU 操作捕获成图，然后反复执行。

普通执行：

```
CPU Launch Kernel A
CPU Launch Kernel B
CPU Launch Kernel C
CPU Launch Kernel D
```

每次执行都需要 CPU 提交操作。

CUDA Graph：

```
CPU Capture
     │
     ▼
CUDA Graph
  ├── Kernel A
  ├── Kernel B
  ├── Kernel C
  └── Kernel D
     │
     ▼
Graph Replay
```

后续只需要提交 Graph Replay，减少 CPU 端重复 Launch 开销。

### 41.7 CUDA Graph 的底层 API 示例

下面是一个独立 CUDA C++ 示例，展示基本原理：

```
#include <cuda_runtime.h>
#include <cstdio>

__global__ void add_one(float* data, int n) {
    int i = blockIdx.x * blockDim.x + threadIdx.x;

    if (i < n) {
        data[i] += 1.0f;
    }
}

int main() {
    const int n = 1024;
    const size_t bytes = n * sizeof(float);

    float* d_data = nullptr;
    cudaMalloc(&d_data, bytes);

    // 预热
    add_one<<<4, 256>>>(d_data, n);
    cudaDeviceSynchronize();

    cudaGraph_t graph;
    cudaGraphExec_t graph_exec;

    // 创建 Stream
    cudaStream_t stream;
    cudaStreamCreate(&stream);

    // 开始捕获
    cudaStreamBeginCapture(
        stream,
        cudaStreamCaptureModeGlobal
    );

    // 捕获 Kernel Launch
    add_one<<<4, 256, 0, stream>>>(d_data, n);

    // 结束捕获
    cudaStreamEndCapture(stream, &graph);

    // 实例化 Graph
    cudaGraphInstantiate(
        &graph_exec,
        graph,
        nullptr,
        nullptr,
        0
    );

    // 多次重放
    for (int i = 0; i < 100; ++i) {
        cudaGraphLaunch(graph_exec, stream);
    }

    cudaStreamSynchronize(stream);

    cudaGraphExecDestroy(graph_exec);
    cudaGraphDestroy(graph);
    cudaStreamDestroy(stream);
    cudaFree(d_data);

    return 0;
}
```

编译：

```
nvcc cuda_graph.cu -o cuda_graph
```

关键 API：

| API                      | 作用            |
| ------------------------ | ------------- |
| `cudaStreamBeginCapture` | 开始捕获          |
| `cudaStreamEndCapture`   | 结束捕获          |
| `cudaGraphInstantiate`   | 实例化 Graph     |
| `cudaGraphLaunch`        | 执行 Graph      |
| `cudaGraphExecDestroy`   | 释放 Graph 执行对象 |
| `cudaGraphDestroy`       | 释放 Graph      |

这说明 CUDA Graph 并不是把模型权重编译成一个文件，而是捕获 GPU 执行操作的依赖关系。

### 41.8 CUDA Graph Capture Size

为什么需要 Capture Size？

CUDA Graph 捕获时，通常需要固定或明确的执行形状及相关内存布局。

假设 Batch Size 分别是：

```
1
2
4
8
16
```

可以为这些规模分别准备 Graph。

配置示例：

```
vllm serve Qwen/Qwen3-8B \
  --compilation-config \
  '{"cudagraph_capture_sizes":[1,2,4,8,16]}'
```

运行时：

```
Batch Size = 4
      │
      ▼
选择 Graph 4
      │
      ▼
Replay
```

如果运行时 Batch Size 不匹配捕获形状，可能需要使用其他 Graph、Padding 或回退到非 Graph 执行路径。

### 41.9 CompilationConfig 的其他重要字段

| 字段                           | 作用               |
| ---------------------------- | ---------------- |
| `mode`                       | 编译模式             |
| `backend`                    | 编译后端             |
| `custom_ops`                 | 自定义算子选择          |
| `splitting_ops`              | 图拆分边界            |
| `cudagraph_mode`             | CUDA Graph 执行模式  |
| `cudagraph_capture_sizes`    | 捕获 Batch Size    |
| `max_cudagraph_capture_size` | 最大捕获规模           |
| `cudagraph_num_of_warmups`   | 捕获前预热次数          |
| `compile_sizes`              | 编译形状规模           |
| `inductor_compile_config`    | TorchInductor 配置 |
| `inductor_passes`            | 图优化 Pass         |
| `cache_dir`                  | 编译缓存目录           |

配置示例：

```
vllm serve Qwen/Qwen3-8B \
  --compilation-config '{
    "mode": 3,
    "cudagraph_capture_sizes": [1,2,4,8,16],
    "cudagraph_num_of_warmups": 2
  }'
```

这是配置结构的示意，字段和 JSON 类型应以目标分支为准。

### 41.10 CUDA Graph 与 TorchInductor 的区别

这是一个容易混淆的地方：

| 项目          | TorchInductor | CUDA Graph       |
| ----------- | ------------- | ---------------- |
| 核心作用        | 编译和优化计算图      | 捕获和重放 GPU 操作     |
| 优化对象        | 算子和计算图        | Kernel 提交流程      |
| 典型优化        | 算子融合、代码生成     | 减少 CPU Launch 开销 |
| 是否生成 Kernel | 通常可以          | 不负责生成 Kernel     |
| 是否依赖 CUDA   | 视编译后端而定       | 是                |

两者可以配合使用，而不是互相替代。

使用建议：

- 初次部署：使用默认编译配置。
- 性能测试：分别测试 Eager、编译和 CUDA Graph。
- 小 Batch Decode：重点观察 CUDA Graph 收益。
- 长 Prompt Prefill：重点观察 Attention Kernel 和矩阵计算性能。
- 编译失败：先关闭或降低编译优化，定位是否为特定算子问题。

## 十一、KernelConfig：GPU 算子优化

### 42. KernelConfig 是什么？

Kernel 是 GPU 上执行具体计算任务的程序。

例如 Transformer 一层包括：

```
Input
  │
  ▼
RMSNorm
  │
  ▼
QKV Projection
  │
  ▼
Attention
  │
  ▼
Output Projection
  │
  ▼
Residual
  │
  ▼
MLP
  │
  ▼
Output
```

每一个计算阶段都可能由一个或多个 GPU Kernel 实现。

KernelConfig 的作用是控制某些算子实现的选择、执行优化和后端行为。

### 42.1 为什么需要专用 Kernel？

以 RMSNorm 为例：

\\[ RMSNorm(x)= \frac{x}{\sqrt{\frac{1}{d}\sum\_{i=1}^{d}x_i^2+\epsilon}} \cdot \gamma \\]

普通 PyTorch 实现可能需要：

1. 平方。
2. 求和。
3. 求平均。
4. 开平方。
5. 除法。
6. 乘权重。

如果每一步都启动一个 Kernel，就会产生多个 Launch。

专用 RMSNorm Kernel 可以把这些操作融合：

```
PyTorch:
Square → Sum → Mean → Sqrt → Divide → Multiply

Fused Kernel:
          RMSNorm
```

这样可以减少中间张量和 Kernel Launch 次数。

### 42.2 常见 Kernel 类型

| Kernel     | 功能        | 优化重点         |
| ---------- | --------- | ------------ |
| RMSNorm    | 归一化       | 算子融合         |
| LayerNorm  | 归一化       | Reduce 优化    |
| GEMM       | 矩阵乘法      | Tensor Core  |
| Attention  | 注意力计算     | IO 和计算效率     |
| MoE        | Expert 计算 | Token 分发与计算  |
| All-Reduce | 集合通信      | 通信带宽         |
| RoPE       | 位置编码      | 向量化计算        |
| Sampling   | Token 采样  | GPU/CPU 执行开销 |

### 42.3 Kernel 的基本执行模型

GPU Kernel 通常由 Grid、Block、Thread 组成：

```
Grid
 ├── Block 0
 │    ├── Thread 0
 │    ├── Thread 1
 │    └── ...
 │
 ├── Block 1
 │    ├── Thread 0
 │    ├── Thread 1
 │    └── ...
 │
 └── Block N
```

CUDA 示例：

```
__global__ void vector_add(
    float* a,
    float* b,
    float* c,
    int n
) {
    int idx =
        blockIdx.x * blockDim.x +
        threadIdx.x;

    if (idx < n) {
        c[idx] = a[idx] + b[idx];
    }
}
```

调用：

```
int threads = 256;
int blocks = (n + threads - 1) / threads;

vector_add<<<blocks, threads>>>(
    a, b, c, n
);
```

其中：

- `threadIdx`：线程在 Block 内的位置。
- `blockIdx`：Block 在 Grid 中的位置。
- `blockDim`：每个 Block 的线程数。
- `gridDim`：Grid 的 Block 数量。

### 42.4 KernelConfig 示例

建议先通过 Python 查看配置对象：

```
from vllm.config import VllmConfighelp(VllmConfig)
```

实际使用时：

```
vllm serve Qwen/Qwen3-8B \
  --compilation-config '{"mode":3}'
```

如果需要控制特定 Kernel，应使用当前分支 `KernelConfig` 暴露的字段，而不是假设所有 CUDA 算子都可以通过一个统一的 `kernel-config` 开关选择。

KernelConfig 与 CompilationConfig 的区别：

| 项目   | KernelConfig        | CompilationConfig          |
| ---- | ------------------- | -------------------------- |
| 主要目标 | 算子实现选择与优化           | 计算图编译                      |
| 关注对象 | 单个或一组 GPU 算子        | 模型计算图                      |
| 典型技术 | CUDA、Triton、CUTLASS | Dynamo、Inductor、CUDA Graph |
| 主要收益 | 提高算子效率              | 降低整体执行开销                   |

## 十二、环境变量：Distributed

### 43. `LOCAL_RANK`

是什么？

表示当前分布式进程在本机 GPU Worker 中的编号。

假设一台服务器有 4 张 GPU：

```
GPU 0 → LOCAL_RANK=0
GPU 1 → LOCAL_RANK=1
GPU 2 → LOCAL_RANK=2
GPU 3 → LOCAL_RANK=3
```

PyTorch 分布式初始化通常使用：

```
import torch.distributed as distdist.init_process_group(    backend="nccl")
```

进程可以通过：

```
import oslocal_rank = int(os.environ["LOCAL_RANK"])
```

获取本机 Rank。

然后设置当前 CUDA Device：

```
import torchtorch.cuda.set_device(local_rank)
```

为什么需要？

多 GPU 环境中，多个 Worker 需要知道自己应该使用哪张 GPU。

LOCAL_RANK 解决的是本机设备编号问题，不代表全局分布式编号。

### 44. `VLLM_DP_RANK`

是什么？

表示当前进程所属的 Data Parallel 副本编号。

例如：

```
DP Size = 4

Replica 0 → VLLM_DP_RANK=0
Replica 1 → VLLM_DP_RANK=1
Replica 2 → VLLM_DP_RANK=2
Replica 3 → VLLM_DP_RANK=3
```

作用：

- 区分不同 DP 副本。
- 确定当前请求或执行进程所属的副本。
- 配合 DP 协调机制进行请求分发和同步。

与 LOCAL_RANK 的区别：

LOCAL_RANK 是 GPU 进程在本机的位置。

VLLM_DP_RANK 是 DP 副本在 DP Group 中的位置。

### 45. `VLLM_DP_SIZE`

是什么？

表示 Data Parallel 的总副本数量。

例如：

```
VLLM_DP_SIZE=4
```

表示 DP Group 包含 4 个副本。

为什么需要？

在 DP 模式下，vLLM 需要知道：

- 有多少个模型副本。
- 当前副本属于哪个 Rank。
- 请求应该分发到哪个副本。
- 如何协调副本之间的状态。

环境变量示意：

```
export VLLM_DP_SIZE=2
export VLLM_DP_RANK=0
```

这些值通常由 vLLM 的分布式启动流程或外部调度器注入，而不是建议用户手动设置后就认为 DP 已经完成部署。

### 45.1 DP、TP、PP 的关系

假设：

```
TP = 2
PP = 2
DP = 2
```

那么理论上的 GPU 进程规模：

\\[ WorldSize=TP\times PP\times DP \\]

\\[ WorldSize=2\times2\times2=8 \\]

对应：

```
DP 0:
  TP 0 / PP 0
  TP 1 / PP 0
  TP 0 / PP 1
  TP 1 / PP 1

DP 1:
  TP 0 / PP 0
  TP 1 / PP 0
  TP 0 / PP 1
  TP 1 / PP 1
```

实际进程 Rank 编排、GPU 映射和通信组构建由分布式初始化逻辑决定。

注意：这只是标准组合并行的规模计算示意，实际还要考虑 EP、CP、外部 DP、API Server 等进程。

## 十三、把这些配置放回 vLLM 整体架构

以上参数可以归纳为六个层次。理解这个关系，比单独记忆参数更重要。

## vLLM 配置与运行时对应关系

1\. 输入层

ModelConfig / Tokenizer

`chat-template`、`tokenizer-mode`、`trust-remote-code`、`runner`

2\. 模型加载层

LoadConfig

`load-format`、`safetensors-load-strategy`、`prefetch`、`ignore-patterns`

3\. 计算层

AttentionConfig / MambaConfig / KernelConfig

Attention、SSM、CUDA Kernel、Triton、FlashAttention

4\. 分布式层

ParallelConfig

TP、PP、DP、EP、All-to-All、NCCL、Ray

5\. 资源管理层

CacheConfig / LoRAConfig / OffloadConfig

KV Cache、Block、Prefix Cache、CPU Offload、Adapter

6\. 执行调度层

SchedulerConfig / CompilationConfig

Batch、Preemption、CUDA Graph、TorchInductor

## 十四、建议重点掌握的参数

如果从 vLLM 系统工程师的角度学习，我建议按下面的优先级掌握。

| 优先级 | 参数/配置                          | 必须理解的知识          |
| --- | ------------------------------ | ---------------- |
| P0  | `max-model-len`                | 上下文与 KV Cache    |
| P0  | `gpu-memory-utilization`       | 显存预算             |
| P0  | `block-size`                   | PagedAttention   |
| P0  | `max-num-seqs`                 | 请求并发             |
| P0  | `max-num-batched-tokens`       | 调度预算             |
| P0  | `tensor-parallel-size`         | 张量并行             |
| P0  | `attention-backend`            | Attention Kernel |
| P0  | `enable-prefix-caching`        | Prefix Cache     |
| P1  | `distributed-executor-backend` | Worker 管理        |
| P1  | `all2all-backend`              | MoE 通信           |
| P1  | `preemption-mode`              | KV Cache 不足时的处理  |
| P1  | `compilation-config`           | TorchInductor    |
| P1  | CUDA Graph                     | Kernel Launch 优化 |
| P1  | `load-format`                  | 权重加载             |
| P2  | `chat-template`                | 模型输入协议           |
| P2  | `runner`                       | 生成与 Pooling      |
| P2  | LoRAConfig                     | 动态适配器            |
| P2  | MambaConfig                    | SSM 推理           |

## 十五、对应源码阅读入口

建议后续阅读时不要从 CLI 参数开始一路追，而是沿着配置对象进入具体执行模块。

| 研究内容         | 源码入口                                                                                                                                                    |
| ------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------- |
| CLI 参数注册     | [`arg_utils.py`](https://github.com/neoguojing/vllm/blob/codespace-scaling-halibut-7q499q7wxqcwq57/vllm/engine/arg_utils.py?utm_source=chatgpt.com)     |
| Attention 配置 | [`attention.py`](https://github.com/neoguojing/vllm/blob/codespace-scaling-halibut-7q499q7wxqcwq57/vllm/config/attention.py?utm_source=chatgpt.com)     |
| 并行配置         | [`parallel.py`](https://github.com/neoguojing/vllm/blob/codespace-scaling-halibut-7q499q7wxqcwq57/vllm/config/parallel.py?utm_source=chatgpt.com)       |
| KV Cache 配置  | [`cache.py`](https://github.com/neoguojing/vllm/blob/codespace-scaling-halibut-7q499q7wxqcwq57/vllm/config/cache.py?utm_source=chatgpt.com)             |
| 编译配置         | [`compilation.py`](https://github.com/neoguojing/vllm/blob/codespace-scaling-halibut-7q499q7wxqcwq57/vllm/config/compilation.py?utm_source=chatgpt.com) |
| 环境变量         | [`envs.py`](https://github.com/neoguojing/vllm/blob/codespace-scaling-halibut-7q499q7wxqcwq57/vllm/envs.py?utm_source=chatgpt.com)                      |

对应的源码阅读路径：

```
arg_utils.py
    │
    ▼
EngineArgs
    │
    ▼
VllmConfig
    │
    ├── ModelConfig
    ├── ParallelConfig
    ├── CacheConfig
    ├── AttentionConfig
    ├── CompilationConfig
    ├── SchedulerConfig
    └── ...
    │
    ▼
Engine Core
    │
    ├── Scheduler
    ├── KV Cache Manager
    ├── Model Runner
    ├── Worker
    └── Distributed Executor
    │
    ▼
GPU Kernels / NCCL / CUDA Graph
```

版本准确性说明： 本次重点解释了你列出的配置及其底层技术原理，但部分高级配置的枚举值、字段名称和默认值仍需要在目标分支逐项核实。尤其是 `AttentionConfig`、`KernelConfig`、`Elastic EP`、Safetensors Prefetch 和 `CompilationConfig`，不能把其他 vLLM 版本的配置直接视为该分支支持的配置。前面的代码示例中，明确标注为示意的字段不应未经核实直接用于生产部署。

下一步最有价值的是把这份说明落实成源码级参数字典：每个参数对应 `CLI 名称 → Python 字段 → 类型 → 默认值 → 校验逻辑 → 实际使用位置`，这样才能形成真正完整、可维护的分支配置手册。
