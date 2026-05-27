# 扩展模式

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

---

## 跨框架数值对齐（CompatibilityTest 模式）

参考：`tests/transformers/qwen3/test_modeling.py:411`（17 个模型均有此模式）

**核心三步骤**：
1. `setUpClass`：创建 tiny 随机权重 HF 模型（hidden_size=16, num_hidden_layers=2），`save_pretrained` 到临时目录
2. 测试方法：torch 侧 `from_pretrained(path, torch_dtype=torch.float32)`，paddle 侧 `from_pretrained(path, dtype="float32", load_checkpoint_format="flex_checkpoint")`
3. 数值比较：取 logits 前 9 个值，`np.allclose(atol=1e-2, rtol=1e-2)`

**必须设置的确定性计算环境变量**（对齐精度关键，在 `CompatibilityTest.setUp` 或脚本开头设置）：
```bash
unset PADDLE_ELASTIC_JOB_ID PADDLE_TRAINER_ENDPOINTS DISTRIBUTED_TRAINER_ENDPOINTS
export NNODES=1 PADDLE_TRAINERS_NUM=1 FLAGS_selected_gpus=0
export FLAGS_use_accuracy_compatible_kernel=1 FLAGS_cudnn_deterministic=1
```

**dtype 容忍度对应关系**：
- `float32`：atol=1e-2，实测 max_diff < 5e-5，mean_diff < 1e-5（N 层内积累误差极小）
- `bfloat16`：atol=1e-1；实测 Qwen3-0.6B（28 层）max_diff ≈ 0.09，mean_diff ≈ 0.04，差值为 bfloat16 最小步长（2^{-4}=0.0625）的整数倍，属正常精度损失
- `float16`：先用 float32 验证通过后再测

**bfloat16 对齐失败的判断准则**：用 float32 复现 → float32 也有较大差距才是 IMPL_BUG；float32 通过而 bfloat16 不通过说明容差太严，不需要修改实现。

**flex_checkpoint 加载时的 Warning 是正常现象**：
- `Missing keys: {qkv_proj, up_gate_proj}` — AOA 在加载时将 q/k/v_proj 融合为 qkv_proj，此 warning 说明 AOA 正在工作
- `Unexpected keys: {q_proj, k_proj, v_proj, gate_proj, up_proj}` — HF 格式的独立 key，已被 AOA 融合消费
- `Unexpected keys: {lm_head.weight}` — tie_word_embeddings=True 时 paddle 模型不单独存 lm_head.weight，属正常

**权重对齐（--check weights）注意事项**：
- `Qwen3RMSNorm.weight`（q_norm、k_norm）显式创建为 float32，对比时需从 bfloat16 state 转换再比较
- GQA interleaved QKV 布局：stride=(num_kv_groups+2)*head_dim，每 KV-group 内依次是 [Q×num_kv_groups, K, V]；参考 `align_qwen3_bfloat16.py` 中的切片逻辑
- AOA statement 中参数名 `num_key_value_groups=config.num_key_value_heads` 实为 KV head 数量（命名有歧义，不是 ratio）

**对齐使用的模型类**：
- `Qwen3ForCausalLMDeprecated` — 标准 from_pretrained 接口，用于对齐
- `Qwen3ForCausalLM` — fleet 版本，用 `__new__` 返回 GPT provider 对象，不支持直接 from_pretrained 对齐

**`--check loss` labels 传参规则（关键差异）**：
- HF `ForCausalLM.forward(input_ids, labels=labels)` **内部移位**：`CE(logits[:-1], labels[1:])`
- PaddleFormers `CriterionLayer`（`Qwen3ForCausalLMDeprecated.forward` 调用）**不移位**：直接 `CE(logits, labels)`, `ignored_index=-100` 的位置被遮蔽
- 对齐写法：HF 传 `labels=input_ids`；PaddleFormers 传预移位 labels：
  ```python
  labels_pf = np.concatenate([input_ids[:, 1:], np.full((batch,1), -100, dtype=np.int64)], axis=1)
  paddle_model(paddle.to_tensor(input_ids), labels=paddle.to_tensor(labels_pf))
  ```
- 实测 float32 loss diff ≈ 6e-6，远低于 `ATOL * 10 = 1e-1`；参考 `align_qwen3_float32.py`

**`--check train` 端到端训练配置要点**（`paddleformers-cli train`）：

- **dtype 匹配**：HF checkpoint 通常是 bfloat16；YAML 必须同时设置 `bf16: true` + `fp16_opt_level: O2`，否则 SFT workflow 默认 float32，AOA 加载时报 dtype 断言错误
- **单卡运行**：在 `subprocess.run` 的 `env` 中设置 `CUDA_VISIBLE_DEVICES: "N"`（选空闲卡），cli 读此变量只启动 1 个进程，避免多卡分布式干扰
- **数据集 prob 字段**：`train_dataset_type: erniekit` 必须同时设置 `train_dataset_prob: "1.0"` 和 `eval_dataset_prob: "1.0"`，否则 `MultiSourceDataset` 报 `ValueError: could not convert string to float: 'None'`
- **torch 侧必须 `.cuda()`**：`from_pretrained` 默认加载到 CPU；训练循环前必须调用 `model.cuda()`，input tensor 也需 `.cuda()`，否则 backward 在 CPU 上极慢（0.6B 模型可挂起 10+ 分钟）
- **GPU 选择**：多卡机器上不同进程应使用不同卡（`CUDA_VISIBLE_DEVICES`），避免同卡竞争导致进程挂起

参考：`align_qwen3_train.py`，`paddleformers/cli/train/sft/workflow.py:247-255`，`paddleformers/cli/cli.py:91-98`


- `/align` 验证通过后，**不自动写入 test_modeling.py**；待用户显式运行其他指令时再固化


**失败后的调试层次**：
1. 钩子捕获逐层激活值 → 定位第一个发散层
2. CPU 确定性检验 → 区分 IMPL_BUG（稳定但与 torch 不符）vs OP_BUG（CPU 也不稳定）
3. IMPL_BUG → 修改 modeling.py；OP_BUG → 输出算子问题报告，不修改实现

参考 `/align` command 获取完整流程。

---

## 标准单元测试结构（模型测试）

参考：`tests/transformers/ernie4_5/test_modeling.py`

```python
import unittest
import numpy as np
import paddle
from paddleformers.transformers import XxxConfig, XxxForCausalLM, XxxModel
from tests.testing_utils import gpu_device_initializer, require_package
from tests.transformers.test_modeling_common import ModelTesterMixin, ids_tensor

class XxxModelTester:
    """配置+数据准备的辅助类"""
    def __init__(self, parent, vocab_size=1000, hidden_size=64, ...):
        self.parent = parent
        ...

    def prepare_config_and_inputs(self):
        input_ids = ids_tensor([self.batch_size, self.seq_length], self.vocab_size)
        config = XxxConfig(vocab_size=self.vocab_size, ...)
        return config, input_ids

    def create_and_check_model(self, config, input_ids):
        model = XxxModel(config)
        model.eval()
        result = model(input_ids)
        self.parent.assertEqual(result.last_hidden_state.shape, [...])

class XxxModelTest(ModelTesterMixin, unittest.TestCase):
    all_model_classes = (XxxModel, XxxForCausalLM)

    def setUp(self):
        self.model_tester = XxxModelTester(self)

    def test_model(self):
        config, input_ids = self.model_tester.prepare_config_and_inputs()
        self.model_tester.create_and_check_model(config, input_ids)
```
