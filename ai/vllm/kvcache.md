/home/neo/workspace/vllm/vllm/v1/core/single_type_kv_cache_manager.py
/home/neo/workspace/vllm/vllm/v1/kv_cache_spec_registry.py
/home/neo/workspace/vllm/vllm/v1/kv_cache_interface.py
/home/neo/workspace/vllm/vllm/v1/core/single_type_kv_cache_manager.py
vllm_config = VllmConfig(model_config=ModelConfig(model='/home/neo/.cache/huggingface/hub/models--cyankiwi--gemma-4-31B-it-AWQ-4bit/snapshots/6f1b616c64bca2a05d67fdd6437c5500a8509327',
model_weights='',
runner='auto',
convert='auto',
tokenizer='/home/neo/.cache/huggingface/hub/models--cyankiwi--gemma-4-31B-it-AWQ-4bit/snapshots/6f1b616c64bca2a05d67fdd6437c5500a8509327',
tokenizer_mode='auto',
trust_remote_code=False,
dtype=torch.float16,
seed=0,
hf_config=Gemma4Config {
 "architectures": [
 "Gemma4ForConditionalGeneration"
 ],
 "audio_config": null,
 "audio_token_id": 258881,
 "boa_token_id": 256000,
 "boi_token_id": 255999,
 "dtype": "float16",
 "eoa_token_id": 258883,
 "eoa_token_index": 258883,
 "eoi_token_id": 258882,
 "eos_token_id": [
 1,
 106
 ],
 "image_token_id": 258880,
 "initializer_range": 0.02,
 "model_type": "gemma4",
 "quantization_config": {
 "config_groups": {
 "group_0": {
 "format": "pack-quantized",
 "input_activations": null,
 "output_activations": null,
 "targets": [
 "Linear"
 ],
 "weights": {
 "actorder": null,
 "block_structure": null,
 "dynamic": false,
 "group_size": 32,
 "num_bits": 4,
 "observer": "mse",
 "observer_kwargs": {},
 "scale_dtype": null,
 "strategy": "group",
 "symmetric": false,
 "type": "int",
 "zp_dtype": "torch.int8"
 }
 }
 },
 "format": "pack-quantized",
 "global_compression_ratio": null,
 "ignore": [
 "model.vision_tower.patch_embedder.input_proj",
 "model.vision_tower.encoder.layers.0.self_attn.q_proj.linear",
 "model.vision_tower.encoder.layers.0.self_attn.k_proj.linear",
 "model.vision_tower.encoder.layers.0.self_attn.v_proj.linear",
 "model.vision_tower.encoder.layers.0.self_attn.o_proj.linear",
 "model.vision_tower.encoder.layers.0.mlp.gate_proj.linear",
 "model.vision_tower.encoder.layers.0.mlp.up_proj.linear",
 "model.vision_tower.encoder.layers.0.mlp.down_proj.linear",
 "model.vision_tower.encoder.layers.1.self_attn.q_proj.linear",
 "model.vision_tower.encoder.layers.1.self_attn.k_proj.linear",
 "model.vision_tower.encoder.layers.1.self_attn.v_proj.linear",
 "model.vision_tower.encoder.layers.1.self_attn.o_proj.linear",
 "model.vision_tower.encoder.layers.1.mlp.gate_proj.linear",
 "model.vision_tower.encoder.layers.1.mlp.up_proj.linear",
 "model.vision_tower.encoder.layers.1.mlp.down_proj.linear",
 "model.vision_tower.encoder.layers.2.self_attn.q_proj.linear",
 "model.vision_tower.encoder.layers.2.self_attn.k_proj.linear",
 "model.vision_tower.encoder.layers.2.self_attn.v_proj.linear",
 "model.vision_tower.encoder.layers.2.self_attn.o_proj.linear",
 "model.vision_tower.encoder.layers.2.mlp.gate_proj.linear",
 "model.vision_tower.encoder.layers.2.mlp.up_proj.linear",
 "model.vision_tower.encoder.layers.2.mlp.down_proj.linear",
 "model.vision_tower.encoder.layers.3.self_attn.q_proj.linear",
 "model.vision_tower.encoder.layers.3.self_attn.k_proj.linear",
 "model.vision_tower.encoder.layers.3.self_attn.v_proj.linear",
 "model.vision_tower.encoder.layers.3.self_attn.o_proj.linear",
 "model.vision_tower.encoder.layers.3.mlp.gate_proj.linear",
 "model.vision_tower.encoder.layers.3.mlp.up_proj.linear",
 "model.vision_tower.encoder.layers.3.mlp.down_proj.linear",
 "model.vision_tower.encoder.layers.4.self_attn.q_proj.linear",
 "model.vision_tower.encoder.layers.4.self_attn.k_proj.linear",
 "model.vision_tower.encoder.layers.4.self_attn.v_proj.linear",
 "model.vision_tower.encoder.layers.4.self_attn.o_proj.linear",
 "model.vision_tower.encoder.layers.4.mlp.gate_proj.linear",
 "model.vision_tower.encoder.layers.4.mlp.up_proj.linear",
 "model.vision_tower.encoder.layers.4.mlp.down_proj.linear",
 "model.vision_tower.encoder.layers.5.self_attn.q_proj.linear",
 "model.vision_tower.encoder.layers.5.self_attn.k_proj.linear",
 "model.vision_tower.encoder.layers.5.self_attn.v_proj.linear",
 "model.vision_tower.encoder.layers.5.self_attn.o_proj.linear",
 "model.vision_tower.encoder.layers.5.mlp.gate_proj.linear",
 "model.vision_tower.encoder.layers.5.mlp.up_proj.linear",
 "model.vision_tower.encoder.layers.5.mlp.down_proj.linear",
 "model.vision_tower.encoder.layers.6.self_attn.q_proj.linear",
 "model.vision_tower.encoder.layers.6.self_attn.k_proj.linear",
 "model.vision_tower.encoder.layers.6.self_attn.v_proj.linear",
 "model.vision_tower.encoder.layers.6.self_attn.o_proj.linear",
 "model.vision_tower.encoder.layers.6.mlp.gate_proj.linear",
 "model.vision_tower.encoder.layers.6.mlp.up_proj.linear",
 "model.vision_tower.encoder.layers.6.mlp.down_proj.linear",
 "model.vision_tower.encoder.layers.7.self_attn.q_proj.linear",
 "model.vision_tower.encoder.layers.7.self_attn.k_proj.linear",
 "model.vision_tower.encoder.layers.7.self_attn.v_proj.linear",
 "model.vision_tower.encoder.layers.7.self_attn.o_proj.linear",
 "model.vision_tower.encoder.layers.7.mlp.gate_proj.linear",
 "model.vision_tower.encoder.layers.7.mlp.up_proj.linear",
 "model.vision_tower.encoder.layers.7.mlp.down_proj.linear",
 "model.vision_tower.encoder.layers.8.self_attn.q_proj.linear",
 "model.vision_tower.encoder.layers.8.self_attn.k_proj.linear",
 "model.vision_tower.encoder.layers.8.self_attn.v_proj.linear",
 "model.vision_tower.encoder.layers.8.self_attn.o_proj.linear",
 "model.vision_tower.encoder.layers.8.mlp.gate_proj.linear",
 "model.vision_tower.encoder.layers.8.mlp.up_proj.linear",
 "model.vision_tower.encoder.layers.8.mlp.down_proj.linear",
 "model.vision_tower.encoder.layers.9.self_attn.q_proj.linear",
 "model.vision_tower.encoder.layers.9.self_attn.k_proj.linear",
 "model.vision_tower.encoder.layers.9.self_attn.v_proj.linear",
 "model.vision_tower.encoder.layers.9.self_attn.o_proj.linear",
 "model.vision_tower.encoder.layers.9.mlp.gate_proj.linear",
 "model.vision_tower.encoder.layers.9.mlp.up_proj.linear",
 "model.vision_tower.encoder.layers.9.mlp.down_proj.linear",
 "model.vision_tower.encoder.layers.10.self_attn.q_proj.linear",
 "model.vision_tower.encoder.layers.10.self_attn.k_proj.linear",
 "model.vision_tower.encoder.layers.10.self_attn.v_proj.linear",
 "model.vision_tower.encoder.layers.10.self_attn.o_proj.linear",
 "model.vision_tower.encoder.layers.10.mlp.gate_proj.linear",
 "model.vision_tower.encoder.layers.10.mlp.up_proj.linear",
 "model.vision_tower.encoder.layers.10.mlp.down_proj.linear",
 "model.vision_tower.encoder.layers.11.self_attn.q_proj.linear",
 "model.vision_tower.encoder.layers.11.self_attn.k_proj.linear",
 "model.vision_tower.encoder.layers.11.self_attn.v_proj.linear",
 "model.vision_tower.encoder.layers.11.self_attn.o_proj.linear",
 "model.vision_tower.encoder.layers.11.mlp.gate_proj.linear",
 "model.vision_tower.encoder.layers.11.mlp.up_proj.linear",
 "model.vision_tower.encoder.layers.11.mlp.down_proj.linear",
 "model.vision_tower.encoder.layers.12.self_attn.q_proj.linear",
 "model.vision_tower.encoder.layers.12.self_attn.k_proj.linear",
 "model.vision_tower.encoder.layers.12.self_attn.v_proj.linear",
 "model.vision_tower.encoder.layers.12.self_attn.o_proj.linear",
 "model.vision_tower.encoder.layers.12.mlp.gate_proj.linear",
 "model.vision_tower.encoder.layers.12.mlp.up_proj.linear",
 "model.vision_tower.encoder.layers.12.mlp.down_proj.linear",
 "model.vision_tower.encoder.layers.13.self_attn.q_proj.linear",
 "model.vision_tower.encoder.layers.13.self_attn.k_proj.linear",
 "model.vision_tower.encoder.layers.13.self_attn.v_proj.linear",
 "model.vision_tower.encoder.layers.13.self_attn.o_proj.linear",
 "model.vision_tower.encoder.layers.13.mlp.gate_proj.linear",
 "model.vision_tower.encoder.layers.13.mlp.up_proj.linear",
 "model.vision_tower.encoder.layers.13.mlp.down_proj.linear",
 "model.vision_tower.encoder.layers.14.self_attn.q_proj.linear",
 "model.vision_tower.encoder.layers.14.self_attn.k_proj.linear",
 "model.vision_tower.encoder.layers.14.self_attn.v_proj.linear",
 "model.vision_tower.encoder.layers.14.self_attn.o_proj.linear",
 "model.vision_tower.encoder.layers.14.mlp.gate_proj.linear",
 "model.vision_tower.encoder.layers.14.mlp.up_proj.linear",
 "model.vision_tower.encoder.layers.14.mlp.down_proj.linear",
 "model.vision_tower.encoder.layers.15.self_attn.q_proj.linear",
 "model.vision_tower.encoder.layers.15.self_attn.k_proj.linear",
 "model.vision_tower.encoder.layers.15.self_attn.v_proj.linear",
 "model.vision_tower.encoder.layers.15.self_attn.o_proj.linear",
 "model.vision_tower.encoder.layers.15.mlp.gate_proj.linear",
 "model.vision_tower.encoder.layers.15.mlp.up_proj.linear",
 "model.vision_tower.encoder.layers.15.mlp.down_proj.linear",
 "model.vision_tower.encoder.layers.16.self_attn.q_proj.linear",
 "model.vision_tower.encoder.layers.16.self_attn.k_proj.linear",
 "model.vision_tower.encoder.layers.16.self_attn.v_proj.linear",
 "model.vision_tower.encoder.layers.16.self_attn.o_proj.linear",
 "model.vision_tower.encoder.layers.16.mlp.gate_proj.linear",
 "model.vision_tower.encoder.layers.16.mlp.up_proj.linear",
 "model.vision_tower.encoder.layers.16.mlp.down_proj.linear",
 "model.vision_tower.encoder.layers.17.self_attn.q_proj.linear",
 "model.vision_tower.encoder.layers.17.self_attn.k_proj.linear",
 "model.vision_tower.encoder.layers.17.self_attn.v_proj.linear",
 "model.vision_tower.encoder.layers.17.self_attn.o_proj.linear",
 "model.vision_tower.encoder.layers.17.mlp.gate_proj.linear",
 "model.vision_tower.encoder.layers.17.mlp.up_proj.linear",
 "model.vision_tower.encoder.layers.17.mlp.down_proj.linear",
 "model.vision_tower.encoder.layers.18.self_attn.q_proj.linear",
 "model.vision_tower.encoder.layers.18.self_attn.k_proj.linear",
 "model.vision_tower.encoder.layers.18.self_attn.v_proj.linear",
 "model.vision_tower.encoder.layers.18.self_attn.o_proj.linear",
 "model.vision_tower.encoder.layers.18.mlp.gate_proj.linear",
 "model.vision_tower.encoder.layers.18.mlp.up_proj.linear",
 "model.vision_tower.encoder.layers.18.mlp.down_proj.linear",
 "model.vision_tower.encoder.layers.19.self_attn.q_proj.linear",
 "model.vision_tower.encoder.layers.19.self_attn.k_proj.linear",
 "model.vision_tower.encoder.layers.19.self_attn.v_proj.linear",
 "model.vision_tower.encoder.layers.19.self_attn.o_proj.linear",
 "model.vision_tower.encoder.layers.19.mlp.gate_proj.linear",
 "model.vision_tower.encoder.layers.19.mlp.up_proj.linear",
 "model.vision_tower.encoder.layers.19.mlp.down_proj.linear",
 "model.vision_tower.encoder.layers.20.self_attn.q_proj.linear",
 "model.vision_tower.encoder.layers.20.self_attn.k_proj.linear",
 "model.vision_tower.encoder.layers.20.self_attn.v_proj.linear",
 "model.vision_tower.encoder.layers.20.self_attn.o_proj.linear",
 "model.vision_tower.encoder.layers.20.mlp.gate_proj.linear",
 "model.vision_tower.encoder.layers.20.mlp.up_proj.linear",
 "model.vision_tower.encoder.layers.20.mlp.down_proj.linear",
 "model.vision_tower.encoder.layers.21.self_attn.q_proj.linear",
 "model.vision_tower.encoder.layers.21.self_attn.k_proj.linear",
 "model.vision_tower.encoder.layers.21.self_attn.v_proj.linear",
 "model.vision_tower.encoder.layers.21.self_attn.o_proj.linear",
 "model.vision_tower.encoder.layers.21.mlp.gate_proj.linear",
 "model.vision_tower.encoder.layers.21.mlp.up_proj.linear",
 "model.vision_tower.encoder.layers.21.mlp.down_proj.linear",
 "model.vision_tower.encoder.layers.22.self_attn.q_proj.linear",
 "model.vision_tower.encoder.layers.22.self_attn.k_proj.linear",
 "model.vision_tower.encoder.layers.22.self_attn.v_proj.linear",
 "model.vision_tower.encoder.layers.22.self_attn.o_proj.linear",
 "model.vision_tower.encoder.layers.22.mlp.gate_proj.linear",
 "model.vision_tower.encoder.layers.22.mlp.up_proj.linear",
 "model.vision_tower.encoder.layers.22.mlp.down_proj.linear",
 "model.vision_tower.encoder.layers.23.self_attn.q_proj.linear",
 "model.vision_tower.encoder.layers.23.self_attn.k_proj.linear",
 "model.vision_tower.encoder.layers.23.self_attn.v_proj.linear",
 "model.vision_tower.encoder.layers.23.self_attn.o_proj.linear",
 "model.vision_tower.encoder.layers.23.mlp.gate_proj.linear",
 "model.vision_tower.encoder.layers.23.mlp.up_proj.linear",
 "model.vision_tower.encoder.layers.23.mlp.down_proj.linear",
 "model.vision_tower.encoder.layers.24.self_attn.q_proj.linear",
 "model.vision_tower.encoder.layers.24.self_attn.k_proj.linear",
 "model.vision_tower.encoder.layers.24.self_attn.v_proj.linear",
 "model.vision_tower.encoder.layers.24.self_attn.o_proj.linear",
 "model.vision_tower.encoder.layers.24.mlp.gate_proj.linear",
 "model.vision_tower.encoder.layers.24.mlp.up_proj.linear",
 "model.vision_tower.encoder.layers.24.mlp.down_proj.linear",
 "model.vision_tower.encoder.layers.25.self_attn.q_proj.linear",
 "model.vision_tower.encoder.layers.25.self_attn.k_proj.linear",
 "model.vision_tower.encoder.layers.25.self_attn.v_proj.linear",
 "model.vision_tower.encoder.layers.25.self_attn.o_proj.linear",
 "model.vision_tower.encoder.layers.25.mlp.gate_proj.linear",
 "model.vision_tower.encoder.layers.25.mlp.up_proj.linear",
 "model.vision_tower.encoder.layers.25.mlp.down_proj.linear",
 "model.vision_tower.encoder.layers.26.self_attn.q_proj.linear",
 "model.vision_tower.encoder.layers.26.self_attn.k_proj.linear",
 "model.vision_tower.encoder.layers.26.self_attn.v_proj.linear",
 "model.vision_tower.encoder.layers.26.self_attn.o_proj.linear",
 "model.vision_tower.encoder.layers.26.mlp.gate_proj.linear",
 "model.vision_tower.encoder.layers.26.mlp.up_proj.linear",
 "model.vision_tower.encoder.layers.26.mlp.down_proj.linear",
 "model.embed_vision.embedding_projection",
 "lm_head"
 ],
 "kv_cache_scheme": null,
 "quant_method": "compressed-tensors",
 "quantization_status": "compressed",
 "sparsity_config": {},
 "total_num_heads": 32,
 "total_num_kv_heads": 16,
 "transform_config": {},
 "version": "0.14.1.a20260326"
 },
 "text_config": {
 "attention_bias": false,
 "attention_dropout": 0.0,
 "attention_k_eq_v": true,
 "bos_token_id": 2,
 "dtype": "float16",
 "enable_moe_block": false,
 "eos_token_id": 1,
 "expert_intermediate_size": null,
 "final_logit_softcapping": 30.0,
 "head_dim": 256,
 "hidden_activation": "gelu_pytorch_tanh",
 "hidden_size": 5376,
 "hidden_size_per_layer_input": 0,
 "initializer_range": 0.02,
 "intermediate_size": 21504,
 "layer_types": [
 "sliding_attention",
 "sliding_attention",
 "sliding_attention",
 "sliding_attention",
 "sliding_attention",
 "full_attention",
 "sliding_attention",
 "sliding_attention",
 "sliding_attention",
 "sliding_attention",
 "sliding_attention",
 "full_attention",
 "sliding_attention",
 "sliding_attention",
 "sliding_attention",
 "sliding_attention",
 "sliding_attention",
 "full_attention",
 "sliding_attention",
 "sliding_attention",
 "sliding_attention",
 "sliding_attention",
 "sliding_attention",
 "full_attention",
 "sliding_attention",
 "sliding_attention",
 "sliding_attention",
 "sliding_attention",
 "sliding_attention",
 "full_attention",
 "sliding_attention",
 "sliding_attention",
 "sliding_attention",
 "sliding_attention",
 "sliding_attention",
 "full_attention",
 "sliding_attention",
 "sliding_attention",
 "sliding_attention",
 "sliding_attention",
 "sliding_attention",
 "full_attention",
 "sliding_attention",
 "sliding_attention",
 "sliding_attention",
 "sliding_attention",
 "sliding_attention",
 "full_attention",
 "sliding_attention",
 "sliding_attention",
 "sliding_attention",
 "sliding_attention",
 "sliding_attention",
 "full_attention",
 "sliding_attention",
 "sliding_attention",
 "sliding_attention",
 "sliding_attention",
 "sliding_attention",
 "full_attention"
 ],
 "max_position_embeddings": 262144,
 "model_type": "gemma4_text",
 "moe_intermediate_size": null,
 "num_attention_heads": 32,
 "num_experts": null,
 "num_hidden_layers": 60,
 "num_key_value_heads": 16,
 "num_kv_shared_layers": 0,
 "pad_token_id": 0,
 "per_layer_config": {
 "05": {
 "head_dim": 512,
 "num_key_value_heads": 4
 },
 "11": {
 "head_dim": 512,
 "num_key_value_heads": 4
 },
 "17": {
 "head_dim": 512,
 "num_key_value_heads": 4
 },
 "23": {
 "head_dim": 512,
 "num_key_value_heads": 4
 },
 "29": {
 "head_dim": 512,
 "num_key_value_heads": 4
 },
 "35": {
 "head_dim": 512,
 "num_key_value_heads": 4
 },
 "41": {
 "head_dim": 512,
 "num_key_value_heads": 4
 },
 "47": {
 "head_dim": 512,
 "num_key_value_heads": 4
 },
 "53": {
 "head_dim": 512,
 "num_key_value_heads": 4
 },
 "59": {
 "head_dim": 512,
 "num_key_value_heads": 4
 }
 },
 "rms_norm_eps": 1e-06,
 "rope_parameters": {
 "full_attention": {
 "partial_rotary_factor": 0.25,
 "rope_theta": 1000000.0,
 "rope_type": "proportional"
 },
 "sliding_attention": {
 "rope_theta": 10000.0,
 "rope_type": "default"
 }
 },
 "sliding_window": 1024,
 "tie_word_embeddings": true,
 "top_k_experts": null,
 "use_bidirectional_attention": "vision",
 "use_cache": true,
 "use_double_wide_mlp": false,
 "vocab_size": 262144,
 "vocab_size_per_layer_input": 262144
 },
 "tie_word_embeddings": true,
 "transformers_version": "5.23.0",
 "video_token_id": 258884,
 "vision_config": {
 "attention_bias": false,
 "attention_dropout": 0.0,
 "default_output_length": 280,
 "dtype": "bfloat16",
 "global_head_dim": 72,
 "head_dim": 72,
 "hidden_activation": "gelu_pytorch_tanh",
 "hidden_size": 1152,
 "initializer_range": 0.02,
 "intermediate_size": 4304,
 "max_position_embeddings": 131072,
 "model_type": "gemma4_vision",
 "num_attention_heads": 16,
 "num_hidden_layers": 27,
 "num_key_value_heads": 16,
 "patch_size": 16,
 "pooling_kernel_size": 3,
 "position_embedding_size": 10240,
 "rms_norm_eps": 1e-06,
 "rope_parameters": {
 "rope_theta": 100.0,
 "rope_type": "axial"
 },
 "standardize": true,
 "use_clipped_linears": false
 },
 "vision_soft_tokens_per_image": 280
 },
hf_text_config=Gemma4TextConfig {
 "attention_bias": false,
 "attention_dropout": 0.0,
 "attention_k_eq_v": true,
 "bos_token_id": 2,
 "dtype": "float16",
 "enable_moe_block": false,
 "eos_token_id": 1,
 "expert_intermediate_size": null,
 "final_logit_softcapping": 30.0,
 "head_dim": 256,
 "hidden_activation": "gelu_pytorch_tanh",
 "hidden_size": 5376,
 "hidden_size_per_layer_input": 0,
 "initializer_range": 0.02,
 "intermediate_size": 21504,
 "layer_types": [
 "sliding_attention",
 "sliding_attention",
 "sliding_attention",
 "sliding_attention",
 "sliding_attention",
 "full_attention",
 "sliding_attention",
 "sliding_attention",
 "sliding_attention",
 "sliding_attention",
 "sliding_attention",
 "full_attention",
 "sliding_attention",
 "sliding_attention",
 "sliding_attention",
 "sliding_attention",
 "sliding_attention",
 "full_attention",
 "sliding_attention",
 "sliding_attention",
 "sliding_attention",
 "sliding_attention",
 "sliding_attention",
 "full_attention",
 "sliding_attention",
 "sliding_attention",
 "sliding_attention",
 "sliding_attention",
 "sliding_attention",
 "full_attention",
 "sliding_attention",
 "sliding_attention",
 "sliding_attention",
 "sliding_attention",
 "sliding_attention",
 "full_attention",
 "sliding_attention",
 "sliding_attention",
 "sliding_attention",
 "sliding_attention",
 "sliding_attention",
 "full_attention",
 "sliding_attention",
 "sliding_attention",
 "sliding_attention",
 "sliding_attention",
 "sliding_attention",
 "full_attention",
 "sliding_attention",
 "sliding_attention",
 "sliding_attention",
 "sliding_attention",
 "sliding_attention",
 "full_attention",
 "sliding_attention",
 "sliding_attention",
 "sliding_attention",
 "sliding_attention",
 "sliding_attention",
 "full_attention"
 ],
 "max_position_embeddings": 262144,
 "model_type": "gemma4_text",
 "moe_intermediate_size": null,
 "num_attention_heads": 32,
 "num_experts": null,
 "num_hidden_layers": 60,
 "num_key_value_heads": 16,
 "num_kv_shared_layers": 0,
 "pad_token_id": 0,
 "per_layer_config": {
 "05": {
 "head_dim": 512,
 "num_key_value_heads": 4
 },
 "11": {
 "head_dim": 512,
 "num_key_value_heads": 4
 },
 "17": {
 "head_dim": 512,
 "num_key_value_heads": 4
 },
 "23": {
 "head_dim": 512,
 "num_key_value_heads": 4
 },
 "29": {
 "head_dim": 512,
 "num_key_value_heads": 4
 },
 "35": {
 "head_dim": 512,
 "num_key_value_heads": 4
 },
 "41": {
 "head_dim": 512,
 "num_key_value_heads": 4
 },
 "47": {
 "head_dim": 512,
 "num_key_value_heads": 4
 },
 "53": {
 "head_dim": 512,
 "num_key_value_heads": 4
 },
 "59": {
 "head_dim": 512,
 "num_key_value_heads": 4
 }
 },
 "rms_norm_eps": 1e-06,
 "rope_parameters": {
 "full_attention": {
 "partial_rotary_factor": 0.25,
 "rope_theta": 1000000.0,
 "rope_type": "proportional"
 },
 "sliding_attention": {
 "rope_theta": 10000.0,
 "rope_type": "default"
 }
 },
 "sliding_window": 1024,
 "tie_word_embeddings": true,
 "top_k_experts": null,
 "transformers_version": "5.23.0",
 "use_bidirectional_attention": "vision",
 "use_cache": true,
 "use_double_wide_mlp": false,
 "vocab_size": 262144,
 "vocab_size_per_layer_input": 262144
 },
word_embeddings_untied_by_checkpoint=False,
hf_config_path=None,
allowed_local_media_path='',
allowed_media_domains=None,
revision=None,
code_revision=None,
tokenizer_revision=None,
max_model_len=16000,
spec_target_max_model_len=None,
quantization='compressed-tensors',
quantization_config=None,
allow_deprecated_quantization=False,
enforce_eager=False,
enable_return_routed_experts=False,
return_sampling_mask=False,
max_logprobs=20,
logprobs_mode='raw_logprobs',
use_fp64_gumbel=False,
enable_trace_replay=False,
disable_sliding_window=False,
disable_cascade_attn=True,
skip_tokenizer_init=False,
enable_prompt_embeds=False,
served_model_name='debug-model',
config_format='auto',
hf_token=None,
hf_overrides={},
model_class_overrides={},
generation_config='auto',
override_generation_config={},
enable_sleep_mode=False,
sleep_mode_backend='cumem',
enable_nccl_comm_suspend=False,
enable_cumem_allocator=False,
model_impl='auto',
logits_processors=None,
io_processor_plugin=None,
renderer_num_workers=1,
pooler_config=None,
multimodal_config=MultiModalConfig(language_model_only=False,
limit_per_prompt={'video': VideoDummyOptions(count=0,
num_frames=None,
width=None,
height=None)},
enable_mm_embeds=False,
media_io_kwargs={},
mm_processor_kwargs=None,
mm_device_do_normalize=False,
mm_processor_cache_gb=4.0,
mm_processor_cache_type='lru',
mm_hasher_algorithm='blake3',
mm_shm_cache_max_object_size_mb=128,
mm_encoder_only=False,
mm_encoder_tp_mode='weights',
mm_encoder_attn_backend=None,
mm_encoder_attn_dtype=None,
mm_encoder_fp8_scale_path=None,
mm_encoder_fp8_scale_save_path=None,
mm_encoder_fp8_scale_save_margin=1.5,
interleave_mm_strings=False,
skip_mm_profiling=False,
video_pruning_rate=None,
video_pruning_method='evs',
mm_tensor_ipc='direct_rpc',
allow_missing_mm_embeddings=False,
mm_ipc_gpu_memory_gb=0.0)),
cache_config=CacheConfig(block_size=16,
user_specified_block_size=False,
user_specified_mamba_block_size=False,
kv_cache_layout='LBNHC',
prefix_match_unit=None,
gpu_memory_utilization=0.9,
cache_dtype='auto',
is_attention_free=False,
num_gpu_blocks_override=None,
sliding_window=None,
enable_prefix_caching=True,
prefix_caching_hash_algo='sha256',
prefix_cache_retention_interval=0,
kv_cache_dtype_skip_layers=[],
mamba_page_size_padded=None,
skip_page_size_padded=None,
mamba_block_size=None,
mamba_cache_dtype='auto',
mamba_ssm_cache_dtype='auto',
mamba_cache_mode='none',
enable_mamba_fine_grained_prefix_cache=False,
replayssm_buffer_len=16,
use_replayssm=False,
use_kda_recoverssm=False,
num_gpu_blocks=2454,
num_cpu_blocks=None,
kv_cache_size_tokens=16462,
kv_cache_max_concurrency=1.0289308176100629,
kv_sharing_fast_prefill=False,
kv_cache_memory_bytes=None,
kv_offloading_size=None,
kv_offloading_backend='native',
_block_size_resolved=True),
parallel_config=ParallelConfig(pipeline_parallel_size=1,
tensor_parallel_size=1,
prefill_context_parallel_size=1,
data_parallel_size=1,
data_parallel_size_local=1,
data_parallel_rank=0,
data_parallel_rank_local=0,
data_parallel_master_ip='127.0.0.1',
data_parallel_rpc_port=29550,
dp_sync_interval=16,
data_parallel_master_port=0,
data_parallel_backend='mp',
data_parallel_external_lb=False,
data_parallel_hybrid_lb=False,
is_moe_model=False,
enable_expert_parallel=False,
enable_batch_sharded_sampling=False,
enable_ep_weight_filter=False,
enable_eplb=False,
eplb_config=EPLBConfig(window_size=1000,
step_interval=3000,
num_redundant_experts=0,
log_balancedness=False,
log_balancedness_interval=1,
use_async=True,
policy='default',
communicator=None),
expert_placement_strategy='linear',
all2all_backend='allgather_reducescatter',
max_parallel_loading_workers=None,
disable_custom_all_reduce=False,
enable_elastic_ep=False,
elastic_ep_max_dp_size=1,
enable_dbo=False,
ubatch_size=0,
dbo_decode_token_threshold=32,
dbo_prefill_token_threshold=512,
disable_nccl_for_dp_synchronization=True,
ray_workers_use_nsight=False,
ray_runtime_env=None,
placement_group=None,
distributed_executor_backend='uni',
worker_cls='vllm.v1.worker.gpu_worker.Worker',
sd_worker_cls='auto',
worker_extension_cls='',
master_addr='127.0.0.1',
master_port=29501,
node_rank=0,
nnodes=1,
numa_bind=False,
numa_bind_nodes=None,
numa_bind_cpus=None,
assigned_physical_gpu_ids=None,
distributed_timeout_seconds=None,
cpu_distributed_timeout_seconds=None,
world_size=1,
rank=0,
_data_parallel_master_port_list=[],
_coord_store_port=0,
decode_context_parallel_size=1,
dcp_kv_cache_interleave_size=1,
dcp_comm_backend='ag_rs',
dcp_q_replicate=False,
cp_kv_cache_interleave_size=1,
_allow_auto_resolve_cp_interleave_size=True,
data_parallel_index=0,
_api_process_count=1,
_api_process_rank=0,
enable_fault_tolerance=False,
fault_tolerance_config=FaultToleranceConfig(engine_recovery_timeout_sec=120)),
scheduler_config=SchedulerConfig(runner_type='generate',
max_num_batched_tokens=2496,
max_num_scheduled_tokens=None,
max_num_seqs=256,
long_prefill_token_threshold=0,
max_num_queued_reqs=None,
max_num_queued_tokens=None,
enable_chunked_prefill=True,
is_multimodal_model=True,
max_num_encoder_input_tokens=2496,
encoder_cache_size=2496,
policy='fcfs',
disable_chunked_mm_input=True,
scheduler_cls=None,
disable_hybrid_kv_cache_manager=False,
scheduler_reserve_full_isl=True,
watermark=0.0,
prefill_schedule_interval=1,
async_scheduling=True,
stream_interval=1),
device_config=DeviceConfig(device=device(type='cuda'),
device_type='cuda'),
load_config=LoadConfig(load_format='auto',
download_dir=None,
safetensors_load_strategy=None,
safetensors_prefetch_num_threads=8,
safetensors_prefetch_block_size=16777216,
model_loader_extra_config={},
device=None,
ignore_patterns=['original/**/*'],
use_tqdm_on_load=True,
pt_load_map_location='cpu'),
offload_config=OffloadConfig(offload_backend='prefetch',
uva=UVAOffloadConfig(cpu_offload_gb=12.0,
cpu_offload_params=set()),
prefetch=PrefetchOffloadConfig(offload_group_size=8,
offload_num_in_group=4,
offload_prefetch_step=1,
offload_params=set())),
attention_config=AttentionConfig(backend=<AttentionBackendEnum.TRITON_ATTN: 'vllm.v1.attention.backends.triton_attn.TritonAttentionBackend'>,
minimax_m3_msa_decode_backend='triton',
backend_per_kind={},
flash_attn_version=None,
flash_attn_max_num_splits_for_cuda_graph=32,
tq_max_kv_splits_for_cuda_graph=32,
use_trtllm_attention=None,
disable_flashinfer_q_quantization=False,
mla_prefill_backend=None,
use_prefill_query_quantization=False,
indexer_kv_dtype='auto',
hisparse_config=None,
use_non_causal=False,
sparse_mla_force_mqa=False,
flex_attn_block_m=None,
flex_attn_block_n=None,
flex_attn_q_block_size=None,
flex_attn_kv_block_size=None),
engram_config=None,
mamba_config=MambaConfig(backend=<MambaBackendEnum.TRITON: 'triton'>,
enable_stochastic_rounding=False,
stochastic_rounding_philox_rounds=0,
ssu_algorithm=None),
kernel_config=KernelConfig(ir_op_priority=IrOpPriorityConfig(rms_norm=['native'],
fused_add_rms_norm=['native'],
gelu_and_mul_sparse=['triton', 'native']),
enable_flashinfer_autotune=True,
enable_cutedsl_warmup=True,
enable_jit_warmup=True,
moe_backend='auto',
sparse_indexer_topk_backend='auto',
linear_backend='auto',
linear_backend_per_quant=None),
lora_config=None,
speculative_config=None,
watermark_config=None,
diffusion_config=None,
structured_outputs_config=StructuredOutputsConfig(backend='auto',
disable_any_whitespace=False,
disable_additional_properties=False,
reasoning_parser='',
reasoning_parser_plugin='',
enable_in_reasoning=False),
observability_config=ObservabilityConfig(show_hidden_metrics_for_version=None,
otlp_traces_endpoint=None,
collect_detailed_traces=None,
per_request_spec_decode_metrics='none',
kv_cache_metrics=False,
kv_cache_metrics_sample=0.01,
cudagraph_metrics=False,
enable_layerwise_nvtx_tracing=False,
enable_mfu_metrics=False,
enable_mm_processor_stats=False,
enable_logging_iteration_details=False,
jit_monitor_mode='warn',
jit_monitor_verbose=False),
quant_config=<vllm.model_executor.layers.quantization.compressed_tensors.compressed_tensors.CompressedTensorsConfig object at 0x753db0170950>,
compilation_config={'mode': <CompilationMode.VLLM_COMPILE: 3>, 'debug_dump_path': None, 'cache_dir': '', 'compile_cache_save_format': 'binary', 'backend': 'inductor', 'custom_ops': ['none'], 'ir_enable_torch_wrap': True, 'splitting_ops': ['vllm::unified_attention_with_output', 'vllm::unified_mla_attention_with_output', 'vllm::mamba_mixer2', 'vllm::mamba_mixer', 'vllm::short_conv', 'vllm::qwen4_exp_ple_short_conv', 'vllm::qwen4_exp_qsa_with_output', 'vllm::linear_attention', 'vllm::qwen_gdn_attention_core', 'vllm::qwen_gdn_attention_core_fused_norm_packed', 'vllm::gdn_attention_core_xpu', 'vllm::olmo_hybrid_gdn_full_forward', 'vllm::sparse_attn_indexer', 'vllm::rocm_aiter_sparse_attn_indexer', 'vllm::deepseek_v4_attention', 'vllm::hpc_rope_norm_forward', 'vllm::unified_kv_cache_update', 'vllm::unified_mla_kv_cache_update'], 'compile_mm_encoder': False, 'cudagraph_mm_encoder': False, 'encoder_cudagraph_token_budgets': [], 'encoder_cudagraph_max_vision_items_per_batch': 0, 'encoder_cudagraph_max_frames_per_batch': None, 'compile_sizes': [], 'compile_ranges_endpoints': [2496], 'inductor_compile_config': {'enable_auto_functionalized_v2': False, 'combo_kernels': True, 'benchmark_combo_kernel': True}, 'inductor_passes': {}, 'cudagraph_mode': <CUDAGraphMode.FULL_AND_PIECEWISE: (2, 1)>, 'cudagraph_num_of_warmups': 1, 'cudagraph_capture_sizes': [1, 2, 4, 8, 16, 24, 32, 40, 48, 56, 64, 72, 80, 88, 96, 104, 112, 120, 128, 136, 144, 152, 160, 168, 176, 184, 192, 200, 208, 216, 224, 232, 240, 248, 256, 272, 288, 304, 320, 336, 352, 368, 384, 400, 416, 432, 448, 464, 480, 496, 512], 'cudagraph_copy_inputs': False, 'cudagraph_specialize_lora': True, 'use_inductor_graph_partition': False, 'pass_config': {'fuse_norm_quant': False, 'fuse_act_quant': False, 'fuse_attn_quant': False, 'enable_sp': False, 'fuse_gemm_comms': False, 'fuse_allreduce_rms': False, 'enable_qk_norm_rope_fusion': False, 'fuse_rope_kvcache_cat_mla': False, 'fuse_act_padding': False, 'fuse_qk_norm_rope_kvcache': False}, 'max_cudagraph_capture_size': 512, 'dynamic_shapes_config': {'type': <DynamicShapesType.BACKED: 'backed'>, 'evaluate_guards': False, 'assume_32_bit_indexing': False}, 'local_cache_dir': '/home/neo/.cache/vllm/torch_compile_cache/torch_aot_compile/f4f1fec5dc8563e5ca5c4ea9ca4c050a118b8c2d057c3b993390bbd1b5a3cd38', 'fast_moe_cold_start': False, 'static_all_moe_layers': []},
profiler_config=ProfilerConfig(profiler=None,
torch_profiler_dir='',
proton_profiler_dir='',
proton_context='shadow',
proton_data='tree',
proton_backend=None,
proton_mode=None,
proton_hook=None,
proton_output_format=None,
proton_graph_attribution=False,
torch_profiler_with_stack=True,
torch_profiler_with_flops=False,
torch_profiler_use_gzip=True,
torch_profiler_dump_cuda_time_total=True,
torch_profiler_record_shapes=False,
torch_profiler_with_memory=False,
capture_torch_profiler=False,
detailed_trace_annotation=False,
ignore_frontend=False,
delay_iterations=0,
max_iterations=0,
warmup_iterations=0,
active_iterations=5,
wait_iterations=0),
kv_transfer_config=None,
kv_events_config=None,
ec_transfer_config=None,
ec_manager_config=EncoderCacheManagerConfig(encoder_cache_manager_cls=None,
manager_config={}),
reasoning_config=None,
additional_config={},
instance_id='1795533739371249132',
optimization_level=<OptimizationLevel.O2: 2>,
performance_mode='balanced',
weight_transfer_config=None,
shutdown_timeout=0)

