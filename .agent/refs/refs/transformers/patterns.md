# transformers 可借鉴的实现模式

---

## 模型目录结构（与 PaddleFormers 完全对应）

- **解决的问题**: 每个模型的文件布局标准化，易于定位
- **核心思路**: HF 每个模型位于 `src/transformers/models/{model_name}/`，包含：
  - `configuration_{model_name}.py` → PaddleFormers: `configuration.py`
  - `modeling_{model_name}.py` → PaddleFormers: `modeling.py`
  - `tokenization_{model_name}.py` → PaddleFormers: `tokenizer.py`
  - `modular_{model_name}.py`（仅 HF，用于代码生成，**PaddleFormers 无此机制**）
  - `__init__.py`（两者均使用懒加载，但实现方式略有不同）
- **参考来源**: transformers/src/transformers/models/llama/
- **迁移建议**: 在 PaddleFormers 中直接对应，用 `configuration.py` / `modeling.py` 命名；`__init__.py` 用 `_LazyModule` + `import_structure` dict 代替 HF 的 `define_import_structure`

---

## Config 类设计（两库接口对应关系）

- **解决的问题**: 新模型 Config 字段的准确来源
- **核心思路**:
  - HF: `class LlamaConfig(PreTrainedConfig)`，字段用**类型注解 + 默认值**声明（`@strict` + `@dataclass` 风格）
  - PaddleFormers: `class LlamaConfig(PretrainedConfig)`，字段在 `__init__` 参数中声明并赋值给 `self.xxx`
  - 字段名称完全一致，默认值以 HF 为准
  - HF 特有字段（`base_model_tp_plan`、`base_model_pp_plan`、`base_model_sp_plan`）是声明式并行配置，PaddleFormers 不需要，可忽略
  - HF 使用 `__post_init__` 做后处理，PaddleFormers 在 `__init__` 内完成，最后调用 `super().__init__(...)`
- **参考来源**: transformers/src/transformers/models/llama/configuration_llama.py:31
- **迁移建议**: 迁移新模型时，直接读取 HF 的 `configuration_xxx.py` 中所有非 `base_model_*_plan` 字段，按原名称和默认值填充到 PaddleFormers 的 `__init__` 参数中

**字段名对照（以 Llama 为例）**:

| HF 字段 | PaddleFormers 字段 | 备注 |
|--------|-------------------|------|
| `vocab_size: int = 32000` | `vocab_size=32000` | 完全一致 |
| `hidden_size: int = 4096` | `hidden_size=4096` | 完全一致 |
| `intermediate_size: int = 11008` | `intermediate_size=11008` | 完全一致 |
| `num_hidden_layers: int = 32` | `num_hidden_layers=32` | 完全一致 |
| `num_attention_heads: int = 32` | `num_attention_heads=32` | 完全一致 |
| `num_key_value_heads` | `num_key_value_heads` | 完全一致 |
| `hidden_act: str = "silu"` | `hidden_act="silu"` | 完全一致 |
| `max_position_embeddings: int = 2048` | `max_position_embeddings=2048` | 完全一致 |
| `rms_norm_eps: float = 1e-6` | `rms_norm_eps=1e-6` | 完全一致 |
| `rope_parameters: RopeParameters` | `rope_scaling` / `rope_theta` | PaddleFormers 拆分成两个字段，需调用 `standardize_rope_params()` 合并 |
| `head_dim: int \| None = None` | `head_dim=None` | 完全一致 |
| `mlp_bias: bool = False` | `mlp_bias=False` | 完全一致 |
| `attention_bias: bool = False` | `attention_bias=False` | 完全一致 |
| `base_model_tp_plan` | **无** | HF 特有，迁移时忽略 |
| `base_model_pp_plan` | **无** | HF 特有，迁移时忽略 |

---

## 模型类层次结构（两库接口对应关系）

- **核心思路**:
  - HF: `PreTrainedModel` → `LlamaPreTrainedModel` → `LlamaModel` / `LlamaForCausalLM`（三层）
  - PaddleFormers: `PretrainedModel` → `LlamaModel` / `LlamaForCausalLM`（两层，无 PreTrained 中间层）
  - HF 的 `LlamaPreTrainedModel` 主要设置类属性（`supports_gradient_checkpointing`、`_no_split_modules` 等），PaddleFormers 直接在 `PretrainedModel` 子类上设置或忽略这些属性
- **参考来源**: transformers/src/transformers/models/llama/modeling_llama.py:336

**类名对照**:

| HF 类 | PaddleFormers 类 | 备注 |
|-------|-----------------|------|
| `PreTrainedModel` | `PretrainedModel` | 大小写差异 |
| `LlamaPreTrainedModel` | **无** | PaddleFormers 无此中间层 |
| `LlamaModel` | `LlamaModel` | 完全一致 |
| `LlamaForCausalLM` | `LlamaForCausalLM` | 完全一致 |
| `LlamaForSequenceClassification` | 暂无 | 按需迁移 |
| `config_class = LlamaConfig` | `config_class = LlamaConfig` | 完全一致 |
| `base_model_prefix = "model"` | `base_model_prefix = "llama"` | **差异**：HF 用 `"model"`，PaddleFormers 用模型名 |

