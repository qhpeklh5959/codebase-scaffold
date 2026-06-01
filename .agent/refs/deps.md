# 依赖代码库注册表

本文件由 `/import-dep` 维护，记录当前代码库依赖的外部代码库及其导入状态。

每个依赖的详细符号表存放在 `.agent/refs/deps/{name}/symbols.md`，规范存放在 `.agent/refs/deps/{name}/conventions.md`（若有）。

---

<!-- 新增格式：
## {name}
- **路径**: 依赖库根目录路径（相对于 PaddleFormers 项目根目录）
- **refs 来源**: 有 refs（直接读取）| 从源码提取
- **导入时间**: YYYY-MM-DD
- **当前库中的使用方**: （本库中依赖它的文件列表）
- **已导入符号数**: N 个类/接口
- **共享规范**: 是 | 否
-->

## PaddleFleet
- **路径**: ../PaddleFleet
- **refs 来源**: 有 refs（直接读取）
- **导入时间**: 2026-05-26
- **当前库中的使用方**:
  - `paddleformers/transformers/gpt_provider.py`
  - `paddleformers/transformers/model_provider.py`
  - `paddleformers/transformers/qwen3_5/modeling_fleet.py`
  - `paddleformers/transformers/qwen3_vl/modeling_fleet.py`
  - `paddleformers/trainer/trainer.py`
  - `paddleformers/trainer/trainer_callback.py`
  - `paddleformers/trainer/training_args.py`
  - `paddleformers/trainer/trainer_utils.py`
  - `paddleformers/peft/lora/lora_model.py`
  - `paddleformers/peft/lora/lora_layers.py`
  - `paddleformers/quantization/quantization_utils.py`
  - `paddleformers/transformers/fp8_utils.py`
  - `paddleformers/cli/train/deepseek_v3_pretrain/moe_layer.py`
  - `paddleformers/cli/train/deepseek_v3_pretrain/moe_utils.py`
  - `paddleformers/cli/train/dpo/dpo_trainer.py`
  - `examples/experiments/paddlefleet/run_pretrain.py`
  - `examples/experiments/deepseek_v3_pretrain/moe_layer.py`
  - `examples/experiments/deepseek_v3_pretrain/moe_utils.py`
- **已导入符号数**: 42 个类/接口/函数
- **共享规范**: 是
