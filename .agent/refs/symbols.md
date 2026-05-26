# 关键符号索引

## 协议 / 抽象类（扩展点）

- `BackendSpecProvider`: 后端算子选择协议，定义各并行层的类型选择接口 | `src/paddlefleet/models/backends.py:52`
  - `column_parallel_linear() -> type`: 列并行线性层类型
  - `row_parallel_linear() -> type`: 行并行线性层类型
  - `fuse_layernorm_and_linear() -> bool`: 是否融合 LayerNorm 和 Linear
  - `column_parallel_layer_norm_linear() -> type | None`: 融合 LN+Linear 层类型
  - `layer_norm(rms_norm, for_qk) -> type`: LayerNorm 层类型
  - `core_attention() -> type`: 核心注意力层类型
  - `grouped_mlp_layers(moe_use_grouped_gemm, moe_use_legacy_grouped_gemm) -> tuple`: MoE grouped MLP 层和 spec
  - `hidden_act() -> type`: 激活函数层类型

- `FleetLayer`: 所有模型层的基类，持有 config | `src/paddlefleet/transformer/layer.py:25`
  - `__init__(config: TransformerConfig)`: 初始化，存储 self.config

## 核心配置类

- `ModelParallelConfig`: 并行配置基类（dataclass） | `src/paddlefleet/model_parallel_config.py:32`
  - 字段：`tensor_model_parallel_size`, `pipeline_model_parallel_size`, `virtual_pipeline_model_parallel_size`, `context_parallel_size` 等

- `TransformerConfig`: Transformer 模型配置（继承 ModelParallelConfig，dataclass） | `src/paddlefleet/transformer/transformer_config.py:34`
  - 字段：`num_hidden_layers`, `num_nextn_predict_layers`, `mtp_loss_scaling_factor`, `normalization`, `use_qk_norm`, `qk_l2_norm`, `n_routed_experts` 等

## 核心层类

- `TransformerLayer`: 单个 Transformer 解码器层 | `src/paddlefleet/transformer/transformer_layer.py`
  - 通过 `TransformerLayerSublayersSpec` 注入子层
  - `forward(dict_args)` (line 426): 入口，处理 MTP hidden_states 拆分、full_recompute 分支，调用 `_forward_impl`
  - `_forward_impl(hidden_states, ...)` (line 680): 核心实现，顺序调用 `_forward_attention` → `_forward_mlp`
  - `_forward_attention(hidden_states, ...)` (line 782): input_layernorm → self_attn → self_attn_bda → (可选) cross_attention
  - `_forward_mlp(hidden_states, ...)` (line 916): post_attention_layernorm → mlp → mlp_bda

- `TransformerLayerNode`: 用于 overlap 调度的 TransformerLayer 节点 | `src/paddlefleet/transformer/transformer_layer.py`

- `SelfAttention`: 标准多头自注意力层（FleetLayer 子类） | `src/paddlefleet/transformer/attention.py`
  - 通过 `SelfAttentionSublayersSpec` 注入 qkv_proj、core_attention、o_proj、q_norm、k_norm

- `MLASelfAttention`: Multi-Latent Attention（DeepSeek 风格） | `src/paddlefleet/transformer/multi_latent_attention.py`

- `DotProductAttention`: 标准点积注意力 | `src/paddlefleet/transformer/dot_product_attention.py`
- `CPDotProductAttention`: Context Parallel 点积注意力 | `src/paddlefleet/transformer/dot_product_attention.py`

- `MLP`: 前馈网络层（FleetLayer 子类） | `src/paddlefleet/transformer/mlp.py`
  - 通过 `MLPSublayersSpec(up_gate_proj, down_proj)` 注入子层

