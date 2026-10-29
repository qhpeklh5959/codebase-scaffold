# 扩展模式

## 编辑本仓库或接入的业务仓库

1. 进入仓库后，读取 `CODEBASE.md` 和与任务相关的 refs；该前置检查由 `AGENTS.md` 要求。`AGENTS.md:7-12`
2. 直接描述要修改的功能，或使用 `$extend` 明确要求实现方法、类或恢复遮蔽目标。`.codex/skills/extend/SKILL.md:6-8`
3. 按仓库已有构建工具执行验证；若失败，记录根因并最多重试三次。`AGENTS.md:28`
4. 任务结束后调用 `$reflect`，将新观察写回 refs，并维护定制 skill 建议。`.codex/skills/reflect/SKILL.md:6-8`

## 陪跑式实现

1. 使用 `$companion` 先生成任务计划、验收条件和可勾选 checklist。`.codex/skills/companion/SKILL.md:6-10`
2. 每完成一个步骤，更新任务状态；只有代码、测试或配置已验证的可复用结论才同步到 refs。`.claude/commands/companion.md:33-38`
3. 业务代码库的任务状态保存在 `.agent/work/`；脚手架自我维护时保存在 `.agent/self/`，两者不可混用。`.claude/commands/companion.md:5-10`

## 初始化接入的业务代码库

1. 将 `AGENTS.md`、`.codex/`、`.claude/` 和 README 指定的空 refs 模板复制到目标仓库根目录；不要复制脚手架自身的 bootstrap 产物或 `.agent/self/`。`README.md:37-48`
2. 执行 `$bootstrap`，生成代码库概览、符号、规范和模式文档。`README.md:46-56`
3. 用 `$trace` 理解入口和核心路径；它只做静态分析。`.codex/skills/trace/SKILL.md:6-8`
4. 如有外部调用依赖，用 `$import-dep` 导入公开 API；如只借鉴思路，用 `$ref-from` 导入模式。`.codex/skills/import-dep/SKILL.md:6-8` `.codex/skills/ref-from/SKILL.md:6-8`

## 添加仓库专属自动化能力

1. 识别重复且多步骤的工作流。
2. 使用 `$gen-command` 生成新 skill；新文件位于 `.codex/skills/{kebab-case-name}/SKILL.md`，并包含 YAML frontmatter。`.codex/skills/gen-command/SKILL.md:8-14`
3. 将建议与状态维护在 `command-suggestions.md`。`.agent/refs/command-suggestions.md:1-14`

## 遮蔽与恢复

1. 仅在显式 `$shadow` 请求下遮蔽目标，并保留备份及状态。`.codex/skills/shadow/SKILL.md:2-8`
2. 用 `$extend` 基于描述恢复，或用 `$rollback` 还原备份；后者覆盖前必须确认。`.codex/skills/extend/SKILL.md:2-8` `.codex/skills/rollback/SKILL.md:2-8`
3. 状态统一记录在 `shadow-state.md`。`.agent/refs/shadow-state.md:1-9`
