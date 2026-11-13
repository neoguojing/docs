下面整理成一份完整的 vLLM 接口说明文档。按 **接口体系 → 在线 API → 特殊接口 → KV Event → Pooling 与 Cohere → Python SDK → 当前实例核对** 组织。

先明确一个重要边界：文档描述的是 vLLM 这一版本的接口能力；你提供的启动日志则用于确认当前实例实际注册了哪些路由。两者不能混为一谈。

# vLLM 接口体系与使用说明

> 适用范围：`neoguojing/vllm` 分支 `codespace-scaling-halibut-7q499q7wxqcwq57`\
> 目标：梳理 vLLM 的在线 HTTP API、Pooling API、特殊推理接口、基础设施接口、Python SDK 和 CLI。\
> 注意：接口是否可用，最终以当前版本源码、启动参数和服务实际注册路由为准。

---

## 1. vLLM 接口体系总览

vLLM 对外提供多个接口层，它们共享底层推理引擎，但服务对象和使用方式不同。

| 接口层           | 入口                        | 主要用途                 | 使用者           |
| ------------- | ------------------------- | -------------------- | ------------- |
| 在线 HTTP API   | `vllm serve`              | 通过 HTTP 调用模型推理       | 应用、Agent、业务后端 |
| 离线 Python API | `vllm.LLM`                | Python 程序直接加载模型并推理   | 算法工程师、离线任务    |
| CLI           | `vllm`                    | 启动服务、客户端调用、Benchmark | 运维、开发者        |
| 配置 API        | `EngineArgs`、`VllmConfig` | 配置模型、并行、调度、缓存        | 引擎开发者         |
| Engine 内部 API | Engine 类及控制接口             | 请求执行、引擎控制、性能管理       | vLLM 内部组件     |

### 1.1 在线服务内部架构

```
Client / Agent / Application
             |
             | HTTP
             v
       API Router
             |
             v
     Serving Implementation
             |
             | 请求转换、参数处理
             v
        Engine Client
             |
             v
       vLLM Engine
             |
       Scheduler
             |
       Model Runner
             |
       GPU / KV Cache
```

阅读源码时需要区分三层：

1. **Router / Protocol**：定义 HTTP 路由、请求和响应格式。
2. **Serving**：将外部协议转换为 vLLM 内部请求。
3. **Engine**：负责调度、执行模型推理和管理 KV Cache。

不能直接用 Engine 内部 API 的参数定义来推断 HTTP API 的请求协议。

---

## 2. 在线 HTTP API

通过 `vllm serve` 启动服务：

```
vllm serve Qwen/Qwen3-8B \
  --host 0.0.0.0 \
  --port 8100
```

在线 API 可以分为五类：

1. 生成式模型 API
2. Pooling 模型 API
3. 特殊推理 API
4. 服务管理 API
5. 推理基础设施 API

### 2.1 OpenAI Compatible API

vLLM 实现了部分 OpenAI API 协议兼容接口，可以直接使用 OpenAI Python SDK 调用本地服务。

| 方法   | 路径                                   | 功能             | 典型模型            |
| ---- | ------------------------------------ | -------------- | --------------- |
| POST | `/v1/chat/completions`               | 对话生成           | Chat / CausalLM |
| POST | `/v1/chat/completions/batch`         | 批量对话生成         | Chat / CausalLM |
| POST | `/v1/completions`                    | 文本补全           | CausalLM        |
| POST | `/v1/responses`                      | Responses 风格生成 | 支持该协议的生成模型      |
| GET  | `/v1/responses/{response_id}`        | 查询 Response    | Responses       |
| POST | `/v1/responses/{response_id}/cancel` | 取消 Response    | Responses       |
| POST | `/v1/embeddings`                     | 文本向量化          | Embedding       |

注意：这些是接口体系中的常见入口，并不意味着所有模型或所有启动配置都支持全部路由。

#### 2.1.1 Chat Completions

最常见的在线推理接口。

```
curl http://localhost:8100/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model": "Qwen/Qwen3-8B",
    "messages": [
      {"role": "system", "content": "You are a helpful assistant."},
      {"role": "user", "content": "Explain KV Cache."}
    ],
    "temperature": 0.7,
    "max_tokens": 512
  }'
```