---

## 模型层实现对照（torch → paddle）

- **核心思路**: 层结构完全一致，仅需替换框架 API

**层级替换关系**:

| HF (torch.nn) | PaddleFormers (paddle.nn / 自定义) | 备注 |
|---------------|-----------------------------------|------|
| `nn.Linear(in, out, bias=...)` | `GeneralLinear(in, out, bias=...)` 或 `Linear` from `paddleformers.nn.linear` | 支持 TP |
| `nn.Embedding(vocab, dim, pad)` | `Embedding` from `paddleformers.nn.embedding` | 支持 Vocab Parallel |
| `nn.Linear` (lm_head) | `LMHead` from `paddleformers.nn.lm_head` | 支持权重共享 + TP |
| `LlamaRMSNorm(nn.Module)` | `Norm` from `paddleformers.nn.norm` (RMSNorm) | 直接使用通用 Norm |
| `LlamaMLP(nn.Module)` | `MLP` from `paddleformers.nn.mlp` | 直接使用通用 MLP |
| `ACT2FN[config.hidden_act]` | `ACT2FN[config.hidden_act]` from `paddleformers.transformers.activations` | 一致 |
| `ALL_ATTENTION_FUNCTIONS.get_interface(...)` | `ALL_ATTENTION_FUNCTIONS[attn_impl]` | 接口略有差异，见下方 |
| `GradientCheckpointingLayer` | `paddle.distributed.fleet.utils.recompute(...)` | 机制不同，手动包装 |
| `GenerationMixin` | `GenerationMixin` from `paddleformers.generation` | 一致 |

---

## Attention 实现对照（关键差异）

- **核心思路**: HF 和 PaddleFormers 均使用注册表模式，但接口细节不同

| 维度 | HF transformers | PaddleFormers |
|------|----------------|---------------|
| 注册表对象 | `ALL_ATTENTION_FUNCTIONS`（`modeling_utils.py`） | `ALL_ATTENTION_FUNCTIONS`（`paddleformers.nn.attention.interface`） |
| 获取方式 | `.get_interface(attn_impl, fallback_fn)` | `[attn_impl]`（dict 式访问） |
| 内置实现 | eager / sdpa / flex_attn / flash_attention_2 | eager / sdpa / flashmask |
| forward 签名 | `eager_attention_forward(module, q, k, v, attn_mask, scaling, **kwargs)` | 不同，需根据 PaddleFormers 实际签名适配 |
| RoPE 传入方式 | `position_embeddings: tuple[cos, sin]` 作为参数传入 Attention | 相同模式 |

- **参考来源**: transformers/src/transformers/models/llama/modeling_llama.py:199,272
- **迁移建议**: 迁移 Attention 时，将 `module.num_key_value_groups`、`scaling` 等属性在 `__init__` 中设置，`forward` 调用方式改为 PaddleFormers 的 `ALL_ATTENTION_FUNCTIONS[attn_impl]` 形式

---

## Forward 签名对照（LlamaModel）

| 参数 | HF | PaddleFormers | 备注 |
|-----|----|---------------|------|
| `input_ids` | `torch.LongTensor \| None` | `paddle.Tensor \| None` | 类型换框架 |
| `attention_mask` | `torch.Tensor \| None` | `paddle.Tensor \| None` | 一致 |
| `position_ids` | `torch.LongTensor \| None` | `paddle.Tensor \| None` | 一致 |
| `past_key_values` | `Cache \| None` | `Cache \| None` | 一致 |
| `inputs_embeds` | `torch.FloatTensor \| None` | `paddle.Tensor \| None` | 一致 |
| `use_cache` | `bool \| None` | `bool \| None` | 一致 |
| `**kwargs` | `Unpack[TransformersKwargs]` | `**kwargs` | PaddleFormers 无 TransformersKwargs |
| 返回值 | `BaseModelOutputWithPast` | `BaseModelOutputWithPast` | 一致 |

**HF 特有参数（迁移时无需**）:
- `logits_to_keep`（ForCausalLM）：HF 的推理优化，PaddleFormers 暂无
- `@can_return_tuple`、`@merge_with_config_defaults`、`@capture_outputs` 装饰器：HF 内部机制，迁移时去掉

---

## Modular 文件系统（理解新模型从哪来）

- **解决的问题**: 快速定位新模型的架构来源，了解它是 Llama/Qwen/Gemma 的哪种变体
- **核心思路**: HF 新模型通常在 `modular_{name}.py` 中通过继承已有模型来声明差异，`modeling_{name}.py` 是自动生成的展开版本
  - 读 `modular` 文件可以快速看出：该模型是哪个基础模型的变体、改动了哪些层
  - 例：`modular_qwen3.py` 继承 `LlamaAttention`，只改了 q/k_norm；`modular_ernie4_5.py` 继承 `LlamaAttention`、`LlamaMLP`、`LlamaForCausalLM`
