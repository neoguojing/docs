# **vLLM VllmConfig 配置清单及说明（Gemma4 31B）**

以下基于你提供的实际启动日志整理。每项包含 **配置名称、实际值、配置作用**，保留 vLLM 中的原始配置名称，方便对照源码和断点调试。

## **1. ModelConfig：模型配置**

负责定义模型路径、模型结构加载方式、计算精度、最大上下文长度以及模型实现方式。

| 配置项                  | 实际值                | 配置说明                               |
| -------------------- | ------------------ | ---------------------------------- |
| model                | Gemma4 本地模型路径      | 模型权重和配置文件所在目录                      |
| model_weights        | 空字符串               | 未单独指定权重路径                          |
| runner               | auto               | 自动确定模型运行任务类型                       |
| convert              | auto               | 自动选择模型转换方式                         |
| tokenizer            | Gemma4 本地模型路径      | Tokenizer 文件路径                     |
| tokenizer_mode       | auto               | 自动选择 Tokenizer 实现                  |
| trust_remote_code    | False              | 不允许执行模型仓库中的远程自定义代码                 |
| dtype                | float16            | 模型计算使用 FP16 精度                     |
| seed                 | 0                  | 随机数种子，用于控制随机行为                     |
| max_model_len        | 16000              | 本次运行允许的最大序列长度                      |
| quantization         | compressed-tensors | 使用 Compressed Tensors 量化实现         |
| model_impl           | auto               | 自动选择 vLLM 或其他模型实现                  |
| enforce_eager        | False              | 不强制使用 Eager 模式，允许使用 CUDA Graph 等优化 |
| served_model_name    | debug-model        | 对外提供服务时使用的模型名称                     |
| config_format        | auto               | 自动选择模型配置文件格式                       |
| generation_config    | auto               | 自动加载模型目录中的生成配置                     |
| skip_tokenizer_init  | False              | 正常初始化 Tokenizer                    |
| enable_sleep_mode    | False              | 不启用 Engine Sleep 模式                |
| enable_prompt_embeds | False              | 不启用 Prompt Embeddings 输入           |

## **2. HF Model Config：模型结构配置**

负责描述 Gemma4 Transformer、Attention、FFN、位置编码等模型结构参数。

### **2.1 文本模型结构**

| 配置项                     | 实际值                            | 配置说明                       |
| ----------------------- | ------------------------------ | -------------------------- |
| model_type              | gemma4_text                    | 文本模型类型                     |
| architectures           | Gemma4ForConditionalGeneration | 模型架构入口                     |
| hidden_size             | 5376                           | Transformer 隐藏层维度          |
| num_hidden_layers       | 60                             | Transformer 层数             |
| num_attention_heads     | 32                             | Query Attention Head 数量    |
| num_key_value_heads     | 16                             | KV Head 数量，采用 GQA 结构       |
| head_dim                | 256                            | 普通 Attention 层的 Head 维度    |
| intermediate_size       | 21504                          | FFN 中间层维度                  |
| vocab_size              | 262144                         | Token 词表大小                 |
| max_position_embeddings | 262144                         | 模型位置编码支持的最大长度              |
| sliding_window          | 1024                           | Sliding Attention 的局部窗口大小  |
| rms_norm_eps            | 1e-6                           | RMSNorm 数值稳定性参数            |
| hidden_activation       | gelu_pytorch_tanh              | FFN 激活函数                   |
| attention_bias          | False                          | Attention Linear 不使用 Bias  |
| attention_dropout       | 0.0                            | Attention Dropout 比例为 0    |
| use_cache               | True                           | 模型支持 KV Cache              |
| tie_word_embeddings     | True                           | 输入 Embedding 和输出权重共享       |
| final_logit_softcapping | 30.0                           | 对最终 Logits 进行 Soft Capping |

### **2.2 特殊 Attention 层配置**

`per_layer_config` 指定部分层使用不同的 KV Head 和 Head Dimension。

| Layer | head_dim | num_key_value_heads | 说明              |
| ----- | -------- | ------------------- | --------------- |
| 5     | 512      | 4                   | 特殊 Attention 配置 |
| 11    | 512      | 4                   | 特殊 Attention 配置 |
| 17    | 512      | 4                   | 特殊 Attention 配置 |
| 23    | 512      | 4                   | 特殊 Attention 配置 |
| 29    | 512      | 4                   | 特殊 Attention 配置 |
| 35    | 512      | 4                   | 特殊 Attention 配置 |
| 41    | 512      | 4                   | 特殊 Attention 配置 |
| 47    | 512      | 4                   | 特殊 Attention 配置 |
| 53    | 512      | 4                   | 特殊 Attention 配置 |
| 59    | 512      | 4                   | 特殊 Attention 配置 |

### **2.3 Attention 类型及 RoPE**

| 配置项                                  | 实际值                      | 配置说明                        |
| ------------------------------------ | ------------------------ | --------------------------- |
| layer_types                          | Sliding / Full Attention | 每层指定不同 Attention 类型         |
| full_attention.rope_theta            | 1000000                  | Full Attention 的 RoPE 基数    |
| full_attention.rope_type             | proportional             | Full Attention 的位置编码类型      |
| full_attention.partial_rotary_factor | 0.25                     | 旋转位置编码维度占比                  |
| sliding_attention.rope_theta         | 10000                    | Sliding Attention 的 RoPE 基数 |
| sliding_attention.rope_type          | default                  | Sliding Attention 的位置编码类型   |

## **3. QuantizationConfig：量化配置**

负责控制模型权重的量化格式、位宽、分组方式和量化对象。

| 配置项                 | 实际值                | 配置说明                                    |
| ------------------- | ------------------ | --------------------------------------- |
| quant_method        | compressed-tensors | 使用 Compressed Tensors 量化框架              |
| format              | pack-quantized     | 权重采用打包量化格式                              |
| quantization_status | compressed         | 权重处于压缩量化状态                              |
| version             | 0.14.1.a20260326   | 量化配置版本                                  |
| num_bits            | 4                  | 权重量化位宽为 4bit                            |
| group_size          | 32                 | 每组包含 32 个权重                             |
| strategy            | group              | 按分组进行量化                                 |
| type                | int                | 整数量化                                    |
| symmetric           | False              | 使用非对称量化                                 |
| observer            | mse                | 使用 MSE 观察策略                             |
| zp_dtype            | torch.int8         | Zero Point 数据类型                         |
| targets             | Linear             | 主要量化 Linear 层                           |
| dynamic             | False              | 不采用动态量化配置                               |
| ignore              | 指定模块列表             | 指定不参与量化的模块，包括部分 Vision Tower 层和 lm_head |

## **4. MultiModalConfig：多模态配置**

负责图像、视频等多模态输入的处理、缓存、Encoder 和数据传输。

| 配置项                             | 实际值        | 配置说明                        |
| ------------------------------- | ---------- | --------------------------- |
| language_model_only             | False      | 不限制为纯语言模型                   |
| limit_per_prompt.video          | 0          | 每个请求允许处理的视频数量为 0            |
| enable_mm_embeds                | False      | 不允许外部直接传入多模态 Embeddings     |
| mm_processor_cache_gb           | 4.0        | 多模态 Processor 缓存容量          |
| mm_processor_cache_type         | lru        | 使用 LRU 缓存策略                 |
| mm_hasher_algorithm             | blake3     | 多模态缓存使用 BLAKE3 Hash         |
| mm_shm_cache_max_object_size_mb | 128        | 共享内存缓存单个对象大小上限              |
| mm_encoder_only                 | False      | 不限制为仅 Encoder 模式            |
| mm_encoder_tp_mode              | weights    | 多模态 Encoder 的 TP 模式         |
| mm_encoder_attn_backend         | None       | 未单独指定多模态 Attention 后端       |
| mm_encoder_attn_dtype           | None       | 未单独指定多模态 Attention 精度       |
| interleave_mm_strings           | False      | 不启用多模态字符串交错处理               |
| skip_mm_profiling               | False      | 不跳过多模态 Profiling            |
| mm_tensor_ipc                   | direct_rpc | 多模态 Tensor 使用 Direct RPC 传输 |

## **5. CacheConfig：KV Cache 配置**