Python SDK：

```
from openai import OpenAI

client = OpenAI(
    base_url="http://localhost:8100/v1",
    api_key="EMPTY"
)

response = client.chat.completions.create(
    model="Qwen/Qwen3-8B",
    messages=[
        {"role": "user", "content": "Explain KV Cache."}
    ],
    temperature=0.7,
    max_tokens=512
)

print(response.choices[0].message.content)
```

流式调用：

```
stream = client.chat.completions.create(
    model="Qwen/Qwen3-8B",
    messages=[
        {"role": "user", "content": "Explain vLLM."}
    ],
    stream=True
)

for chunk in stream:
    delta = chunk.choices[0].delta
    if delta.content:
        print(delta.content, end="", flush=True)
```

核心参数：

| 参数            | 含义                  |
| ------------- | ------------------- |
| `model`       | 服务端暴露的模型名称          |
| `messages`    | 对话消息列表              |
| `temperature` | 采样温度                |
| `top_p`       | Nucleus Sampling 参数 |
| `max_tokens`  | 最大生成 Token 数        |
| `stream`      | 是否流式返回              |
| `stop`        | 停止词                 |
| `tools`       | 工具定义                |
| `tool_choice` | 工具选择策略              |

#### 2.1.2 Completions

传统文本补全接口，输入是 Prompt，而不是 Chat Messages。

```
curl http://localhost:8100/v1/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model": "Qwen/Qwen3-8B",
    "prompt": "vLLM is",
    "max_tokens": 64
  }'
```

适合传统 Completion 模型或需要直接控制 Prompt 的应用。

#### 2.1.3 Responses

Responses API 是另一种生成式交互协议，支持通过 Response 对象管理生成结果。

| 接口                                        | 作用          |
| ----------------------------------------- | ----------- |
| `POST /v1/responses`                      | 创建 Response |
| `GET /v1/responses/{response_id}`         | 查询 Response |
| `POST /v1/responses/{response_id}/cancel` | 取消 Response |

它与 Chat Completions 的主要区别在于请求和响应对象的组织方式，以及部分交互能力。是否支持具体功能取决于 vLLM 版本和模型能力。

#### 2.1.4 Batch Chat Completions

```
POST /v1/chat/completions/batch
```

作用：一次请求提交多组独立对话，服务端并发执行，再汇总结果。

适用场景：

- 离线批量问答
- 多条文本分类或抽取任务
- Agent 批量评估
- 批量 Prompt 测试

请求结构示意：

```
{
  "model": "Qwen/Qwen3-8B",
  "conversations": [
    [
      {"role": "user", "content": "What is vLLM?"}
    ],
    [
      {"role": "user", "content": "What is KV Cache?"}
    ]
  ],
  "max_tokens": 128
}
```

返回结果中的 `choices.index` 对应输入中的会话索引。

注意：该分支实现中，Batch Chat 不支持 Streaming、Tool Calling 和 Beam Search。它是一个批量请求入口，不等同于底层连续批处理调度器。

---

## 3. Anthropic Compatible API

vLLM 也提供 Anthropic Messages 风格的兼容入口。

| 方法   | 路径                          | 功能             |
| ---- | --------------------------- | -------------- |
| POST | `/v1/messages`              | 创建消息生成请求       |
| POST | `/v1/messages/count_tokens` | 计算消息输入 Token 数 |

示例：

```
curl http://localhost:8100/v1/messages \
  -H "Content-Type: application/json" \
  -H "anthropic-version: 2023-06-01" \
  -d '{
    "model": "Qwen/Qwen3-8B",
    "max_tokens": 256,
    "messages": [
      {"role": "user", "content": "Explain vLLM."}
    ]
  }'
```

意义：让使用 Anthropic SDK 或 Anthropic 协议的应用可以接入 vLLM，而不必修改为 OpenAI 请求格式。

---

## 4. Cohere Compatible API 与 Pooling 模型

### 4.1 Cohere 是什么？

Cohere 是一家 AI 公司，提供 Embedding、Rerank 和生成式模型等能力。

