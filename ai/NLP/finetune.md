# 模型微调技术
- https://github.com/huggingface/peft
## LoRA 低秩适应大型语言模型
- 它通过利用低秩矩阵分解来减少微调过程中的计算和内存需求
- 通过将语言模型的权重矩阵进行低秩分解来解决这个问题
- 进行微调时，只需要更新低秩分解后得到的因子矩阵，相比于更新完整的权重矩阵，这需要更少的计算和内存资源
### 奇异值分解（SVD）是一种常用的低秩分解方法，
- 可以将一个矩阵分解为三个部分：左奇异向量矩阵、奇异值对角矩阵和右奇异向量矩阵的转置
- 优点是能够找到最佳的低秩逼近，并且在计算上相对高效
- 奇异值分解对任何矩阵都有效，甚至适用于非方阵
- https://mp.weixin.qq.com/s?__biz=MzU0MDQ1NjAzNg==&mid=2247539565&idx=3&sn=12e3834d5949f3fb6296910ee1a9fa74&chksm=fb3afe66cc4d7770beac01549e482fd83d88513e8bbd7d127115cedc790d9045976ce27ff2b6&scene=27
### 特征分解（Eigenvalue Decomposition）是另一种常见的低秩分解方法
- 它将一个矩阵分解为特征向量矩阵和特征值对角矩阵的乘积形式
- 特征分解在某些情况下可以提供更好的可解释性
- 特征分解的计算复杂度较高
- 特征分解只对方形矩阵有效
### 参数说明
- r（低秩矩阵的维度）：指定LoRA（低秩注意力）层中低秩矩阵的维度。低秩矩阵是一种通过降低注意力机制的复杂度来减少计算和内存需求的方法。
- lora_alpha（低秩矩阵的缩放因子）：用于缩放低秩矩阵的因子。通过调整该因子，可以控制低秩矩阵在模型中的重要性或影响力。
- lora_dropout（LoRA层的dropout概率）：指定LoRA层中使用的dropout概率。dropout是一种在训练过程中随机丢弃一部分神经元的技术，有助于减少过拟合并提高模型的泛化能力。
### 原理
- 对于 Transformer 模型，LoRA 可能会被添加到模型的每个 Transformer 层。在每个层中，LoRA 会生成一个低秩矩阵，用来调整该层的输出
- 
### 使用LoRA进行模型微调
```
from peft import get_peft_model, LoraConfig, TaskType
## 配置lora
peft_config = LoraConfig(
    task_type=TaskType.SEQ_2_SEQ_LM, inference_mode=False, r=8, lora_alpha=32, lora_dropout=0.1
)

## get_peft_model包装基础模型
model = AutoModelForSeq2SeqLM.from_pretrained(model_name_or_path)
model = get_peft_model(model, peft_config)
model.print_trainable_parameters()
## 保存Peft权重
model.save_pretrained("output_dir") 
```
### 使用LoRA进行推理
```
from peft import PeftModel, PeftConfig
## 合并模型 model为基模型，peft_model_id为PEFT模型路径
model = PeftModel.from_pretrained(model, peft_model_id)
```
## QLoRA
- 通过使用高精度权重进行模型微调
- 利用可学习的低秩适配器调整预训练模型的权重
### 技术细节
- 4 位 Normalfloat，一种理论上最佳量化数据类型，该数据类型对正态分布数据产生比 4 位整数和 4 位 Float 更好的实证结果
- NF4 的格点按照正态分布的分位数截取，格点分布两端稀疏，中间密集，格点分布与数据分布一致。
- 双量化，一种量化量化常数的方法，每个参数保存平均约 0.37 位（65B 模型大约 3 GB）
- QLoRA 将每 64 个参数为做一个 block，即 block_size = 64，每个 block 计算一个 Scale。由于量化后的 Scale 通常以 FP32 存储，在 block 数众多的情况下，Scale 占用的显存也不可忽视。因此，QLoRA 对 Scale 进一步量化成 FP8，取 Double Quant 的 block size = 256，因而进一步降低了显存消耗。
- Paged Optimizers，使用 NVIDIA 统一内存，以防止梯度检查点期间的内存峰值导致传统上对大型模型困难的单个机器进行微调的内存不足错误
- NVIDIA 统一内存：当 GPU 运行内存不足时，当优化器更新步骤中需要内存时，这些状态会被自动门出到 CPU RAM。
- 存储数据类型：4 位 Normalfloat
- 计算数据类型：BF16
```
model = AutoModelForCausalLM.from_pretrained(
        model_name_or_path='/name/or/path/to/your/model',
        load_in_4bit=True,
        device_map='auto',
        max_memory=max_memory,
        torch_dtype=torch.bfloat16,
        quantization_config=BitsAndBytesConfig(
            load_in_4bit=True,
            bnb_4bit_compute_dtype=torch.bfloat16,
            bnb_4bit_use_double_quant=True,
            bnb_4bit_quant_type='nf4'
        ),
    )
```
## GPTQ
- GPTQ 对某个 block 内的所有参数逐个量化，每个参数量化后，需要适当调整这个 block 内其他未量化的参数，以弥补量化造成的精度损失
- 发现直接按顺序做参数量化，对精度影响也不大，因此参数矩阵每一行的量化可以做并行的矩阵计算
- Lazy Batch-Updates，延迟一部分参数的更新，它能够缓解 bandwidth 的压力
- Cholesky Reformulation，用 Cholesky 分解求海森矩阵的逆，在增强数值稳定性的同时，不再需要对海森矩阵做更新计算，进一步减少了计算量
### OBD：Optimal Brain 
- 任务参数之间相互独立
- 实际上是一种剪枝方法，用于降低模型复杂度，提高泛化能力
- 计算参数对结果的影响，排序，从而知道剪枝的顺序
### OBS：Optimal Brain Surgeon
- 参数之间的独立性不成立，我们还是要考虑交叉项
- 寻找最合适的权重 ，使得删除它对目标的影响最小。
### OBC
- OBD 和 OBS 都存在一个缺点，就是剪枝需要计算全参数的海森矩阵（或者它的逆矩阵）
- 假设参数矩阵的同一行参数互相之间是相关的，而不同行之间的参数互不相关
- 这样，海森矩阵就只需要在每一行内单独计算
### OBQ
- 剪枝是一种特殊的量化（即剪枝的参数等价于量化到 0 点）
- 算法复杂度太高
- 采用贪心策略，先量化对目标影响最小的参数
## Accelerate:Hugging Face Pytorch GPU多机多卡加速器
- DeepSpeed：https://huggingface.co/docs/accelerate/usage_guides/deepspeed
- https://huggingface.co/docs/accelerate/index

