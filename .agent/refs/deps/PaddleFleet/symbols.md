# PaddleFleet 公共 API（当前库使用的部分）

## 协议 / 抽象类（扩展点）

- `BackendSpecProvider`: 后端算子选择协议，定义各并行层的类型选择接口 | `src/paddlefleet/models/backends.py:52`
  - `column_parallel_linear() -> type`: 列并行线性层类型
  - `row_parallel_linear() -> type`: 行并行线性层类型
  - `fuse_layernorm_and_linear() -> bool`: 是否融合 LayerNorm 和 Linear
  - `layer_norm(rms_norm, for_qk) -> type`: LayerNorm 层类型
  - `core_attention() -> type`: 核心注意力层类型

- `FleetLayer`: 所有模型层的基类，持有 config | `src/paddlefleet/transformer/layer.py:25`
  - `__init__(config: TransformerConfig)`: 初始化，存储 self.config

## 核心配置类

- `TransformerConfig`: Transformer 模型配置（继承 ModelParallelConfig，dataclass） | `src/paddlefleet/transformer/transformer_config.py:34`
  - 字段：`num_hidden_layers`, `normalization`, `use_qk_norm`, `n_routed_experts`, `mtp_loss_scaling_factor` 等

- `GPTConfig`: GPT 模型配置（继承 TransformerConfig） | `src/paddlefleet/models/gpt/gpt_config.py:21`

## 模型类

- `GPTModel`: GPT 主模型（继承 PipelineLayer） | `src/paddlefleet/models/gpt/gpt_model.py:148`
- `GPTEmbedding`: GPT 词嵌入层（FleetLayer 子类） | `src/paddlefleet/models/gpt/gpt_embedding.py:55`
- `GPTLMHead`: GPT 语言模型头（继承 ColumnParallelLinear） | `src/paddlefleet/models/gpt/lm_head.py:32`
- `LLaVAModel`: 多模态 LLaVA 模型（FleetLayer 子类） | `src/paddlefleet/models/multimodal/llava_model.py:50`

## 核心层类

- `TransformerLayer`: 单个 Transformer 解码器层 | `src/paddlefleet/transformer/transformer_layer.py:144`
- `TransformerBlock`: Transformer 解码器块（FleetLayer 子类） | `src/paddlefleet/transformer/transformer_block.py:108`
- `SelfAttention`: 标准多头自注意力层 | `src/paddlefleet/transformer/attention.py`
- `DotProductAttention`: 标准点积注意力 | `src/paddlefleet/transformer/dot_product_attention.py`
- `MLP`: 前馈网络层（FleetLayer 子类） | `src/paddlefleet/transformer/mlp.py`
- `MoELayer`: Mixture-of-Experts 层 | `src/paddlefleet/transformer/moe/moe_layer.py`
- `EmptyLayer`: 空占位层（FleetLayer 子类） | `src/paddlefleet/models/common/empty_layer.py:28`
- `VisionLayer`: 视觉层基类（FleetLayer 子类） | `src/paddlefleet/models/common/vision_layer/vision_layer.py:20`
- `IdentityOp`: 恒等操作层 | `src/paddlefleet/transformer/identity_op.py:18`

## SublayersSpec 数据类

- `MLPSublayersSpec`: MLP 子层规格 | `src/paddlefleet/transformer/mlp.py:60`
- `SelfAttentionSublayersSpec`: 自注意力子层规格 | `src/paddlefleet/transformer/attention.py`
- `TransformerBlockSublayersSpec`: TransformerBlock 子层规格 | `src/paddlefleet/transformer/transformer_block.py:48`
- `TransformerLayerSublayersSpec`: TransformerLayer 子层规格 | `src/paddlefleet/transformer/transformer_layer.py`

## 模型构建函数（工厂）

- `gpt_builder(config, **kwargs)`: GPT 模型构建入口 | `src/paddlefleet/gpt_builders.py:33`
- `get_gpt_layer_local_spec(config, ...)`: 构建单层 GPT LayerSpec | `src/paddlefleet/models/gpt/gpt_layer_specs.py`
- `get_gpt_decoder_layers_spec(config, ...)`: 构建 MoE decoder block spec | `src/paddlefleet/models/gpt/gpt_layer_specs.py:475`
- `get_gpt_mtp_layers_spec(config, ...)`: 构建 MTP 层 spec | `src/paddlefleet/models/gpt/gpt_layer_specs.py`
- `get_qwen3_5_vision_spec(config: TransformerConfig) -> LayerSpec`: Qwen3.5 视觉层 spec | `src/paddlefleet/models/qwen3_5/layer_specs.py:52`

