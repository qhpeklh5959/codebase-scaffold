# PaddleFleet Codebase Agent

本目录是 [Ducc](https://ducc.baidu.com) 为 PaddleFleet 代码库维护的知识库，配合 `/bootstrap`、`/trace`、`/reflect`、`/extend`、`/test-gen` 等 skill 使用。

```
.agent/
├── refs/               # 知识库（由 Agent 维护，勿手动大幅修改）
│   ├── CODEBASE.md     # 代码库架构概览、扩展点索引
│   ├── symbols.md      # 关键类/方法的路径和签名
│   ├── patterns.md     # 扩展模式和执行路径
│   ├── conventions.md  # 编码规范
│   ├── failures.md     # 已知错误及修正方式
│   └── command-suggestions.md  # 推荐生成的定制 command
└── README.md           # 本文件
```

---

## 应用示例

### 1. 初始化知识库

第一次使用前，让 Agent 扫描代码库、建立 `refs/` 索引：

```
/bootstrap
```

对子目录单独初始化（如只关注 MoE 模块）：

```
/bootstrap src/paddlefleet/transformer/moe
```

---

### 2. 静态追踪调用链

追踪一个方法的完整调用树，不需要运行代码：

```
/trace GPTModel.forward
/trace MoELayer.forward
/trace FusionMoePyLayer.forward
```

输出示例（节选）：

```
MoELayer.forward (moe_layer.py:942)
├── TopKRouter.forward (moe_router.py:770)
│   ├── gate_detach_matmul(...)   [paddlefleet_ops]
│   └── self.gate_score_func(logits)  [dynamic: sigmoid/softmax]
├── self.dispatch(...)
│   └── token_dispatcher.token_dispatch(...)  [** AllToAll 通信 **]
└── FusionMoePyLayer.apply(...)
    └── MlpNode.forward(...)
        ├── UnZipNode.forward → paddle.nn.functional.moe_permute(...)
        ├── ExpertsGroupGemmContiguousNode.forward(...)
        │   ├── fwd_gate_up_fp8 → split_group_gemm / deep_gemm
        │   └── fwd_down_fp8   → fuse_weighted_swiglu_fp8_quant
        └── ZipNode.forward → paddle.nn.functional.moe_unpermute(...)
```

追踪时标注 `[dynamic]`（接口/回调/反射）、`[depth limit]`（超过 5 层）、`[** 通信 **]`（分布式通信节点），便于定位性能瓶颈或调试问题。

---

### 3. 实现一个新方法

用 `/extend` 让 Agent 根据 `refs/` 中的模式和规范自动实现代码：

**场景：为 TransformerLayer 添加一个新的 post-processing hook**

```
/extend TransformerLayer._post_process_hook
```

Agent 会：
1. 读取 `refs/CODEBASE.md` 了解模块边界
2. 读取 `refs/patterns.md` 找相似的扩展模式
3. 读取 `refs/conventions.md` 确保风格一致
4. 生成符合 PaddleFleet 规范的实现

**场景：为新的 attention 变体补全实现**

```
/shadow SelfAttention   # 先将目标设为 stub
# 描述需求后：
/extend SelfAttention   # Agent 读取 shadow 时记录的描述并实现
```

---

### 4. 生成测试

为指定目标生成覆盖率达标的单元测试：

```
/test-gen src/paddlefleet/transformer/moe/moe_router.py 80
/test-gen TopKRouter.forward 90
```

生成的测试会放入 `tests/single_card_tests/ai_edited_test/` 目录，遵循 `refs/conventions.md` 中的测试规范（`unittest.TestCase`，`MagicMock`，`assertEqual` 等）。

---

### 5. 更新知识库

完成一次 `/trace` 或 `/extend` 后，让 Agent 把新发现沉淀回 `refs/`：

```
/reflect
```

也可以带说明，指定更新方向：

```
/reflect 刚刚修改了 MoELayer 的分支逻辑，更新 patterns.md 中的路由决策树
```

`/reflect` 会自动：
- 补充 `symbols.md` 中缺失的类/方法
- 修正 `patterns.md` 中错误或过时的模式
- 将本次发现的失败案例写入 `failures.md`
- 在 `command-suggestions.md` 中建议值得封装的定制 command

---

### 6. 遮蔽（临时移除）一个模块

在重构期间，将某个实现替换为 stub，阻止其他代码依赖它：

```
/shadow MultiTokenPredictionLayer
```

Agent 会保留原始代码备份，并在 `refs/shadow-state.md` 中记录遮蔽状态，避免后续操作误改 stub 代码。恢复时：

```
/rollback MultiTokenPredictionLayer
```

---

### 7. 生成定制 Command

根据本项目的 `refs/command-suggestions.md` 中的建议，生成可复用的 command：

```
/gen-command --create trace-moe-path
/gen-command --create trace-pp-stage
```

生成后，后续可直接用简短命令替代复杂的手动操作：

```
/trace-moe-path --ep-size 8 --fusion-node true
/trace-pp-stage --pp-size 4 --stage-id 2
```

---

## 知识库维护说明

`refs/` 下的文件由 Agent 在每次 `/reflect` 时自动维护，记录的是**观察到的结论**，不粘贴原始代码。

| 文件 | 内容 | 更新时机 |
|------|------|---------|
| `CODEBASE.md` | 目录结构、模块职责、扩展点索引 | 新增模块时 |
| `symbols.md` | 类/方法的文件路径、行号、签名 | trace/extend 发现新符号时 |
| `patterns.md` | 扩展步骤、执行路径、设计决策 | 理解新模式时 |
| `conventions.md` | 命名、注释、测试、导入规范 | 发现新约定时 |
| `failures.md` | 已踩过的坑及修正方式 | 遭遇失败时 |

如果 `refs/` 内容与代码出现冲突，**以代码为准**，并通过 `/reflect` 修正 refs。
