## trace-moe-path（trace 任务观察）
- **触发场景**: 需要根据运行时配置快速定位 MoELayer 走哪条执行路径（fusion_moe_forward / custom_forward / 单卡路径）
- **解决的问题**: MoELayer.forward 有 6+ 条分支路径，每次手动查分支条件耗时；需要一个工具输入 config 标志就能告知执行路径和关键调用
- **建议的步骤摘要**: 1. 读取用户提供的 config 字段（EP size, moe_use_fusion_node, moe_expert_fusion, using_sonic_moe）2. 按 patterns.md#MoE 路由配置决策树 推导执行路径 3. 输出该路径的调用树摘要 + 通信节点标注
- **依赖的 refs**: patterns.md#MoE 执行三段式, patterns.md#MoE 路由配置决策树, symbols.md#MoELayer
- **状态**: 待处理
- **建议时间**: 2026-05-26

## trace-pp-stage（trace 任务观察）
- **触发场景**: 需要追踪 GPTModel 在特定 PP stage 的执行路径（哪些层被分配到该 stage）
- **解决的问题**: GPTModel.forward 委托给 PipelineLayer，无法直接静态确定某 stage 执行哪些层；需要结合 stage_id + get_sequential_layers 分析
- **建议的步骤摘要**: 1. 读取 `GPTSublayersSpec` 中各层列表 2. 模拟 `get_layer_desc_list` 的顺序编号 3. 按 PP size 和 stage_id 计算 layer 归属 4. 输出该 stage 的层列表及其 forward 路径
- **依赖的 refs**: patterns.md#GPTModel forward 执行路径, symbols.md#GPTModel
- **状态**: 待处理
- **建议时间**: 2026-05-26

## add-attention（bootstrap 分析）
- **触发场景**: 需要为 PaddleFleet 添加新的注意力机制变体（如新型线性注意力、稀疏注意力）
- **解决的问题**: 添加注意力机制涉及 4+ 个文件（新 attention 文件、gpt_layer_specs、__init__ 导出、测试），步骤繁琐易遗漏
- **建议的步骤摘要**: 1. 创建 `transformer/<name>_attention.py`（SublayersSpec + FleetLayer 子类） 2. 在 `gpt_layer_specs.py` 的 `get_attention_spec()` 注册新 type 3. 更新 `transformer/__init__.py` 导出 4. 生成对应单元测试
- **依赖的 refs**: patterns.md#添加新注意力机制, symbols.md#SublayersSpec
- **状态**: 待处理
- **建议时间**: 2026-05-25

## add-model（bootstrap 分析）
- **触发场景**: 需要接入新的 LLM 架构（如新版 Qwen、LLaMA 变体等）
- **解决的问题**: 新模型需要创建目录结构、layer_specs、model 类、builders，并在 models/__init__.py 注册，步骤超过 5 步
- **建议的步骤摘要**: 1. 创建 `models/<name>/` 目录结构 2. 实现 `layer_specs.py`（复用 get_gpt_layer_local_spec） 3. 实现 `<name>_model.py` 4. 实现 `<name>_builders.py` 5. 注册到 `models/__init__.py`
- **依赖的 refs**: patterns.md#添加新模型, symbols.md#模型构建函数
- **状态**: 待处理
- **建议时间**: 2026-05-25

## run-tests（bootstrap 分析）
- **触发场景**: 需要快速运行指定模块的单卡或多卡测试，并查看覆盖率
- **解决的问题**: 单卡/多卡测试命令不同，覆盖率收集需要额外配置，频繁手动输入
- **建议的步骤摘要**: 1. 识别目标模块对应的测试目录 2. 选择单卡/多卡模式 3. 运行 pytest 并收集 coverage 4. 输出未覆盖分支报告
- **依赖的 refs**: CODEBASE.md#测试框架和运行命令
- **状态**: 待处理
- **建议时间**: 2026-05-25