## 并行状态管理

- `parallel_state` 模块：`get_tensor_model_parallel_group()`, `get_pipeline_model_parallel_group()`, `get_context_parallel_group()` 等 | `src/paddlefleet/parallel_state.py`
- `get_tensor_model_parallel_group_if_none(group)`: 若 group 为 None 则返回默认 TP group | `src/paddlefleet/utils.py:199`

## 张量并行层

- `ColumnParallelLinear`: 列并行线性层 | `src/paddlefleet/tensor_parallel/layers.py`
- `RowParallelLinear`: 行并行线性层 | `src/paddlefleet/tensor_parallel/layers.py`
- `scatter_to_sequence_parallel_region(input_, group=None)`: 将张量 scatter 到序列并行区域 | `src/paddlefleet/tensor_parallel/mappings.py:545`

## 归一化层

- `WrappedPaddleNorm`: 封装 Paddle LayerNorm/RMSNorm | `src/paddlefleet/transformer/paddle_norm.py`
- `WrappedPaddleNormPipe`: Pipeline 版本 | `src/paddlefleet/transformer/paddle_norm.py`
- `FusedRMSNorm`: 融合 RMSNorm | `src/paddlefleet/transformer/paddle_norm.py:141`
- `LayerNorm`: 标准 LayerNorm 封装 | `src/paddlefleet/transformer/paddle_norm.py:91`
- `Qwen3_5RMSNorm`: Qwen3.5 专用 RMSNorm | `src/paddlefleet/models/qwen3_5/qwen3_5_model.py:52`
- `Qwen3_5RMSNormPipe`: Pipeline 版本 | `src/paddlefleet/models/qwen3_5/qwen3_5_model.py:120`

## MoE 相关

- `StandardMoERouter`: 标准 MoE 路由器 | `src/paddlefleet/transformer/moe/moe_router.py:210`
- `GroupedMLPExpert`: Grouped MLP Expert（FleetLayer 子类） | `src/paddlefleet/transformer/moe/moe_expert.py:128`
- `BMMFunction`: BMM 自定义 autograd 函数 | `src/paddlefleet/transformer/moe/moe_expert.py:46`
- `DeepGEMMBMMFunction`: DeepGEMM BMM 自定义 autograd 函数 | `src/paddlefleet/transformer/moe/moe_expert.py:78`

## 枚举类型

- `ModelType`: 模型类型枚举 | `src/paddlefleet/transformer/enums.py:18`
- `AttnMaskType`: 注意力掩码类型枚举 | `src/paddlefleet/transformer/enums.py:59`

## 数据结构

- `PackedSeqParams`: 打包序列参数（dataclass） | `src/paddlefleet/packed_seq_params.py:25`
- `ProcessGroupCollection`: 进程组集合 | `src/paddlefleet/process_groups_config.py:28`
- `WrappedTensor`: 封装张量，携带额外元信息 | `src/paddlefleet/utils.py:60`

## 工具函数

- `get_model_config(model)`: 从模型实例获取 TransformerConfig | `src/paddlefleet/utils.py:461`
- `get_batch_on_this_cp_rank(inputs)`: 获取当前 CP rank 的 batch 切片 | `src/paddlefleet/utils.py:270`
- `deprecate_inference_params(inference_context, inference_params)`: 推理参数废弃兼容处理 | `src/paddlefleet/utils.py:524`
- `get_bias_dropout_add(training, fused)`: 获取 bias+dropout+add 融合函数 | `src/paddlefleet/fusions/fused_bias_dropout.py:74`
- `set_profile_timers(timers)`: 设置全局 profile timers | `src/paddlefleet/training/global_vars.py:58`
- `initialize_fleet(...)`: 初始化 PaddleFleet 分布式训练环境 | `src/paddlefleet/training/initialize.py:25`

## 注意事项

- 所有 import 均为条件导入（`try/except ImportError`），PaddleFleet 是可选依赖
- `get_gpt_mtp_block_spec` 在 PaddleFleet 源码中未找到定义，可能是别名或已移除，使用时需确认版本
- `correct_amax_history_if_needed`（`paddlefleet.fp8_utils`）在源码中未找到，可能已迁移至 `paddlefleet.transformer.moe.fp8_utils`
- `deep_gemm`（`paddlefleet.ops`）为底层算子，依赖硬件支持
