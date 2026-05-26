# transformers 值得借鉴的编码规范

## Config 字段声明方式（HF 更简洁）

- **参考库的做法**: 使用 `@strict` 装饰器 + 类型注解 + 默认值直接声明字段，`__post_init__` 做后处理；字段均在类体中，一目了然
  ```python
  @strict
  class LlamaConfig(PreTrainedConfig):
      vocab_size: int = 32000
      hidden_size: int = 4096
      num_key_value_heads: int | None = None
      def __post_init__(self, **kwargs):
          if self.num_key_value_heads is None:
              self.num_key_value_heads = self.num_attention_heads
          super().__post_init__(**kwargs)
  ```
- **当前库的现状**: 字段在 `__init__` 参数列表中声明，通过 `self.xxx = xxx` 赋值，最后 `super().__init__(...)`
- **建议**: 仅供参考；PaddleFormers 当前方式兼容性更好，无需强制迁移，但新模型 Config 可以参考 HF 字段列表的完整性

---

## Modular 文件机制（HF 特有，可借鉴思路）

- **参考库的做法**: 新模型通过 `modular_xxx.py` 继承现有模型，只写差异部分；`make fix-repo` 自动展开为完整的 `modeling_xxx.py`，避免大量重复代码
- **当前库的现状**: PaddleFormers 无代码生成机制，每个模型文件均手写完整实现（会有较多重复）
- **建议**: 迁移时，读 HF 的 `modular_xxx.py` 了解"最小差异"，在 PaddleFormers 中尽量复用 `paddleformers.nn.*` 通用组件减少重复

---

## `__all__` 明确导出列表（HF 强制）

- **参考库的做法**: 每个 `configuration_xxx.py` 和 `modeling_xxx.py` 文件末尾都有 `__all__ = ["XxxConfig"]` / `__all__ = ["XxxModel", "XxxForCausalLM", ...]`，明确声明公开符号
- **当前库的现状**: 通过 `__init__.py` 的 `import_structure` 控制导出，文件内无 `__all__`
- **建议**: 可选采纳；迁移时在新文件末尾加 `__all__` 是好习惯，但不强制

---

## base_model_prefix 差异（重要）

- **参考库的做法**: HF 中 `LlamaForCausalLM` 有 `self.model = LlamaModel(config)`，因此 `base_model_prefix = "model"`（通用）
- **当前库的现状**: PaddleFormers 中通常用 `self.{model_name} = XxxModel(config)`，`base_model_prefix = "{model_name}"`
- **建议**: 迁移时注意此差异；若 checkpoint 来自 HF，权重 key 中的前缀可能需要做映射转换（通过 `_checkpoint_conversion_mapping` 处理）