## 常用库
- transformers： https://huggingface.co/docs/transformers/index
- deepspeed ：https://huggingface.co/docs/transformers/main/main_classes/deepspeed
- https://www.deepspeed.ai/tutorials/

## bitsandbytes
- https://github.com/TimDettmers/bitsandbytes
- 轻量级的CUDA自定义函数包装器，特别用于PyTorch中的8位优化器、矩阵乘法（LLM.int8()）和量化函数。
- 8位优化器：bitsandbytes提供了8位优化器，用于在深度学习模型中进行训练和优化。您可以使用8位优化器替换torch.optim中的优化器，并通过修改代码中的相应部分来配置和使用它们。
- LLM.int8()矩阵乘法：bitsandbytes提供了LLM.int8()函数，用于执行8位整数矩阵乘法操作。这种矩阵乘法的目的是在保持计算精度的同时减少内存和计算需求。
- 量化函数：bitsandbytes还提供了一些用于量化和处理数据的函数，以提高模型的效率和性能。
## deepspeed
- 提高大规模模型训练的效率和可扩展性
- 模型并行化、梯度累积、动态精度缩放、本地模式混合精度等
- 分布式训练管理、内存优化和模型压缩
- 语言模型、图像分类、目标检测
- 大模型训练加速库，位于模型训练框架和模型之间，用来提升训练、推理等。
### ZeRO(零冗余优化器)
- 用于大规模分布式深度学习的新型内存优化技术
- 速度提升3倍
- 用于提高显存效率和计算效率
- 通过在数据并行进程之间划分模型状态参数、梯度和优化器状态来消除数据并行进程中的内存冗余，而不是复制它们
- 在训练期间使用动态通信调度来在分布式设备之间共享必要的状态，以保持数据并行的计算粒度和通信量
- 数据并行:一张GPU无法存储足够的数据，此时，需要对数据进行分块，放到不同的GPU上进行处理，反向传播后，再通过通信归约梯度，保证优化器在各个机器进行相同的更新
- 模型并行。数据并行时，每个GPU会加载完整的模型结构，但是如果一个模型参数非常多，一个GPU无法加载所有的参数时，需要对模型进行分层，每个GPU处理一层，该方法就是模型并行
- 流水线并行：必须同时计算多个 micro-batch，确保流水线的各个阶段能并行计算
#### Zero-1 消除了内存冗余
- Optimizer State Partitioning（Pos）优化器状态：减少4倍内存，通信量与数据并行性相同
#### Zero-2 仅用于训练
- 梯度：添加梯度分区（Pos+g）：减少8倍内存，通信量与数据并行性相同
#### Zero-2 可用于推理和训练
- 参数分区：添加参数分区（Pos+g+p）：内存减少与数据并行度Nd呈线性关系。例如，在64个GPU（Nd=64）之间进行拆分将产生64倍的内存缩减。通信量有50%的适度增长。
#### ZeRO-Infinity (CPU and NVME offload)
