# transformers 参考概览

## 技术栈
- **语言**: Python 3.8+
- **框架**: PyTorch (torch + torch.nn)
- **权重格式**: safetensors / pytorch bin
- **模型协作**: HuggingFace Hub (huggingface_hub)
- **代码组织**: modular 文件系统（新模型通过 `modular_<name>.py` + 代码生成得到 `modeling_<name>.py`）

## 与当前库的关系
HuggingFace transformers 是 PaddleFormers 的**上游参考实现**：
- 两者 API 设计几乎一一对应（PretrainedConfig/PretrainedModel、AutoConfig、ModelOutput 等）
- 绝大多数模型在 HF 先发布，PaddleFormers 需要将其迁移到 PaddlePaddle 框架
- 核心差异：torch.nn → paddle.nn，Linear/Embedding 需替换为 PaddleFormers 的并行版本
- HF 有 modular 代码生成机制，PaddleFormers 目前无此机制

## 最有参考价值的领域
- **模型架构（Config + Model）**：迁移新模型的直接参考，字段一一对应
- **modular 继承模式**：理解新模型在 HF 侧是从哪个基础模型继承的，对应要迁移哪个已有实现
- **Config 字段与默认值**：`configuration_*.py` 是最权威的字段参考
- **chat_template 位置**：在 `tokenizer_config.json` 中，可从 HF Hub 直接读取
- **模型目录结构**：每个模型的文件组织与 PaddleFormers 高度一致

## 来源
路径: /root/paddlejob/share-storage/gpfs/system-public/qinhuapeng/transformers
导入时间: 2026-05-25
聚焦领域: 具体模型实现，以及两边的接口对应关系，方便将模型迁移到本库
