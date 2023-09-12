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
## Accelerate:Hugging Face Pytorch GPU多机多卡加速器
- DeepSpeed
- https://huggingface.co/docs/accelerate/index

## 常用库
- transformers： https://huggingface.co/docs/transformers/index
- 

## bitsandbytes
- 轻量级的CUDA自定义函数包装器，特别用于PyTorch中的8位优化器、矩阵乘法（LLM.int8()）和量化函数。
- 8位优化器：bitsandbytes提供了8位优化器，用于在深度学习模型中进行训练和优化。您可以使用8位优化器替换torch.optim中的优化器，并通过修改代码中的相应部分来配置和使用它们。
- LLM.int8()矩阵乘法：bitsandbytes提供了LLM.int8()函数，用于执行8位整数矩阵乘法操作。这种矩阵乘法的目的是在保持计算精度的同时减少内存和计算需求。
- 量化函数：bitsandbytes还提供了一些用于量化和处理数据的函数，以提高模型的效率和性能。