负责 GPU KV Cache 内存预算、Block 分配、Prefix Cache 和 KV Cache 精度管理。

| 配置项                             | 实际值    | 配置说明                            |
| ------------------------------- | ------ | ------------------------------- |
| block_size                      | 16     | 每个 KV Cache Block 存储 16 个 Token |
| gpu_memory_utilization          | 0.9    | GPU 显存使用目标比例为 90%               |
| cache_dtype                     | auto   | 自动确定 KV Cache 数据类型              |
| is_attention_free               | False  | 当前模型不是 Attention-Free 模型        |
| num_gpu_blocks_override         | None   | 不手动覆盖 GPU Block 数量              |
| enable_prefix_caching           | True   | 开启 Prefix Cache                 |
| prefix_caching_hash_algo        | sha256 | Prefix Cache 使用 SHA256 Hash     |
| prefix_cache_retention_interval | 0      | Prefix Cache 保留时间间隔配置           |
| kv_cache_dtype_skip_layers      | []     | 没有指定跳过 KV Cache 精度设置的层          |
| num_gpu_blocks                  | 2454   | 实际 GPU KV Cache Block 数量        |
| num_cpu_blocks                  | None   | 未单独指定 CPU KV Cache Block 数量     |
| kv_cache_size_tokens            | 16462  | 日志记录的 KV Cache Token 容量         |
| kv_cache_max_concurrency        | 1.0289 | KV Cache 容量对应的估算并发指标            |
| disable_sliding_window          | False  | 保留 Sliding Window 机制            |
| disable_cascade_attn            | True   | 禁用 Cascade Attention            |
| kv_sharing_fast_prefill         | False  | 不启用 KV Sharing Fast Prefill     |
| kv_cache_memory_bytes           | None   | 未直接指定 KV Cache 字节数              |
| kv_offloading_size              | None   | 未单独指定 KV Offload 容量             |
| kv_offloading_backend           | native | 使用 Native KV Offload 后端         |

## **6. ParallelConfig：并行配置**

负责模型在 GPU、进程和节点之间的并行部署与通信。

| 配置项                                 | 实际值        | 配置说明                          |
| ----------------------------------- | ---------- | ----------------------------- |
| tensor_parallel_size                | 1          | TP=1，不进行 Tensor Parallel 切分   |
| pipeline_parallel_size              | 1          | PP=1，不进行 Pipeline Parallel 切分 |
| data_parallel_size                  | 1          | DP=1，只有一个数据并行副本               |
| data_parallel_size_local            | 1          | 本地 DP 数量为 1                   |
| data_parallel_rank                  | 0          | 当前 DP 全局 Rank                 |
| data_parallel_rank_local            | 0          | 当前 DP 本地 Rank                 |
| prefill_context_parallel_size       | 1          | Prefill Context Parallel 数量   |
| decode_context_parallel_size        | 1          | Decode Context Parallel 数量    |
| nnodes                              | 1          | 使用一个计算节点                      |
| node_rank                           | 0          | 当前节点 Rank                     |
| world_size                          | 1          | 分布式总进程数                       |
| rank                                | 0          | 当前进程 Rank                     |
| distributed_executor_backend        | uni        | 使用单进程 Executor                |
| worker_cls                          | GPU Worker | GPU Worker 实现类                |
| data_parallel_backend               | mp         | DP 使用多进程通信后端                  |
| data_parallel_master_ip             | 127.0.0.1  | DP 主节点 IP                     |
| data_parallel_rpc_port              | 29550      | DP RPC 通信端口                   |
| master_addr                         | 127.0.0.1  | 分布式通信 Master 地址               |
| master_port                         | 29501      | 分布式通信端口                       |
| disable_custom_all_reduce           | False      | 不禁用自定义 AllReduce              |
| disable_nccl_for_dp_synchronization | True       | DP 同步不使用 NCCL                 |
| enable_expert_parallel              | False      | 不启用 Expert Parallel           |
| enable_eplb                         | False      | 不启用 Expert Load Balancing     |
| enable_dbo                          | False      | 不启用 Dual Batch Overlap        |
| enable_elastic_ep                   | False      | 不启用弹性 Expert Parallel         |
| enable_fault_tolerance              | False      | 不启用分布式容错                      |

## **7. SchedulerConfig：调度配置**

负责请求队列、Batch 组织、Token 预算、Prefill 和 Decode 调度。

| 配置项                             | 实际值      | 配置说明                            |
| ------------------------------- | -------- | ------------------------------- |
| runner_type                     | generate | 使用文本生成模式                        |
| max_num_batched_tokens          | 2496     | 单次调度迭代的最大 Token 预算              |
| max_num_scheduled_tokens        | None     | 不单独指定调度 Token 上限                |
| max_num_seqs                    | 256      | 单次调度允许的最大序列数                    |
| long_prefill_token_threshold    | 0        | 未设置长 Prefill Token 阈值           |
| max_num_queued_reqs             | None     | 不单独限制排队请求数量                     |
| max_num_queued_tokens           | None     | 不单独限制排队 Token 数量                |
| enable_chunked_prefill          | True     | 开启分块 Prefill                    |
| is_multimodal_model             | True     | 标记为多模态模型                        |
| max_num_encoder_input_tokens    | 2496     | Encoder 输入 Token 预算             |
| encoder_cache_size              | 2496     | Encoder Cache 容量配置              |
| policy                          | fcfs     | 采用 First Come First Served 调度策略 |
| disable_chunked_mm_input        | True     | 禁用分块多模态输入                       |
| disable_hybrid_kv_cache_manager | False    | 不禁用 Hybrid KV Cache Manager     |
| scheduler_reserve_full_isl      | True     | 为完整输入序列长度预留相关调度资源               |
| watermark                       | 0.0      | KV Cache 调度水位参数                 |
| prefill_schedule_interval       | 1        | Prefill 调度间隔                    |
| async_scheduling                | True     | 启用异步调度                          |
| stream_interval                 | 1        | 流式输出间隔为 1                       |

## **8. DeviceConfig：设备配置**

负责指定模型运行设备及底层设备类型。

| 配置项         | 实际值  | 配置说明           |
| ----------- | ---- | -------------- |
| device      | cuda | 使用 CUDA 设备     |
| device_type | cuda | GPU 设备类型为 CUDA |

## **9. LoadConfig：模型加载配置**

负责模型权重加载格式、文件读取、预取和加载过程中的设备映射。

| 配置项                              | 实际值              | 配置说明                   |
| -------------------------------- | ---------------- | ---------------------- |
| load_format                      | auto             | 自动选择权重加载格式             |
| download_dir                     | None             | 未指定模型下载目录              |
| safetensors_load_strategy        | None             | 未指定特殊 Safetensors 加载策略 |
| safetensors_prefetch_num_threads | 8                | Safetensors 文件预取线程数    |
| safetensors_prefetch_block_size  | 16777216         | 预取块大小为 16MB            |
| model_loader_extra_config        |                  | 没有额外模型加载配置             |
| ignore_patterns                  | original/\*\*/\* | 忽略匹配的权重文件路径            |
| use_tqdm_on_load                 | True             | 加载过程中显示进度条             |
| pt_load_map_location             | cpu              | PyTorch 权重加载时映射到 CPU   |

## **10. OffloadConfig：权重卸载配置**

负责将部分模型权重放在 CPU 内存中，并通过预取机制在需要时传输到 GPU。

| 配置项                   | 实际值                   | 配置说明                   |
| --------------------- | --------------------- | ---------------------- |
| offload_backend       | prefetch              | 使用 Prefetch 权重卸载后端     |
| cpu_offload_gb        | 12.0                  | 配置 12GB CPU Offload 容量 |
| offload_group_size    | 8                     | Offload 分组大小           |
| offload_num_in_group  | 4                     | 每组参与 Offload 的参数数量配置   |
| offload_prefetch_step | 1                     | 预取步长                   |
| offload_params        | set()                 | 未指定额外的 Offload 参数集合    |
| uva                   | UVAOffloadConfig      | UVA 相关卸载配置             |
| prefetch              | PrefetchOffloadConfig | Prefetch 相关卸载配置        |

## **11. AttentionConfig：Attention 后端配置**

