# PaddleFleet 代码库概览

## 技术栈
- 语言：Python 3.10+
- 框架：PaddlePaddle (paddlepaddle-gpu)
- 包管理：uv + setuptools，workspace 含 `packages/paddlefleet_ops`
- Lint：ruff ~0.14.0，行宽 80，target py310
- 测试：pytest + unittest + parameterized
- CI：GitHub Actions，codecov 覆盖率

## 目录结构及职责
```
src/paddlefleet/
  models/              # 具体模型实现（GPT、Qwen3、LLaVA、KimiK25 等）
    gpt/               # GPT 系列：config、embedding、layer_specs、model
    qwen3_5/           # Qwen3.5 模型
    qwen3_vl/          # Qwen3-VL 多模态模型
    kimi_k25/          # Kimi K2.5 模型
    multimodal/        # 多模态通用组件
    vision/            # 视觉编码器（CLIP ViT、RADIO）
    common/            # 共享组件：embedding、loss、vision_layer
    backends.py        # BackendSpecProvider 协议 + LocalSpecProvider 实现
  transformer/         # Transformer 核心组件
    layer.py           # FleetLayer 基类
    transformer_config.py  # TransformerConfig（继承 ModelParallelConfig）
    transformer_layer.py   # TransformerLayer / TransformerLayerNode
    transformer_block.py   # TransformerBlock
    transformer_encoder.py # TransformerEncoder
    attention.py       # SelfAttention（标准 MHA）
    dot_product_attention.py  # DotProductAttention / CPDotProductAttention
    mlp.py             # MLP + MLPSublayersSpec
    moe/               # MoE 相关：router、expert、token_dispatcher、fusion_layer_utils、fp8_utils
                       #   fusion_layer_utils.py: FusionMoePyLayer / HybridEPMoePyLayer / MlpNode / UnZipNode / ZipNode
                       #   fp8_utils.py: ExpertsGroupGemmContiguousNode（FP8/BF16 GroupedGEMM 节点）
    multi_latent_attention.py  # MLA（Multi-Latent Attention）
    multi_token_prediction.py  # MTP（Multi-Token Prediction）
    hyper_connection.py        # Manifold Hyper Connection
    enums.py           # AttnMaskType 等枚举
  tensor_parallel/     # 张量并行：layers、mappings、cross_entropy
  pipeline_parallel/   # 流水线并行：p2p 通信、VPP 模拟器
  fp8/                 # FP8 量化：linear、quantization
  fusions/             # 融合算子：bias_dropout、bias_gelu、rms_norm 等
  training/            # 训练入口：arguments、initialize、global_vars
  refined_recompute/   # 精细化重计算
  gpt_builders.py      # GPT 模型构建入口
  model_parallel_config.py  # ModelParallelConfig 基础配置
  parallel_state.py    # 并行状态管理（TP/PP/CP group）
  process_groups_config.py  # ProcessGroupCollection
tests/
  single_card_tests/   # 单卡单元测试（unittest）
  multi_card_tests/    # 多卡集成测试
```

## 关键入口文件
- `src/paddlefleet/gpt_builders.py`：GPT 模型构建主入口，调用 `get_gpt_spec` 组装 PipelineLayer
- `src/paddlefleet/models/gpt/gpt_layer_specs.py`：所有 GPT layer spec 工厂函数
- `src/paddlefleet/training/initialize.py`：训练初始化（并行组、随机种子等）
- `src/paddlefleet/training/arguments.py`：命令行参数解析 `parse_args()`

## 扩展点索引（→ 详见 symbols.md）
- `BackendSpecProvider`：后端算子选择协议（Protocol）
- `FleetLayer`：所有模型层的基类
- `TransformerConfig`：模型配置数据类（继承 ModelParallelConfig）
- `*SublayersSpec`：各层的子层规格 dataclass（SelfAttentionSublayersSpec、MLPSublayersSpec 等）
- `get_attention_spec()`：注意力层 spec 工厂，通过 `attention_layer_type` 字符串分发

## 测试框架和运行命令
```bash
# 单卡测试
pytest tests/single_card_tests/

# 多卡测试（需要多 GPU 环境）
pytest tests/multi_card_tests/

# 代码风格检查
ruff check src/
ruff format src/
```
