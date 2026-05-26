# 编码规范

## forward 方法约定

来源：`gpt_model.py`, `transformer_layer.py`, `attention.py`, `lm_head.py`

Pipeline boundary 层（GPTEmbedding、TransformerLayer、GPTLMHead 等）的 `forward` 接受 `dict_args: dict`，通过字段名取输入，返回 dict_args 或 Tensor。原因：PP p2p 通信需要固定结构的张量 tuple，dict 在跨 stage 边界时由 `dict_to_tuple_helper` 转换。

- 内部计算层（SelfAttention、MLP、DotProductAttention）接受显式 Tensor 参数（非 dict）
- 含 recompute 的层用 `_forward_impl` 分离出可重计算部分，`forward` 只做分发

## 命名规范
- 类名：PascalCase（`TransformerLayer`, `SelfAttention`, `MLPSublayersSpec`）
- 函数/方法名：snake_case（`get_gpt_layer_local_spec`, `gpt_builder`）
- 常量：UPPER_SNAKE_CASE（`LNImpl`, `_LOG_LAYER_MD5`）
- 私有函数：前缀 `_`（`_md5`, `_apply_ec_complex_3d_mrope`）
- 模块级 logger：`logger = logging.getLogger(__name__)`
- 配置字段：snake_case dataclass 字段，附 docstring 注释

## 包/模块组织规则
- 按功能分包：`transformer/`（核心层）、`models/`（具体模型）、`tensor_parallel/`、`pipeline_parallel/`
- 每个模型在 `models/<model_name>/` 下独立目录，含 `__init__.py`、`*_model.py`、`layer_specs.py`、`*_builders.py`
- 公共组件放 `models/common/`（embeddings、loss、vision_layer）
- 融合算子放 `fusions/`，FP8 相关放 `fp8/`
- 测试镜像源码结构：`tests/single_card_tests/` 和 `tests/multi_card_tests/`，子目录对应源码模块

## 错误处理规范
- 使用 `assert` 做前置条件检查（如 `assert config is not None, "config must be specified."`）
- 使用标准 Python 异常（`ValueError`, `RuntimeError`）
- 通过 `logger.warning()` / `logger.info()` 记录运行时信息
- 不使用裸 `except`（ruff E722 已配置忽略，但仍应避免）

## 类型注解规范
- 文件头部加 `from __future__ import annotations`（延迟求值）
- 使用 `TYPE_CHECKING` 块做仅类型检查的导入，避免循环依赖
- dataclass 字段使用类型注解 + 默认值 + 行内 docstring

## 注释风格
- 类和函数使用 Google 风格 docstring（Args/Returns/Raises）
- 行内注释用 `#`，复杂逻辑附说明
- 版权头：Apache 2.0 License 标准头，含 PaddlePaddle 和 NVIDIA 双重版权（部分文件）

## 测试规范
- 测试类命名：`Test<功能名>` 继承 `unittest.TestCase`
- 测试方法命名：`test_<场景描述>`（snake_case）
- Mock 框架：`unittest.mock`（`MagicMock`, `patch`）
- 断言：`self.assertEqual`, `self.assertIsNone`, `self.assertIsNotNone` 等
- 多卡测试通过 `paddle.distributed` 初始化，单卡测试直接 import 运行
- AI 生成测试放 `ai_edited_test/` 子目录，人工测试直接放对应模块目录

## 导入规范
- 标准库 → 第三方（paddle）→ 本地（paddlefleet）顺序
- isort：`combine-as-imports = true`，`known-first-party = ["paddlefleet"]`
- `__init__.py` 中允许 re-export（忽略 PLC0414）