- `MoELayer`: Mixture-of-Experts 层（nn.Layer 子类） | `src/paddlefleet/transformer/moe/moe_layer.py:139`
  - `forward(hidden_states, input_ids)` (line 942): 顶层入口，按 EP/SP 配置分发到以下路径
  - **EP > 1 路径**：
    - `fusion_moe_forward(...)` (line 636)：fusion node 路径，依次 dispatch → FusionMoePyLayer（本库 Python PyLayer, fusion_layer_utils.py:1808）/SonicMoE/HybridEPMoePyLayer → combine
    - `custom_forward(...)` (line 601)：标准 EP 路径，dispatch → routed_experts_compute → combine
  - **EP = 1 路径**：
    - `_forward_single_card_grouped_gemm_moe(...)` (line 1115)：Grouped GEMM，含 SonicMoE kernel
    - `_forward_single_card_moe(...)` (line 1059)：Python loop over experts（测试/少量专家用）
  - 公共子方法：`dispatch(line 552)` / `combine(line 583)` / `routed_experts_compute(line 589)` / `expert_forward(line 525)`

- `TopKRouter`: MoE 路由器，继承 `StandardMoERouter` | `src/paddlefleet/transformer/moe/moe_router.py:762`
  - `forward(input, input_ids)` (line 770): reshape → `gate_detach_matmul` 线性投影 → `gate_score_func` 评分 → topk 选择（MoETopkFusion 或 `_call_topk_method`）→ routing map → aux_loss/z_loss
  - 返回：`(capacity, topk_weights, topk_indices, probs, mask, priorities, aux_loss, z_loss)`
  - 关键配置：`config.scoring_func`（sigmoid/softmax）、`config.moe_topk_fusion`（Triton kernel）、`config.topk_method`

- `StandardMLPExpert`: 单个 expert MLP，继承 `MLP` | `src/paddlefleet/transformer/moe/moe_expert.py:337`
  - 无独立 forward，直接复用 `MLP.forward`

- `GroupedMLPExpert`: Grouped GEMM Expert，继承 `FleetLayer` | `src/paddlefleet/transformer/moe/moe_expert.py:128`
  - `forward(permuted_local_hidden_states, tokens_per_expert)` (line 235): BMM(w1) → activation → BMM(w2)，支持 DeepGEMM（`moe_deep_gemm=True`）

- `MoEFlexTokenDispatcher`: 主力 token 分发器，处理 EP AllToAll 通信 | `src/paddlefleet/transformer/moe/token_dispatcher.py:716`
  - `dispatch_preprocess`: 路由元数据初始化
  - `token_dispatch`: 委托 `_comm_manager.dispatch(...)` 执行 AllToAll
  - `dispatch_postprocess`: 按 expert 重排 tokens
  - `combine_preprocess` / `token_combine` / `combine_postprocess`: 逆序恢复 + AllToAll

- `FusionMoePyLayer`: EP 场景下的融合 MoE 前向 PyLayer | `src/paddlefleet/transformer/moe/fusion_layer_utils.py:1808`
  - `forward(ctx, hidden_states, dispatched_probs, dispatched_indices, custom_map, ...)` (line 1814)
  - 作用：包装 `MlpNode`，管理前向缓存（`cached_tensors`）+ 手动 backward
  - 处理 FP8 dispatch handle → 构造 `MlpNode` → `ctx.node.forward(...)` → 保存缓存 → 清理缓存
  - 关键标志：`use_fp8_mlp`, `moe_deep_gemm`, `recompute_moe_gate_up`, `use_auto_subbatch`, `moe_expert_fusion`

- `HybridEPMoePyLayer`: HybridEP 专用 PyLayer，跳过 unzip/zip 直接走 contiguous GEMM | `src/paddlefleet/transformer/moe/fusion_layer_utils.py:1957`
  - `forward(ctx, hidden_states, dispatched_probs, custom_map, ...)` (line 1967)
  - 构造 `ExpertsGroupGemmContiguousNode`，调用 `node.forward(...)` 后直接返回

- `MlpNode`: FusionMoePyLayer 内部的 unzip→GEMM→zip 计算节点 | `src/paddlefleet/transformer/moe/fusion_layer_utils.py:248`
  - `forward(hs_2d_dispatched, dispatched_indices, dispatched_probs)` (line 1506)
    - `use_auto_subbatch=True` → `forward_auto_subbatch(...)` (VMM 感知动态 subbatch)
    - 否则：`_prepare_forward(...)` → group_gemm 路径 或 per_expert subbatch 路径
  - `_prepare_forward(...)` (line 805): 调用 `UnZipNode.forward` + 可选 FP8 量化
  - `forward_auto_subbatch(...)` (line 939): 动态决定 zip_unzip_fusion + subbatch_rows

