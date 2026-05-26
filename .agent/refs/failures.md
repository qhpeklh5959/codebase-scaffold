# 失败记录
<!-- 新增格式：
## [错误类型，如：ImportNotFound / TypeMismatch / CoverageInsufficient]
- 现象：
- 根因：
- 修正方式：
- 相关 refs 修正：（修改了哪个 refs 文件的哪条规则）
-->

## [AlignNonDeterministic] 对齐时 paddle 结果每次运行不同

- 现象：多次运行 `from_pretrained` + forward，logits 数值不一致，无法与 torch 稳定对齐
- 根因：未清除分布式环境变量，或未设置确定性计算 flags；分布式残留变量（`PADDLE_ELASTIC_JOB_ID` 等）会触发意外的多卡逻辑
- 修正方式：运行任何对齐脚本前执行以下环境变量设置：
  ```bash
  unset PADDLE_ELASTIC_JOB_ID PADDLE_TRAINER_ENDPOINTS DISTRIBUTED_TRAINER_ENDPOINTS FLAGS_START_PORT PADDLE_ELASTIC_TIMEOUT
  export NNODES=1 PADDLE_TRAINERS_NUM=1 FLAGS_selected_gpus=0
  export FLAGS_use_accuracy_compatible_kernel=1 FLAGS_cudnn_deterministic=1
  ```
  也可在 Python 脚本中用 `os.environ` 设置（`FLAGS_cudnn_deterministic` 等）
- 相关 refs 修正：patterns.md 新增"跨框架数值对齐"章节

## [AlignTrainPathMismatch] 训练 loss 对齐，但 paddleformers-cli 端到端训练趋势不一致

- 现象：裸 Python 循环的 loss 与 torch 对齐，但 `paddleformers-cli train` 跑出来的 loss 曲线偏差大
- 根因：裸循环绕过了 `Trainer` 内部的梯度裁剪、优化器参数分组、`CriterionLayer` 的权重等逻辑
- 修正方式：训练趋势对齐时必须通过 `paddleformers-cli train <yaml>` 端到端执行，不能用裸 Python 循环替代
- 相关 refs 修正：align.md Step 5 训练趋势检查已改为调用 paddleformers-cli

## [DepSymbolNotFound] import-dep 时部分符号在依赖库源码中找不到定义

- 现象：PaddleFormers 中 import 了 `paddlefleet.models.gpt.gpt_layer_specs.get_gpt_mtp_block_spec`、`paddlefleet.fp8_utils.correct_amax_history_if_needed`，但在 PaddleFleet 源码中 grep 不到对应定义
- 根因：两种可能——①符号已被重命名/迁移（如 `fp8_utils` 已移至 `transformer/moe/fp8_utils.py`）；②符号通过 `__init__.py` re-export 但定义在子模块中，grep 类名/函数名时未命中（如通过 `from x import *` 导出）
- 修正方式：遇到 grep 不到的符号时，额外检查对应包的 `__init__.py` 是否有 re-export；同时检查是否有同名但路径不同的定义
- 相关 refs 修正：deps/PaddleFleet/symbols.md 注意事项中已标注这两个符号待确认
