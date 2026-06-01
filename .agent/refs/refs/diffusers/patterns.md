# diffusers 可借鉴的实现模式

## ConfigMixin + register_to_config（调度器/模型配置自动序列化）
- **解决的问题**: 调度器和模型的超参数需要随 checkpoint 保存/加载，避免手写序列化逻辑
- **核心思路**: `@register_to_config` 装饰 `__init__`，自动将所有参数存入 `self.config`（FrozenDict）；
  `ConfigMixin` 提供 `save_config`/`from_config` 方法，config 以 `config.json` 持久化
- **参考来源**: src/diffusers/configuration_utils.py:56（FrozenDict）、:register_to_config
- **迁移建议**: PaddleFormers 的 `PretrainedConfig` 已有类似机制；调度器若要引入，
  可参考此模式为 `SchedulerMixin` 基类添加 `register_to_config` 支持

## Scheduler 接口（SchedulerMixin）
- **解决的问题**: 统一不同噪声调度算法（DDPM/DDIM/DPM-Solver 等）的调用接口
- **核心思路**: 所有调度器实现 `set_timesteps(num_inference_steps)` + `step(model_output, timestep, sample)` 两个核心方法；
  `step` 返回 `SchedulerOutput(prev_sample, pred_original_sample)`；
  `_compatibles` 列表声明可互换的调度器
- **参考来源**: src/diffusers/schedulers/scheduling_ddpm.py:137（DDPMScheduler）
  src/diffusers/schedulers/scheduling_utils.py（SchedulerMixin）
- **迁移建议**: 在 PaddleFormers 中实现调度器时，遵循此接口；
  `step` 输出用 `@dataclass` + `BaseOutput` 包装，与 `ModelOutput` 风格一致

## Beta Schedule（噪声方差调度）
- **解决的问题**: 扩散过程中每步加噪量的设计，影响生成质量
- **核心思路**: 支持 `linear`/`scaled_linear`/`squaredcos_cap_v2`/`sigmoid` 四种 schedule；
  `betas_for_alpha_bar` 函数通过 alpha_bar 函数（cosine/exp/laplace）生成 beta 序列；
  预计算 `alphas_cumprod`、`sqrt_alphas_cumprod`、`sqrt_one_minus_alphas_cumprod` 等常量
- **参考来源**: src/diffusers/schedulers/scheduling_ddpm.py:48（betas_for_alpha_bar）、:212（__init__ beta 计算）
- **迁移建议**: paddle 中 `torch.linspace` → `paddle.linspace`，`torch.cumprod` → `paddle.cumprod`

## Timestep Embedding（时间步嵌入）
- **解决的问题**: 将离散时间步 t 编码为连续向量，注入 UNet 各层
- **核心思路**: `get_timestep_embedding` 生成正弦位置编码（类似 Transformer PE）；
  `Timesteps` 模块封装嵌入生成；`TimestepEmbedding` 用两层 MLP 将嵌入投影到模型维度；
  UNet 在每个 ResBlock 中通过加法将 time emb 注入特征图
- **参考来源**: src/diffusers/models/embeddings.py:26（get_timestep_embedding）
- **迁移建议**: paddle 中实现完全等价，注意 `flip_sin_to_cos` 参数控制 sin/cos 顺序

## UNet 条件架构（UNet2DConditionModel）
- **解决的问题**: 扩散模型的去噪网络，需同时接受噪声图像、时间步、文本条件
- **核心思路**: 通过 `down_block_types`/`mid_block_type`/`up_block_types` 字符串列表动态组装 block；
  `get_down_block`/`get_up_block` 工厂函数按名称实例化 block（CrossAttnDownBlock2D 等）；
  skip connection 通过 `down_block_res_samples` 列表在 down→up 间传递；
  cross-attention 将文本 encoder_hidden_states 注入每个 transformer block
- **参考来源**: src/diffusers/models/unets/unet_2d_condition.py:75（UNet2DConditionModel）
  src/diffusers/models/unets/unet_2d_blocks.py（block 工厂）
