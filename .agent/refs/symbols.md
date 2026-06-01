# 关键符号索引

## 接口 / 抽象类（扩展点）

- **PretrainedConfig**: 所有模型配置的基类，含 from_pretrained/save_pretrained/to_dict 等 | paddleformers/transformers/configuration_utils.py:613
  - `__init__(self, **kwargs)`: 初始化通用 config 字段（pad/bos/eos token id、parallel 参数等）
  - `from_pretrained(cls, pretrained_model_name_or_path, **kwargs)`: 从本地或 hub 加载 config
  - `save_pretrained(self, save_directory, **kwargs)`: 保存 config 到目录
  - `to_dict(self) -> Dict`: 序列化为字典

- **PretrainedModel**: 所有预训练模型的基类，继承自 `paddle.nn.Layer + GenerationMixin + ConversionMixin` | paddleformers/transformers/model_utils.py:1184
  - `from_pretrained(cls, pretrained_model_name_or_path, *model_args, **kwargs)`: 加载权重
  - `save_pretrained(self, save_directory, **kwargs)`: 保存模型权重
  - `forward(self, ...)`: 子类必须实现
  - 类属性 `model_type`、`base_model_prefix`、`config_class` 需在子类中覆盖

- **TrainerCallback**: 训练钩子基类，所有自定义训练回调须继承此类 | paddleformers/trainer/trainer_callback.py:219
  - `on_init_end(args, state, control, **kwargs)`: 初始化结束
  - `on_train_begin/end(args, state, control, **kwargs)`: 训练开始/结束
  - `on_epoch_begin/end(args, state, control, **kwargs)`: epoch 开始/结束
  - `on_step_begin/end(args, state, control, **kwargs)`: step 开始/结束
  - `on_evaluate(args, state, control, **kwargs)`: 评估时
  - `on_save(args, state, control, **kwargs)`: 保存 checkpoint 时
  - `on_log(args, state, control, logs=None, **kwargs)`: 日志时

- **GeneralInterface**: dict-like 函数注册基类，子类通过 `_global_mapping` 注册函数 | paddleformers/nn/general.py:19
  - `register(cls, key, value)`: 类方法，全局注册新函数
  - `__setitem__(key, value)`: 实例级局部覆盖

- **AttentionInterface(GeneralInterface)**: attention 函数注册表 | paddleformers/nn/attention/interface.py:22
  - 内置 key: `"eager"`, `"sdpa"`, `"flashmask"`
  - 扩展方式: `AttentionInterface.register("new_attn", my_fn)`

- **LossInterface(GeneralInterface)**: loss 函数注册表 | paddleformers/nn/criterion/interface.py:27
  - 内置 key: `"sft"`, `"dpo"`, `"kto"`, `"mtp_sft"`
  - 扩展方式: `LossInterface.register("new_loss", my_fn)`

- **BasePlugin** (多模态插件基类): 多模态数据处理扩展点 | paddleformers/datasets/template/mm_plugin.py
  - 子类示例: `ErnieVLPlugin`, `Qwen2VLPlugin`, `Qwen3VLPlugin`, `GLM4VPlugin`
  - 注册方式: `register_mm_plugin(name, cls)` / `get_mm_plugin(name)`

- **Template**: 对话模板基类 | paddleformers/datasets/template/template.py
  - 子类: `ReasoningTemplate`, `Llama2Template`
  - 注册方式: `register_template(name, template)` / `get_template_and_fix_tokenizer(...)`

- **MOELayerBase**: MoE 层抽象基类 | paddleformers/nn/moe/abstract.py
  - 扩展方式: 实现子类后通过 `create_moe_block()` 工厂函数创建

- **CriterionLayer(nn.Layer)**: 统一 loss 层，包装 LossInterface 调用 | paddleformers/nn/criterion/interface.py:40

## 核心类

- **Qwen3Config(PretrainedConfig)**: Qwen3 模型配置 | paddleformers/transformers/qwen3/configuration.py:21
  - `model_type = "qwen3"`
  - 关键字段：`vocab_size`, `hidden_size`, `intermediate_size`, `num_hidden_layers`, `num_attention_heads`, `num_key_value_heads`, `head_dim`, `rms_norm_eps`, `rope_theta`, `rope_scaling`, `use_sliding_window`, `sliding_window`, `max_window_layers`, `layer_types`, `attention_bias`
  - `layer_types`：每层 attention 类型列表（`"full_attention"` / `"sliding_attention"`），None 时自动生成

- **Qwen3PretrainedModel(PretrainedModel)**: Qwen3 模型基类 | paddleformers/transformers/qwen3/modeling.py
  - 持有 `_gen_aoa_config` / `_gen_inv_aoa_config` 静态方法，定义 HF↔PaddleFormers 权重映射（AOA）