- `UnZipNode`: token 按 expert 重排节点（unzip = moe_permute） | `src/paddlefleet/transformer/moe/fusion_layer_utils.py:35`
  - `forward(hs_2d_dispatched, dispatched_indices, dispatched_probs, ...)` (line 79)
  - 核心调用：`paddle.nn.functional.moe_permute(hidden_states, scale, dispatched_indices, dispatched_probs, ...)`

- `ZipNode`: token 合并回原顺序节点（zip = moe_unpermute） | `src/paddlefleet/transformer/moe/fusion_layer_utils.py:168`
  - `forward(expert_out, zipped_expertwise_rowmap, routemap_topk, unzipped_probs, ...)` (line 196)
  - 核心调用：`paddle.nn.functional.moe_unpermute(expert_out, zipped_expertwise_rowmap, ...)`

- `ExpertsGroupGemmContiguousNode`: FP8/BF16 双路 GroupedGEMM expert 计算节点 | `src/paddlefleet/transformer/moe/fp8_utils.py:321`
  - `forward(hs_out, unzipped_probs, tokens_per_expert, output, scale)` (line 1246)
    - `fwd_gate_up(x, expert_w1, ...)` (line 505): BF16 → `fwd_gate_up_bf16` / FP8 → `fwd_gate_up_fp8`
    - `fwd_down(o1, unzipped_probs, expert_w2, ...)` (line 672): BF16 → `fwd_down_bf16` / FP8 → `fwd_down_fp8`
  - FP8 路径关键算子：`fused_stack_quant` → `fp8_quant_blockwise` → `split_group_gemm` / `m_grouped_fp8_gemm_nt_contiguous` → `fuse_weighted_swiglu_fp8_quant`
  - 两种权重存储：`self.experts`（list，逐专家）或 `self.grouped_gemm_experts`（stacked tensor，融合）
  - `forward/backward`: 前向直传 output，反向将 aux_loss 梯度叠加到主干梯度流

- `MultiTokenPredictionLayer`: MTP 层 | `src/paddlefleet/transformer/multi_token_prediction.py`

- `GatedDeltaNet`: 线性注意力变体 | `src/paddlefleet/transformer/gated_delta_net.py`

- `CompressedSparseAttention`: CSA 稀疏注意力 | `src/paddlefleet/transformer/csa_attention.py`

- `DSAttention`: DSA 注意力 | `src/paddlefleet/transformer/dsa_attention.py`

- `DSv4HybridSelfAttention`: DSv4 混合注意力 | `src/paddlefleet/transformer/dsv4_hybrid_attention.py`

## SublayersSpec 数据类（扩展点）

- `MLPSublayersSpec`: MLP 子层规格 | `src/paddlefleet/transformer/mlp.py:60`
  - 字段：`up_gate_proj`, `down_proj`

- `SelfAttentionSublayersSpec`: 自注意力子层规格 | `src/paddlefleet/transformer/attention.py`
  - 字段：`qkv_proj`, `core_attention`, `o_proj`, `q_norm`, `k_norm`

- `TransformerLayerSublayersSpec`: TransformerLayer 子层规格 | `src/paddlefleet/transformer/transformer_layer.py`

- `MLASelfAttentionSublayersSpec`: MLA 子层规格 | `src/paddlefleet/transformer/multi_latent_attention.py`

## GPT 模型层类

- `GPTModel`: GPT Transformer 语言模型，继承 `PipelineLayer`，**无自定义 `forward`**，完全委托给 PaddlePaddle `PipelineLayer.forward` | `src/paddlefleet/models/gpt/gpt_model.py:148`
  - `__init__(sublayers_spec: GPTSublayersSpec, **kwargs)`: 构建各层 LayerDesc，调用 `super().__init__(layers=...)`
  - `get_layer_desc_list(spec, tie_word_embeddings)`: 将 spec 中各组件转为 `LayerDesc` / `SharedLayerDesc` 列表
  - `overlapped_forward_backward(...)`: overlap 调度下的前向+反向融合执行（含 `build_overlapped_nodes`）
  - `state_dict / set_state_dict / sharded_state_dict`: PP stage 权重名映射（PP index → 单机权重名）

