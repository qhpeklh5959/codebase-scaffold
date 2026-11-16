按 `.agent/refs/` 中已形成的规范与模式审查当前改动，输出带引用的只读报告。用法：`/review [<路径>|<提交范围>]`

本命令只读：不修改源码、refs 或 shadow 状态；唯一例外是审查过程本身失败时按 Step 7 记录 `failures.md`。代码修复交给 `/extend`，知识沉淀交给 `/reflect`。

---

## Step 1：加载既成规范

解析 `$ARGUMENTS` 为可选的路径或提交范围（在 Step 2 中使用），然后读取：

- `.agent/refs/CODEBASE.md` — 技术栈、目录职责、产物边界。若缺失，停止并提示用户先运行 `/bootstrap`。
- `AGENTS.md`（若存在）与 `.agent/refs/conventions.md` — 前置检查、refs 记录规则、失败与输出规范。`AGENTS.md` 缺失时以 `conventions.md` 为行为规则来源。
- `.agent/refs/patterns.md` — 已形成的扩展模式。
- `.agent/refs/symbols.md` — 关键入口与扩展点。
- `.agent/refs/shadow-state.md` 的 `## Active` 节（到下一个 `##` 为止）— 当前被遮蔽的目标。
- 仅当改动涉及已导入依赖或参考模式时，读取 `.agent/refs/deps.md`、`.agent/refs/refs-registry.md` 及其子文件。
- 仓库根存在 `AGENT-DEVELOPMENT.md` 时，将其作为该仓库自身的附加边界。

只以上述文件为标准；不得臆造 refs 中不存在的规则。某条规则依赖的文件或小节不存在时，在报告中标为 `Unverifiable`，不要用通用最佳实践替代；`AGENTS.md` 缺失时，依赖它的规则按此处理。

## Step 2：确定审查范围

- 无参数：审查整个工作区改动（已暂存、未暂存、未跟踪），使用 `git status --porcelain=v1`、`git diff --cached`、`git diff`。
- 路径参数：只审查该路径下的改动。
- 提交范围（如 `HEAD~2..HEAD`）：使用 `git diff <range>` 审查。
- 没有改动时直接说明并结束，不得编造发现。
- 每个改动文件记录为仓库相对路径及改动行号；不得用本机绝对路径建立引用。

## Step 3：逐项对照 refs

对每个改动文件应用下表中匹配的行。每条 Blocker 或 Warning 必须同时给出 refs 依据和改动侧行号。

| 检查项 | 触发条件 | 规则来源 |
|---|---|---|
| 前置条件与遮蔽 | 改动文件命中 Active 遮蔽目标 | `AGENTS.md` 的前置检查节、`conventions.md` 的任务前置条件节、`shadow-state.md` 的 `## Active` |
| refs 记录 | 改动涉及 `.agent/refs/**` | `AGENTS.md` 的知识库维护节、`conventions.md` 的记录规则节：只写结论不贴原始代码；相对路径+行号；失败按错误类型合并；超过 200 行拆分 |
| skill 与 command 结构 | 改动新增或编辑 `.codex/skills/**` 或 `.claude/commands/**` | `gen-command.md` 的生成模板、`README.md` 的三种 CLI 入口节：kebab-case 目录；`name` 与可区分的 `description` frontmatter；`$name` 用法；三入口齐备（`.codex`、`.claude`、`.dsh` 软链） |
| 依赖与参考 | 改动使用外部符号或借鉴模式 | `AGENTS.md` 的依赖与参考库节、`symbols.md`、`deps.md`、`refs-registry.md` |
| 模式一致性 | 改动新增实现或扩展点 | `patterns.md` 对应小节；偏离时必须有明确理由 |
| 失败与验证 | 改动包含测试或验证记录 | `AGENTS.md` 的失败与输出节、`conventions.md`：三次失败上限；失败按错误类型合并；列出未覆盖分支 |
| 部署边界 | 改动涉及部署清单或模板 | `AGENT-DEVELOPMENT.md`（若存在）、`README.md` 的部署节：不得把本仓库 bootstrap 产物或 `.agent/self/` 当作模板部署 |

refs 中没有规则覆盖的改动标为 `uncovered`，不算违规；建议用 `/reflect` 补充规则。

## Step 4：定位并校验 refs 依据

Step 3 的规则来源是“文件 + 小节/规则描述”，不要依赖固定行号：

1. 打开被引用文件，找到与规则描述一致的小节或条目，记录其当前 `文件:行号`。
2. 找不到对应内容时，将该条标记为 `[refs-stale]` 或 `Unverifiable`，展示已读取到的内容与期望规则，并建议 `/reflect`。
3. 规则缺失不构成改动违规，只说明该规则无法验证。

## Step 5：输出审查报告

按严重级别排序输出表格：

```
文件:行号 | 级别 | 规则（refs 引用） | 证据 | 建议修复
```

- 级别：`Blocker`（明确违反 refs）、`Warning`（模式偏离或风险）、`Unverifiable`（refs 缺失或过期）。
- 每条 Blocker 与 Warning 都要同时给出 refs 的实际行号（Step 4 定位结果）和改动行号。
- 末尾给出 Blocker / Warning / uncovered 的汇总，以及未审查的范围和原因。
- 只引用关键行，不粘贴大段代码。

## Step 6：交接

- 需要改代码的发现：列出 `文件:行号` 和修复建议，由用户决定是否用 `/extend` 执行。
- 发现可复用的新规则：建议 `/reflect` 写回 refs；本命令不直接修改 refs。
- 发现重复的多步骤工作流：建议 `/gen-command --suggest`。
- 遮蔽相关风险：转入 `/extend` 或 `/rollback` 流程，不得直接修改 stub。

## Step 7：更新 failures.md

仅当审查过程本身失败（git 不可用、refs 无法读取、范围无法解析）时追加，不记录审查发现的问题：

```
{错误类型} / {根因} / {修正方式}
```

与同类错误的已有条目合并，不重复追加。

## 完成输出

- 审查范围（工作区 / 路径 / 提交范围）与被审查文件数。
- 按级别排序的问题表；没有问题时明确写“未发现违反 refs 的改动”。
- refs 过期或缺失的锚点，以及对应的 `/reflect` 建议。
- 明确列出未能评估的检查域。

## 重试规则

审查本身失败时：1. 解析错误；2. 查阅 `.agent/refs/failures.md`；3. 修正后重试，最多 3 次；第 3 次失败后报告根因并停止。