- **Qwen3Model(Qwen3PretrainedModel)**: Qwen3 Transformer decoder 主体 | paddleformers/transformers/qwen3/modeling.py:480
  - `@register_base_model`；`forward` 返回 `BaseModelOutputWithPast`
  - 子层：`embed_tokens`（GeneralEmbedding）、`layers`（Qwen3DecoderLayer 列表）、`norm`（GeneralNorm RMSNorm）、`rotary_emb`（Qwen3RotaryEmbedding）

- **Qwen3ForCausalLM(Qwen3PretrainedModel)**: Fleet 版 Causal LM（生产训练用）| paddleformers/transformers/qwen3/modeling.py:643
  - `is_fleet = True`；`__new__` 返回 `Qwen3ModelProvider.provide()` 的 GPT provider 对象
  - 不支持直接 `from_pretrained` 对齐；用于 `paddleformers-cli train` 端到端训练

- **Qwen3ForCausalLMDeprecated(Qwen3PretrainedModel)**: formers 原生 Causal LM（fleet 版出现前的实现，已废弃，保留用于对齐/推理测试）| paddleformers/transformers/qwen3/modeling.py:665
  - 标准 `PretrainedModel` 子类，支持 `from_pretrained`；含 `Qwen3Model` + `GeneralLMHead` + `CriterionLayer`
  - `_tied_weights_keys = ["lm_head.weight"]`；`enable_to_static_method = True`

- **Qwen3ForSequenceClassification(Qwen3PretrainedModel)**: 序列分类 | paddleformers/transformers/qwen3/modeling.py:737
- **Qwen3ForTokenClassification(Qwen3PretrainedModel)**: Token 分类 | paddleformers/transformers/qwen3/modeling.py:825
- **Qwen3SentenceEmbedding(Qwen3PretrainedModel)**: 句向量/对比学习 | paddleformers/transformers/qwen3/modeling.py:882
- **Qwen3ForCausalLMPipe(Qwen3PretrainedModel, GeneralModelForCausalLMPipe)**: Pipeline Parallel 版 | paddleformers/transformers/qwen3/modeling.py:936

- **Qwen3RMSNorm(nn.Layer)**: 每头 QK-Norm，作用于 head_dim 维度 | paddleformers/transformers/qwen3/modeling.py:119
  - `weight` 显式创建为 `float32`（即使模型为 bfloat16）；forward 内部转 float32 计算后还原输入 dtype

- **Qwen3ModelProvider(GPTModelProvider)**: Fleet 模型 provider dataclass | paddleformers/transformers/qwen3/modeling.py:61
  - `use_qk_norm=True`；`transform_rules = {"dtype": "params_dtype"}`

- **LlamaConfig(PretrainedConfig)**: Llama 系列模型配置 | paddleformers/transformers/llama/configuration.py:19
  - 字段: `vocab_size`, `hidden_size`, `num_hidden_layers`, `num_attention_heads`, `num_key_value_heads`, `rope_theta`, `rope_scaling`
  - `model_type = "llama"`

- **LlamaForCausalLM(PretrainedModel)**: Llama Causal LM 模型 | paddleformers/transformers/llama/modeling.py
  - 依赖 `LlamaModel`（base）+ `LMHead`
  - `forward(input_ids, attention_mask, labels, ...)`: 返回 CausalLMOutputWithPast

- **Trainer**: 主训练器，支持 TP/PP/SP/Sharding/FSDP 并行 | paddleformers/trainer/trainer.py
  - `__init__(self, model, args, train_dataset, eval_dataset, ...)`: 初始化
  - `train(self, resume_from_checkpoint=None)`: 启动训练
  - `evaluate(self, eval_dataset=None)`: 执行评估

- **TrainingArguments**: 训练参数数据类（@dataclass）| paddleformers/trainer/training_args.py
  - 包含 learning_rate、per_device_train_batch_size、tensor_parallel_degree、pipeline_parallel_degree 等

- **AutoConfig**: 自动根据 config.json 中 model_type 选择 Config 类 | paddleformers/transformers/auto/configuration.py
- **AutoModelForCausalLM**: 自动根据 model_type 选择模型类 | paddleformers/transformers/auto/modeling.py
- **AutoTokenizer**: 自动选择 tokenizer 类 | paddleformers/transformers/auto/tokenizer.py

- **SFTDataSet**: SFT 训练数据集 | paddleformers/datasets/SFTDataset.py
- **DPODataSet**: DPO 训练数据集 | paddleformers/datasets/DPODataset.py

- **Embedding(GeneralLinear)**: 通用 Embedding 层，含并行支持 | paddleformers/nn/embedding.py
- **Linear**: 通用 Linear 层，含 TP/Sequence Parallel 支持 | paddleformers/nn/linear.py
- **LMHead**: 语言模型头，通常与 Embedding 权重共享 | paddleformers/nn/lm_head.py
- **MLP**: 通用 MLP 层（含 gate_proj / up_proj / down_proj） | paddleformers/nn/mlp.py
- **Norm**: 通用 Norm 层（LayerNorm / RMSNorm） | paddleformers/nn/norm.py
- **GeneralModelForCausalLMPipe**: Pipeline Parallel 模型基类 | paddleformers/nn/pp_model.py