kv_cache_specs = self.model_executor.get_kv_cache_specs()
def get_kv_cache_specs(self) -> list[dict[str, KVCacheSpec]]:
        return self.collective_rpc("get_kv_cache_spec")
GPUModelRunner.get_kv_cache_spec

kv_cache_specs = [{'language_model.model.layers.0.self_attn.attn': SlidingWindowSpec(block_size=16,
num_kv_heads=16,
head_size=256,
dtype=torch.float16,
...ding_window=1024,
extra_retained_tokens=0),
'language_model.model.layers.1.self_attn.attn': SlidingWindowSpec(block_size=16,
num_kv_heads=16,
head_size=256,
dtype=torch.float16,
...ding_window=1024,
extra_retained_tokens=0),
'language_model.model.layers.2.self_attn.attn': SlidingWindowSpec(block_size=16,
num_kv_heads=16,
head_size=256,
dtype=torch.float16,
...ding_window=1024,
extra_retained_tokens=0),
'language_model.model.layers.3.self_attn.attn': SlidingWindowSpec(block_size=16,
num_kv_heads=16,
head_size=256,
dtype=torch.float16,
...ding_window=1024,
extra_retained_tokens=0),
'language_model.model.layers.4.self_attn.attn': SlidingWindowSpec(block_size=16,
num_kv_heads=16,
head_size=256,
dtype=torch.float16,
...ding_window=1024,
extra_retained_tokens=0),
'language_model.model.layers.5.self_attn.attn': FullAttentionSpec(block_size=16,
num_kv_heads=4,
head_size=512,
dtype=torch.float16,
...tention_chunk_size=None,
non_causal=False),
'language_model.model.layers.6.self_attn.attn': SlidingWindowSpec(block_size=16,
num_kv_heads=16,
head_size=256,
dtype=torch.float16,
...ding_window=1024,
extra_retained_tokens=0),
'language_model.model.layers.7.self_attn.attn': SlidingWindowSpec(block_size=16,
num_kv_heads=16,
head_size=256,
dtype=torch.float16,
...ding_window=1024,
extra_retained_tokens=0),
'language_model.model.layers.8.self_attn.attn': SlidingWindowSpec(block_size=16,
num_kv_heads=16,
head_size=256,
dtype=torch.float16,
...ding_window=1024,
extra_retained_tokens=0),
'language_model.model.layers.9.self_attn.attn': SlidingWindowSpec(block_size=16,
num_kv_heads=16,
head_size=256,
dtype=torch.float16,
...ding_window=1024,
extra_retained_tokens=0),
'language_model.model.layers.10.self_attn.attn': SlidingWindowSpec(block_size=16,
num_kv_heads=16,
head_size=256,
dtype=torch.float16,
...ding_window=1024,
extra_retained_tokens=0),
'language_model.model.layers.11.self_attn.attn': FullAttentionSpec(block_size=16,
num_kv_heads=4,
head_size=512,
dtype=torch.float16,
...tention_chunk_size=None,
non_causal=False),
'language_model.model.layers.12.self_attn.attn': SlidingWindowSpec(block_size=16,
num_kv_heads=16,
head_size=256,
dtype=torch.float16,
...ding_window=1024,
extra_retained_tokens=0),
'language_model.model.layers.13.self_attn.attn': SlidingWindowSpec(block_size=16,
num_kv_heads=16,
head_size=256,
dtype=torch.float16,
...ding_window=1024,
extra_retained_tokens=0),
'language_model.model.layers.14.self_attn.attn': SlidingWindowSpec(block_size=16,
num_kv_heads=16,
head_size=256,
dtype=torch.float16,
...ding_window=1024,
extra_retained_tokens=0),
'language_model.model.layers.15.self_attn.attn': SlidingWindowSpec(block_size=16,
num_kv_heads=16,
head_size=256,
dtype=torch.float16,
...ding_window=1024,
extra_retained_tokens=0),
'language_model.model.layers.16.self_attn.attn': SlidingWindowSpec(block_size=16,
num_kv_heads=16,
head_size=256,
dtype=torch.float16,
...ding_window=1024,
extra_retained_tokens=0),
'language_model.model.layers.17.self_attn.attn': FullAttentionSpec(block_size=16,
num_kv_heads=4,
head_size=512,
dtype=torch.float16,
...tention_chunk_size=None,
non_causal=False),
'language_model.model.layers.18.self_attn.attn': SlidingWindowSpec(block_size=16,
num_kv_heads=16,
head_size=256,
dtype=torch.float16,
...ding_window=1024,
extra_retained_tokens=0), ...}]

