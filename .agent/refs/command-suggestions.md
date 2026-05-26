# Command 建议清单

本文件由 `/gen-command --suggest`、`/bootstrap`、`/reflect` 自动写入，记录针对当前代码库建议新增的定制 commands。

运行 `/gen-command --create <name>` 将某条建议生成为实际的 command 文件。
运行 `/gen-command --list` 查看所有待处理建议。

---

<!-- 新增格式：
## {name}（{来源：bootstrap 分析 / reflect 分析 / 用户需求}）
- **触发场景**: 什么时候用
- **解决的问题**: 没有它时需要手动做什么
- **建议的步骤摘要**: 1. ... 2. ... 3. ...
- **依赖的 refs**: CODEBASE.md / patterns.md / ...
- **状态**: 待处理 | 已生成（YYYY-MM-DD）| 已忽略
- **建议时间**: YYYY-MM-DD
-->

## add-model（bootstrap 分析）
- **触发场景**: 需要为 PaddleFormers 添加一个新模型（如接入新的开源 LLM）
- **解决的问题**: 手动创建 configuration.py、modeling.py、__init__.py、注册到 auto 模块、补充顶层 __init__ 导出，步骤繁琐且易遗漏
- **建议的步骤摘要**:
  1. 读取 patterns.md#添加新模型，收集目标模型架构信息
  2. 在 paddleformers/transformers/{model_name}/ 创建 configuration.py（继承 PretrainedConfig）
  3. 创建 modeling.py（继承 PretrainedModel，使用 nn/ 积木层）
  4. 创建标准 __init__.py（懒加载模式）
  5. 注册到 auto/ 的 CONFIG_MAPPING / MODEL_FOR_CAUSAL_LM_MAPPING
  6. 更新顶层 paddleformers/transformers/__init__.py
  7. 生成对应测试骨架 tests/transformers/{model_name}/test_modeling.py
- **依赖的 refs**: patterns.md#添加新模型, symbols.md, conventions.md#懒加载模式
- **状态**: 已生成（2026-05-25），已更新（2026-05-25）：移除 --template 参数，改为从参考库 chat_template 自动判断；新增 Step 3 读取 refs-registry.md 并使用 HF 参考库架构
- **建议时间**: 2026-05-25

## add-callback（bootstrap 分析）
- **触发场景**: 需要在训练过程中插入自定义监控、日志或控制逻辑
- **解决的问题**: 用户不熟悉 TrainerCallback 的事件接口和注册方式，需要查文档
- **建议的步骤摘要**:
  1. 根据用户描述确定需要的事件钩子（on_train_begin / on_step_end / on_evaluate 等）
  2. 生成继承 TrainerCallback 的类
  3. 生成使用示例（传入 Trainer(callbacks=[...])）
- **依赖的 refs**: symbols.md#TrainerCallback, patterns.md#添加新的TrainerCallback
- **状态**: 待处理
- **建议时间**: 2026-05-25

## train-model（用户需求）
- **触发场景**: 需要使用 paddleformers-cli 对模型进行端到端训练（SFT/PT/DPO/LoRA）
- **解决的问题**: 手动编写训练 YAML 配置、处理路径注入、确定 chat template、组织分布式启动命令
- **建议的步骤摘要**:
  1. 解析参数（model、stage、lora、tp、pp、output、template）
  2. 自动检测 chat template（tokenizer_config.json → TEMPLATES dict）
  3. 基于 examples/config/sft/full.yaml 或 lora.yaml 生成 train_config.yaml
  4. 按 CI 脚本（tests/integration_test/）的 yq 模式注入绝对路径
  5. 执行 paddleformers-cli train train_config.yaml（多卡时用 paddle.distributed.launch）
- **依赖的 refs**: CODEBASE.md, patterns.md, examples/config/sft/
- **状态**: 已生成（2026-05-25）
- **建议时间**: 2026-05-25

## align（用户需求）
- **触发场景**: 迁移新模型后，需要验证 PaddleFormers 与 HF transformers 加载同样权重时的数值一致性
- **解决的问题**: 手动编写 CompatibilityTest 代码、处理 dtype 转换、挑选合适的 atol/rtol、对比 logits/loss/训练趋势
- **建议的步骤摘要**:
  1. 确定模型类名对照（HF ↔ PaddleFormers）
  2. 准备权重（指定路径或自动创建 tiny 随机权重）
  3. 生成对齐脚本（logits/loss/train 三种检查模式）
  4. 按 dtype（float32/bfloat16/float16）设置合适的 atol/rtol 容忍度
  5. 执行脚本，失败时按常见场景表（key 映射/reduce 方式/bf16 精度）排查
  6. 通过后固化为 CompatibilityTest 类写入 test_modeling.py
- **依赖的 refs**: CODEBASE.md, refs/transformers/patterns.md, tests/transformers/qwen3/test_modeling.py:411
- **状态**: 已生成（2026-05-25）
- **建议时间**: 2026-05-25

## run-model-test（bootstrap 分析）
- **触发场景**: 添加或修改模型后，快速运行对应模型的单元测试
- **解决的问题**: 记住完整的 pytest 命令和必要环境变量（DOWNLOAD_SOURCE、PYTHONPATH）
- **建议的步骤摘要**:
  1. 定位 tests/transformers/{model_name}/ 目录
  2. 以 `DOWNLOAD_SOURCE=aistudio PYTHONPATH=. pytest tests/transformers/{model_name}/ -v` 运行
  3. 输出失败摘要，写入 failures.md
- **依赖的 refs**: CODEBASE.md#测试框架和运行命令
- **状态**: 待处理
- **建议时间**: 2026-05-25

## refresh-dep（reflect 分析）
- **触发场景**: 依赖库（如 PaddleFleet）升级后，需要验证 deps/ 中记录的符号签名是否仍然有效
- **解决的问题**: 手动 grep 每个已导入符号、比对路径和签名，步骤重复且容易遗漏
- **建议的步骤摘要**:
  1. 读取 `.agent/refs/deps.md`，列出所有已注册依赖
  2. 对每个依赖，遍历 `deps/{name}/symbols.md` 中的符号，在依赖库源码中验证路径和签名
  3. 标记失效符号（路径不存在 / 签名变更），输出差异报告
  4. 自动更新 symbols.md，将失效条目标注 `[待确认]`
  5. 若变更较大，提示用户运行 `/import-dep {name} --refresh`
- **依赖的 refs**: deps.md, deps/{name}/symbols.md
- **状态**: 已生成（2026-05-26）
- **建议时间**: 2026-05-26
