# 脚手架自身开发说明

本文件仅适用于 `codebase-scaffold` 仓库：它帮助 AI 修改本脚手架的行为、skill 和文档，**不得复制到使用该脚手架的业务代码库**。

## 修改边界

- `AGENTS.md` 定义会随框架部署到目标仓库的通用行为规则。`AGENTS.md:1-31`
- `.codex/skills/*/SKILL.md` 是 Codex 的 skill 入口；其内容应指向对应 `.claude/commands/*.md` 完整工作流。`.codex/skills/bootstrap/SKILL.md:6-13`
- `.claude/commands/` 是工作流参考；修改流程时应同步检查对应 skill 的映射。`README.md:177-188`
- `.agent/refs/` 是**当前仓库一次 bootstrap 的产物**。其中 `CODEBASE.md`、`symbols.md`、`conventions.md`、`patterns.md` 只描述这个脚手架，部署到业务仓库时不能复制。`.agent/refs/CODEBASE.md:1-31`
- `.agent/self/` 是脚手架自我维护隔离区；`$reflect` 在此模式下只能更新该目录，复盘产物只适用于本仓库。`.agent/self/README.md:1-7`
- `$companion` 在自我维护模式下同样只将计划、checklist 和过程性结论写入 `.agent/self/`；业务库模式才维护业务库的 refs。`.codex/skills/companion/SKILL.md:1-11`

## 修改流程

1. 先读 `.agent/refs/CODEBASE.md`、本文件及关联 skill/工作流。
2. 修改框架配置、skill 或文档时，保持 `$skill-name`、目录名和 legacy workflow 引用一致。
3. 如果修改影响部署内容，检查 README 的部署步骤只复制通用框架与空 refs 模板，不复制当前仓库的 bootstrap 产物。
4. 当前仓库没有构建配置或自动化测试；至少执行链接、文件存在性和 Markdown 内容一致性检查。`.agent/refs/CODEBASE.md:5-6`
5. 执行 `$reflect` 时，只更新 `.agent/self/reflect.md`；它不能修改可部署的 `.agent/refs/` 模板或产生业务库规则。
6. 目标业务库应在部署后自行运行 `$bootstrap` 生成自身 refs，并在其自身 `.agent/refs/` 中运行常规 `$reflect`。

## 非目标

- 不在此文件记录任何业务代码库的架构、领域规则或实现细节。
- 不把本文件或当前仓库生成的 `CODEBASE.md` 等文档作为模板部署。