self.model_executor.get_supported_kv_cache_layouts(),
def get_supported_kv_cache_layouts(self) -> list[list[str]]:
        """Layouts each worker's backends support, most preferred first."""
        return self.collective_rpc("get_supported_kv_cache_layouts")
WorkerBase.get_supported_kv_cache_layouts
layout = <KVCacheLayout.LBNHC: (0, 1, 3, 2, 4)>

self.model_executor.set_kv_cache_layout(layout.name)
WorkerBase.set_kv_cache_layout

available_gpu_memory = self.model_executor.determine_available_memory()
def determine_available_memory(self) -> list[int]:  # in bytes
        return self.collective_rpc("determine_available_memory")
Worker.determine_available_memory
torch.compile + Capturing CUDA graphs
available_gpu_memory = [6435619021]

self.model_executor.initialize_from_config(kv_cache_configs)
def initialize_from_config(self, kv_cache_configs: list[KVCacheConfig]) -> None:
        """Initialize the KV caches on the underlying workers."""
        self.collective_rpc("initialize_from_config", args=(kv_cache_configs,))
Worker.initialize_from_config
GPUModelRunner.initialize_kv_cache

self.model_executor.compile_or_warm_up_model()
Worker.compile_or_warm_up_model

scheduler_kv_cache_config = KVCacheConfig(num_blocks=2454,
kv_cache_tensors=[KVCacheTensor(size=6433013760,
layers=['language_model.model.layers.0.self_attn.attn', 'language_model.model.layers.6.self_attn.attn', 'language_model.model.layers.12.self_attn.attn', 'language_model.model.layers.18.self_attn.attn', 'language_model.model.layers.24.self_attn.attn', 'language_model.model.layers.30.self_attn.attn', 'language_model.model.layers.36.self_attn.attn', 'language_model.model.layers.42.self_attn.attn', 'language_model.model.layers.48.self_attn.attn', 'language_model.model.layers.54.self_attn.attn'],
layer_stride=643301376,
block_stride=262144,
offset=0,
host_resident=False),
KVCacheTensor(size=6433013760,
layers=['language_model.model.layers.1.self_attn.attn', 'language_model.model.layers.7.self_attn.attn', 'language_model.model.layers.13.self_attn.attn', 'language_model.model.layers.19.self_attn.attn', 'language_model.model.layers.25.self_attn.attn', 'language_model.model.layers.31.self_attn.attn', 'language_model.model.layers.37.self_attn.attn', 'language_model.model.layers.43.self_attn.attn', 'language_model.model.layers.49.self_attn.attn', 'language_model.model.layers.55.self_attn.attn'],
layer_stride=643301376,
block_stride=262144,
offset=0,
host_resident=False),
KVCacheTensor(size=6433013760,
layers=['language_model.model.layers.2.self_attn.attn', 'language_model.model.layers.8.self_attn.attn', 'language_model.model.layers.14.self_attn.attn', 'language_model.model.layers.20.self_attn.attn', 'language_model.model.layers.26.self_attn.attn', 'language_model.model.layers.32.self_attn.attn', 'language_model.model.layers.38.self_attn.attn', 'language_model.model.layers.44.self_attn.attn', 'language_model.model.layers.50.self_attn.attn', 'language_model.model.layers.56.self_attn.attn'],
layer_stride=643301376,
block_stride=262144,
offset=0,
host_resident=False),
KVCacheTensor(size=6433013760,
layers=['language_model.model.layers.3.self_attn.attn', 'language_model.model.layers.9.self_attn.attn', 'language_model.model.layers.15.self_attn.attn', 'language_model.model.layers.21.self_attn.attn', 'language_model.model.layers.27.self_attn.attn', 'language_model.model.layers.33.self_attn.attn', 'language_model.model.layers.39.self_attn.attn', 'language_model.model.layers.45.self_attn.attn', 'language_model.model.layers.51.self_attn.attn', 'language_model.model.layers.57.self_attn.attn'],
layer_stride=643301376,
block_stride=262144,
offset=0,
host_resident=False),
KVCacheTensor(size=6433013760,
layers=['language_model.model.layers.4.self_attn.attn', 'language_model.model.layers.10.self_attn.attn', 'language_model.model.layers.16.self_attn.attn', 'language_model.model.layers.22.self_attn.attn', 'language_model.model.layers.28.self_attn.attn', 'language_model.model.layers.34.self_attn.attn', 'language_model.model.layers.40.self_attn.attn', 'language_model.model.layers.46.self_attn.attn', 'language_model.model.layers.52.self_attn.attn', 'language_model.model.layers.58.self_attn.attn'],
layer_stride=643301376,
block_stride=262144,
offset=0,
host_resident=False),
KVCacheTensor(size=6433013760,
layers=['language_model.model.layers.5.self_attn.attn', 'language_model.model.layers.11.self_attn.attn', 'language_model.model.layers.17.self_attn.attn', 'language_model.model.layers.23.self_attn.attn', 'language_model.model.layers.29.self_attn.attn', 'language_model.model.layers.35.self_attn.attn', 'language_model.model.layers.41.self_attn.attn', 'language_model.model.layers.47.self_attn.attn', 'language_model.model.layers.53.self_attn.attn', 'language_model.model.layers.59.self_attn.attn'],
layer_stride=643301376,
block_stride=262144,
offset=0,
host_resident=False)],
kv_cache_groups=[KVCacheGroupSpec(layer_names=['language_model.model.layers.0.self_attn.attn', 'language_model.model.layers.6.self_attn.attn', 'language_model.model.layers.12.self_attn.attn', 'language_model.model.layers.18.self_attn.attn', 'language_model.model.layers.24.self_attn.attn', 'language_model.model.layers.30.self_attn.attn', 'language_model.model.layers.36.self_attn.attn', 'language_model.model.layers.42.self_attn.attn', 'language_model.model.layers.48.self_attn.attn', 'language_model.model.layers.54.self_attn.attn'],
kv_cache_spec=SlidingWindowSpec(block_size=16,
num_kv_heads=16,
head_size=256,
dtype=torch.float16,
head_size_v=256,
kv_quant_mode=<KVQuantMode.NONE: 0>,
page_size_padded=None,
num_head_slots=None,
state_content_bytes=None,
tokens_per_state=1,
sliding_window=1024,
extra_retained_tokens=0),
is_eagle_group=False,
host_resident=False,
enable_kv_transfer=True,
role=<KVCacheGroupRole.DEFAULT: 'default'>),
KVCacheGroupSpec(layer_names=['language_model.model.layers.1.self_attn.attn', 'language_model.model.layers.7.self_attn.attn', 'language_model.model.layers.13.self_attn.attn', 'language_model.model.layers.19.self_attn.attn', 'language_model.model.layers.25.self_attn.attn', 'language_model.model.layers.31.self_attn.attn', 'language_model.model.layers.37.self_attn.attn', 'language_model.model.layers.43.self_attn.attn', 'language_model.model.layers.49.self_attn.attn', 'language_model.model.layers.55.self_attn.attn'],
kv_cache_spec=SlidingWindowSpec(block_size=16,
num_kv_heads=16,
head_size=256,
dtype=torch.float16,
head_size_v=256,
kv_quant_mode=<KVQuantMode.NONE: 0>,
page_size_padded=None,
num_head_slots=None,
state_content_bytes=None,
tokens_per_state=1,
sliding_window=1024,
extra_retained_tokens=0),
is_eagle_group=False,
host_resident=False,
enable_kv_transfer=True,
role=<KVCacheGroupRole.DEFAULT: 'default'>),
KVCacheGroupSpec(layer_names=['language_model.model.layers.2.self_attn.attn', 'language_model.model.layers.8.self_attn.attn', 'language_model.model.layers.14.self_attn.attn', 'language_model.model.layers.20.self_attn.attn', 'language_model.model.layers.26.self_attn.attn', 'language_model.model.layers.32.self_attn.attn', 'language_model.model.layers.38.self_attn.attn', 'language_model.model.layers.44.self_attn.attn', 'language_model.model.layers.50.self_attn.attn', 'language_model.model.layers.56.self_attn.attn'],
kv_cache_spec=SlidingWindowSpec(block_size=16,
num_kv_heads=16,
head_size=256,
dtype=torch.float16,
head_size_v=256,
kv_quant_mode=<KVQuantMode.NONE: 0>,
page_size_padded=None,
num_head_slots=None,
state_content_bytes=None,
tokens_per_state=1,
sliding_window=1024,
extra_retained_tokens=0),
is_eagle_group=False,
host_resident=False,
enable_kv_transfer=True,
role=<KVCacheGroupRole.DEFAULT: 'default'>),
KVCacheGroupSpec(layer_names=['language_model.model.layers.3.self_attn.attn', 'language_model.model.layers.9.self_attn.attn', 'language_model.model.layers.15.self_attn.attn', 'language_model.model.layers.21.self_attn.attn', 'language_model.model.layers.27.self_attn.attn', 'language_model.model.layers.33.self_attn.attn', 'language_model.model.layers.39.self_attn.attn', 'language_model.model.layers.45.self_attn.attn', 'language_model.model.layers.51.self_attn.attn', 'language_model.model.layers.57.self_attn.attn'],
kv_cache_spec=SlidingWindowSpec(block_size=16,
num_kv_heads=16,
head_size=256,
dtype=torch.float16,
head_size_v=256,
kv_quant_mode=<KVQuantMode.NONE: 0>,
page_size_padded=None,
num_head_slots=None,
state_content_bytes=None,
tokens_per_state=1,
sliding_window=1024,
extra_retained_tokens=0),
is_eagle_group=False,
host_resident=False,
enable_kv_transfer=True,
role=<KVCacheGroupRole.DEFAULT: 'default'>),
KVCacheGroupSpec(layer_names=['language_model.model.layers.4.self_attn.attn', 'language_model.model.layers.10.self_attn.attn', 'language_model.model.layers.16.self_attn.attn', 'language_model.model.layers.22.self_attn.attn', 'language_model.model.layers.28.self_attn.attn', 'language_model.model.layers.34.self_attn.attn', 'language_model.model.layers.40.self_attn.attn', 'language_model.model.layers.46.self_attn.attn', 'language_model.model.layers.52.self_attn.attn', 'language_model.model.layers.58.self_attn.attn'],
kv_cache_spec=SlidingWindowSpec(block_size=16,
num_kv_heads=16,
head_size=256,
dtype=torch.float16,
head_size_v=256,
kv_quant_mode=<KVQuantMode.NONE: 0>,
page_size_padded=None,
num_head_slots=None,
state_content_bytes=None,
tokens_per_state=1,
sliding_window=1024,
extra_retained_tokens=0),
is_eagle_group=False,
host_resident=False,
enable_kv_transfer=True,
role=<KVCacheGroupRole.DEFAULT: 'default'>),
KVCacheGroupSpec(layer_names=['language_model.model.layers.5.self_attn.attn', 'language_model.model.layers.11.self_attn.attn', 'language_model.model.layers.17.self_attn.attn', 'language_model.model.layers.23.self_attn.attn', 'language_model.model.layers.29.self_attn.attn', 'language_model.model.layers.35.self_attn.attn', 'language_model.model.layers.41.self_attn.attn', 'language_model.model.layers.47.self_attn.attn', 'language_model.model.layers.53.self_attn.attn', 'language_model.model.layers.59.self_attn.attn'],
kv_cache_spec=FullAttentionSpec(block_size=32,
num_kv_heads=4,
head_size=512,
dtype=torch.float16,
head_size_v=512,
kv_quant_mode=<KVQuantMode.NONE: 0>,
page_size_padded=None,
num_head_slots=None,
state_content_bytes=None,
tokens_per_state=1,
sliding_window=None,
attention_chunk_size=None,
non_causal=False),
is_eagle_group=False,
host_resident=False,
enable_kv_transfer=True,
role=<KVCacheGroupRole.DEFAULT: 'default'>)],
prefix_cache_retention_interval=0,
kv_cache_layout='LBNHC',
hisparse_host_num_blocks=None,
hisparse_host_block_stride=None,
hisparse_shared_host_pool=False),
kv_cache_configs = [KVCacheConfig(num_blocks=2454,
kv_cache_tensors=[KVCacheTensor(size=6433013760,
layer...ide=None,
hisparse_shared_host_pool=False)]


def collective_rpc(  # type: ignore[override]
        self,
        method: str | Callable,
        timeout: float | None = None,
        args: tuple = (),
        kwargs: dict | None = None,
        non_block: bool = False,
        single_value: bool = False,
    ) -> Any:
        if kwargs is None:
            kwargs = {}

        if not non_block:
            result = run_method(self.driver_worker, method, args, kwargs)
            if isinstance(result, AsyncModelRunnerOutput):
                result = result.get_output()
            return result if single_value else [result]

        try:
            result = run_method(self.driver_worker, method, args, kwargs)
            if isinstance(result, AsyncModelRunnerOutput):
                return AsyncOutputFuture(result, single_value)
            future = Future[Any]()
            future.set_result(result if single_value else [result])
        except Exception as e:
            future = Future[Any]()
            future.set_exception(e)
        return future

```
Request ID
    |
    v
req_to_blocks
    |
    +-- Request A -> [B0, B1, B2]
    |
    +-- Request B -> [B0, B3, B4]
    |
    +-- Request C -> [B5, B6, B7]
                       |
                       v
                   BlockPool
                       |
                       v
               实际 Block 资源管理

Request
   |
   v
计算 Request Block Hash
   |
   v
get_cached_block()
   |
   v
查询 cached_block_hash_to_block
   |
   +----------+
   |          |
   v          v
 Cache Hit  Cache Miss
   |          |
   v          v
 touch()   get_new_blocks()
   |          |
   v          v
ref_cnt++   分配新 Block
   |          |
   +-----+----+
         |
         v
     执行模型推理
         |
         v
    生成 KV Cache
         |
         v
 cache_full_blocks()
         |
         v
 注册 Prefix Cache Hash

           BlockPool
               |
       +-------+-------+
       |               |
       v               v
  get_new_blocks   free_blocks
       |               |
       v               v
   popleft_n()      ref_cnt--
       |               |
       v               v
  检查旧 Hash       ref_cnt == 0
       |               |
       v          +----+----+
  驱逐旧缓存       |         |
       |          v         v
       v        无 Hash    有 Hash
   ref_cnt++      |         |
       |          v         v
       v       prepend    append
    使用         队首      队尾
                  |         |
                  +----+----+
                       |
                       v
                   等待分配


                 BlockPool
                     |
          +----------+----------+
          |                     |
          v                     v
 cached_block_hash_to_block  cached_block_hashes_by_block
          |                     |
          v                     v
      Hash -> Block          Block -> Hash Set
          |                     |
          +----------+----------+
                     |
                     v
                KVCacheBlock
                     |
          +----------+----------+
          |          |          |
          v          v          v
       block_id   ref_cnt   block_hash
```
下面整理成一份可直接放入 GitHub 的 Markdown 文档。重点突出 **注册机制 → Spec 类型与容量计算 → Manager 核心接口**，保留关键参数和公式。
下面是完整修订版：**所有数学公式全部改为纯文本表达**，不使用 LaTeX、`$`、`$$`，确保可以直接复制到 GitHub Markdown 中正常显示。

# vLLM KV Cache Spec 与 Manager 设计

## 1. KVCacheSpec 注册机制

### 1.1 作用

vLLM 通过 `KVCacheSpecRegistry` 建立 KV Cache Spec 与 Manager 的映射关系，使不同类型的 Cache 可以使用对应的管理器实现。

主要解决三个问题：

1. **类型识别**：识别模型各层需要使用的 KV Cache 类型。
2. **Manager 选择**：根据 Spec 类型找到对应的 Manager。
3. **统一类型判断**：通过 `uniform_type_base_spec` 判断不同 Spec 是否属于同一类 Cache。

注册关系：

```
KVCacheSpecRegistry
        |
        | register()
        v
Spec Class -> KVCacheSpecMetadata
                    |
                    +-- kvcache_spec_cls
                    +-- manager_class
                    +-- uniform_type_base_spec
```

### 1.2 注册过程

核心入口：

```
register_all_kvcache_specs(vllm_config)
```

主要完成：

- 注册各种 KVCacheSpec 对应的 Manager。
- 建立 Spec 类型到 Metadata 的映射。
- 校验 Spec 与 uniform base 类型之间的继承关系。
- 检查重复注册时 Manager 和 uniform base 是否一致。

**注意：注册只建立类型映射，不创建 Cache，也不分配 GPU 显存。**

### 1.3 典型注册关系

| Spec 类型                   | Manager 类型                   | Uniform Base Spec         |
| ------------------------- | ---------------------------- | ------------------------- |
| FullAttentionSpec         | FullAttentionManager         | FullAttentionSpec         |
| SlidingWindowSpec         | SlidingWindowManager         | SlidingWindowSpec         |
| ChunkedLocalAttentionSpec | ChunkedLocalAttentionManager | ChunkedLocalAttentionSpec |
| CrossAttentionSpec        | CrossAttentionManager        | CrossAttentionSpec        |
| MLAAttentionSpec          | FullAttentionManager         | FullAttentionSpec         |
| SlidingWindowMLASpec      | SlidingWindowManager         | SlidingWindowMLASpec      |
| SinkFullAttentionSpec     | SinkFullAttentionManager     | FullAttentionSpec         |
| RSWASpec                  | RSWAManager                  | FullAttentionSpec         |
| HiddenStateCacheSpec      | FullAttentionManager         | FullAttentionSpec         |
| MambaSpec                 | MambaManager                 | MambaSpec                 |
| CircularBufferSpec        | CircularBufferManager        | CircularBufferSpec        |
| KpoolTailSpec             | KpoolTailManager             | KpoolTailSpec             |
| HiSparseHotSpec           | HiSparseHotManager           | HiSparseHotSpec           |
| HiSparseResidentSpec      | HiSparseResidentManager      | HiSparseResidentSpec      |

另外，Role Manager 还存在独立注册关系，例如：

```
KVCacheGroupRole.HISPARSE_SOURCE
              |
              v
      HiSparseSourceManager
```

### 1.4 Uniform Type 的作用

`uniform_type_base_spec` 用于判断一组 Spec 是否属于同一种统一类型。

相关接口：

- `KVCacheSpecRegistry.get_uniform_type_base_spec()`
- `KVCacheSpec.is_uniform_with_collection()`
- `UniformTypeKVCacheSpecs.from_specs()`
- `UniformTypeKVCacheSpecs.is_uniform_type()`

注意：Uniform Type 表示类型兼容，不代表所有 Cache 参数完全相同。实际合并仍然需要满足对应的 Spec 约束。

---

## 2. KVCacheSpec 类型与容量计算

`KVCacheSpec` 描述 Cache 的数据结构、Block 大小及容量计算方式。

不同模型结构需要不同的 Cache 管理方式：

- 普通 Attention：保存 K/V。
- Sliding Window Attention：只保留局部窗口。
- MLA：使用不同于传统 K/V 的 Cache 表示。
- Mamba：保存状态张量。
- Circular Buffer：循环复用固定容量。
- HiSparse：区分 Hot Cache 和 Resident Cache。

### 2.1 通用容量计算

KV Cache 通常以 Block 为单位管理。

定义：

- `T`：需要缓存的 Token 数量。
- `B`：一个 Block 对应的 Token 数量。
- `S_block`：一个 Block 占用的字节数。
- `N_blocks`：需要的 Block 数量。

**Block 数量：**

```
N_blocks = ceil(T / B)
```

其中 `ceil()` 表示向上取整。

**总 Cache 容量：**

```
S_cache = N_blocks * S_block
```

**传统 Attention 单个 Block 的理论容量：**

```
S_block = 2 * H_kv * B * D_head * S_dtype
```

其中：

- `2`：分别保存 Key 和 Value。
- `H_kv`：KV Head 数量。
- `B`：Block Size。
- `D_head`：Head Dimension。
- `S_dtype`：每个元素占用的字节数。

该公式适用于传统 K/V 存储模型；特殊 Cache 类型需要根据各自 Spec 计算。

### 2.2 Cache 类型对比

| Spec 类型                     | 作用与为什么需要                                                                | 关键参数                                                                                                                                                                         | 容量计算与特点                                            |
| --------------------------- | ----------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | -------------------------------------------------- |
| `FullAttentionSpec`         | 标准全量 Attention，保存历史 Token 的 K/V，支持完整上下文注意力                              | `num_kv_heads`、`head_size`、`head_size_v`、`dtype`、`block_size`                                                                                                                | `S_block = 2 * H_kv * B * D_head * S_dtype`        |
| `SlidingWindowSpec`         | 滑动窗口 Attention，只需要保留最近窗口内的 Token，降低长上下文 Cache 占用                        | `sliding_window`、`extra_retained_tokens`                                                                                                                                     | 有效 Token 数受窗口限制；`N_blocks ≈ ceil(T_effective / B)` |
| `ChunkedLocalAttentionSpec` | 分块局部 Attention，限制每次计算所需的局部历史，降低 Cache 需求                                | `attention_chunk_size`                                                                                                                                                       | 容量取决于局部 Chunk 的保留策略，不能简单按完整上下文长度计算                 |
| `CrossAttentionSpec`        | 用于 Cross Attention，保存供 Decoder 查询的 Encoder 侧 K/V                        | Attention 相关维度参数                                                                                                                                                             | Cache 需求与 Encoder 输出及其 K/V 表示有关                    |
| `MLAAttentionSpec`          | 适配 Multi-head Latent Attention，使用 MLA 特定的 Cache 表示，减少传统 K/V Cache 的存储开销 | `cache_dtype_str`、`alignment`、`model_version`、`cache_role` 等                                                                                                                 | 根据 MLA Cache 的实际表示、对齐和数据类型计算，不能直接套用传统 K/V 公式       |
| `SlidingWindowMLASpec`      | 将滑动窗口机制与 MLA Cache 结合                                                   | `sliding_window`、`extra_retained_tokens`、`alignment`、`model_version` 等                                                                                                       | 容量同时受窗口限制和 MLA Cache 布局影响                          |
| `SinkFullAttentionSpec`     | 支持 Attention Sink，保留开头的 Sink Token，同时维护后续 Attention Cache               | `sink_len`                                                                                                                                                                   | 容量受 Sink Token 与普通历史 Cache 的保留策略影响                 |
| `RSWASpec`                  | 实现 RSW​​A 窗口管理，控制历史 Cache 的保留范围                                         | `rswa_window`                                                                                                                                                                | 容量受窗口参数影响；可能通过回收中间 Block 控制占用                      |
| `HiddenStateCacheSpec`      | 管理 Hidden State Cache，满足需要缓存隐藏状态的模型执行场景                                 | 以具体 Spec 定义为准                                                                                                                                                                | 根据 Hidden State 的张量形状和数据类型计算                       |
| `MambaSpec`                 | 管理 Mamba 等状态空间模型的状态 Cache，不采用传统 Attention K/V 存储形式                      | `shapes`、`dtypes`、`page_size_padded`、`mamba_type`、`mamba_cache_mode`、`num_speculative_blocks`、`num_prefill_checkpoint_blocks`、`prefill_checkpoint_alignment`、`tp_replicated` | 按状态张量大小计算，见下文                                      |
| `CircularBufferSpec`        | 使用固定容量循环 Buffer，适用于不需要持续保存完整历史的场景                                       | Attention 相关参数、`block_size`                                                                                                                                                  | 单请求固定一个 Block，通过循环复用控制容量                           |
| `KpoolTailSpec`             | 管理 KPool 尾部 Cache，采用固定 Block 的管理方式                                      | 继承 Sliding Window 相关参数                                                                                                                                                       | 单请求固定一个 Block                                      |
| `HiSparseHotSpec`           | 管理 HiSparse 中的 Hot Cache，保存热点数据                                         | `page_size`、`blocks_per_request`                                                                                                                                             | 容量与 `blocks_per_request * page_size` 相关            |
| `HiSparseResidentSpec`      | 管理 HiSparse 中的 Resident Cache，保存驻留数据                                    | `page_size`                                                                                                                                                                  | 按 Resident Cache 的 Block 和 Page 布局计算               |

注：

1. 表中容量公式是理解 Cache 规模的通用模型，实际实现还可能考虑对齐、量化、Padding、并行切分和额外保留空间。
2. `SlidingWindowSpec`、`ChunkedLocalAttentionSpec`、`RSWASpec` 等类型的实际 Block 分配量还取决于 Manager 的保留和回收策略。
3. 特殊 Spec 的字段和容量计算应以对应源码实现为准。

### 2.3 Mamba Cache 容量计算

Mamba 不保存传统 Attention 的 K/V，而是保存状态张量。

设第 `i` 个状态张量的 Shape 为 `shape_i`，数据类型字节数为 `S_dtype_i`：

```
S_state = sum(prod(shape_i) * S_dtype_i)
```

其中：

- `prod(shape_i)`：该状态张量所有维度相乘，得到元素总数。
- `S_dtype_i`：每个元素占用的字节数。
- `sum()`：累加所有状态张量的存储大小。

如果使用 Padding 后的 Page Size：

```
S_block = S_state_padded
```

相关参数：

- `shapes`：状态张量的形状。
- `dtypes`：状态张量的数据类型。
- `page_size_padded`：Padding 后的 Page 大小。
- `mamba_cache_mode`：影响 Cache 的管理和容量策略。
- `num_speculative_blocks`：Speculative Decoding 相关 Block 数量。
- `num_prefill_checkpoint_blocks`：Prefill Checkpoint 相关 Block 数量。

Mamba Cache 的实际容量还需要结合 Cache Mode、Checkpoint 和 Speculative 配置计算。

---

## 3. SingleTypeKVCacheManager

### 3.1 核心职责

`SingleTypeKVCacheManager` 是单一 KV Cache Group 的抽象管理器。

它负责：

1. 计算请求需要多少 Cache Block。
2. 为请求分配和回收 Block。
3. 管理请求与 Block 的映射关系。
4. 处理 Prefix Cache 命中。
5. 处理 Copy-on-Write。
6. 管理新分配 Block 的通知和状态交接。

它不负责：

- 执行 Attention。
- 执行 GPU Tensor 分配。
- 执行模型推理。
- 直接计算 K/V 内容。

### 3.2 相关组件关系

```
                 Scheduler
                     |
                     v
              KVCacheManager
                     |
                     v
        SingleTypeKVCacheManager
                     |
          +----------+----------+
          |                     |
          v                     v
      BlockPool              KVCacheSpec
          |                     |
          v                     v
     KVCacheBlock         Cache 类型和容量
          |
          v
    Block 生命周期管理

Worker:
    根据 KVCacheConfig 初始化实际 GPU Cache Tensor
```

组件职责：

| 组件                         | 核心职责                          |
| -------------------------- | ----------------------------- |
| `KVCacheSpec`              | 描述 Cache 类型、数据布局和容量           |
| `KVCacheManager`           | 管理多个 KV Cache Group           |
| `SingleTypeKVCacheManager` | 管理单个 Cache Group 的 Block 生命周期 |
| `BlockPool`                | 实际分配、释放、缓存和引用计数管理             |
| `KVCacheBlock`             | 表示一个逻辑 Cache Block            |
| Worker                     | 初始化并使用 GPU 上的实际 Cache Tensor  |

### 3.3 初始化关键状态

| 成员                                  | 作用                        |
| ----------------------------------- | ------------------------- |
| `kv_cache_spec`                     | 当前 Group 的 Cache Spec     |
| `block_pool`                        | 负责 Block 分配和回收            |
| `enable_caching`                    | 是否启用 Cache                |
| `kv_cache_group_id`                 | 当前 Cache Group ID         |
| `block_size`                        | 当前 Manager 的 Block Size   |
| `scheduler_block_size`              | 所有 Group Block Size 的 LCM |
| `cache_hit_alignment_tokens`        | Cache Hit 对齐 Token 数      |
| `dcp_world_size`                    | DCP 并行规模                  |
| `pcp_world_size`                    | PCP 并行规模                  |
| `_max_admission_blocks_per_request` | 单请求最大允许使用的 Block 数        |
| `req_to_blocks`                     | Request ID 到 Block 列表的映射  |
| `num_cached_block`                  | 当前请求已缓存 Block 数量          |
| `_null_block`                       | 跳过 Block 的占位对象            |
| `new_block_ids`                     | 本轮新分配 Block ID            |
| `_record_new_block_ids`             | 是否记录需要通知 Worker 的新 Block  |
| `_partial_hit_reqs`                 | 部分 Prefix Cache 命中请求      |
| `_pending_cow_copies`               | 等待执行的 Copy-on-Write 操作    |
| `_pending_boundary_state_offloads`  | 等待处理的边界状态 Offload         |

其中，`req_to_blocks` 是请求和逻辑 Cache Block 之间最重要的映射：

```
Request A -> [Block 1, Block 2, Block 3]
Request B -> [Block 4, Block 5]
Request C -> [Block 6, Block 7, Block 8]
```

注意：`block_size` 在 DCP 并行场景下可能根据 `dcp_world_size` 调整；`scheduler_block_size` 用于协调不同 Cache Group 的调度粒度。

### 3.4 核心接口

| 功能分类          | 接口                                       | 作用                          |
| ------------- | ---------------------------------------- | --------------------------- |
| Block 计算      | `get_num_blocks_to_allocate()`           | 计算当前请求需要新增多少 Block          |
| 容量计算          | `_get_num_evictable_blocks()`            | 计算可以通过驱逐获得的 Block 数量        |
| Cache 命中      | `_has_partial_local_hit()`               | 判断是否存在部分本地 Cache 命中         |
| 跳过 Token      | `get_num_skipped_tokens()`               | 计算需要跳过的 Token 数量            |
| 本地 Cache      | `add_local_computed_blocks()`            | 将本地计算得到的 Cache Block 加入请求管理 |
| 外部 Cache      | `allocate_external_computed_blocks()`    | 分配外部计算结果对应的 Cache Block     |
| 新 Block 分配    | `allocate_new_blocks()`                  | 为请求分配新的 Cache Block         |
| Prefix Cache  | `find_longest_cache_hit()`               | 查找最长 Prefix Cache 命中，抽象接口   |
| Prefix Cache  | `cache_blocks()`                         | 将请求 Block 加入 Cache          |
| Prefix Cache  | `get_num_common_prefix_blocks()`         | 计算多个请求的公共 Prefix Block，抽象接口 |
| Block 可达性     | `reachable_block_mask()`                 | 计算 Block 的可达性 Mask          |
| Block 回收      | `remove_skipped_blocks()`                | 移除被跳过的 Block                |
| Block 回收      | `_remove_blocks_in_range()`              | 移除指定范围的 Block               |
| Block 回收      | `pop_blocks_for_free()`                  | 从请求映射中取出待释放 Block           |
| Block 回收      | `free()`                                 | 释放请求占用的 Block               |
| Copy-on-Write | `_apply_cow()`                           | 执行 CoW 相关 Block 处理          |
| Copy-on-Write | `take_pending_cow_copies()`              | 获取等待执行的 CoW 操作              |
| Worker 通知     | `take_new_block_ids()`                   | 获取新分配 Block ID              |
| Worker 通知     | `take_pending_boundary_state_offloads()` | 获取待处理的边界状态 Offload          |
| Worker 通知     | `finalize_partial_tail_offload()`        | 完成 Partial Tail Offload     |
| 生命周期          | `new_step_starts()`                      | 开始新的调度 Step                 |

其中，以下三个接口由具体 Manager 子类实现或提供差异化逻辑：

- `find_longest_cache_hit()`
- `get_num_common_prefix_blocks()`
- `get_num_skipped_tokens()`

这也是不同 Cache 类型在 Prefix Cache 和 Block 回收策略上的主要差异之一。

---

## 4. Manager 核心计算原理

### 4.1 普通 Block 分配

设：

- `T`：请求需要缓存的 Token 数量。
- `B`：Block Size。
- `N_existing`：请求当前已有 Block 数量。

需要的 Block 数量：

```
N_required = ceil(T / B)
```

需要新增的 Block 数量：

```
N_new = max(N_required - N_existing, 0)
```

### 4.2 Prefix Cache 命中与跳过 Block

当请求存在 Prefix Cache 命中或跳过 Token 时，部分历史 Token 不需要重新分配 Block。

简化理解：

```
N_new = max(N_required - max(N_skipped, N_cached), 0)
```

其中：

- `N_skipped`：跳过的 Block 数量。
- `N_cached`：已命中的本地 Cache Block 数量。

实际实现还需要考虑部分命中、Block 对齐、CoW 和驱逐等情况，不能仅用该公式替代源码中的完整分配逻辑。

### 4.3 Admission 限制

对于 Sliding Window 等具有最大保留容量的 Cache，单请求实际需要的 Block 数可能受到 Admission 上限限制。

```
N_required = min(ceil(T / B), N_admission_max)
```

这样可以避免请求长度持续增长时，Cache 占用也无限增长。

### 4.4 Copy-on-Write

多个请求可能共享相同的 Prefix Cache Block。

当其中一个请求需要修改共享 Block 时，需要执行 Copy-on-Write：

```
Request A ----+
              |
              +---- Shared Block
              |
Request B ----+

Request B 需要修改 Block：

Request A -------- Shared Block
Request B -------- New Block
```

核心思想：

- 只读共享：多个请求可以引用同一个 Block。
- 写入修改：为需要修改的请求分配独立 Block。
- 避免直接修改共享数据，保证其他请求的 Cache 内容不受影响。

因此，Block 分配时还可能需要额外预留 CoW Block。

### 4.5 Block 回收

请求完成或不再需要某些历史 Cache 时，Manager 将 Block 从请求映射中移除，再交给 `BlockPool` 处理释放。

```
Scheduler
    |
    v
KVCacheManager.free()
    |
    v
SingleTypeKVCacheManager.free()
    |
    v
pop_blocks_for_free()
    |
    v
BlockPool.free_blocks()
    |
    v
Block 回到可复用池
```

Block 是否能立即复用，还取决于引用计数、Prefix Cache 和 BlockPool 的管理状态。

---

## 5. 核心流程总结

### 5.1 请求分配 Cache

```
Scheduler
    |
    v
KVCacheManager.allocate_slots()
    |
    v
get_num_blocks_to_allocate()
    |
    +---- 计算需求 Block 数量
    +---- 检查 Prefix Cache 命中
    +---- 计算跳过 Block
    +---- 检查可驱逐容量
    +---- 考虑 CoW
    |
    v
allocate_new_blocks()
    |
    v
BlockPool.get_new_blocks()
    |
    v
更新 req_to_blocks
    |
    v
Worker 使用对应 Cache Block
```

### 5.2 请求释放 Cache

```
Scheduler
    |
    v
KVCacheManager.free()
    |
    v
SingleTypeKVCacheManager.free()
    |
    v
pop_blocks_for_free()
    |
    v
BlockPool.free_blocks()
    |
    v
Block 进入可复用状态
```

### 5.3 核心概念关系

| 概念                         | 解决的问题                             |
| -------------------------- | --------------------------------- |
| `KVCacheSpec`              | Cache 应该如何存储？                     |
| `KVCacheSpecRegistry`      | 根据 Spec 类型选择哪个 Manager？           |
| `KVCacheGroupSpec`         | 哪些模型层共享同一组 Block 管理关系？            |
| `KVCacheTensor`            | Worker 如何组织实际 Cache Tensor 的存储位置？ |
| `SingleTypeKVCacheManager` | 请求如何申请、使用和释放 Block？               |
| `BlockPool`                | Block 如何分配、复用、缓存和回收？              |
| `KVCacheBlock`             | 一个逻辑 Cache Block 如何表示？            |

**总结：**

vLLM 将 KV Cache 管理拆分为三个层次：

1. **Spec 层**：描述 Cache 数据结构和容量。
2. **Manager 层**：管理请求与 Block 的生命周期。
3. **BlockPool 层**：执行 Block 的分配、复用和释放。

这种设计使不同模型结构可以复用统一的 Block 管理框架，同时通过不同 Spec 和 Manager 实现 Attention、MLA、Mamba、Sliding Window 等 Cache 类型的差异化管理。

用纯文本替换所有公式检查转义字符和乱码统一公式中的变量说明