- `GPTSublayersSpec`: GPT 各阶段层规格的容器 dataclass | `src/paddlefleet/models/gpt/gpt_model.py:129`
  - 字段：`embedding`, `head_empty_layers`, `mhc_expand`, `transformer_layers`, `mhc_contract`, `tail_empty_layers`, `mtp`, `layer_norm`, `lm_head`, `mtp_lm_head`, `mtp_loss`

- `GPTEmbedding`: GPT 嵌入层（FleetLayer 子类），处理 token embedding + RoPE 生成，输出完整 dict_args | `src/paddlefleet/models/gpt/gpt_embedding.py:121`
  - `forward(dict_args, decoder_input, packed_seq_params)`: 从 dict_args 取 input_ids/position_ids，调用 `self.embedding`，可选 fill_feature（EP>1）、MTP 输入拆分

- `GPTLMHead`: LM Head 基类（继承 ColumnParallelLinear），含 recompute 和 SP 支持 | `src/paddlefleet/models/gpt/lm_head.py`
  - `_forward(hidden_states)`: 核心线性投影，支持 fused_linear_ce_loss_chunk 路径
  - `forward(dict_args)`: 标准路径；含 MTP split（若 `num_nextn_predict_layers > 0`）

- `GPTMainLMHead`: 主干 LM Head，含 block_attn_res，单次预测 | `src/paddlefleet/models/gpt/lm_head.py:202`
  - `forward(dict_args)`: 返回 `{logits, mtp_loss}` dict

- `GPTMTPLMHead`: MTP LM Head，对拼接 hidden_states 拆分后逐 MTP depth 计算 logits | `src/paddlefleet/models/gpt/lm_head.py:244`
  - `forward(dict_args)`: 返回带 `mtp_logits` 列表的 dict_args

## 模型构建函数（工厂）

- `gpt_builder(config, **kwargs)`: GPT 模型构建入口 | `src/paddlefleet/gpt_builders.py:33`
- `get_gpt_layer_local_spec(config, ...)`: 构建单层 GPT LayerSpec | `src/paddlefleet/models/gpt/gpt_layer_specs.py`
- `get_gpt_decoder_layers_spec(config, ...)`: 构建 MoE decoder block spec | `src/paddlefleet/models/gpt/gpt_layer_specs.py`
- `get_gpt_mtp_layers_spec(config, ...)`: 构建 MTP 层 spec | `src/paddlefleet/models/gpt/gpt_layer_specs.py`
- `get_attention_spec(config, attention_layer_type, ...)`: 注意力层 spec 工厂 | `src/paddlefleet/models/gpt/gpt_layer_specs.py:109`
- `get_moe_layer_spec_for_backend(config, ...)`: MoE 层 spec 工厂 | `src/paddlefleet/models/gpt/moe_layer_specs.py`

## 并行状态管理

- `parallel_state` 模块：`get_tensor_model_parallel_group()`, `get_pipeline_model_parallel_group()`, `get_context_parallel_group()` 等 | `src/paddlefleet/parallel_state.py`

## 张量并行层

- `ColumnParallelLinear`: 列并行线性层 | `src/paddlefleet/tensor_parallel/layers.py`
- `RowParallelLinear`: 行并行线性层 | `src/paddlefleet/tensor_parallel/layers.py`
- `Linear`: 普通线性层（TP 兼容） | `src/paddlefleet/tensor_parallel/layers.py`

## 归一化层

- `WrappedPaddleNorm`: 封装 Paddle LayerNorm/RMSNorm | `src/paddlefleet/transformer/paddle_norm.py`
- `WrappedPaddleNormPipe`: Pipeline 版本 | `src/paddlefleet/transformer/paddle_norm.py`
- `L2Norm`: L2 归一化 | `src/paddlefleet/transformer/paddle_norm.py`