- **迁移建议**: PaddleFormers 的 `nn/attention/` 已有 attention 实现；
  block 工厂模式可复用 `GeneralInterface` 注册机制

## Attention Processor（可插拔 Attention）
- **解决的问题**: 支持在不修改模型代码的情况下替换 attention 实现（xformers/flash/默认）
- **核心思路**: `Attention` 模块持有 `processor` 属性（默认 `AttnProcessor`）；
  `set_attn_processor(processor)` 递归替换所有 attention 层的 processor；
  processor 实现 `__call__(attn, hidden_states, ...)` 接口
- **参考来源**: src/diffusers/models/attention_processor.py:52（Attention 类）
- **迁移建议**: PaddleFormers 已有 `AttentionInterface`（GeneralInterface 子类），
  思路相似但用注册表而非实例属性；diffusers 的 processor 模式更灵活，支持 per-layer 替换

## VAE（AutoencoderKL）与 Latent Scaling
- **解决的问题**: 在潜在空间而非像素空间做扩散，降低计算量
- **核心思路**: Encoder 将图像压缩为均值+方差，采样得到 latent z；
  `scaling_factor`（默认 0.18215）将 latent 归一化到单位方差；
  推理时：`latents = vae.encode(image).latent_dist.sample() * scaling_factor`；
  解码时：`image = vae.decode(latents / scaling_factor).sample`；
  `DiagonalGaussianDistribution` 封装重参数化采样
- **参考来源**: src/diffusers/models/autoencoders/autoencoder_kl.py:36（AutoencoderKL）
  src/diffusers/models/autoencoders/vae.py（DiagonalGaussianDistribution）
- **迁移建议**: VAE 的 Encoder/Decoder 是标准 CNN 结构，paddle 迁移直接；
  `scaling_factor` 是模型特定常量，需从预训练 config 读取

## Pipeline 编排模式（DiffusionPipeline）
- **解决的问题**: 将 VAE、UNet、Scheduler、TextEncoder 等多个组件组合为端到端推理流程
- **核心思路**: `DiffusionPipeline.__init__` 接收所有子模块，通过 `self.register_modules()` 注册；
  `model_cpu_offload_seq` 声明 CPU offload 顺序（"text_encoder->unet->vae"）；
  推理循环：encode text → prepare latents → denoise loop（scheduler.step）→ decode；
  CFG（Classifier-Free Guidance）通过拼接 unconditional+conditional 批次实现，
  `noise_pred = uncond + guidance_scale * (cond - uncond)`
- **参考来源**: src/diffusers/pipelines/stable_diffusion/pipeline_stable_diffusion.py:154
- **迁移建议**: PaddleFormers 目前无 Pipeline 层；若要实现，
  可在 `paddleformers/diffusion/` 下新建，参考此模式组织推理流程

## Classifier-Free Guidance（CFG）
- **解决的问题**: 在无需训练分类器的情况下，用文本条件引导生成方向
- **核心思路**: 训练时随机将条件置空（drop_rate ~10%）；
  推理时同时预测有条件和无条件噪声，线性插值：
  `noise_pred = noise_uncond + guidance_scale * (noise_cond - noise_uncond)`；
  `guidance_rescale` 可进一步修正过曝问题（rescale_noise_cfg）
- **参考来源**: src/diffusers/pipelines/stable_diffusion/pipeline_stable_diffusion.py:69（rescale_noise_cfg）
- **迁移建议**: 训练侧需在 dataset/template 层支持条件 drop；推理侧在 pipeline 的 denoise loop 中实现

## ModelHook 推理加速框架（FasterCache / PyramidAttentionBroadcast）
- **解决的问题**: 扩散模型推理时每步计算量大，相邻时间步的 attention 输出高度相似，可复用
- **核心思路**:
  - `ModelHook` 基类提供 `pre_forward` / `post_forward` 钩子，通过 `HookRegistry` 附加到模型层
  - `FasterCache`：每 N 步才真正计算 attention，中间步直接复用缓存；支持按 timestep 范围控制 skip 区间
  - `PyramidAttentionBroadcast`：对 spatial/temporal/cross-attention 分别配置不同的 skip_range；
    pyramid 结构指浅层频繁计算、深层可更多跳过
  - `StateManager` 管理每个 hook 的状态，支持多上下文（unconditional/conditional 分开缓存）
