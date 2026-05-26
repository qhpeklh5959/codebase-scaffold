# PaddleFormers Codebase Agent

基于 Claude Code 的 PaddleFormers 专属研发 Agent，内置代码库知识库（`.agent/refs/`）和一套定制 commands，覆盖模型接入、训练、对齐、测试、依赖管理等核心研发场景。

---

## 快速开始

首次使用时，初始化知识库：

```
/bootstrap
```

之后所有 commands 均可直接使用，Agent 会自动读取 `.agent/refs/` 中的上下文。

---

## Commands 一览

### 模型开发

| Command | 用法 | 说明 |
|---------|------|------|
| `/add-model` | `/add-model <model_name> [--ref <hf_model_id>]` | 为 PaddleFormers 添加新模型，自动创建 Config/Model/Tokenizer、注册 Auto 模块和 Chat Template |
| `/extend` | `/extend <描述>` | 在代码库中实现一个方法或类，自动处理 shadow stub 状态 |
| `/trace` | `/trace <方法全限定名>` | 静态追踪方法完整调用链，标注动态调用风险 |
| `/shadow` | `/shadow <目标>` | 将方法/类替换为 stub 或从代码库卸载，保留备份 |
| `/rollback` | `/rollback <目标>` | 从备份恢复原始代码 |

### 训练与对齐

| Command | 用法 | 说明 |
|---------|------|------|
| `/train-model` | `/train-model <model> [--stage SFT\|PT\|DPO\|VL-SFT] [--lora] [--tp N] [--pp N]` | 生成训练配置并执行端到端训练 |
| `/align` | `/align <model_name> [--hf <hf_path>] [--pf <pf_path>] [--mode logits\|loss\|train]` | 对齐 PaddleFormers 与 HuggingFace 的数值输出 |

### 测试

| Command | 用法 | 说明 |
|---------|------|------|
| `/test-gen` | `/test-gen <目标文件或类路径> <覆盖率>` | 生成测试脚本，达到目标覆盖率 |

### 资源受限场景

| Command | 用法 | 说明 |
|---------|------|------|
| `/shrink-config` | `/shrink-config <模型路径> [--layers N] [--ratio R] [--vram G] [--auto-vram] [--out PATH]` | 生成缩层 config.json，自动探测 GPU 显存估算目标层数，同步生成过滤后的权重索引文件 |

### 知识库维护

| Command | 用法 | 说明 |
|---------|------|------|
| `/bootstrap` | `/bootstrap [路径]` | 分析代码库，初始化 `.agent/refs/` 知识库 |
| `/reflect` | `/reflect [说明]` | 根据近期任务经验更新知识库（failures/patterns/symbols） |
| `/import-dep` | `/import-dep <依赖库路径> [--name 别名] [--refresh]` | 将外部依赖库的公共 API 导入为依赖描述（仅导入本库实际用到的部分） |
| `/refresh-dep` | `/refresh-dep [<name>]` | 验证已导入依赖的符号签名是否仍有效，标记失效条目 |
| `/ref-from` | `/ref-from <参考库路径>` | 从另一个代码库提取实现模式作为参考（无调用关系） |
| `/gen-command` | `/gen-command [需求描述\|--suggest\|--list\|--create <name>]` | 生成新的定制 command，或分析代码库建议应新增哪些 command |

---

## 知识库结构

```
.agent/refs/
├── CODEBASE.md          # 代码库架构概览、扩展点索引
├── conventions.md       # 编码规范（命名、错误处理、导入顺序等）
├── patterns.md          # 扩展模式（添加新模型、新 Callback 等步骤）
├── symbols.md           # 核心类/接口符号表（路径 + 签名）
├── failures.md          # 失败经验积累（错误类型、根因、修正方式）
├── shadow-state.md      # 当前处于 stub 状态的目标列表
├── refs-registry.md     # 已注册的参考库列表
├── command-suggestions.md  # 待生成的 command 建议
├── deps.md              # 外部依赖库注册表
├── deps/
│   └── PaddleFleet/
│       ├── symbols.md   # PaddleFleet 中本库实际使用的符号（42 个）
│       └── conventions.md  # 两库共享的编码约定
└── refs/                # 参考库知识（由 /ref-from 导入）
```

---

## 已导入依赖

| 依赖 | 路径 | 符号数 | 导入时间 |
|------|------|--------|----------|
| PaddleFleet | `/root/.../PaddleFleet` | 42 | 2026-05-26 |

---

## 行为规则摘要

- 执行任何代码任务前，自动读取 `CODEBASE.md` 了解架构
- 在 `symbols.md` 找不到符号时，依次查阅 `deps/{name}/symbols.md`
- 任务失败时，查阅 `failures.md` 中的已知解法，最多重试 3 次
- 每次重试前更新 refs（缺什么补什么）
- 生成代码时，注释中标注参考了哪条 refs 规则

详见 `CLAUDE.md`。