在 vLLM 的语境中，Cohere API 主要指兼容 Cohere 请求和响应格式的接口协议。

这意味着可以使用 Cohere 风格的请求调用本地部署的兼容模型，不代表请求一定发往 Cohere 云服务。

### 4.2 Cohere 相关接口

| 方法   | 路径           | 功能             | 对应任务      |
| ---- | ------------ | -------------- | --------- |
| POST | `/v2/embed`  | 文本向量化          | Embedding |
| POST | `/rerank`    | 文档相关性排序        | Scoring   |
| POST | `/v1/rerank` | 兼容旧版 Rerank 协议 | Scoring   |
| POST | `/v2/rerank` | 新版 Rerank 协议   | Scoring   |

具体路由是否注册取决于分支实现和服务配置。

### 4.3 Embed 示例

```
curl http://localhost:8100/v2/embed \
  -H "Content-Type: application/json" \
  -d '{
    "model": "intfloat/e5-small",
    "input_type": "search_document",
    "texts": [
      "What is vLLM?",
      "What is KV Cache?"
    ],
    "embedding_types": ["float"]
  }'
```

作用：将文本转换为向量，用于：

- RAG 知识库
- 语义搜索
- 向量数据库检索
- 文本相似度计算

### 4.4 Rerank 示例

```
curl http://localhost:8100/v2/rerank \
  -H "Content-Type: application/json" \
  -d '{
    "model": "BAAI/bge-reranker-base",
    "query": "What is KV Cache?",
    "documents": [
      "KV Cache stores previously computed keys and values.",
      "vLLM is an inference and serving engine.",
      "Redis is an in-memory database."
    ],
    "top_n": 2
  }'
```

返回结构示意：

```
{
  "results": [
    {
      "index": 0,
      "relevance_score": 0.98
    },
    {
      "index": 1,
      "relevance_score": 0.52
    }
  ]
}
```

Rerank 的核心是对 Query-Document 组合进行相关性评分，再根据分数排序。

### 4.5 Pooling 模型接口

Pooling 模型不一定生成自然语言，而是将模型隐藏状态经过 Pooling 或任务特定处理，得到向量、类别或分数。

常见任务：

| 任务              | 输出          | 典型用途     |
| --------------- | ----------- | -------- |
| Embedding       | 文本向量        | RAG、语义检索 |
| Token Embedding | Token 级向量   | 特征抽取     |
| Classification  | 类别或分类分数     | 文本分类     |
| Scoring         | 输入文本或文本对的分数 | 相关性评估    |
| Rerank          | 多个候选文档的排序   | 检索增强     |

常见接口对应关系：

| 接口                                  | 模型任务           | 说明            |
| ----------------------------------- | -------------- | ------------- |
| `/v1/embeddings`                    | Embedding      | OpenAI 风格向量化  |
| `/v2/embed`                         | Embedding      | Cohere 风格向量化  |
| `/classify`                         | Classification | 分类任务          |
| `/score`                            | Scoring        | 评分任务          |
| `/v1/score`                         | Scoring        | 兼容协议入口，视版本而定  |
| `/rerank`、`/v1/rerank`、`/v2/rerank` | Scoring        | Rerank 协议     |
| `/pooling`                          | Pooling        | 通用 Pooling 入口 |

**关键区别：**

- `/v1/embeddings` 和 `/v2/embed` 是 Embedding 专用接口。
- `/rerank` 是面向候选文档排序的协议接口。
- `/pooling` 是通用 Pooling 入口，适合需要更灵活输出的场景。
- 这些接口不能仅凭名称推断所有 Pooling 模型都能使用，必须匹配模型任务类型。

---

## 5. vLLM 自定义推理接口

除了 OpenAI、Anthropic、Cohere 兼容接口，该分支还提供一些 vLLM 自有接口。

### 5.1 `/generative_scoring`

```
POST /generative_scoring
```

**用途：使用 CausalLM 生成模型计算指定 Token 作为下一个 Token 的概率。**

这与普通 Pooling Scoring 不同：它使用生成模型的 Next Token Logits 进行评分。

请求示意：

