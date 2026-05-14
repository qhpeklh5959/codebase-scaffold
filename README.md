# Codebase Agent 脚手架

将 Claude Code 的能力与大型代码库结合，提供**扩展实现、测试生成、调用链追踪、遮蔽与恢复**能力，并通过持续学习机制形成代码库专属知识库。

## 快速开始

### 1. 部署到目标代码库

将以下内容复制到你的代码库根目录：

```
your-project/
├── CLAUDE.md                     ← 从本脚手架复制
├── .claude/
│   └── commands/
│       ├── bootstrap.md          ← 从本脚手架复制
│       ├── extend.md
│       ├── test-gen.md
│       ├── trace.md
│       ├── import-dep.md
│       ├── shadow.md
│       ├── restore.md
│       ├── rollback.md
│       └── reflect.md
└── .agent/
    ├── refs/
    │   ├── failures.md           ← 从本脚手架复制（空模板）
    │   ├── shadow-state.md       ← 从本脚手架复制（空模板）
    │   └── deps.md               ← 从本脚手架复制（空模板）
    ├── shadows/                  ← 由 /shadow 自动创建，存放原始代码备份
    └── refs/deps/                ← 由 /import-dep 自动创建，存放依赖库接口描述
```

或直接在项目根目录运行：

```bash
cp -r /path/to/codebase-agent/CLAUDE.md .
cp -r /path/to/codebase-agent/.claude .
cp -r /path/to/codebase-agent/.agent .
```

### 2. 初始化知识库（仅需一次）

```
/bootstrap
```

或指定路径：

```
/bootstrap /path/to/your/project
```

### 3. 开始使用

```bash
# 实现一个方法
/extend 实现 UserService.createUser 方法，要求持久化到数据库并发送欢迎邮件

# 实现一个新类
/extend 实现 EmailNotifier 类，实现 Notifier 接口

# 为某个文件生成测试，目标覆盖率 85%
/test-gen src/service/UserService.java 85

# 导入另一个代码库的公共 API 作为依赖描述
/import-dep ../codebase-a --name codebase-a

# 依赖库升级后刷新
/import-dep ../codebase-a --name codebase-a --refresh

# 追踪方法调用链（自动识别跨库调用节点）
/trace UserService.createUser

# 追踪调用链（同时显示谁调用了它）
/trace UserService.createUser --callers

# 遮蔽一个方法（替换为 stub，原始代码自动备份）
/shadow UserService.createUser --mode throw --reason "隔离排查依赖问题"

# 遮蔽整个类
/shadow PaymentService --mode log

# 查看当前所有遮蔽状态
/restore --list

# 恢复被遮蔽的方法（agent 重写）
/restore UserService.createUser

# 直接回滚到原始代码（跳过 agent 重写）
/rollback UserService.createUser

# 查看所有可回滚的项
/rollback --list

# 更新知识库（建议每次任务后运行）
/reflect
```

## 目录说明

| 路径 | 说明 |
|------|------|
| `CLAUDE.md` | Agent 行为规则（前置检查、重试规则、输出规范） |
| `.claude/commands/bootstrap.md` | 分析代码库，初始化 `.agent/refs/` |
| `.claude/commands/extend.md` | 实现方法或类 |
| `.claude/commands/test-gen.md` | 生成测试，达到指定覆盖率 |
| `.claude/commands/trace.md` | 静态调用链追踪 |
| `.claude/commands/import-dep.md` | 导入依赖库的公共 API 描述，建立依赖接口层 |
| `.claude/commands/shadow.md` | 遮蔽方法/类为 stub，备份原始代码 + 生成功能描述 |
| `.claude/commands/restore.md` | agent 重写被遮蔽的方法/类，读功能描述不读备份代码 |
| `.claude/commands/rollback.md` | 直接将备份代码覆写回代码库，绕过 agent 重写 |
| `.claude/commands/reflect.md` | 更新知识库，总结经验 |
| `.agent/refs/CODEBASE.md` | 代码库概览（由 bootstrap 生成） |
| `.agent/refs/symbols.md` | 关键符号索引（由 bootstrap 生成） |
| `.agent/refs/conventions.md` | 编码规范（由 bootstrap 生成） |
| `.agent/refs/patterns.md` | 扩展模式（由 bootstrap 生成） |
| `.agent/refs/failures.md` | 失败记录，持续更新 |
| `.agent/refs/deps.md` | 依赖库注册表（由 /import-dep 维护） |
| `.agent/refs/deps/{name}/` | 依赖库的接口描述（symbols + 可选 conventions） |
| `.agent/refs/shadow-state.md` | 遮蔽状态记录（Active / Restored / Rolled Back） |
| `.agent/shadows/` | 原始代码备份（由 /shadow 自动创建） |

## 设计原则

- **高内聚**：每个 skill 职责单一，refs 文件按主题隔离
- **低耦合**：skill 之间不互相调用，只通过 `.agent/refs/` 共享知识
- **可复用**：skills 是通用的，refs 是代码库专属的
- **持续学习**：failures.md 随每次任务积累，避免重复犯错

## 注意事项

- `bootstrap` 生成的 refs 是起点，会随使用逐渐完善
- `trace` 是纯静态分析，接口多态场景会标注 `[dynamic]` 提示
- `shadow` / `restore` / `rollback` 具有依赖关系：必须先 shadow 才能 restore 或 rollback；restore 之后仍可 rollback（备份默认保留）
- `restore` 使用 agent 重写，与当前代码库设计语言一致但结果可能与原始有差异；`rollback` 直接复原原始代码，结果确定但可能与当前代码库存在兼容性问题
- 建议将 `.agent/refs/` 提交到版本控制，团队共享知识库；`.agent/shadows/` 按需决定是否提交
- `failures.md` 尤其值得保留，记录了代码库特有的"坑"
