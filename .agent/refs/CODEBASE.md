# PaddleFormers 代码库概览

## 技术栈
- **语言**: Python 3.8+
- **核心框架**: PaddlePaddle (paddle)
- **辅助依赖**: numpy, tqdm, huggingface_hub, safetensors, transformers-like API
- **代码风格**: black (line-length=119) + isort
- **测试框架**: pytest（含 pytest-cov、pytest-retry）
- **包管理**: pip + requirements.txt；构建用 setup.py

## 目录结构及职责
```
paddleformers/
├── transformers/       # 模型定义核心：Config、Model、Tokenizer（每个模型一个子目录）
│   ├── auto/           # AutoConfig、AutoModel 等工厂/注册机制
│   ├── llama/          # 示例模型目录（configuration.py / modeling.py / tokenizer.py / __init__.py）
│   ├── ernie4_5/       # ERNIE 4.5 密集模型
│   ├── ernie4_5_moe/   # ERNIE 4.5 MoE 模型
│   ├── ernie4_5_moe_vl/# ERNIE 4.5 多模态（含 VL）
│   ├── qwen*/          # Qwen 系列（qwen2, qwen2_vl, qwen3, qwen3_moe, qwen3_vl 等）
│   ├── deepseek_v3/    # DeepSeek-V3
│   ├── configuration_utils.py  # PretrainedConfig 基类
│   ├── model_utils.py          # PretrainedModel 基类
│   ├── tokenizer_utils.py      # PretrainedTokenizer 基类
│   └── model_outputs.py        # ModelOutput 数据类（BaseModelOutputWithPast 等）
├── nn/                 # 通用神经网络组件（可复用积木）
│   ├── general.py      # GeneralInterface（dict-like 函数注册基类）
│   ├── attention/      # AttentionInterface + eager/sdpa/flashmask 实现
│   ├── criterion/      # LossInterface + sft/dpo/kto 损失实现 + CriterionLayer
│   ├── moe/            # MoE 相关组件（gate、all-to-all、layer 等）
│   ├── embedding.py / linear.py / lm_head.py / mlp.py / norm.py / pp_model.py
├── trainer/            # 训练循环
│   ├── trainer.py      # Trainer 主类（大量并行策略集成）
│   ├── trainer_callback.py  # TrainerCallback 事件钩子基类
│   ├── training_args.py     # TrainingArguments 数据类
│   ├── trainer_utils.py     # 工具函数
│   └── unified_checkpoint/ # 统一 checkpoint 读写
├── datasets/           # 数据集加载与处理
│   ├── template/       # Template、Formatter、MMPlugin（多模态插件）、ToolUtils
│   ├── reader/         # FileReader、HuggingFaceReader、MixDataset
│   ├── SFTDataset.py / DPODataset.py
├── peft/lora/          # LoRA 相关实现
├── quantization/       # 量化工具
├── generation/         # GenerationMixin、GenerationConfig
├── mergekit/           # 模型合并工具
├── cli/                # 命令行入口（sft、dpo、pretrain、export 等子命令）
│   ├── train/sft/      # SFT 训练入口
│   ├── train/dpo/      # DPO 训练入口
│   ├── train/ernie_pretrain/  # ERNIE 预训练
│   ├── train/deepseek_v3_pretrain/
│   └── export/         # 模型导出
└── utils/              # 日志、下载、环境变量、序列化等通用工具
tests/                  # 单元测试（与 paddleformers/ 子目录对应）
examples/               # 配置示例（YAML 训练配置文件）
```

## 关键入口文件
- `paddleformers/transformers/configuration_utils.py:613` — PretrainedConfig 基类，所有 Config 的父类
- `paddleformers/transformers/model_utils.py:1184` — PretrainedModel 基类，所有模型的父类
- `paddleformers/trainer/trainer.py` — 主训练器 Trainer，支持 TP/PP/SP/FSDP/Sharding 等并行策略
- `paddleformers/trainer/trainer_callback.py:219` — TrainerCallback 事件钩子基类
- `paddleformers/nn/general.py:19` — GeneralInterface，注册/扩展 attention/loss 函数的基础
- `paddleformers/transformers/auto/` — AutoConfig、AutoModel 等工厂注册

## 扩展点索引（→ 详见 symbols.md）
- `PretrainedConfig` — 新模型 Config 的必须继承基类
- `PretrainedModel` — 新模型 Model 的必须继承基类
- `TrainerCallback` — 自定义训练钩子的扩展点
- `GeneralInterface` — 注册新 attention 函数或 loss 函数
- `AttentionInterface` — attention 实现注册表（eager/sdpa/flashmask）
- `LossInterface` — loss 函数注册表（sft/dpo/kto/mtp_sft）
- `BasePlugin` (datasets/template/mm_plugin.py) — 多模态插件扩展点
- `Template` (datasets/template/template.py) — 对话模板扩展点

## 双版本模型模式（Fleet vs Deprecated）
仅当模型在 fleet 中已有完整实现时才出现双版本：
- `XxxForCausalLM`：fleet 版（正式），`__new__` 返回 GPT provider，用于生产训练
- `XxxForCausalLMDeprecated`：formers 原生版（已废弃），保留用于对齐验证和推理测试
正常情况（fleet 无实现）只有单版本 `XxxForCausalLM`，直接继承 `PretrainedModel`。
详见 `patterns-model-extension.md#Fleet 版 vs Deprecated 版`

## 测试框架和运行命令
```bash
# 运行全部测试
DOWNLOAD_SOURCE=aistudio PYTHONPATH=. pytest -v --retries 1 --retry-delay 1 --durations 20 --cov=./paddleformers --cov-report=xml:coverage.xml

# 等价 make 命令
make test

# 运行特定模块测试
pytest tests/transformers/ernie4_5/
pytest tests/nn/
```