```
{
  "model": "Qwen/Qwen3-8B",
  "query": "Is this document relevant?",
  "items": [
    "This document discusses KV Cache.",
    "This document discusses cooking."
  ],
  "label_token_ids": [1234, 5678],
  "apply_softmax": true
}
```

主要参数：

| 参数                | 含义                          |
| ----------------- | --------------------------- |
| `query`           | 公共查询文本                      |
| `items`           | 候选文本列表                      |
| `label_token_ids` | 用于评分的标签 Token ID            |
| `item_first`      | 是否将 item 放在 query 前面        |
| `apply_softmax`   | 是否只对指定标签 Token 的分数做 Softmax |

核心计算：

设标签 Token 为 (t_1,t_2)，模型输出的 Next Token Logits 为 (z_1,z_2)，则：

[\
P(t_i)=\frac{e^{z_i}}{\sum_j e^{z_j}}\
]

当只有两个标签时，归一化标签概率为：

[\
P(t_1\mid t_1,t_2)=\
\frac{e^{z_1}}{e^{z_1}+e^{z_2}}\
]

注意：这里的归一化只在指定标签集合内进行，不等同于模型对整个词表的原始概率。

**典型用途：**

- 生成式分类
- 文档相关性评分
- 基于标签 Token 的判别
- 利用生成模型实现轻量级决策

注意：`label_token_ids` 应通过实际模型 Tokenizer 获取，不能随意填写。

### 5.2 `/v1/systemone`

```
POST /v1/systemone
```

**用途：通过生成模型的 Next Token 概率实现结构化决策。**

它将模型的生成式能力封装成结构化输出，避免每次都要求模型生成完整自然语言。

主要决策类型：

| 类型       | 含义            | 使用场景      |
| -------- | ------------- | --------- |
| `noul`   | 无标签约束或自由决策类判断 | 开放式判断     |
| `choice` | 在候选选项中选择      | 意图路由、工具选择 |
| `score`  | 对候选项进行评分      | 相关性、优先级判断 |

请求结构示意：

```
{
  "model": "Qwen/Qwen3-8B",
  "state": "用户想查询订单物流状态",
  "questions": [
    {
      "type": "choice",
      "question": "应该调用哪个工具？",
      "choices": [
        "query_order",
        "cancel_order",
        "refund_order"
      ]
    }
  ]
}
```

实际请求字段应以该分支接口定义为准。

典型 Agent 场景：

```
用户请求
   |
   v
/v1/systemone
   |
   v
意图/决策判断
   |
   +---- query_order
   |
   +---- cancel_order
   |
   +---- refund_order
   |
   v
Agent 执行对应工具
```

与 `/generative_scoring` 的区别：

| 对比   | `/generative_scoring`           | `/v1/systemone`   |
| ---- | ------------------------------- | ----------------- |
| 核心能力 | 指定 Token 概率评分                   | 结构化决策             |
| 输入   | Query + Items + Label Token IDs | State + Questions |
| 输出目标 | 评分结果                            | 决策结果              |
| 典型场景 | 相关性、生成式分类                       | Agent 路由、结构化判断    |
| 模型类型 | CausalLM                        | 支持该能力的文本生成模型      |

---

## 6. 批量与弹性扩缩容接口

### 6.1 `/scale_elastic_ep`

```
POST /scale_elastic_ep
```

**用途：动态调整 MoE Elastic Expert Parallelism 的 Data Parallel Engine 数量。**

它不是普通的增加 GPU 显存或扩展模型副本，而是调整启用了 Elastic EP 的推理集群中 DP Engine 的规模。

请求：

```
curl -X POST http://localhost:8100/scale_elastic_ep \
  -H "Content-Type: application/json" \
  -d '{
    "new_data_parallel_size": 4,
    "drain_timeout": 120
  }'
```

参数：

| 参数                       | 含义                    |
| ------------------------ | --------------------- |
| `new_data_parallel_size` | 目标 DP Size，必填正整数      |
| `drain_timeout`          | 缩容等操作的排空超时时间，默认 120 秒 |

示意：