负责指定 Attention 的计算后端，以及不同 Attention 实现的相关参数。

| 配置项                                      | 实际值         | 配置说明                                  |
| ---------------------------------------- | ----------- | ------------------------------------- |
| backend                                  | TRITON_ATTN | 使用 Triton Attention Backend           |
| minimax_m3_msa_decode_backend            | triton      | Minimax MSA Decode 使用 Triton          |
| backend_per_kind                         |             | 没有针对不同 Attention 类型单独指定后端             |
| flash_attn_version                       | None        | 未指定 FlashAttention 版本                 |
| flash_attn_max_num_splits_for_cuda_graph | 32          | FlashAttention CUDA Graph 最大 Split 配置 |
| tq_max_kv_splits_for_cuda_graph          | 32          | TQ KV CUDA Graph 最大 Split 配置          |
| use_trtllm_attention                     | None        | 未指定 TensorRT-LLM Attention            |
| disable_flashinfer_q_quantization        | False       | 不禁用 FlashInfer Query 量化               |
| mla_prefill_backend                      | None        | 未指定 MLA Prefill 后端                    |
| use_prefill_query_quantization           | False       | 不启用 Prefill Query 量化                  |
| indexer_kv_dtype                         | auto        | Indexer KV 数据类型自动选择                   |
| use_non_causal                           | False       | 不使用非因果 Attention                      |
| sparse_mla_force_mqa                     | False       | 不强制使用 MQA                             |
| flex_attn_block_m                        | None        | 未指定 Flex Attention M 维度 Block         |
| flex_attn_block_n                        | None        | 未指定 Flex Attention N 维度 Block         |

## **12. KernelConfig：Kernel 算子配置**

负责选择底层算子实现、自动调优和 JIT 预热。

| 配置项                         | 实际值  | 配置说明                        |
| --------------------------- | ---- | --------------------------- |
| enable_flashinfer_autotune  | True | 开启 FlashInfer 自动调优          |
| enable_cutedsl_warmup       | True | 开启 CuteDSL Kernel 预热        |
| enable_jit_warmup           | True | 开启 JIT Kernel 预热            |
| moe_backend                 | auto | 自动选择 MoE 算子后端               |
| sparse_indexer_topk_backend | auto | 自动选择 Sparse Indexer TopK 后端 |
| linear_backend              | auto | 自动选择 Linear 算子后端            |
| linear_backend_per_quant    | None | 不针对不同量化类型单独指定 Linear 后端     |

IR 算子优先级：

| 算子                  | 优先实现          | 配置说明                            |
| ------------------- | ------------- | ------------------------------- |
| rms_norm            | native        | RMSNorm 优先使用 Native 实现          |
| fused_add_rms_norm  | native        | 融合 Add + RMSNorm 优先使用 Native 实现 |
| gelu_and_mul_sparse | triton、native | 优先 Triton，之后 Native             |

## **13. CompilationConfig：编译配置**

负责模型计算图编译、Torch Inductor 优化、CUDA Graph 捕获和运行时图复用。

| 配置项                          | 实际值                    | 配置说明                         |
| ---------------------------- | ---------------------- | ---------------------------- |
| mode                         | VLLM_COMPILE           | 使用 vLLM 编译模式                 |
| backend                      | inductor               | 使用 Torch Inductor 编译后端       |
| custom_ops                   | [none]                 | 未指定自定义算子替换                   |
| compile_mm_encoder           | False                  | 不编译多模态 Encoder               |
| cudagraph_mm_encoder         | False                  | 不对多模态 Encoder 使用 CUDA Graph  |
| compile_sizes                | []                     | 未指定固定编译尺寸                    |
| compile_ranges_endpoints     | [2496]                 | 编译范围端点为 2496                 |
| cudagraph_mode               | FULL_AND_PIECEWISE     | 同时使用完整和分段 CUDA Graph         |
| cudagraph_num_of_warmups     | 1                      | CUDA Graph 捕获前预热 1 次         |
| cudagraph_copy_inputs        | False                  | 不采用输入复制模式                    |
| max_cudagraph_capture_size   | 512                    | 最大 CUDA Graph 捕获规模           |
| cudagraph_specialize_lora    | True                   | CUDA Graph 针对 LoRA 进行特化      |
| use_inductor_graph_partition | False                  | 不启用 Inductor Graph Partition |
| fast_moe_cold_start          | False                  | 不启用 MoE 冷启动优化                |
| local_cache_dir              | torch_compile_cache 路径 | 编译缓存存储目录                     |

其他编译配置：

| 配置项                                            | 实际值                   | 配置说明                        |
| ---------------------------------------------- | --------------------- | --------------------------- |
| compile_cache_save_format                      | binary                | 编译缓存保存为二进制格式                |
| ir_enable_torch_wrap                           | True                  | 启用 Torch IR Wrapper         |
| splitting_ops                                  | 多个 vLLM Attention 等算子 | 指定编译图拆分边界                   |
| dynamic_shapes_config.type                     | backed                | 使用 Backed Dynamic Shapes    |
| dynamic_shapes_config.evaluate_guards          | False                 | 不启用 Guard 求值                |
| dynamic_shapes_config.assume_32_bit_indexing   | False                 | 不假设使用 32 位索引                |
| inductor_compile_config.combo_kernels          | True                  | 开启 Combo Kernels            |
| inductor_compile_config.benchmark_combo_kernel | True                  | 对 Combo Kernel 进行 Benchmark |
| pass_config.fuse_norm_quant                    | False                 | 不启用 Norm + Quant 融合         |
| pass_config.fuse_act_quant                     | False                 | 不启用 Activation + Quant 融合   |
| pass_config.fuse_attn_quant                    | False                 | 不启用 Attention + Quant 融合    |
| pass_config.enable_sp                          | False                 | 不启用 Sequence Parallel       |
| pass_config.fuse_gemm_comms                    | False                 | 不启用 GEMM + 通信融合             |
| pass_config.fuse_allreduce_rms                 | False                 | 不启用 AllReduce + RMSNorm 融合  |

## **14. ObservabilityConfig：可观测性配置**

负责运行指标、Trace、KV Cache 监控、CUDA Graph 监控及性能统计。

| 配置项                              | 实际值   | 配置说明                |
| -------------------------------- | ----- | ------------------- |
| otlp_traces_endpoint             | None  | 未指定 OTLP Trace 上报地址 |
| collect_detailed_traces          | None  | 未指定详细 Trace 采集模式    |
| kv_cache_metrics                 | False | 不启用 KV Cache 指标采集   |
| kv_cache_metrics_sample          | 0.01  | KV Cache 指标采样比例配置   |
| cudagraph_metrics                | False | 不启用 CUDA Graph 指标采集 |
| enable_mfu_metrics               | False | 不启用 MFU 计算指标        |
| enable_mm_processor_stats        | False | 不启用多模态 Processor 统计 |
| enable_logging_iteration_details | False | 不记录详细 Iteration 日志  |
| enable_layerwise_nvtx_tracing    | False | 不启用逐层 NVTX Trace    |
| jit_monitor_mode                 | warn  | JIT 监控使用警告模式        |
| jit_monitor_verbose              | False | 不输出详细 JIT 监控信息      |
| show_hidden_metrics_for_version  | None  | 未指定隐藏指标版本           |

## **15. ProfilerConfig：性能分析配置**

负责 PyTorch Profiler、Proton Profiler 等性能分析工具的采集参数。

| 配置项                                 | 实际值    | 配置说明                     |
| ----------------------------------- | ------ | ------------------------ |
| profiler                            | None   | 未指定 Profiler             |
| torch_profiler_dir                  | 空字符串   | 未指定 Torch Profiler 输出目录  |
| proton_profiler_dir                 | 空字符串   | 未指定 Proton Profiler 输出目录 |
| proton_context                      | shadow | Proton Context 模式        |
| proton_data                         | tree   | Proton 数据组织方式            |
| torch_profiler_with_stack           | True   | 允许采集调用栈                  |
| torch_profiler_with_flops           | False  | 不采集 FLOPs                |
| torch_profiler_with_gzip            | True   | Profiler 输出使用 GZIP       |
| torch_profiler_dump_cuda_time_total | True   | 输出 CUDA 总耗时              |
| torch_profiler_record_shapes        | False  | 不记录 Tensor Shape         |
| torch_profiler_with_memory          | False  | 不采集内存分析数据                |
| capture_torch_profiler              | False  | 不启用 Torch Profiler 捕获    |

