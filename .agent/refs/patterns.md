# 扩展模式

## FP8 Expert 计算管线（FusionMoePyLayer 内部）

来源：`src/paddlefleet/transformer/moe/fp8_utils.py:505,672`

FP8 路径下单次 expert 前向的完整计算链：

```
gate_up (fwd_gate_up_fp8):
  fused_stack_quant(expert_w1, transpose=True)  → w1_fp8_stacked + scale
  fp8_quant_blockwise(x)                        → x_fp8 + x_scale (行粒度 1x128)
  split_group_gemm / m_grouped_fp8_gemm_nt_contiguous
    (x_fp8 @ w1_fp8.T) → o1 [m_sum, 2*ffn_dim]  (gate + up 拼接)

down (fwd_down_fp8):
  fuse_weighted_swiglu_fp8_quant(o1, unzipped_probs) → o2_fp8 + o2_scale
    (SwiGLU 激活 + expert 权重 × + FP8 量化 三步融合)
  split_group_gemm / m_grouped_fp8_gemm_nt_contiguous
    (o2_fp8 @ w2_fp8.T) → o3 [m_sum, hidden_size]
```

关键优化点：
- `fused_stack_quant`：将多个 expert 权重 offline 堆叠成 stacked tensor，避免每次 forward 拼接
- `fuse_weighted_swiglu_fp8_quant`：SwiGLU + 权重缩放 + FP8 量化三合一，节省一次中间内存分配
- `recompute_moe_gate_up=True`：不保存 o1，反向重算 gate_up，以内存换计算

## PyLayer 包装模式（FusionMoePyLayer）

来源：`src/paddlefleet/transformer/moe/fusion_layer_utils.py:1808`

`FusionMoePyLayer` 是一个 `paddle.autograd.PyLayer`，其作用是把 `MlpNode`（内部维护缓存状态的计算节点）包装成 Paddle 自动微分图中的一个节点：

- `forward`：构造 `MlpNode` → 执行 `ctx.node.forward()` → 保存 `cached_tensors` 到 ctx → 清空 node 内部缓存
- `backward`：从 ctx 恢复缓存 → `ctx.node.backward(output_grad)` → 返回梯度

**推论**：在 `FusionMoePyLayer.forward` 内部，所有计算都在 `@paddle.no_grad()` 域下执行（MlpNode 的 forward 方法加了 `@paddle.no_grad()`），梯度完全由 `backward` 手动实现，不走 Paddle 的自动微分。

## MoE 执行三段式：dispatch → compute → combine

来源：`src/paddlefleet/transformer/moe/moe_layer.py:601,636`

所有 EP 路径均遵循统一的三段结构：

```
dispatch(hidden_states, probs, routing_map)
  ├── dispatch_preprocess: 构建路由元数据（每 token 分配的 expert index）
  └── token_dispatch: AllToAll 通信，将 token 送到持有对应 expert 的 EP rank

compute（routed_experts_compute / FusionMoePyLayer / SonicMoE）
  ├── permute: 按 expert 重排输入 tokens
  ├── expert_forward: 对各 expert 的 token 分片分别执行 MLP（up_gate → act → down）
  └── unpermute: 逆向恢复 token 顺序

combine(hidden_states)
  ├── token_combine: AllToAll 通信，将 expert 输出送回原 rank
  └── combine_postprocess: reshape 回原始形状
```

两条主执行路径：
- `fusion_moe_forward`（EP > 1，moe_use_fusion_node=True）：dispatch + FusionMoePyLayer/SonicMoE + `_comm_manager.combine`
- `custom_forward`（EP > 1，moe_use_fusion_node=False）：dispatch + routed_experts_compute + combine

单卡路径（EP=1）绕过 AllToAll，直接用 Grouped GEMM 或 expert loop。

**关键推论**：任何 MoE 性能/正确性问题，先确认问题出现在哪一段（dispatch/compute/combine）再定位代码。

## MoE 路由配置决策树

来源：`src/paddlefleet/transformer/moe/moe_layer.py:942` forward 分支

```
expert_model_parallel_size > 1?
├── Yes: moe_use_fusion_node?
│   ├── Yes: fusion_moe_forward
│   │   └── _use_hybrid_ep_fusion() ? HybridEPMoePyLayer
│   │       : using_sonic_moe?       SonicMoE kernel
│   │       : FusionMoePyLayer (default)
│   └── No:  custom_forward
│           → routed_experts_compute → expert_forward（StandardMLPExpert loop）
└── No: moe_expert_fusion?
    ├── Yes: _forward_single_card_grouped_gemm_moe
    │   └── using_sonic_moe? SonicMoE : FusionMoePyLayer
    └── No:  _forward_single_card_moe（Python expert loop）
```

## GPTModel forward 执行路径（PipelineLayer 委托模式）

来源：`src/paddlefleet/models/gpt/gpt_model.py:148`

关键点：
- `GPTModel` **没有** 自定义 `forward` 方法，完全委托给父类 `PipelineLayer.forward`（外部 PaddlePaddle 库）
- `PipelineLayer` 在每个 PP stage 只执行分配给该 stage 的层子集（`self.run_function`）
- 层的注册顺序（来自 `get_layer_desc_list`）决定了数据流：
  1. `GPTEmbedding` → N × `TransformerLayer` → `WrappedPaddleNorm` → (可选 MTP 层) → `GPTLMHead`