```
扩容前：DP Size = 2
       |
       v
POST /scale_elastic_ep
new_data_parallel_size = 4
       |
       v
扩容后：DP Size = 4
```

启动时需要相应的 Elastic EP 配置，例如：

```
--enable-expert-parallel \
--enable-elastic-ep \
--enable-eplb \
--data-parallel-backend ray
```

具体启动参数和后端要求以当前分支配置为准。

常见错误响应：

| HTTP 状态码 | 含义      |
| -------- | ------- |
| 400      | 参数不合法   |
| 408      | 扩缩容操作超时 |
| 500      | 内部异常    |

### 6.2 `/is_scaling_elastic_ep`

```
POST /is_scaling_elastic_ep
```

用途：查询当前是否正在执行 Elastic EP 扩缩容，以及相关状态信息。

它与 `/scale_elastic_ep` 配合使用：

- `/scale_elastic_ep`：发起扩缩容。
- `/is_scaling_elastic_ep`：检查扩缩容状态。

---

## 7. 服务管理与监控接口

这些接口不直接执行模型生成，而是提供服务状态、模型信息和运行指标。

| 方法   | 路径                  | 功能                 |
| ---- | ------------------- | ------------------ |
| GET  | `/health`           | 健康检查               |
| GET  | `/version`          | 查询 vLLM 版本         |
| GET  | `/load`             | 查询服务负载             |
| GET  | `/metrics`          | Prometheus 监控指标    |
| GET  | `/v1/models`        | 查询服务暴露的模型          |
| GET  | `/kv_event_sources` | 查询 KV Event Source |
| POST | `/tokenize`         | 文本 Tokenization    |
| POST | `/detokenize`       | Token ID 转文本       |
| GET  | `/openapi.json`     | OpenAPI Schema     |
| GET  | `/docs`             | Swagger 文档         |
| GET  | `/redoc`            | ReDoc 文档           |

### 7.1 `/health`

```
curl http://localhost:8100/health
```

检查服务是否健康。

常用于：

- Kubernetes Liveness Probe
- Kubernetes Readiness Probe
- 负载均衡器健康检查
- 服务部署后的自动检测

### 7.2 `/version`

```
curl http://localhost:8100/version
```

查询服务端 vLLM 版本，便于排查客户端与服务端版本差异。

### 7.3 `/load`

```
curl http://localhost:8100/load
```

返回结构示意：

```
{
  "server_load": 5
}
```

`server_load` 表示服务负载计数，不是 GPU 利用率百分比。

可供外部负载均衡器参考，但不能直接将它理解为 GPU 使用率。

### 7.4 `/metrics`

```
curl http://localhost:8100/metrics
```

输出 Prometheus 格式的监控数据。

重点关注：

- Running Requests：正在运行的请求数量。
- Waiting Requests：等待调度的请求数量。
- Token Throughput：Token 吞吐量。
- Request Latency：请求延迟。
- KV Cache Usage：KV Cache 使用情况。

适合接入 Prometheus + Grafana。

### 7.5 `/v1/models`

```
curl http://localhost:8100/v1/models
```

查询服务暴露的模型名称和模型信息。客户端请求中的 `model` 通常需要与服务端暴露的模型名称一致。

### 7.6 `/tokenize` 与 `/detokenize`

`/tokenize`：文本转 Token ID。

`/detokenize`：Token ID 转文本。

它们适合：

- Prompt 长度预估
- Tokenizer 调试
- 上下文窗口检查
- Agent 请求构造

### 7.7 `/kv_event_sources`

```
GET /kv_event_sources
```

**用途：查询 KV Cache Event 发布端点。**

它不是查询 KV Cache 里的 Tensor 内容，而是查询外部组件订阅 KV Event 所需的事件源信息。

开启 KV Event 后，vLLM 可以发布 KV Cache 变化事件，例如：

| 事件                 | 含义           |
| ------------------ | ------------ |
| `BlockStored`      | KV Block 被存储 |
| `BlockRemoved`     | KV Block 被移除 |
| `AllBlocksCleared` | KV Cache 被清空 |

整体关系：

