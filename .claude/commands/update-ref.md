拉取远端分支并把本地快进到最新，再按本次拉入的变更检查并更新 `.agent/refs/` 知识库，最后给出 trace 建议。用法：`/update-ref [<remote>/<branch>] [--check]`

本命令只更新本地分支和 refs：不推送、不强制、不自动 stash。`--check` 只同步并报告拟更新项，不写文件。

---

## Step 1：加载上下文

- `.agent/refs/CODEBASE.md` — 缺失则停止并提示先运行 `/bootstrap`。
- `AGENTS.md`（若存在）与 `.agent/refs/conventions.md` — refs 记录规则。
- `.agent/refs/patterns.md`、`.agent/refs/symbols.md`、`.agent/refs/shadow-state.md` 的 `## Active` 节。
- 仓库根 `AGENT-DEVELOPMENT.md`（若存在）作为该仓库的附加边界。

## Step 2：同步前检查

1. `git status --porcelain` — 有未提交改动时停止，要求用户先提交或手动 stash；不得自动 stash、reset 或丢弃改动。
2. `git rev-parse --abbrev-ref HEAD` — 当前分支。
3. 解析 `$ARGUMENTS`：识别 `--check` 与可选的 `<remote>/<branch>`；否则读取上游 `git rev-parse --abbrev-ref --symbolic-full-name @{u}`。两者都没有时停止并询问。
4. 记录 `OLD=$(git rev-parse HEAD)`。

## Step 3：拉取并快进

1. `git fetch --prune <remote>`。
2. 用 `git rev-list --left-right --count HEAD...<upstream>` 判断 ahead/behind：
   - ahead > 0 且 behind > 0：分叉，停止并报告，不 merge、不 rebase。
   - ahead > 0 且 behind == 0：本地领先，停止并提示先推送或合并；本命令不推送。
   - behind > 0：`git merge --ff-only <upstream>`。
3. 记录 `NEW=$(git rev-parse HEAD)`。`OLD == NEW` 时跳过变更分析，只执行 Step 6 的 refs 一致性检查。

## Step 4：分析本次变更

- `git log --oneline OLD..NEW`
- `git diff --name-status OLD..NEW`
- 按类型归类：源码、测试、文档、skill/command、refs、依赖/构建。
- 只处理该区间内的实际改动；不重写无关 refs。

## Step 5：更新知识库

沿用 `/reflect` 的记录规则：

1. 只记录观察到的结论和规律，不粘贴原始代码；每条带相对路径和行号。
2. 符号变化 → `symbols.md`；架构/职责变化 → `CODEBASE.md`；规范变化 → `conventions.md`；新增实现模式 → `patterns.md`。
3. 删除或重命名文件时，同步删除/修正对应 refs 条目，不保留失效引用。
4. 失败或工具问题按错误类型合并到 `failures.md`，不按时间堆积。
5. 单个 refs 文件超过 200 行时按主题拆分，并保留索引。
6. 不写本机绝对路径；跨库引用使用 `deps/{name}/` 别名。
7. 只写入可复用、已验证的结论；未确认的观察标为待验证项交回用户。
8. 若拉取内容已包含 `.agent/refs/**` 改动，以拉取结果为准，只补充缺失结论，不覆盖远端已提交的 refs。
9. 发现重复的多步骤工作流时，更新 `command-suggestions.md`，不自动生成新 skill。

`--check` 模式下只输出差异和拟更新项，不写文件。

## Step 6：校验

1. 逐条回读本次新增/修改的 refs 锚点，确认 `文件:行号` 仍指向预期内容。
2. 检查 refs 文件是否超过 200 行；超限则拆分。
3. 检查 Markdown 链接与被引用文件存在。
4. 本次拉取导致已有 refs 锚点行号变化时（如 README 行号），同步修正引用。

## Step 7：输出

- 分支与同步结果：`<branch>`，OLD → NEW，拉入 N 个提交、M 个文件；或“已是最新”。
- refs 变更清单：每个文件改了什么、依据哪个改动文件和行号。
- 未更新项及原因（无法验证、超出范围、待用户确认）。
- `--check` 模式下的建议清单。
- 明确声明：未提交、未推送；git 状态与 refs 改动由用户复核。

### Trace 任务建议

基于 `symbols.md` 与本次变更涉及的符号，按优先级给出 1~3 个 `/trace` 建议：

1. **入口方法**：本次变更影响的启动/入口方法。
2. **核心业务方法**：变更较多的核心 public 方法（create/process/handle 等）。
3. **扩展点实现**：接口或抽象类变更对应的已有实现类方法。

输出格式：

```
建议的 trace 任务（按优先级排序）：
1. /trace ClassName.methodName — 理由：一句话说明为什么这条链路值得追踪（依据 symbols.md 或 改动文件:行号）
```

选取原则：

- 只从 `symbols.md` 或本次改动中出现的真实符号选取，不臆造类名或方法名。
- 变更未触及业务符号，或仓库本身没有业务源码时，明确写“本次变更无可追踪的业务符号”，不输出占位建议。
- `--check` 模式同样输出 trace 建议，但不写 refs。

## Step 8：更新 failures.md

仅当同步或 refs 更新过程本身失败（fetch 失败、fast-forward 失败、refs 读写失败）时，按错误类型记录根因与修正方式；不记录普通代码变更。

## 重试规则

同步失败时：1. 解析错误；2. 查阅 `.agent/refs/failures.md`；3. 修正后重试，最多 3 次；3 次失败后报告根因并停止。网络类失败先确认远端可达，不盲目重试。