- overlap 调度时通过 `overlapped_forward_backward` + `build_overlapped_nodes` 合并前反向

推论：追踪 forward 调用链时，必须进入各组件的 `forward` 方法，而不是 `GPTModel` 本身。

## TransformerLayer forward 内部分解

来源：`src/paddlefleet/transformer/transformer_layer.py:426`

标准（非 overlap）路径：
```
TransformerLayer.forward(dict_args)
  └── full_recompute?
      ├── True  → recompute(self._forward_impl, ...)
      └── False → self._forward_impl(**dict_args)
          ├── _forward_attention: input_layernorm → self_attn → self_attn_bda
          └── _forward_mlp:       post_attn_layernorm → mlp → mlp_bda
```

MTP 分支：若 `config.num_nextn_predict_layers > 0`，`forward` 会在进入 `_forward_impl` 前将拼接的 hidden_states 按 `num_nextn_predict_layers+1` split，主干走 `_forward_impl`，MTP 分量传递给后续 MTP 层。

## Spec 注入模式（动态子层）

来源：`src/paddlefleet/transformer/attention.py:255`, `src/paddlefleet/transformer/mlp.py:161`

所有 `self.xxx` 子层均通过 `build_spec_layer(spec, ...)` 在 `__init__` 时动态实例化，运行时具体类型由 spec 决定。静态分析只能确认接口，无法确认运行时实现。受影响的关键子层：
- `self.qkv_proj` / `self.o_proj`（attention）→ ColumnParallelLinear 或 FP8 变体
- `self.core_attention`（attention）→ `DotProductAttention` 或 `CPDotProductAttention`
- `self.input_layernorm` / `self.post_attention_layernorm` → RMSNorm / LayerNorm
- `self.mlp` → `MLP` 或 `MoELayer`（取决于 `config.n_routed_experts`）
- `self.up_gate_proj` / `self.down_proj`（MLP）→ ColumnParallelLinear / RowParallelLinear

## 添加新注意力机制

参考示例：`src/paddlefleet/transformer/attention.py`（SelfAttention）

步骤：
1. 在 `src/paddlefleet/transformer/` 下新建 `<name>_attention.py`
2. 定义 `<Name>SublayersSpec` dataclass（字段为子层类型）
3. 定义 `<Name>Attention(FleetLayer)` 类，`__init__` 接收 `config: TransformerConfig` 和 `sublayers_spec`
4. 在 `src/paddlefleet/models/gpt/gpt_layer_specs.py` 的 `get_attention_spec()` 中添加新的 `attention_layer_type` 分支
5. 在 `src/paddlefleet/models/gpt/gpt_layer_specs.py` 顶部 import 新类
6. 在 `src/paddlefleet/transformer/__init__.py` 中 re-export（如需对外暴露）

## 添加新模型

参考示例：`src/paddlefleet/models/qwen3_5/`

步骤：
1. 在 `src/paddlefleet/models/<model_name>/` 下创建目录，含 `__init__.py`
2. 创建 `layer_specs.py`：定义 `get_<model>_layer_spec()` 工厂函数，复用 `get_gpt_layer_local_spec` 或自定义
3. 创建 `<model>_model.py`：定义模型类（继承 `GPTModel` 或直接用 `PipelineLayer`）
4. 创建 `<model>_builders.py`（可选）：定义 `<model>_builder(config)` 入口函数
5. 在 `src/paddlefleet/models/__init__.py` 中注册新模型
6. 若需要自定义 embedding，参考 `src/paddlefleet/models/kimi_k25/embedding.py`

## 添加新后端（BackendSpecProvider 实现）

参考示例：`src/paddlefleet/models/backends.py`（LocalSpecProvider）

步骤：
1. 在 `src/paddlefleet/models/backends.py` 或新文件中定义类实现 `BackendSpecProvider` Protocol
2. 实现所有 `@abstractmethod` 方法（column_parallel_linear、row_parallel_linear、layer_norm、core_attention 等）
3. 在 `gpt_layer_specs.py` 中将 `LocalSpecProvider()` 替换为新 provider，或通过参数传入

## 添加新 MoE 配置

参考示例：`src/paddlefleet/models/gpt/moe_layer_specs.py`

步骤：
1. 在 `moe_layer_specs.py` 中扩展 `get_moe_layer_spec_for_backend()` 函数
2. 在 `TransformerConfig` 中添加新配置字段（`src/paddlefleet/transformer/transformer_config.py`）
3. 在 `gpt_builders.py` 中根据新配置字段选择对应 spec

## 标准单元测试结构

参考示例：`tests/single_card_tests/ai_edited_test/config_and_utils/test_ai_arguments.py`

```python
import unittest
from unittest.mock import MagicMock, patch

class Test<功能名>(unittest.TestCase):
    def test_<场景>(self):
        from paddlefleet.<module> import <target>
        # arrange
        # act
        result = <target>(...)
        # assert
        self.assertEqual(result, expected)

if __name__ == "__main__":
    unittest.main()
```

## 添加新融合算子

参考示例：`src/paddlefleet/fusions/fused_bias_swiglu.py`

步骤：
1. 在 `src/paddlefleet/fusions/` 下新建 `fused_<name>.py`
2. 实现融合函数，优先使用 `paddlefleet_ops` 中的 C++ 算子，fallback 到 paddle 原生实现
3. 在使用方（如 `mlp.py`）中 import 并替换原有实现
