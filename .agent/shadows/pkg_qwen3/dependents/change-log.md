# pkg_qwen3 Unload 依赖方修改记录

## paddleformers/transformers/__init__.py
- 第 255-267 行：移除 `"qwen3.configuration": ["Qwen3Config"]` 和 `"qwen3.modeling": [...]` 懒加载注册块（含 Qwen3Config、Qwen3Model、Qwen3PretrainedModel 等 10 个符号）
- 第 288 行：移除 `"qwen3": []` 子包占位注册项
- 第 398 行：移除 `from .qwen3 import *` TYPE_CHECKING 导入语句

## paddleformers/transformers/auto/configuration.py
- 第 47 行：`("qwen3", "Qwen3Config")` → 注释 `# [UNLOADED: pkg_qwen3]`（CONFIG_MAPPING_NAMES 中的条目）
- 第 82 行：`("qwen3", "Qwen3")` → 注释 `# [UNLOADED: pkg_qwen3]`（MODEL_NAMES_MAPPING 中的条目）

## paddleformers/transformers/auto/modeling.py
- 第 66 行：`("Qwen3", "qwen3")` → 注释 `# [UNLOADED: pkg_qwen3]`（MAPPING_NAMES 中的条目）

## paddleformers/cli/utils/llm_utils.py
- 第 124-135 行：移除 `elif model.config.model_type == "qwen3":` LoRA target_modules 分支，替换为注释 `# [UNLOADED: pkg_qwen3]`

## paddleformers/datasets/template/template.py
- 第 668-682 行：移除 `register_template(name="qwen3", ...)` 调用块，替换为注释 `# [UNLOADED: pkg_qwen3]`
