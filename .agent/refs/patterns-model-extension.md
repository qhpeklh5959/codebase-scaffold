# 扩展模式 — 模型扩展

## 添加新模型（最常见扩展场景）

参考示例：`paddleformers/transformers/llama/`

**步骤 1：创建模型目录和 configuration.py**
```python
# paddleformers/transformers/{model_name}/configuration.py
from ..configuration_utils import PretrainedConfig
from ..modeling_rope_utils import rope_config_validation, standardize_rope_params  # 若需要 RoPE

class XxxConfig(PretrainedConfig):
    model_type = "xxx"  # 必须唯一，用于 AutoConfig 注册

    def __init__(self, vocab_size=..., hidden_size=..., **kwargs):
        # 设置模型特有字段
        self.vocab_size = vocab_size
        ...
        super().__init__(**kwargs)
```

**步骤 2：创建 modeling.py**
```python
# paddleformers/transformers/{model_name}/modeling.py
from ...nn.attention.interface import ALL_ATTENTION_FUNCTIONS
from ...nn.criterion.interface import CriterionLayer
from ...nn.embedding import Embedding as GeneralEmbedding
from ...nn.linear import Linear as GeneralLinear
from ...nn.lm_head import LMHead as GeneralLMHead
from ...nn.mlp import MLP
from ...nn.norm import Norm as GeneralNorm
from ...nn.pp_model import GeneralModelForCausalLMPipe
from ..model_outputs import BaseModelOutputWithPast, CausalLMOutputWithPast
from ..model_utils import PretrainedModel, register_base_model
from .configuration import XxxConfig

@register_base_model
class XxxModel(PretrainedModel):
    config_class = XxxConfig
    base_model_prefix = "xxx"
    ...

class XxxForCausalLM(PretrainedModel):
    config_class = XxxConfig
    base_model_prefix = "xxx"
    ...
```

**步骤 3：创建 `__init__.py`（懒加载）**
```python
import sys
from typing import TYPE_CHECKING
from ...utils.lazy_import import _LazyModule

import_structure = {
    "configuration": ["XxxConfig"],
    "modeling": ["XxxModel", "XxxForCausalLM", "XxxForCausalLMPipe"],
    "tokenizer": ["XxxTokenizer"],  # 若有
}

if TYPE_CHECKING:
    from .configuration import *
    from .modeling import *
else:
    sys.modules[__name__] = _LazyModule(
        __name__, globals()["__file__"], import_structure, module_spec=__spec__
    )
```

**步骤 4：注册到 auto 模块**
- 在 `paddleformers/transformers/auto/configuration.py` 的 `CONFIG_MAPPING` 中添加 `"xxx": XxxConfig`
- 在 `paddleformers/transformers/auto/modeling.py` 的 `MODEL_FOR_CAUSAL_LM_MAPPING` 中添加映射
- 在 `paddleformers/transformers/auto/tokenizer.py` 的 `TOKENIZER_MAPPING` 中添加（若有 tokenizer）

**步骤 5：在顶层 `paddleformers/transformers/__init__.py` 导出**
- 将 `"transformers.{model_name}"` 及相关类添加到 `import_structure`

示例参考：`paddleformers/transformers/llama/` (configuration.py:19, modeling.py:1+)

---

## Fleet 版 vs Deprecated 版（过渡期双版本）

当一个模型在 fleet（PaddleFleet）中已有完整实现时，PaddleFormers 侧的原生实现会被标记为 Deprecated，形成过渡期双版本：

- **`XxxForCausalLM`（fleet 版，正式版）**：`__new__` 返回 `GPTModelProvider.provide()` 的对象，用于生产训练（TP/PP/SP 并行）；不支持直接 `from_pretrained` 对齐
- **`XxxForCausalLMDeprecated`（formers 原生版，已废弃）**：标准 `PretrainedModel` 子类，是 fleet 版出现前的原始实现；保留用于对齐验证和推理测试，长期将被移除

**正常情况（fleet 无实现）**：只有单版本 `XxxForCausalLM`，直接继承 `PretrainedModel`，无 Deprecated 后缀。

对齐脚本和 `CompatibilityTest` 必须使用 `XxxForCausalLMDeprecated`；训练 YAML 使用 `XxxForCausalLM`（fleet 版）。

参考：`paddleformers/transformers/qwen3/modeling.py:643-674`

---

## 添加新的 Attention 函数

参考：`paddleformers/nn/attention/eager_attention.py`

**步骤 1：实现 attention forward 函数**
函数签名约定：`def my_attention_forward(query, key, value, attention_mask, ...) -> Tensor`

**步骤 2：注册到 AttentionInterface**
```python
from paddleformers.nn.attention.interface import AttentionInterface
AttentionInterface.register("my_attn", my_attention_forward)
```

**步骤 3：模型使用时**
```python
attn_fn = ALL_ATTENTION_FUNCTIONS[config.attn_implementation]
output = attn_fn(q, k, v, attention_mask, ...)
```

---

## 添加新的 Loss 函数

参考：`paddleformers/nn/criterion/sft_loss.py`

**步骤 1：实现 loss forward 函数**
函数签名约定参考 sft_loss_forward

**步骤 2：注册到 LossInterface**
```python
from paddleformers.nn.criterion.interface import LossInterface
LossInterface.register("my_loss", my_loss_forward)
```

---

## 添加新的 TrainerCallback

参考：`paddleformers/trainer/trainer_callback.py:262`（PrinterCallback 示例）

```python
from paddleformers.trainer import TrainerCallback

class MyCallback(TrainerCallback):
    def on_train_begin(self, args, state, control, **kwargs):
        print("Training started!")

    def on_log(self, args, state, control, logs=None, **kwargs):
        # 自定义日志逻辑
        pass

# 使用
trainer = Trainer(..., callbacks=[MyCallback()])
```

---

## 添加新的多模态插件（MM Plugin）

参考：`paddleformers/datasets/template/mm_plugin.py`（ErnieVLPlugin 等）

**步骤 1：继承 BasePlugin**
```python
from paddleformers.datasets.template.mm_plugin import BasePlugin, register_mm_plugin

class MyVLPlugin(BasePlugin):
    def process_messages(self, messages, images, videos, processor):
        ...
    def get_mm_inputs(self, images, videos, imglens, vidlens, seqlens, processor):
        ...
```

**步骤 2：注册**
```python
register_mm_plugin("my_model", MyVLPlugin)
```