## **16. 其他子配置**

| 配置模块                      | 配置项                           | 实际值       | 配置说明                       |
| ------------------------- | ----------------------------- | --------- | -------------------------- |
| MambaConfig               | backend                       | triton    | Mamba 使用 Triton 后端         |
| MambaConfig               | enable_stochastic_rounding    | False     | 不启用随机舍入                    |
| StructuredOutputsConfig   | backend                       | auto      | 结构化输出后端自动选择                |
| StructuredOutputsConfig   | disable_any_whitespace        | False     | 不禁用任意空白字符                  |
| StructuredOutputsConfig   | disable_additional_properties | False     | 不禁用额外属性                    |
| StructuredOutputsConfig   | reasoning_parser              | 空字符串      | 未指定 Reasoning Parser       |
| LoRAConfig                | lora_config                   | None      | 未配置 LoRA                   |
| SpeculativeConfig         | speculative_config            | None      | 未配置推测解码                    |
| KVTransferConfig          | kv_transfer_config            | None      | 未配置 KV Cache 传输            |
| KVEventsConfig            | kv_events_config              | None      | 未配置 KV Cache Events        |
| EncoderCacheManagerConfig | encoder_cache_manager_cls     | None      | 使用默认 Encoder Cache Manager |
| VllmConfig                | optimization_level            | O2        | 使用 O2 优化级别                 |
| VllmConfig                | performance_mode              | balanced  | 使用平衡性能模式                   |
| VllmConfig                | shutdown_timeout              | 0         | 关闭超时配置为 0                  |
| VllmConfig                | instance_id                   | 日志中的实例 ID | 当前 Engine 实例标识             |

## **17. 配置之间的关系总结**

从源码架构上，可以把以上配置归纳为以下几组：

| 配置模块                | 主要负责           | 关联核心组件                      |
| ------------------- | -------------- | --------------------------- |
| ModelConfig         | 模型定义与加载参数      | Model Registry、Model Loader |
| HF Config           | 模型网络结构         | Gemma4 Model                |
| QuantizationConfig  | 权重量化           | CompressedTensors           |
| MultiModalConfig    | 多模态输入处理        | MM Processor、MM Encoder     |
| CacheConfig         | KV Cache 内存管理  | KVCacheManager、BlockPool    |
| ParallelConfig      | 多卡和多进程并行       | Executor、Worker             |
| SchedulerConfig     | 请求调度和 Batch 管理 | Scheduler                   |
| DeviceConfig        | 设备选择           | CUDA Device                 |
| LoadConfig          | 权重加载           | Model Loader                |
| OffloadConfig       | CPU/GPU 权重交换   | Offloader、Prefetch          |
| AttentionConfig     | Attention 后端选择 | Attention Backend           |
| KernelConfig        | 算子实现和调优        | Triton、Native、FlashInfer    |
| CompilationConfig   | 编译和图优化         | Torch Inductor、CUDA Graph   |
| ObservabilityConfig | 运行监控           | Metrics、Tracing             |
| ProfilerConfig      | 性能分析           | Torch Profiler、Proton       |

**阅读建议：** 这份配置清单可以作为后续源码断点分析的索引。优先研究 `CacheConfig → KVCacheConfig → KVCacheManager`、`SchedulerConfig → Scheduler`、`ParallelConfig → Executor → Worker` 三条链路。

以上配置值来自你提供的日志；对于未展示完整嵌套内容的字段，我没有补造具体配置值。