```
vLLM Worker
    |
    | KV Cache 状态变化
    v
KV Event Publisher
    |
    | ZMQ PUB
    v
Router / Observer
    |
    | 维护 KV Block 索引
    v
KV-aware Routing
```

调用方式：

```
curl http://localhost:8100/kv_event_sources
```

接口返回的具体字段和地址以当前版本的实际响应为准。通常需要关注：

- DP Rank：事件源属于哪个 Data Parallel Rank。
- 发布端点：订阅实时 KV Event 的地址。
- 回放端点：如果该版本提供回放能力，用于补齐历史事件。
- 事件配置：例如 Topic、缓存或队列相关配置。

注意：不能把某个示例中的端口、Topic 或字段名称当成固定协议，应该直接检查当前实例的 JSON 响应和对应源码。

外部 Router 的典型流程：

1. 调用 `/kv_event_sources` 获取事件源信息。
2. 根据 DP Rank 建立 ZMQ SUB 订阅。
3. 接收 KV Block 创建、删除等事件。
4. 维护 Worker 与 KV Block 的对应关系。
5. 新请求到达时，计算 Prefix Cache 命中情况。
6. 优先选择缓存命中较高的 Worker，减少重复 Prefill 计算。

**接口定位：HTTP 服务发现 + ZMQ 事件通信。**

`/kv_event_sources` 本身只是 HTTP 查询接口，不负责传输实时 KV Event。

---

## 8. SageMaker 适配接口

当前启动日志中还有：

| 方法         | 路径             | 功能                 |
| ---------- | -------------- | ------------------ |
| GET / POST | `/ping`        | SageMaker 健康检查相关入口 |
| POST       | `/invocations` | SageMaker 推理调用入口   |

这类接口用于适配 AWS SageMaker 部署和调用规范。

`/invocations` 是平台适配入口，不是 OpenAI API 的一部分。

---

## 9. 离线 Python SDK

在线 API 适合远程 HTTP 调用；离线 Python API 则直接在 Python 进程中加载模型并执行推理。

主要入口：

```
from vllm import LLM
```

### 9.1 LLM 类

初始化：

```
from vllm import LLM

llm = LLM(
    model="Qwen/Qwen3-8B",
    tensor_parallel_size=1,
    gpu_memory_utilization=0.85
)
```

常用参数：

| 参数                       | 作用                      |
| ------------------------ | ----------------------- |
| `model`                  | 模型路径或 Hugging Face 模型名称 |
| `tensor_parallel_size`   | Tensor Parallel 大小      |
| `pipeline_parallel_size` | Pipeline Parallel 大小    |
| `gpu_memory_utilization` | GPU 显存利用率目标             |
| `max_model_len`          | 最大上下文长度                 |
| `dtype`                  | 模型数据类型                  |
| `trust_remote_code`      | 是否允许加载模型仓库自定义代码         |

### 9.2 生成模型接口

| 方法           | 用途        |
| ------------ | --------- |
| `generate()` | Prompt 补全 |
| `chat()`     | 对话生成      |

示例：

```
from vllm import LLM, SamplingParams

llm = LLM(model="Qwen/Qwen3-8B")

sampling_params = SamplingParams(
    temperature=0.7,
    max_tokens=128
)

outputs = llm.generate(
    ["Explain KV Cache."],
    sampling_params
)

for output in outputs:
    print(output.outputs[0].text)
```

### 9.3 Pooling 模型接口

| 方法           | 用途                |
| ------------ | ----------------- |
| `embed()`    | Embedding         |
| `classify()` | Classification    |
| `score()`    | Scoring           |
| `encode()`   | 更通用的 Pooling 编码入口 |

这些方法用于非传统文本生成任务，具体方法是否可用取决于模型任务类型和版本实现。

### 9.4 参数对象

| 对象               | 作用           |
| ---------------- | ------------ |
| `SamplingParams` | 生成模型采样参数     |
| `PoolingParams`  | Pooling 任务参数 |
| `EngineArgs`     | 引擎启动参数       |
| `VllmConfig`     | 引擎运行配置       |

可以将它们理解为：

