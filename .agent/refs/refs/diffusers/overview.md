# diffusers 参考概览

## 技术栈
- 语言: Python 3.8+
- 框架: PyTorch（torch.nn）
- 依赖: transformers（文本编码器）、huggingface_hub、safetensors、accelerate、peft

## 与当前库的关系
diffusers 与 PaddleFormers 技术栈相近（都是 HuggingFace 风格的 PretrainedModel/ConfigMixin 体系），
但专注于扩散模型（图像/视频生成），而 PaddleFormers 专注于语言模型训练。
迁移时需将 torch.nn → paddle.nn，torch.Tensor → paddle.Tensor。

## 最有参考价值的领域
- **Scheduler（噪声调度器）**: 扩散过程的核心，beta schedule、加噪/去噪步骤的标准实现
- **UNet 架构**: 条件 UNet 的 down/mid/up block 组合模式，timestep embedding 注入方式
- **VAE（AutoencoderKL）**: 潜在扩散模型的图像编解码器，latent scaling 策略
- **Pipeline 编排**: 多组件（VAE+UNet+Scheduler+TextEncoder）的推理流程组织
- **ConfigMixin + register_to_config**: 调度器/模型配置的自动序列化机制
- **Attention Processor**: 可插拔的 attention 实现（AttnProcessor），支持运行时替换
- **ModelHook 推理加速缓存**: FasterCache / PyramidAttentionBroadcast / GroupOffloading，通过 hook 机制跳过冗余计算
- **BaseGuidance 引导策略体系**: CFG 及多种高级引导算法的统一抽象层
- **SNR 加权训练损失**: Min-SNR 损失加权，改善扩散模型训练稳定性

## 来源
路径: ../diffusers（相对于 PaddleFormers 项目根目录）
导入时间: 2026-05-28
聚焦领域: 全量（--refresh 更新，新增 hooks/guiders/training_utils 领域）