vllm_config = VllmConfig(model_config=ModelConfig(model='/home/neo/.cache/huggingface/hub/models--cyankiwi--gemma-4-31B-it-AWQ-4bit/snapshots/6f1b616c64bca2a05d67fdd6437c5500a8509327', model_weights='', runner='auto', convert='auto', tokenizer='/home/neo/.cache/huggingface/hub/models--cyankiwi--gemma-4-31B-it-AWQ-4bit/snapshots/6f1b616c64bca2a05d67fdd6437c5500a8509327', tokenizer_mode='auto', trust_remote_code=False, dtype=torch.float16, seed=0, hf_config=Gemma4Config { "architectures": [ "Gemma4ForConditionalGeneration" ], "audio_config": null, "audio_token_id": 258881, "boa_token_id": 256000, "boi_token_id": 255999, "dtype": "float16", "eoa_token_id": 258883, "eoa_token_index": 258883, "eoi_token_id": 258882, "eos_token_id": [ 1, 106 ], "image_token_id": 258880, "initializer_range": 0.02, "model_type": "gemma4", "quantization_config": { "config_groups": { "group_0": { "format": "pack-quantized", "input_activations": null, "output_activations": null, "targets": [ "Linear" ], "weights": { "actorder": null, "block_structure": null, "dynamic": false, "group_size": 32, "num_bits": 4, "observer": "mse", "observer_kwargs": {}, "scale_dtype": null, "strategy": "group", "symmetric": false, "type": "int", "zp_dtype": "torch.int8" } } }, "format": "pack-quantized", "global_compression_ratio": null, "ignore": [ "model.vision_tower.patch_embedder.input_proj", "model.vision_tower.encoder.layers.0.self_attn.q_proj.linear", "model.vision_tower.encoder.layers.0.self_attn.k_proj.linear", "model.vision_tower.encoder.layers.0.self_attn.v_proj.linear", "model.vision_tower.encoder.layers.0.self_attn.o_proj.linear", "model.vision_tower.encoder.layers.0.mlp.gate_proj.linear", "model.vision_tower.encoder.layers.0.mlp.up_proj.linear", "model.vision_tower.encoder.layers.0.mlp.down_proj.linear", "model.vision_tower.encoder.layers.1.self_attn.q_proj.linear", "model.vision_tower.encoder.layers.1.self_attn.k_proj.linear", "model.vision_tower.encoder.layers.1.self_attn.v_proj.linear", "model.vision_tower.encoder.layers.1.self_attn.o_proj.linear", "model.vision_tower.encoder.layers.1.mlp.gate_proj.linear", "model.vision_tower.encoder.layers.1.mlp.up_proj.linear", "model.vision_tower.encoder.layers.1.mlp.down_proj.linear", "model.vision_tower.encoder.layers.2.self_attn.q_proj.linear", "model.vision_tower.encoder.layers.2.self_attn.k_proj.linear", "model.vision_tower.encoder.layers.2.self_attn.v_proj.linear", "model.vision_tower.encoder.layers.2.self_attn.o_proj.linear", "model.vision_tower.encoder.layers.2.mlp.gate_proj.linear", "model.vision_tower.encoder.layers.2.mlp.up_proj.linear", "model.vision_tower.encoder.layers.2.mlp.down_proj.linear", "model.vision_tower.encoder.layers.3.self_attn.q_proj.linear", "model.vision_tower.encoder.layers.3.self_attn.k_proj.linear", "model.vision_tower.encoder.layers.3.self_attn.v_proj.linear", "model.vision_tower.encoder.layers.3.self_attn.o_proj.linear", "model.vision_tower.encoder.layers.3.mlp.gate_proj.linear", "model.vision_tower.encoder.layers.3.mlp.up_proj.linear", "model.vision_tower.encoder.layers.3.mlp.down_proj.linear", "model.vision_tower.encoder.layers.4.self_attn.q_proj.linear", "model.vision_tower.encoder.layers.4.self_attn.k_proj.linear", "model.vision_tower.encoder.layers.4.self_attn.v_proj.linear", "model.vision_tower.encoder.layers.4.self_attn.o_proj.linear", "model.vision_tower.encoder.layers.4.mlp.gate_proj.linear", "model.vision_tower.encoder.layers.4.mlp.up_proj.linear", "model.vision_tower.encoder.layers.4.mlp.down_proj.linear", "model.vision_tower.encoder.layers.5.self_attn.q_proj.linear", "model.vision_tower.encoder.layers.5.self_attn.k_proj.linear", "model.vision_tower.encoder.layers.5.self_attn.v_proj.linear", "model.vision_tower.encoder.layers.5.self_attn.o_proj.linear", "model.vision_tower.encoder.layers.5.mlp.gate_proj.linear", "model.vision_tower.encoder.layers.5.mlp.up_proj.linear", "model.vision_tower.encoder.layers.5.mlp.down_proj.linear", "model.vision_tower.encoder.layers.6.self_attn.q_proj.linear", "model.vision_tower.encoder.layers.6.self_attn.k_proj.linear", "model.vision_tower.encoder.layers.6.self_attn.v_proj.linear", "model.vision_tower.encoder.layers.6.self_attn.o_proj.linear", "model.vision_tower.encoder.layers.6.mlp.gate_proj.linear", "model.vision_tower.encoder.layers.6.mlp.up_proj.linear", "model.vision_tower.encoder.layers.6.mlp.down_proj.linear", "model.vision_tower.encoder.layers.7.self_attn.q_proj.linear", "model.vision_tower.encoder.layers.7.self_attn.k_proj.linear", "model.vision_tower.encoder.layers.7.self_attn.v_proj.linear", "model.vision_tower.encoder.layers.7.self_attn.o_proj.linear", "model.vision_tower.encoder.layers.7.mlp.gate_proj.linear", "model.vision_tower.encoder.layers.7.mlp.up_proj.linear", "model.vision_tower.encoder.layers.7.mlp.down_proj.linear", "model.vision_tower.encoder.layers.8.self_attn.q_proj.linear", "model.vision_tower.encoder.layers.8.self_attn.k_proj.linear", "model.vision_tower.encoder.layers.8.self_attn.v_proj.linear", "model.vision_tower.encoder.layers.8.self_attn.o_proj.linear", "model.vision_tower.encoder.layers.8.mlp.gate_proj.linear", "model.vision_tower.encoder.layers.8.mlp.up_proj.linear", "model.vision_tower.encoder.layers.8.mlp.down_proj.linear", "model.vision_tower.encoder.layers.9.self_attn.q_proj.linear", "model.vision_tower.encoder.layers.9.self_attn.k_proj.linear", "model.vision_tower.encoder.layers.9.self_attn.v_proj.linear", "model.vision_tower.encoder.layers.9.self_attn.o_proj.linear", "model.vision_tower.encoder.layers.9.mlp.gate_proj.linear", "model.vision_tower.encoder.layers.9.mlp.up_proj.linear", "model.vision_tower.encoder.layers.9.mlp.down_proj.linear", "model.vision_tower.encoder.layers.10.self_attn.q_proj.linear", "model.vision_tower.encoder.layers.10.self_attn.k_proj.linear", "model.vision_tower.encoder.layers.10.self_attn.v_proj.linear", "model.vision_tower.encoder.layers.10.self_attn.o_proj.linear", "model.vision_tower.encoder.layers.10.mlp.gate_proj.linear", "model.vision_tower.encoder.layers.10.mlp.up_proj.linear", "model.vision_tower.encoder.layers.10.mlp.down_proj.linear", "model.vision_tower.encoder.layers.11.self_attn.q_proj.linear", "model.vision_tower.encoder.layers.11.self_attn.k_proj.linear", "model.vision_tower.encoder.layers.11.self_attn.v_proj.linear", "model.vision_tower.encoder.layers.11.self_attn.o_proj.linear", "model.vision_tower.encoder.layers.11.mlp.gate_proj.linear", "model.vision_tower.encoder.layers.11.mlp.up_proj.linear", "model.vision_tower.encoder.layers.11.mlp.down_proj.linear", "model.vision_tower.encoder.layers.12.self_attn.q_proj.linear", "model.vision_tower.encoder.layers.12.self_attn.k_proj.linear", "model.vision_tower.encoder.layers.12.self_attn.v_proj.linear", "model.vision_tower.encoder.layers.12.self_attn.o_proj.linear", "model.vision_tower.encoder.layers.12.mlp.gate_proj.linear", "model.vision_tower.encoder.layers.12.mlp.up_proj.linear", "model.vision_tower.encoder.layers.12.mlp.down_proj.linear", "model.vision_tower.encoder.layers.13.self_attn.q_proj.linear", "model.vision_tower.encoder.layers.13.self_attn.k_proj.linear", "model.vision_tower.encoder.layers.13.self_attn.v_proj.linear", "model.vision_tower.encoder.layers.13.self_attn.o_proj.linear", "model.vision_tower.encoder.layers.13.mlp.gate_proj.linear", "model.vision_tower.encoder.layers.13.mlp.up_proj.linear", "model.vision_tower.encoder.layers.13.mlp.down_proj.linear", "model.vision_tower.encoder.layers.14.self_attn.q_proj.linear", "model.vision_tower.encoder.layers.14.self_attn.k_proj.linear", "model.vision_tower.encoder.layers.14.self_attn.v_proj.linear", "model.vision_tower.encoder.layers.14.self_attn.o_proj.linear", "model.vision_tower.encoder.layers.14.mlp.gate_proj.linear", "model.vision_tower.encoder.layers.14.mlp.up_proj.linear", "model.vision_tower.encoder.layers.14.mlp.down_proj.linear", "model.vision_tower.encoder.layers.15.self_attn.q_proj.linear", "model.vision_tower.encoder.layers.15.self_attn.k_proj.linear", "model.vision_tower.encoder.layers.15.self_attn.v_proj.linear", "model.vision_tower.encoder.layers.15.self_attn.o_proj.linear", "model.vision_tower.encoder.layers.15.mlp.gate_proj.linear", "model.vision_tower.encoder.layers.15.mlp.up_proj.linear", "model.vision_tower.encoder.layers.15.mlp.down_proj.linear", "model.vision_tower.encoder.layers.16.self_attn.q_proj.linear", "model.vision_tower.encoder.layers.16.self_attn.k_proj.linear", "model.vision_tower.encoder.layers.16.self_attn.v_proj.linear", "model.vision_tower.encoder.layers.16.self_attn.o_proj.linear", "model.vision_tower.encoder.layers.16.mlp.gate_proj.linear", "model.vision_tower.encoder.layers.16.mlp.up_proj.linear", "model.vision_tower.encoder.layers.16.mlp.down_proj.linear", "model.vision_tower.encoder.layers.17.self_attn.q_proj.linear", "model.vision_tower.encoder.layers.17.self_attn.k_proj.linear", "model.vision_tower.encoder.layers.17.self_attn.v_proj.linear", "model.vision_tower.encoder.layers.17.self_attn.o_proj.linear", "model.vision_tower.encoder.layers.17.mlp.gate_proj.linear", "model.vision_tower.encoder.layers.17.mlp.up_proj.linear", "model.vision_tower.encoder.layers.17.mlp.down_proj.linear", "model.vision_tower.encoder.layers.18.self_attn.q_proj.linear", "model.vision_tower.encoder.layers.18.self_attn.k_proj.linear", "model.vision_tower.encoder.layers.18.self_attn.v_proj.linear", "model.vision_tower.encoder.layers.18.self_attn.o_proj.linear", "model.vision_tower.encoder.layers.18.mlp.gate_proj.linear", "model.vision_tower.encoder.layers.18.mlp.up_proj.linear", "model.vision_tower.encoder.layers.18.mlp.down_proj.linear", "model.vision_tower.encoder.layers.19.self_attn.q_proj.linear", "model.vision_tower.encoder.layers.19.self_attn.k_proj.linear", "model.vision_tower.encoder.layers.19.self_attn.v_proj.linear", "model.vision_tower.encoder.layers.19.self_attn.o_proj.linear", "model.vision_tower.encoder.layers.19.mlp.gate_proj.linear", "model.vision_tower.encoder.layers.19.mlp.up_proj.linear", "model.vision_tower.encoder.layers.19.mlp.down_proj.linear", "model.vision_tower.encoder.layers.20.self_attn.q_proj.linear", "model.vision_tower.encoder.layers.20.self_attn.k_proj.linear", "model.vision_tower.encoder.layers.20.self_attn.v_proj.linear", "model.vision_tower.encoder.layers.20.self_attn.o_proj.linear", "model.vision_tower.encoder.layers.20.mlp.gate_proj.linear", "model.vision_tower.encoder.layers.20.mlp.up_proj.linear", "model.vision_tower.encoder.layers.20.mlp.down_proj.linear", "model.vision_tower.encoder.layers.21.self_attn.q_proj.linear", "model.vision_tower.encoder.layers.21.self_attn.k_proj.linear", "model.vision_tower.encoder.layers.21.self_attn.v_proj.linear", "model.vision_tower.encoder.layers.21.self_attn.o_proj.linear", "model.vision_tower.encoder.layers.21.mlp.gate_proj.linear", "model.vision_tower.encoder.layers.21.mlp.up_proj.linear", "model.vision_tower.encoder.layers.21.mlp.down_proj.linear", "model.vision_tower.encoder.layers.22.self_attn.q_proj.linear", "model.vision_tower.encoder.layers.22.self_attn.k_proj.linear", "model.vision_tower.encoder.layers.22.self_attn.v_proj.linear", "model.vision_tower.encoder.layers.22.self_attn.o_proj.linear", "model.vision_tower.encoder.layers.22.mlp.gate_proj.linear", "model.vision_tower.encoder.layers.22.mlp.up_proj.linear", "model.vision_tower.encoder.layers.22.mlp.down_proj.linear", "model.vision_tower.encoder.layers.23.self_attn.q_proj.linear", "model.vision_tower.encoder.layers.23.self_attn.k_proj.linear", "model.vision_tower.encoder.layers.23.self_attn.v_proj.linear", "model.vision_tower.encoder.layers.23.self_attn.o_proj.linear", "model.vision_tower.encoder.layers.23.mlp.gate_proj.linear", "model.vision_tower.encoder.layers.23.mlp.up_proj.linear", "model.vision_tower.encoder.layers.23.mlp.down_proj.linear", "model.vision_tower.encoder.layers.24.self_attn.q_proj.linear", "model.vision_tower.encoder.layers.24.self_attn.k_proj.linear", "model.vision_tower.encoder.layers.24.self_attn.v_proj.linear", "model.vision_tower.encoder.layers.24.self_attn.o_proj.linear", "model.vision_tower.encoder.layers.24.mlp.gate_proj.linear", "model.vision_tower.encoder.layers.24.mlp.up_proj.linear", "model.vision_tower.encoder.layers.24.mlp.down_proj.linear", "model.vision_tower.encoder.layers.25.self_attn.q_proj.linear", "model.vision_tower.encoder.layers.25.self_attn.k_proj.linear", "model.vision_tower.encoder.layers.25.self_attn.v_proj.linear", "model.vision_tower.encoder.layers.25.self_attn.o_proj.linear", "model.vision_tower.encoder.layers.25.mlp.gate_proj.linear", "model.vision_tower.encoder.layers.25.mlp.up_proj.linear", "model.vision_tower.encoder.layers.25.mlp.down_proj.linear", "model.vision_tower.encoder.layers.26.self_attn.q_proj.linear", "model.vision_tower.encoder.layers.26.self_attn.k_proj.linear", "model.vision_tower.encoder.layers.26.self_attn.v_proj.linear", "model.vision_tower.encoder.layers.26.self_attn.o_proj.linear", "model.vision_tower.encoder.layers.26.mlp.gate_proj.linear", "model.vision_tower.encoder.layers.26.mlp.up_proj.linear", "model.vision_tower.encoder.layers.26.mlp.down_proj.linear", "model.embed_vision.embedding_projection", "lm_head" ], "kv_cache_scheme": null, "quant_method": "compressed-tensors", "quantization_status": "compressed", "sparsity_config": {}, "total_num_heads": 32, "total_num_kv_heads": 16, "transform_config": {}, "version": "0.14.1.a20260326" }, "text_config": { "attention_bias": false, "attention_dropout": 0.0, "attention_k_eq_v": true, "bos_token_id": 2, "dtype": "float16", "enable_moe_block": false, "eos_token_id": 1, "expert_intermediate_size": null, "final_logit_softcapping": 30.0, "head_dim": 256, "hidden_activation": "gelu_pytorch_tanh", "hidden_size": 5376, "hidden_size_per_layer_input": 0, "initializer_range": 0.02, "intermediate_size": 21504, "layer_types": [ "sliding_attention", "sliding_attention", "sliding_attention", "sliding_attention", "sliding_attention", "full_attention", "sliding_attention", "sliding_attention", "sliding_attention", "sliding_attention", "sliding_attention", "full_attention", "sliding_attention", "sliding_attention", "sliding_attention", "sliding_attention", "sliding_attention", "full_attention", "sliding_attention", "sliding_attention", "sliding_attention", "sliding_attention", "sliding_attention", "full_attention", "sliding_attention", "sliding_attention", "sliding_attention", "sliding_attention", "sliding_attention", "full_attention", "sliding_attention", "sliding_attention", "sliding_attention", "sliding_attention", "sliding_attention", "full_attention", "sliding_attention", "sliding_attention", "sliding_attention", "sliding_attention", "sliding_attention", "full_attention", "sliding_attention", "sliding_attention", "sliding_attention", "sliding_attention", "sliding_attention", "full_attention", "sliding_attention", "sliding_attention", "sliding_attention", "sliding_attention", "sliding_attention", "full_attention", "sliding_attention", "sliding_attention", "sliding_attention", "sliding_attention", "sliding_attention", "full_attention" ], "max_position_embeddings": 262144, "model_type": "gemma4_text", "moe_intermediate_size": null, "num_attention_heads": 32, "num_experts": null, "num_hidden_layers": 60, "num_key_value_heads": 16, "num_kv_shared_layers": 0, "pad_token_id": 0, "per_layer_config": { "05": { "head_dim": 512, "num_key_value_heads": 4 }, "11": { "head_dim": 512, "num_key_value_heads": 4 }, "17": { "head_dim": 512, "num_key_value_heads": 4 }, "23": { "head_dim": 512, "num_key_value_heads": 4 }, "29": { "head_dim": 512, "num_key_value_heads": 4 }, "35": { "head_dim": 512, "num_key_value_heads": 4 }, "41": { "head_dim": 512, "num_key_value_heads": 4 }, "47": { "head_dim": 512, "num_key_value_heads": 4 }, "53": { "head_dim": 512, "num_key_value_heads": 4 }, "59": { "head_dim": 512, "num_key_value_heads": 4 } }, "rms_norm_eps": 1e-06, "rope_parameters": { "full_attention": { "partial_rotary_factor": 0.25, "rope_theta": 1000000.0, "rope_type": "proportional" }, "sliding_attention": { "rope_theta": 10000.0, "rope_type": "default" } }, "sliding_window": 1024, "tie_word_embeddings": true, "top_k_experts": null, "use_bidirectional_attention": "vision", "use_cache": true, "use_double_wide_mlp": false, "vocab_size": 262144, "vocab_size_per_layer_input": 262144 }, "tie_word_embeddings": true, "transformers_version": "5.23.0", "video_token_id": 258884, "vision_config": { "attention_bias": false, "attention_dropout": 0.0, "default_output_length": 280, "dtype": "bfloat16", "global_head_dim": 72, "head_dim": 72, "hidden_activation": "gelu_pytorch_tanh", "hidden_size": 1152, "initializer_range": 0.02, "intermediate_size": 4304, "max_position_embeddings": 131072, "model_type": "gemma4_vision", "num_attention_heads": 16, "num_hidden_layers": 27, "num_key_value_heads": 16, "patch_size": 16, "pooling_kernel_size": 3, "position_embedding_size": 10240, "rms_norm_eps": 1e-06, "rope_parameters": { "rope_theta": 100.0, "rope_type": "axial" }, "standardize": true, "use_clipped_linears": false }, "vision_soft_tokens_per_image": 280 }, hf_text_config=Gemma4TextConfig { "attention_bias": false, "attention_dropout": 0.0, "attention_k_eq_v": true, "bos_token_id": 2, "dtype": "float16", "enable_moe_block": false, "eos_token_id": 1, "expert_intermediate_size": null, "final_logit_softcapping": 30.0, "head_dim": 256, "hidden_activation": "gelu_pytorch_tanh", "hidden_size": 5376, "hidden_size_per_layer_input": 0, "initializer_range": 0.02, "intermediate_size": 21504, "layer_types": [ "sliding_attention", "sliding_attention", "sliding_attention", "sliding_attention", "sliding_attention", "full_attention", "sliding_attention", "sliding_attention", "sliding_attention", "sliding_attention", "sliding_attention", "full_attention", "sliding_attention", "sliding_attention", "sliding_attention", "sliding_attention", "sliding_attention", "full_attention", "sliding_attention", "sliding_attention", "sliding_attention", "sliding_attention", "sliding_attention", "full_attention", "sliding_attention", "sliding_attention", "sliding_attention", "sliding_attention", "sliding_attention", "full_attention", "sliding_attention", "sliding_attention", "sliding_attention", "sliding_attention", "sliding_attention", "full_attention", "sliding_attention", "sliding_attention", "sliding_attention", "sliding_attention", "sliding_attention", "full_attention", "sliding_attention", "sliding_attention", "sliding_attention", "sliding_attention", "sliding_attention", "full_attention", "sliding_attention", "sliding_attention", "sliding_attention", "sliding_attention", "sliding_attention", "full_attention", "sliding_attention", "sliding_attention", "sliding_attention", "sliding_attention", "sliding_attention", "full_attention" ], "max_position_embeddings": 262144, "model_type": "gemma4_text", "moe_intermediate_size": null, "num_attention_heads": 32, "num_experts": null, "num_hidden_layers": 60, "num_key_value_heads": 16, "num_kv_shared_layers": 0, "pad_token_id": 0, "per_layer_config": { "05": { "head_dim": 512, "num_key_value_heads": 4 }, "11": { "head_dim": 512, "num_key_value_heads": 4 }, "17": { "head_dim": 512, "num_key_value_heads": 4 }, "23": { "head_dim": 512, "num_key_value_heads": 4 }, "29": { "head_dim": 512, "num_key_value_heads": 4 }, "35": { "head_dim": 512, "num_key_value_heads": 4 }, "41": { "head_dim": 512, "num_key_value_heads": 4 }, "47": { "head_dim": 512, "num_key_value_heads": 4 }, "53": { "head_dim": 512, "num_key_value_heads": 4 }, "59": { "head_dim": 512, "num_key_value_heads": 4 } }, "rms_norm_eps": 1e-06, "rope_parameters": { "full_attention": { "partial_rotary_factor": 0.25, "rope_theta": 1000000.0, "rope_type": "proportional" }, "sliding_attention": { "rope_theta": 10000.0, "rope_type": "default" } }, "sliding_window": 1024, "tie_word_embeddings": true, "top_k_experts": null, "transformers_version": "5.23.0", "use_bidirectional_attention": "vision", "use_cache": true, "use_double_wide_mlp": false, "vocab_size": 262144, "vocab_size_per_layer_input": 262144 }, word_embeddings_untied_by_checkpoint=False, hf_config_path=None, allowed_local_media_path='', allowed_media_domains=None, revision=None, code_revision=None, tokenizer_revision=None, max_model_len=16000, spec_target_max_model_len=None, quantization='compressed-tensors', quantization_config=None, allow_deprecated_quantization=False, enforce_eager=False, enable_return_routed_experts=False, return_sampling_mask=False, max_logprobs=20, logprobs_mode='raw_logprobs', use_fp64_gumbel=False, enable_trace_replay=False, disable_sliding_window=False, disable_cascade_attn=True, skip_tokenizer_init=False, enable_prompt_embeds=False, served_model_name='debug-model', config_format='auto', hf_token=None, hf_overrides={}, model_class_overrides={}, generation_config='auto', override_generation_config={}, enable_sleep_mode=False, sleep_mode_backend='cumem', enable_nccl_comm_suspend=False, enable_cumem_allocator=False, model_impl='auto', logits_processors=None, io_processor_plugin=None, renderer_num_workers=1, pooler_config=None, multimodal_config=MultiModalConfig(language_model_only=False, limit_per_prompt={'video': VideoDummyOptions(count=0, num_frames=None, width=None, height=None)}, enable_mm_embeds=False, media_io_kwargs={}, mm_processor_kwargs=None, mm_device_do_normalize=False, mm_processor_cache_gb=4.0, mm_processor_cache_type='lru', mm_hasher_algorithm='blake3', mm_shm_cache_max_object_size_mb=128, mm_encoder_only=False, mm_encoder_tp_mode='weights', mm_encoder_attn_backend=None, mm_encoder_attn_dtype=None, mm_encoder_fp8_scale_path=None, mm_encoder_fp8_scale_save_path=None, mm_encoder_fp8_scale_save_margin=1.5, interleave_mm_strings=False, skip_mm_profiling=False, video_pruning_rate=None, video_pruning_method='evs', mm_tensor_ipc='direct_rpc', allow_missing_mm_embeddings=False, mm_ipc_gpu_memory_gb=0.0)), cache_config=CacheConfig(block_size=16, user_specified_block_size=False, user_specified_mamba_block_size=False, kv_cache_layout='LBNHC', prefix_match_unit=None, gpu_memory_utilization=0.9, cache_dtype='auto', is_attention_free=False, num_gpu_blocks_override=None, sliding_window=None, enable_prefix_caching=True, prefix_caching_hash_algo='sha256', prefix_cache_retention_interval=0, kv_cache_dtype_skip_layers=[], mamba_page_size_padded=None, skip_page_size_padded=None, mamba_block_size=None, mamba_cache_dtype='auto', mamba_ssm_cache_dtype='auto', mamba_cache_mode='none', enable_mamba_fine_grained_prefix_cache=False, replayssm_buffer_len=16, use_replayssm=False, use_kda_recoverssm=False, num_gpu_blocks=2454, num_cpu_blocks=None, kv_cache_size_tokens=16462, kv_cache_max_concurrency=1.0289308176100629, kv_sharing_fast_prefill=False, kv_cache_memory_bytes=None, kv_offloading_size=None, kv_offloading_backend='native', _block_size_resolved=True), parallel_config=ParallelConfig(pipeline_parallel_size=1, tensor_parallel_size=1, prefill_context_parallel_size=1, data_parallel_size=1, data_parallel_size_local=1, data_parallel_rank=0, data_parallel_rank_local=0, data_parallel_master_ip='127.0.0.1', data_parallel_rpc_port=29550, dp_sync_interval=16, data_parallel_master_port=0, data_parallel_backend='mp', data_parallel_external_lb=False, data_parallel_hybrid_lb=False, is_moe_model=False, enable_expert_parallel=False, enable_batch_sharded_sampling=False, enable_ep_weight_filter=False, enable_eplb=False, eplb_config=EPLBConfig(window_size=1000, step_interval=3000, num_redundant_experts=0, log_balancedness=False, log_balancedness_interval=1, use_async=True, policy='default', communicator=None), expert_placement_strategy='linear', all2all_backend='allgather_reducescatter', max_parallel_loading_workers=None, disable_custom_all_reduce=False, enable_elastic_ep=False, elastic_ep_max_dp_size=1, enable_dbo=False, ubatch_size=0, dbo_decode_token_threshold=32, dbo_prefill_token_threshold=512, disable_nccl_for_dp_synchronization=True, ray_workers_use_nsight=False, ray_runtime_env=None, placement_group=None, distributed_executor_backend='uni', worker_cls='vllm.v1.worker.gpu_worker.Worker', sd_worker_cls='auto', worker_extension_cls='', master_addr='127.0.0.1', master_port=29501, node_rank=0, nnodes=1, numa_bind=False, numa_bind_nodes=None, numa_bind_cpus=None, assigned_physical_gpu_ids=None, distributed_timeout_seconds=None, cpu_distributed_timeout_seconds=None, world_size=1, rank=0, _data_parallel_master_port_list=[], _coord_store_port=0, decode_context_parallel_size=1, dcp_kv_cache_interleave_size=1, dcp_comm_backend='ag_rs', dcp_q_replicate=False, cp_kv_cache_interleave_size=1, _allow_auto_resolve_cp_interleave_size=True, data_parallel_index=0, _api_process_count=1, _api_process_rank=0, enable_fault_tolerance=False, fault_tolerance_config=FaultToleranceConfig(engine_recovery_timeout_sec=120)), scheduler_config=SchedulerConfig(runner_type='generate', max_num_batched_tokens=2496, max_num_scheduled_tokens=None, max_num_seqs=256, long_prefill_token_threshold=0, max_num_queued_reqs=None, max_num_queued_tokens=None, enable_chunked_prefill=True, is_multimodal_model=True, max_num_encoder_input_tokens=2496, encoder_cache_size=2496, policy='fcfs', disable_chunked_mm_input=True, scheduler_cls=None, disable_hybrid_kv_cache_manager=False, scheduler_reserve_full_isl=True, watermark=0.0, prefill_schedule_interval=1, async_scheduling=True, stream_interval=1), device_config=DeviceConfig(device=device(type='cuda'), device_type='cuda'), load_config=LoadConfig(load_format='auto', download_dir=None, safetensors_load_strategy=None, safetensors_prefetch_num_threads=8, safetensors_prefetch_block_size=16777216, model_loader_extra_config={}, device=None, ignore_patterns=['original/**/*'], use_tqdm_on_load=True, pt_load_map_location='cpu'), offload_config=OffloadConfig(offload_backend='prefetch', uva=UVAOffloadConfig(cpu_offload_gb=12.0, cpu_offload_params=set()), prefetch=PrefetchOffloadConfig(offload_group_size=8, offload_num_in_group=4, offload_prefetch_step=1, offload_params=set())), attention_config=AttentionConfig(backend=<AttentionBackendEnum.TRITON_ATTN: 'vllm.v1.attention.backends.triton_attn.TritonAttentionBackend'>, minimax_m3_msa_decode_backend='triton', backend_per_kind={}, flash_attn_version=None, flash_attn_max_num_splits_for_cuda_graph=32, tq_max_kv_splits_for_cuda_graph=32, use_trtllm_attention=None, disable_flashinfer_q_quantization=False, mla_prefill_backend=None, use_prefill_query_quantization=False, indexer_kv_dtype='auto', hisparse_config=None, use_non_causal=False, sparse_mla_force_mqa=False, flex_attn_block_m=None, flex_attn_block_n=None, flex_attn_q_block_size=None, flex_attn_kv_block_size=None), engram_config=None, mamba_config=MambaConfig(backend=<MambaBackendEnum.TRITON: 'triton'>, enable_stochastic_rounding=False, stochastic_rounding_philox_rounds=0, ssu_algorithm=None), kernel_config=KernelConfig(ir_op_priority=IrOpPriorityConfig(rms_norm=['native'], fused_add_rms_norm=['native'], gelu_and_mul_sparse=['triton', 'native']), enable_flashinfer_autotune=True, enable_cutedsl_warmup=True, enable_jit_warmup=True, moe_backend='auto', sparse_indexer_topk_backend='auto', linear_backend='auto', linear_backend_per_quant=None), lora_config=None, speculative_config=None, watermark_config=None, diffusion_config=None, structured_outputs_config=StructuredOutputsConfig(backend='auto', disable_any_whitespace=False, disable_additional_properties=False, reasoning_parser='', reasoning_parser_plugin='', enable_in_reasoning=False), observability_config=ObservabilityConfig(show_hidden_metrics_for_version=None, otlp_traces_endpoint=None, collect_detailed_traces=None, per_request_spec_decode_metrics='none', kv_cache_metrics=False, kv_cache_metrics_sample=0.01, cudagraph_metrics=False, enable_layerwise_nvtx_tracing=False, enable_mfu_metrics=False, enable_mm_processor_stats=False, enable_logging_iteration_details=False, jit_monitor_mode='warn', jit_monitor_verbose=False), quant_config=<vllm.model_executor.layers.quantization.compressed_tensors.compressed_tensors.CompressedTensorsConfig object at 0x753db0170950>, compilation_config={'mode': <CompilationMode.VLLM_COMPILE: 3>, 'debug_dump_path': None, 'cache_dir': '', 'compile_cache_save_format': 'binary', 'backend': 'inductor', 'custom_ops': ['none'], 'ir_enable_torch_wrap': True, 'splitting_ops': ['vllm::unified_attention_with_output', 'vllm::unified_mla_attention_with_output', 'vllm::mamba_mixer2', 'vllm::mamba_mixer', 'vllm::short_conv', 'vllm::qwen4_exp_ple_short_conv', 'vllm::qwen4_exp_qsa_with_output', 'vllm::linear_attention', 'vllm::qwen_gdn_attention_core', 'vllm::qwen_gdn_attention_core_fused_norm_packed', 'vllm::gdn_attention_core_xpu', 'vllm::olmo_hybrid_gdn_full_forward', 'vllm::sparse_attn_indexer', 'vllm::rocm_aiter_sparse_attn_indexer', 'vllm::deepseek_v4_attention', 'vllm::hpc_rope_norm_forward', 'vllm::unified_kv_cache_update', 'vllm::unified_mla_kv_cache_update'], 'compile_mm_encoder': False, 'cudagraph_mm_encoder': False, 'encoder_cudagraph_token_budgets': [], 'encoder_cudagraph_max_vision_items_per_batch': 0, 'encoder_cudagraph_max_frames_per_batch': None, 'compile_sizes': [], 'compile_ranges_endpoints': [2496], 'inductor_compile_config': {'enable_auto_functionalized_v2': False, 'combo_kernels': True, 'benchmark_combo_kernel': True}, 'inductor_passes': {}, 'cudagraph_mode': <CUDAGraphMode.FULL_AND_PIECEWISE: (2, 1)>, 'cudagraph_num_of_warmups': 1, 'cudagraph_capture_sizes': [1, 2, 4, 8, 16, 24, 32, 40, 48, 56, 64, 72, 80, 88, 96, 104, 112, 120, 128, 136, 144, 152, 160, 168, 176, 184, 192, 200, 208, 216, 224, 232, 240, 248, 256, 272, 288, 304, 320, 336, 352, 368, 384, 400, 416, 432, 448, 464, 480, 496, 512], 'cudagraph_copy_inputs': False, 'cudagraph_specialize_lora': True, 'use_inductor_graph_partition': False, 'pass_config': {'fuse_norm_quant': False, 'fuse_act_quant': False, 'fuse_attn_quant': False, 'enable_sp': False, 'fuse_gemm_comms': False, 'fuse_allreduce_rms': False, 'enable_qk_norm_rope_fusion': False, 'fuse_rope_kvcache_cat_mla': False, 'fuse_act_padding': False, 'fuse_qk_norm_rope_kvcache': False}, 'max_cudagraph_capture_size': 512, 'dynamic_shapes_config': {'type': <DynamicShapesType.BACKED: 'backed'>, 'evaluate_guards': False, 'assume_32_bit_indexing': False}, 'local_cache_dir': '/home/neo/.cache/vllm/torch_compile_cache/torch_aot_compile/f4f1fec5dc8563e5ca5c4ea9ca4c050a118b8c2d057c3b993390bbd1b5a3cd38', 'fast_moe_cold_start': False, 'static_all_moe_layers': []}, profiler_config=ProfilerConfig(profiler=None, torch_profiler_dir='', proton_profiler_dir='', proton_context='shadow', proton_data='tree', proton_backend=None, proton_mode=None, proton_hook=None, proton_output_format=None, proton_graph_attribution=False, torch_profiler_with_stack=True, torch_profiler_with_flops=False, torch_profiler_use_gzip=True, torch_profiler_dump_cuda_time_total=True, torch_profiler_record_shapes=False, torch_profiler_with_memory=False, capture_torch_profiler=False, detailed_trace_annotation=False, ignore_frontend=False, delay_iterations=0, max_iterations=0, warmup_iterations=0, active_iterations=5, wait_iterations=0), kv_transfer_config=None, kv_events_config=None, ec_transfer_config=None, ec_manager_config=EncoderCacheManagerConfig(encoder_cache_manager_cls=None, manager_config={}), reasoning_config=None, additional_config={}, instance_id='1795533739371249132', optimization_level=<OptimizationLevel.O2: 2>, performance_mode='balanced', weight_transfer_config=None, shutdown_timeout=0)