- **参考来源**: transformers/src/transformers/models/qwen3/modular_qwen3.py, transformers/src/transformers/models/ernie4_5/modular_ernie4_5.py
- **迁移建议**: 迁移一个新模型前，**先读其 `modular_xxx.py`**（若存在），而非直接读展开后的 `modeling_xxx.py`；`modular` 文件会告诉你：
  1. 继承自哪个基础模型（对应 PaddleFormers 中已有哪个实现可以参考）
  2. 覆盖了哪些层（只需在 PaddleFormers 中改这些层）
  3. 新增了哪些字段（Config 差异）

---

## chat_template 位置

- **解决的问题**: 判断模型是否有对话模板，以及模板格式
- **核心思路**: HF 的 `chat_template` 存储在模型的 `tokenizer_config.json` 中，字段名为 `chat_template`，值为 Jinja2 模板字符串
  - 可从 HF Hub 获取：`https://huggingface.co/{model_id}/raw/main/tokenizer_config.json`
  - 或读取本地克隆目录中该文件
  - Jinja2 模板中通常包含 user/assistant/system 的 token 格式，以及 EOS 标记方式
- **参考来源**: HF Hub 上各模型的 tokenizer_config.json
- **迁移建议**: 在 `add-model` 中，读取参考模型的 `tokenizer_config.json`，解析 `chat_template` 字段，将 Jinja2 格式的模板转化为 PaddleFormers 的 `register_template(name=..., format_user=..., ...)` 调用

---

## 模型注册机制对照（Auto 模块）

| HF | PaddleFormers | 备注 |
|----|---------------|------|
| `CONFIG_MAPPING`（`auto/configuration_auto.py`） | `CONFIG_MAPPING_NAMES`（`auto/configuration.py`） | PaddleFormers 只存字符串名称，运行时懒加载 |
| `MODEL_FOR_CAUSAL_LM_MAPPING` | `MAPPING_NAMES`（`auto/modeling.py`） | 命名不同但功能一致 |
| `AutoConfig.register(model_type, config_cls)` | 直接在 `CONFIG_MAPPING_NAMES` 和 `MODEL_NAMES_MAPPING` 中追加元组 | 无运行时注册方法，需修改源文件 |
| `model_type = "llama"`（Config 类属性） | `model_type = "llama"`（Config 类属性） | 完全一致 |
| 通过 `__init_subclass__` 自动发现 | **不自动发现**，需手动在 auto/ 文件中注册 | PaddleFormers 需要额外步骤 |

---

## Qwen3 对齐要点（AOA 权重验证）

- **解决的问题**: `/align --check weights` 时准确验证 Qwen3 的 GQA-interleaved QKV 和 fused FFN 权重
- **核心思路**:
  - **QKV 融合布局（GQA interleaved）**: paddle `qkv_proj.weight` 形状 `[hidden, num_kv_heads*(num_kv_groups+2)*head_dim]`；对每个 KV-group `g`，列偏移 `base=g*stride`（stride=(num_kv_groups+2)*head_dim），布局为 `[Q-head₀, Q-head₁, ..., K-head, V-head]` × head_dim
  - **FFN 融合**：`up_gate_proj.weight = cat([gate_proj^T, up_proj^T], axis=1)`，gate 在前 up 在后（尽管名字叫 `up_gate`）
  - **q_norm / k_norm 权重**：`Qwen3RMSNorm` 显式用 `dtype=paddle.float32` 创建，即使整体模型是 bfloat16，这两个参数也是 float32；比较时需先把 HF 权重转 float32
  - **AOA 参数命名陷阱**：语句中 `num_key_value_groups=config.num_key_value_heads`，含义是 KV head 数量（不是 Q/KV 的比值 num_kv_groups）
- **参考来源**: `paddleformers/transformers/qwen3/modeling.py:337`（`_gen_aoa_config`），`align_qwen3_bfloat16.py`（weights check 完整实现）
- **迁移建议**: 其他 GQA 模型（Llama3、Qwen2 等）若用同样 interleaved 融合，可直接复用 align 脚本中的切片逻辑

从 HF 迁移一个新模型（以某新模型 `xxx` 为例）：

1. **读 `modular_xxx.py`**（若存在）→ 了解它继承自哪个基础模型、改了哪些层
2. **读 `configuration_xxx.py`** → 提取所有字段名称和默认值（忽略 `base_model_*_plan`）
3. **读 `tokenizer_config.json`** → 检查是否有 `chat_template`
4. **创建 `paddleformers/transformers/xxx/configuration.py`** → 字段一一对应
5. **创建 `modeling.py`** → 参考 HF 层结构，替换 torch.nn → paddle/paddleformers.nn
6. **注册 Auto 模块** → 手动修改 `auto/configuration.py` 和 `auto/modeling.py`
7. **注册 Template**（若有 chat_template）→ 在 `datasets/template/template.py` 末尾添加

- **参考来源**: 以上各文件综合
