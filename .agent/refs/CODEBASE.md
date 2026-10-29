# Codebase Agent 脚手架代码库概览

## 技术栈

- Markdown 配置型脚手架：以 `AGENTS.md` 定义项目行为，以 `.codex/skills/*/SKILL.md` 提供 Codex skill 入口，并由 `.claude/commands/*.md` 保存详细工作流。`README.md:1-3`
- 当前没有应用源码、包管理清单或构建配置，因此未识别到编程语言、可执行入口或可运行的测试命令。`README.md:1-216`

## 目录结构及职责

- `AGENTS.md`：进入本目录后应遵循的前置检查、知识库维护、失败处理与输出规则。`AGENTS.md:1-31`
- `.codex/skills/`：可由 Codex 发现并调用的 skill 入口；每个入口链接到对应完整工作流。`.codex/skills/bootstrap/SKILL.md:1-13`
- `.claude/commands/`：各 skill 的详细工作流参考文件，供 `.codex/skills/` 引用。`README.md:188`
- `.agent/refs/`：随仓库工作积累的 AI 可读知识库；本文件和下列索引文件均应提交并持续维护。`README.md:189-199`
- `.agent/shadows/`：仅在 `$shadow` 后创建，保存受遮蔽代码的备份。`README.md:200`

## 关键入口文件

- `AGENTS.md`：所有代码任务开始前要求读取 `CODEBASE.md`；缺失时要求执行 `$bootstrap`。`AGENTS.md:7-12`
- `.codex/skills/bootstrap/SKILL.md`：初始化或重建 `.agent/refs/` 的入口。`.codex/skills/bootstrap/SKILL.md:6-13`
- `README.md`：安装、初始化和各项 skill 的使用说明。`README.md:5-170`

## 扩展点索引

详见 [symbols.md](symbols.md)。当前扩展点是 11 个 Codex skills，而非业务代码接口。

## 测试框架和运行命令

- 未配置测试框架或构建工具，暂无可执行测试命令。`README.md:1-216`
- `$test-gen` 的工作流入口已存在；待本仓库加入实际源码和测试配置后，按 `patterns.md` 更新测试命令与模板。`.codex/skills/test-gen/SKILL.md:1-8`
- `$companion` 为实现任务创建计划和 checklist，并将已验证的可复用事实同步到业务库 refs。`.codex/skills/companion/SKILL.md:1-11`

## 产物边界

- 本文件、`symbols.md`、`conventions.md` 与 `patterns.md` 是对当前脚手架仓库的观察结果，不能作为目标业务代码库的模板。`AGENT-DEVELOPMENT.md:3-11`
- 部署到目标代码库时，只复制通用框架与空 refs 模板，再由目标库执行 `$bootstrap` 生成自己的知识库。`README.md:37-48`