```
LLM
 |
 +-- SamplingParams  -> 控制文本生成
 |
 +-- PoolingParams   -> 控制向量/评分等任务
 |
 +-- EngineArgs      -> 初始化参数
 |
 +-- VllmConfig      -> 引擎运行配置
```

### 9.5 Engine 控制接口

vLLM 的部分 Engine 实现还提供控制能力，例如：

| 接口                     | 作用              |
| ---------------------- | --------------- |
| `sleep()`              | 使引擎进入休眠状态       |
| `wake_up()`            | 唤醒引擎            |
| `start_profile()`      | 开始性能分析          |
| `stop_profile()`       | 停止性能分析          |
| `reset_prefix_cache()` | 重置 Prefix Cache |
| `reset_mm_cache()`     | 重置多模态缓存         |
| `get_metrics()`        | 获取引擎指标          |
| `collective_rpc()`     | 向执行进程发起集体 RPC   |

这些属于版本相关的 Engine 控制 API，不应当默认视为所有 `LLM` 实例都公开的稳定接口。使用前应检查对应类的 API Reference 和当前版本源码。

---

## 10. CLI 接口

CLI 主要负责配置解析、组件初始化和命令执行。

常用命令：

```
# 查看总帮助
vllm --help

# 启动服务
vllm serve Qwen/Qwen3-8B

# 查看 serve 参数
vllm serve --help

# 离线推理/客户端相关命令
vllm chat --help

# Benchmark
vllm bench --help
```

具体子命令以本地安装版本的 `vllm --help` 为准。

CLI 不是推理核心，而是启动和管理入口。

---

## 11. 当前实例接口核对

根据当前提供的启动日志，服务监听：

```
http://0.0.0.0:8100
```

已注册路由如下：

| 方法         | 路径                                   | 分类                 |
| ---------- | ------------------------------------ | ------------------ |
| GET / HEAD | `/openapi.json`                      | API Schema         |
| GET / HEAD | `/docs`                              | Swagger            |
| GET / HEAD | `/docs/oauth2-redirect`              | Swagger            |
| GET / HEAD | `/redoc`                             | ReDoc              |
| GET        | `/load`                              | 监控                 |
| GET        | `/version`                           | 版本                 |
| GET        | `/kv_event_sources`                  | KV Event           |
| GET        | `/health`                            | 健康检查               |
| GET        | `/metrics`                           | Prometheus         |
| POST       | `/tokenize`                          | Tokenizer          |
| POST       | `/detokenize`                        | Tokenizer          |
| GET        | `/v1/models`                         | 模型查询               |
| GET / POST | `/ping`                              | SageMaker          |
| POST       | `/invocations`                       | SageMaker          |
| POST       | `/v1/chat/completions`               | OpenAI Chat        |
| POST       | `/v1/chat/completions/batch`         | Batch Chat         |
| POST       | `/v1/responses`                      | OpenAI Responses   |
| GET        | `/v1/responses/{response_id}`        | Responses 查询       |
| POST       | `/v1/responses/{response_id}/cancel` | Responses 取消       |
| POST       | `/v1/completions`                    | OpenAI Completion  |
| POST       | `/v1/messages`                       | Anthropic Messages |
| POST       | `/v1/messages/count_tokens`          | Anthropic Token 统计 |
| POST       | `/generative_scoring`                | vLLM 自定义评分         |
| POST       | `/v1/systemone`                      | vLLM 自定义决策         |
| POST       | `/scale_elastic_ep`                  | Elastic EP 扩缩容     |
| POST       | `/is_scaling_elastic_ep`             | Elastic EP 状态      |

### 11.1 当前日志中没有显示的接口

| 接口               | 当前日志是否确认注册 |
| ---------------- | ---------- |
| `/v1/embeddings` | 否          |
| `/v2/embed`      | 否          |
| `/v2/rerank`     | 否          |
| `/rerank`        | 否          |
| `/pooling`       | 否          |
| `/classify`      | 否          |
| `/score`         | 否          |

这不代表 vLLM 整个版本不支持，而是当前这份日志没有显示它们。

可以直接通过以下方式检查运行实例：

