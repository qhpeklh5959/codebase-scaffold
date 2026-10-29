# 编码与知识库规范

## 任务前置条件

- 任何代码任务先读取 `CODEBASE.md`；若其不存在，先执行 `$bootstrap`。`AGENTS.md:7-12`
- 目标可能被遮蔽时，先检查 `shadow-state.md` 的 Active 节。`AGENTS.md:12`

## 知识库记录规则

- refs 只记录观察到的结论和规律，不复制原始实现；每条结论带相对路径和行号。`AGENTS.md:16`
- 不记录本机绝对路径；跨库资料以 `deps/{name}/` 别名引用。`AGENTS.md:17`
- 发现 refs 遗漏或错误时立即修正；失败按错误类型合并到 `failures.md`。`AGENTS.md:18`
- 单个 refs 文件超过 200 行时，拆为按主题命名的子文件，并保留原文件作为索引。`AGENTS.md:19`

## 依赖与参考资料

- 本地符号缺失时，依次查询 `deps.md` 和 `.agent/refs/deps/{name}/symbols.md`；仍未找到时建议 `$import-dep`。`AGENTS.md:23`
- 当前库缺少实现模式时，查询 `refs-registry.md` 与 `.agent/refs/refs/{name}/patterns.md`；参考模式必须适配当前技术栈。`AGENTS.md:24`

## 失败、验证与输出

- 编译、测试或工具失败时，先查询 `failures.md`，补充根因与 refs 后重试；最多三次。`AGENTS.md:28`
- 生成代码的关键决策应标注相应 refs 来源。`AGENTS.md:29`
- 静态调用链以带相对路径和行号的树展示；动态风险标注 `[dynamic]`。`AGENTS.md:30`
- 覆盖率不足时，列出未覆盖分支或路径。`AGENTS.md:31`

## 当前仓库的文档风格

- 面向用户的说明使用中文 Markdown，以小节、代码块和表格组织。`README.md:1-216`
- Codex skill 使用 YAML frontmatter（`name`、`description`）后接 Markdown 工作流。`.codex/skills/bootstrap/SKILL.md:1-13`
