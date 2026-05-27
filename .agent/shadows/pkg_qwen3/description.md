# pkg_qwen3 功能描述

本描述供 `/extend` 恢复实现时参考，不含任何原始代码。

## 包边界与整体职责

`paddleformers/transformers/qwen3/` 是 Qwen3 系列基础（dense）模型的完整实现包，对应 model_type `"qwen3"`（Qwen3-0.6B/1.7B/4B/8B/14B/32B 等）。包内三个文件分工如下：

---

## `configuration.py` — `Qwen3Config`

继承 `PretrainedConfig`，负责存储 Qwen3 架构超参数并在初始化时验证合法性。

**主要职责**：
- 保存词表大小、隐藏层维度、中间层维度、层数、注意力头数（含 GQA key-value 头数）、head_dim、激活函数、最大序列长度等架构参数
- 管理 RoPE 配置（rope_theta、rope_scaling），在 `__init__` 末尾调用 `standardize_rope_params` 和 `rope_config_validation` 验证参数
- 管理滑动窗口注意力参数（use_sliding_window、sliding_window、max_window_layers），自动生成每层的 layer_types（full_attention 或 sliding_attention）
- 通过 `layer_type_validation` 校验 layer_types 与 num_hidden_layers 的一致性

**契约**：`model_type = "qwen3"`，`keys_to_ignore_at_inference = ["past_key_values"]`

---

## `modeling.py` — 模型实现

包含以下公开类（均注册到 `paddleformers.transformers` 命名空间）：

### `Qwen3ModelProvider`
继承 `GPTModelProvider`，为 Qwen3 模型提供 PaddleFleet 适配器配置。关键属性：`model_type = "qwen3"`，`use_qk_norm = True`，`bias_activation_fusion = True`，`persist_layer_norm = True`，`share_embeddings_and_output_weights = False`。提供 `save_pretrained` 方法将配置序列化为 JSON。

### `Qwen3Attention`
多头注意力层，支持 GQA（Grouped Query Attention）和滑动窗口注意力。
- 使用 `GeneralLinear` 做 QKV 融合投影（`qkv_proj`）和输出投影（`o_proj`），支持 Tensor Parallel（colwise/rowwise 切分）
- 使用 `GeneralNorm` 对 Q/K 做 RMS 归一化（QK-Norm）
- 支持 Sequence Parallel，通过 `ScatterOp` 处理序列维度的并行散射
- 调用 `ALL_ATTENTION_FUNCTIONS` 注册表执行具体 attention 计算（eager/sdpa/flashmask）

### `Qwen3MLP`（来自 `nn.mlp`）
复用 `paddleformers.nn.MLP` 作为 Qwen3 的前馈层，别名导入为 `Qwen3MLP`。

### `Qwen3DecoderLayer`
单个 Transformer 解码器层：先 self-attention（含 pre-norm），再 MLP（含 pre-norm），支持 recompute 梯度检查点。

### `Qwen3PretrainedModel`
继承 `PretrainedModel`，提供模型权重初始化（`_init_weights`）。

### `Qwen3Model`（@register_base_model）
完整的 Qwen3 基础模型：`Embedding` → N × `Qwen3DecoderLayer` → `RMS Norm`。
- 支持 `past_key_values`（KV 缓存）、`use_cache`、`output_hidden_states`、`output_attentions`、`return_dict`
- 支持 Pipeline Parallelism（通过 `pipeline_model_parallel_size`）

### `Qwen3ForCausalLM`
因果语言模型头：`Qwen3Model` + `GeneralLMHead`，包含 `Qwen3ModelProvider` 适配器。
- 支持 `CriterionLayer` 集成损失计算
- 支持 `generate` 接口（继承 `GenerationMixin`）

### `Qwen3ForCausalLMPipe`
Pipeline 并行版 ForCausalLM，继承 `GeneralModelForCausalLMPipe`，支持 PP + TP 混合并行。

### `Qwen3ForSequenceClassification`
序列分类头：`Qwen3Model` + score 线性层，使用 `CrossEntropyLoss`。

### `Qwen3ForTokenClassification`
Token 分类头：`Qwen3Model` + dropout + score 线性层，使用 `CrossEntropyLoss`。

### `Qwen3SentenceEmbedding`
句向量表示模型：`Qwen3Model` + `SimpleContrastiveLoss`，支持跨设备 in-batch negative。

### `Qwen3ForCausalLMDeprecated` / `Qwen3ForCausalLMPipeDeprecated`
旧版兼容类，功能同上，通过别名方式保持向后兼容。

---

## `__init__.py` — 懒加载入口

使用 `_LazyModule` 机制注册 `Qwen3Config`、`Qwen3Model`、`Qwen3ForCausalLM` 等所有公开符号，仅在实际访问时触发模块加载。

---

## 在代码库中的注册位置

- `paddleformers/transformers/__init__.py`：懒加载结构注册（`qwen3.configuration`、`qwen3.modeling`）
- `paddleformers/transformers/auto/configuration.py`：`CONFIG_MAPPING_NAMES["qwen3"] = "Qwen3Config"`、`MODEL_NAMES_MAPPING["qwen3"] = "Qwen3"`
- `paddleformers/transformers/auto/modeling.py`：`MAPPING_NAMES["Qwen3"] = "qwen3"`
- `paddleformers/cli/utils/llm_utils.py`：LoRA target_modules 分支（`model_type == "qwen3"`）
- `paddleformers/datasets/template/template.py`：`register_template(name="qwen3", ...)`（ReasoningTemplate，qwen 格式对话模板）
