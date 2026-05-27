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

## [AddModelUsedShadowBackup] add-model 误用 shadow 备份而非从参考库重新实现

- 现象：执行 `/add-model` 时，发现 `.agent/shadows/pkg_{model_name}/` 下存在同名备份，直接 `cp` 备份文件到目标路径，而非从 HF 参考库实现
- 根因：未明确区分 `/add-model`（从参考库重新实现）和 `/rollback`（恢复备份）的职责；备份目录的存在被误判为"可复用资源"
- 修正方式：`/add-model` 执行时，即使存在同名备份也**完全忽略备份 .py 文件**，从参考库（HF transformers 等）重新实现。可读取 `description.md` 了解功能描述，但禁止读取或复制任何 `.py` 备份
- 相关 refs 修正：`.claude/commands/add-model.md` Step 2.5 新增 shadow 备份忽略强制规则

## [AddModelGeneratedTests] add-model 生成了测试代码，应由 /test-gen 负责

- 现象：`/add-model` 在 `tests/transformers/{model_name}/` 下生成了 `test_modeling.py`，但测试生成是 `/test-gen` 的职责
- 根因：add-model 命令包含 Step 13（生成测试骨架），职责边界不清晰
- 修正方式：已从 add-model.md 中移除 Step 13；需要测试时运行 `/test-gen {model_name}`
- 相关 refs 修正：`.claude/commands/add-model.md` 删除 Step 13，完成输出中下一步改为引导 `/test-gen`

## [AlignBFloat16ToleranceTooTight] bfloat16 对齐时 atol=1e-2 失败但实现正确

- 现象：float32 logits 完全对齐（max_diff ≈ 2e-5），相同模型 bfloat16 下 atol=1e-2 失败（max_diff ≈ 0.094，mean_diff ≈ 0.039），差值呈 2^{-4}=0.0625 步长
- 根因：bfloat16 尾数 7 bit，精度约 2^{-7}≈0.0078；N 层累积后误差以 bfloat16 最小步长倍数增长，对 Qwen3-0.6B（28 层）实测约 0.09，非实现问题；3 次推理 max_delta=0 确认非算子 bug
- 修正方式：bfloat16 对齐使用 atol=1e-1；若怀疑是 IMPL_BUG，先用 float32 复现——float32 差距大才是实现问题，float32 通过则 bfloat16 差距是精度预期
- 相关 refs 修正：patterns.md#dtype 容忍度对应关系 已包含此说明

## [AlignTrainAOADtypeMismatch] paddleformers-cli train 加载 bfloat16 checkpoint 时 AOA dtype 断言失败

- 现象：`AssertionError: Direct assignment of Tensors with different types is prohibited in AOA. src_var.dtype is bfloat16, the target.dtype is float32`，发生在 `from_pretrained` 的 `dist.load_state_dict` 阶段
- 根因：SFT workflow（`paddleformers/cli/train/sft/workflow.py:247-255`）只有在 `fp16_opt_level == "O2"` 且 `bf16=True` 时才将 `dtype` 设为 `"bfloat16"`；否则默认 `float32`。AOA 的跨 key 映射语句（如 `model.embed_tokens.weight -> model.embedding.embed_tokens.weight`）不含 `dtype=` 修饰符，因此 AOA 引擎拒绝 bfloat16→float32 的直接赋值
- 修正方式：训练配置 YAML 中同时设置 `bf16: true` 和 `fp16_opt_level: O2`，使 workflow 将模型以 bfloat16 实例化，与 checkpoint dtype 一致
- 相关 refs 修正：patterns.md#align-train 新增此说明

## [AlignTrainMultiGPU] paddleformers-cli train 默认启动多卡，单卡对齐需显式限制

- 现象：`paddleformers-cli train` 通过 `paddle.distributed.launch` 启动，自动检测所有可见 GPU（`CUDA_VISIBLE_DEVICES` 或 GPUtil），在多卡机器上会启动 8 个进程
- 根因：`paddleformers/cli/cli.py:91-98` 用 `GPUtil.getGPUs()` 获取所有 GPU 作为 `visible_cards`，传给 `--gpus` 参数
- 修正方式：在调用 `paddleformers-cli` 的 `subprocess.run` 的 `env` 中设置 `CUDA_VISIBLE_DEVICES=<单卡编号>`（如 `"4"`），cli 会读取此变量只启动 1 个进程
- 相关 refs 修正：patterns.md#align-train 新增单卡配置说明

## [AlignTrainDatasetProbMissing] erniekit 数据集类型缺少 task_group_prob 字段报错

- 现象：`ValueError: could not convert string to float: 'None'`，发生在 `MultiSourceDataset.__init__` 的 `task_group_prob` 解析
- 根因：`train_dataset_type: erniekit` 要求同时提供 `train_dataset_prob`（字符串，如 `"1.0"`）；YAML 中未设置时值为 `None`，无法转 float
- 修正方式：训练配置中同时设置 `train_dataset_prob: "1.0"` 和 `eval_dataset_prob: "1.0"`
- 相关 refs 修正：无

## [AlignTrainTorchCPU] torch 模型未移到 GPU，backward 在 CPU 上运行极慢

- 现象：HF 模型加载完成后训练循环无任何输出，挂起超过 10 分钟
- 根因：`from_pretrained` 默认将模型加载到 CPU；未调用 `.cuda()` 时，第一步 backward 在 CPU 上运行 0.6B 参数的反向传播，极慢
- 修正方式：`torch_model = HFQwen3ForCausalLM.from_pretrained(...).cuda()`；input tensor 也需 `.cuda()`
- 相关 refs 修正：无

## [DepSymbolNotFound] import-dep 时部分符号在依赖库源码中找不到定义

- 现象：PaddleFormers 中 import 了 `paddlefleet.models.gpt.gpt_layer_specs.get_gpt_mtp_block_spec`、`paddlefleet.fp8_utils.correct_amax_history_if_needed`，但在 PaddleFleet 源码中 grep 不到对应定义
- 根因：两种可能——①符号已被重命名/迁移（如 `fp8_utils` 已移至 `transformer/moe/fp8_utils.py`）；②符号通过 `__init__.py` re-export 但定义在子模块中，grep 类名/函数名时未命中（如通过 `from x import *` 导出）
- 修正方式：遇到 grep 不到的符号时，额外检查对应包的 `__init__.py` 是否有 re-export；同时检查是否有同名但路径不同的定义
- 相关 refs 修正：deps/PaddleFleet/symbols.md 注意事项中已标注这两个符号待确认