```
# 查看全部路由
curl http://localhost:8100/openapi.json

# 查看 Swagger
# 浏览器访问：
http://localhost:8100/docs

# 查看模型
curl http://localhost:8100/v1/models

# 查看服务健康
curl http://localhost:8100/health
```

---

## 12. 接口选型总结

| 需求                 | 推荐入口                           |
| ------------------ | ------------------------------ |
| Agent 对话生成         | `/v1/chat/completions`         |
| 传统 Prompt 补全       | `/v1/completions`              |
| Responses 协议应用     | `/v1/responses`                |
| 多条独立对话批量执行         | `/v1/chat/completions/batch`   |
| 文本向量化              | `/v1/embeddings` 或 `/v2/embed` |
| 文档相关性排序            | `/v2/rerank`                   |
| 分类/评分              | `/classify`、`/score`           |
| 通用 Pooling 推理      | `/pooling`                     |
| 生成式 Token 概率评分     | `/generative_scoring`          |
| Agent 结构化决策        | `/v1/systemone`                |
| KV Cache 状态感知      | `/kv_event_sources` + ZMQ      |
| MoE Elastic EP 扩缩容 | `/scale_elastic_ep`            |
| 服务健康检查             | `/health`                      |
| 服务负载               | `/load`                        |
| 性能监控               | `/metrics`                     |
| 离线 Python 推理       | `vllm.LLM`                     |
| 服务启动与 Benchmark    | `vllm` CLI                     |

## 13. 源码阅读建议

按以下顺序阅读，可以从外部接口逐步进入推理引擎：

1. **接口总览**：`docs/zh/interfaces.md`
2. **服务启动**：`vllm serve` CLI 入口
3. **HTTP 路由**：OpenAI / Anthropic / 自定义 API Router
4. **Serving 实现**：`OpenAIServingChat`、`OpenAIServingChatBatch` 等
5. **Engine Client**：HTTP 请求如何转换为 Engine 请求
6. **Engine**：请求调度、执行和 KV Cache 管理
7. **Model Runner**：模型前向计算和 GPU 执行

重点建立下面这条调用链：

```
HTTP Request
     |
     v
API Router
     |
     v
Serving Layer
     |
     v
Engine Client
     |
     v
Engine / Scheduler
     |
     v
Model Runner
     |
     v
GPU Execution
     |
     v
Response
```

**核心认识：**

vLLM 的接口不只是模型推理 API。它同时提供了协议兼容层、模型任务接口、服务运维接口和推理基础设施接口。

对 Agent 系统开发而言，最值得重点掌握的是：

- `/v1/chat/completions`：模型生成入口。
- `/v1/chat/completions/batch`：批量推理入口。
- `/generative_scoring` 和 `/v1/systemone`：结构化评分和决策能力。
- `/metrics`、`/load`：服务监控和负载感知。
- `/kv_event_sources`：KV Cache 事件发现和缓存感知路由。
- Python `LLM` 与 Engine API：理解服务层下面的推理执行过程。



参考源码与文档：

- [分支接口总览 `interfaces.md`](https://github.com/neoguojing/vllm/blob/codespace-scaling-halibut-7q499q7wxqcwq57/docs/zh/interfaces.md?utm_source=chatgpt.com)&#x20;

  [image](https://www.google.com/s2/favicons?domain=https://github.com\&sz=32)

  GitHub

- [分支代码：Batch Chat 实现](https://github.com/neoguojing/vllm/blob/codespace-scaling-halibut-7q499q7wxqcwq57/vllm/entrypoints/openai/batch_serving.py?utm_source=chatgpt.com)

- [分支代码：Elastic EP 路由](https://github.com/neoguojing/vllm/blob/codespace-scaling-halibut-7q499q7wxqcwq57/vllm/entrypoints/...?utm_source=chatgpt.com)

- [Generative Scoring 文档](https://github.com/neoguojing/vllm/blob/codespace-scaling-halibut-7q499q7wxqcwq57/docs/...?utm_source=chatgpt.com)

**说明：** 文档中的示例 JSON 是帮助理解协议的示意数据，不保证所有字段在该分支中完全一致。尤其是 Cohere、Pooling 路由，应以 `/openapi.json` 和当前模型任务类型为准。
