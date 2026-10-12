# Codebase Agent 行为规则

本仓库的 Codebase Agent 由项目内 `.codex/skills/` 中的 skills 提供。用户可用 `$bootstrap`、`$companion`、`$extend`、`$test-gen`、`$trace`、`$gen-command`、`$import-dep`、`$ref-from`、`$shadow`、`$rollback` 或 `$reflect` 显式调用；也可用自然语言提出同类任务。

## 前置检查

执行任何代码任务前：

1. 检查 `.agent/refs/CODEBASE.md` 是否存在。
2. 若存在，先读取它，了解架构和扩展点，再开始工作。
3. 若不存在，停止并提示用户先运行 `$bootstrap`。
4. 任务涉及已知可能被遮蔽的目标时，检查 `.agent/refs/shadow-state.md` 的 Active 节，避免对 stub 代码误操作。
5. 若仓库根目录存在 `AGENT-DEVELOPMENT.md`，仅读取其作为该仓库自身开发补充；不得将它或该仓库的 bootstrap 产物复制到其他代码库。

## 知识库维护

- `.agent/refs/` 只记录观察到的结论和规律，不粘贴原始代码；每条记录标注代码库相对路径和行号。
- `.agent/work/` 仅记录当前任务的计划、checklist、验证和未确认项；经过验证且可跨任务复用的结论才可从 work 同步到 refs。
- 不记录本机绝对路径；跨库引用使用 `deps/codebase-a/` 这样的别名。
- 发现 refs 有误或遗漏时立即修正；`failures.md` 按错误类型合并，不按时间堆积。
- 单个 refs 文件超过 200 行时，按主题拆为 `{原文件名}-{主题}.md`。原文件变为链接到子文件的索引；任务中先读索引，再按需读取子文件。

## 依赖与参考库

- `symbols.md` 找不到符号时，检查 `deps.md`，再查 `.agent/refs/deps/{name}/symbols.md`；若找到，在生成代码注释中标注来源。仍找不到时建议用户运行 `$import-dep`。
- 新功能在 `patterns.md` 没有相应模式时，检查 `refs-registry.md` 和 `.agent/refs/refs/{name}/patterns.md`。参考库仅提供思路，须适配当前技术栈。

## 失败与输出

- 编译、测试或工具失败时，先分析错误并查询 `failures.md`；没有已知解法则记录根因、补齐 refs 后重试，最多 3 次。第三次失败后报告根因并请求用户介入。
- 生成代码的关键决策以注释标注相应 refs 来源。
- 静态调用链用树形展示，每层标注相对路径和行号；多态、反射等动态调用标注 `[dynamic]`。
- 覆盖率不够时明确列出未覆盖的分支或路径。
- 用户要求陪跑、计划先行或持续同步文档时，使用 `$companion`：先创建计划/checklist，再实施，并将已验证的可复用事实即时写入 refs。
