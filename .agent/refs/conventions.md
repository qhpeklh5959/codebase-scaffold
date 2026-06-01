# 编码规范

## 命名规范
- **类名**: PascalCase（如 `LlamaForCausalLM`, `TrainerCallback`, `GeneralInterface`）
- **函数/方法名**: snake_case（如 `from_pretrained`, `save_pretrained`, `on_train_begin`）
- **常量**: UPPER_SNAKE_CASE（如 `PADDLE_WEIGHTS_NAME`, `CONFIG_NAME`, `ALL_ATTENTION_FUNCTIONS`）
- **私有属性/方法**: 下划线前缀（如 `_global_mapping`, `_local_mapping`）
- **模型目录名**: 小写+下划线（如 `ernie4_5`, `qwen2_vl`, `deepseek_v3`）
- **测试文件**: `test_*.py`（普通测试）或 `test_ai_*.py`（AI 生成测试，在 `tests/ai_edited_test/` 下）

## 包 / 模块组织规则
- 每个模型独立子目录 `paddleformers/transformers/{model_name}/`，固定包含：
  - `configuration.py`：`XxxConfig(PretrainedConfig)`
  - `modeling.py`：`XxxModel`、`XxxForCausalLM`（及 Pipe 变体）
  - `tokenizer.py`（可选）/`tokenizer_fast.py`（可选）
  - `__init__.py`：使用 `_LazyModule` 做懒加载导出
- 所有公共符号通过 `__init__.py` 中的 `import_structure` dict 声明（懒加载模式）
- `paddleformers/nn/` 存放跨模型共享的积木层
- 测试目录与源码目录镜像对应（如 `tests/transformers/llama/` 对应 `paddleformers/transformers/llama/`）

## 懒加载模式（重要）
```python
# __init__.py 固定模式
import sys
from typing import TYPE_CHECKING
from ...utils.lazy_import import _LazyModule

import_structure = {
    "configuration": ["XxxConfig"],
    "modeling": ["XxxModel", "XxxForCausalLM"],
    "tokenizer": ["XxxTokenizer"],
}

if TYPE_CHECKING:
    from .configuration import *
    from .modeling import *
else:
    sys.modules[__name__] = _LazyModule(
        __name__, globals()["__file__"], import_structure, module_spec=__spec__
    )
```

## 错误处理规范
- 使用 `from ..utils.log import logger` 获取统一 logger
- 警告用 `logger.warning(...)` / `warnings.warn(...)`
- 断言用原生 `assert`，含可读错误消息（如 `assert a == b, f"Found {a} and {b}"`）
- 条件导入模式（可选依赖）：
  ```python
  try:
      from some_optional_pkg import Foo
  except ImportError:
      Foo = None
  ```
  或用 `is_paddlefleet_available()` 等工具函数判断

## 注释风格
- 文件头：Apache 2.0 License 声明 + `# This file is modified from ...`（若改自其他开源库）
- 类/函数：Google-style docstring，含 `Args:`、`Returns:` 节
- 行内注释：简洁说明，格式 `# comment`
- 无强制 JSDoc/Javadoc 要求，但公开 API 方法须有 docstring

## 并行策略相关约定
- Config 中通过 `tensor_model_parallel_size`、`pipeline_model_parallel_size` 等字段控制并行度
- 模型层使用 `paddleformers.nn.Linear`（不直接用 `paddle.nn.Linear`）以支持 TP
- Pipe 变体（`XxxForCausalLMPipe`）继承 `GeneralModelForCausalLMPipe` 以支持 PP

## 跨库路径约定

- `deps.md` 和 `refs-registry.md` 中的 **路径** 字段均相对于 **PaddleFormers 项目根目录**（即 `../SiblingLib`，而非相对于文件自身）
- overview.md 等子文件中的路径说明同样以项目根为基准，需标注"相对于 PaddleFormers 项目根目录"
- Agent 解析路径时，以 `os.path.join(project_root, path)` 展开，而非相对于 `.agent/refs/` 目录

来源：`.agent/refs/deps.md`、`.agent/refs/refs-registry.md`（2026-05-29 规范化）

---

## 测试规范
- 测试类命名：`class XxxModelTester`（配置 tester）+ `class XxxModelTest(ModelTesterMixin, unittest.TestCase)`
- 混入类：`ModelTesterMixin`、`ModelTesterPretrainedMixin`、`GenerationTesterMixin`（在 `tests/transformers/test_modeling_common.py`）
- 常用工具：`ids_tensor`, `random_attention_mask`（来自 `test_modeling_common`）
- 装饰器：`@require_package("paddle")` / `@gpu_device_initializer`（来自 `tests/testing_utils.py`）
- 运行命令：`DOWNLOAD_SOURCE=aistudio PYTHONPATH=. pytest tests/transformers/xxx/`