- **参考来源**: src/diffusers/hooks/hooks.py:59（ModelHook/HookRegistry）
  src/diffusers/hooks/faster_cache.py:50（FasterCacheConfig）
  src/diffusers/hooks/pyramid_attention_broadcast.py:40（PyramidAttentionBroadcastConfig）
- **迁移建议**: PaddleFormers 已有 `TrainerCallback` 钩子体系（训练侧）；
  推理侧若实现扩散模型，可参考此模式在 `paddleformers/generation/` 或 `paddleformers/diffusion/` 下
  引入类似的 `ModelHook` 机制，对 attention 层附加缓存钩子

## BaseGuidance 引导策略体系
- **解决的问题**: CFG 之外存在多种引导算法（APG、SEG、SKL、频率解耦引导等），需统一接口管理
- **核心思路**:
  - `BaseGuidance(ConfigMixin)` 定义通用接口：`prepare(num_steps)` → `__call__(pred_dict)` → `post_process()`
  - 子类通过 `_input_predictions` 列表声明所需预测名（如 `["pred_cond", "pred_uncond"]`）
  - `start`/`stop` 参数控制在推理过程哪个比例区间内启用引导（如只在中段启用）
  - 通过 `@register_to_config` 自动序列化，引导策略可独立保存/加载（`guider_config.json`）
- **参考来源**: src/diffusers/guiders/guider_utils.py:38（BaseGuidance）
  src/diffusers/guiders/classifier_free_guidance.py:30（ClassifierFreeGuidance）
  src/diffusers/guiders/adaptive_projected_guidance.py（APG）
- **迁移建议**: PaddleFormers 的推理目前通过 `GenerationMixin` + beam search 等控制；
  若扩展至扩散模型推理，可参考此层次在 `generation/` 下增加 `BaseGuidance` 抽象，
  解耦噪声预测与引导策略

## SNR 加权训练损失（Min-SNR）
- **解决的问题**: 扩散训练中不同时间步的损失权重不均衡，高噪声步（小 SNR）梯度信号弱
- **核心思路**: `compute_snr(noise_scheduler, timesteps)` 计算每个 timestep 的信噪比（SNR = α²/σ²）；
  训练损失乘以 `min(SNR, gamma) / SNR`（gamma 通常为 5）；
  使高 SNR 步权重降低，低 SNR 步权重提升，使整体训练更均衡
- **参考来源**: src/diffusers/training_utils.py:81（compute_snr）
- **迁移建议**: PaddleFormers 的 `LossInterface` 支持自定义 loss；
  可在 sft_loss 或新建 diffusion_loss 中加入 SNR 加权，用 `paddle.tensor.indexing` 替代 torch 索引

## GroupOffloading（显存-内存分组卸载）
- **解决的问题**: 大模型推理时显存不足，但逐层 CPU offload 传输开销大
- **核心思路**: 将模型层分组，每次推理只将当前组加载到 GPU，处理完后卸载回 CPU（或 disk）；
  通过 `ModelHook` 机制在 `pre_forward` 时加载、`post_forward` 时卸载；
  支持 `pinned_memory`（锁页内存）和 CUDA stream 异步传输，减少等待
- **参考来源**: src/diffusers/hooks/group_offloading.py（GroupOffloading）
- **迁移建议**: PaddleFormers 的 Trainer 已有 PP（Pipeline Parallel）和 CPU offload 支持；
  推理侧若需单卡低显存推理，可参考此 hook 模式实现轻量级 offload，
  paddle 中对应 `paddle.device.set_device` + `tensor.cpu()/cuda()` 替代 `send_to_device`
