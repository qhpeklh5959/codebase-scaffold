在当前代码库中实现一个方法或类。若目标当前处于遮蔽状态（stub），自动读取 shadow 时生成的功能描述作为需求输入，完成后迁移 shadow 状态。

用法：`/extend <描述，如 "实现 UserService.createUser 方法" 或 "实现 EmailNotifier 类">`

---

## Step 1：加载上下文

读取以下 refs 文件（若 CODEBASE.md 不存在则停止，提示用户先运行 `/bootstrap`）：

1. `.agent/refs/CODEBASE.md` — 了解整体架构和目录结构
2. `.agent/refs/symbols.md` — 定位目标类/方法所在的接口或基类
3. `.agent/refs/conventions.md` — 了解命名和错误处理规范
4. `.agent/refs/patterns.md` — 了解扩展步骤
5. `.agent/refs/failures.md` — 避免已知错误

解析 `$ARGUMENTS`，确定：

- 目标是**方法实现**还是**新类实现**
- 目标方法/类的名称
- 所属的接口或基类（若 $ARGUMENTS 未指定，从 symbols.md 中推断）

**Shadow 感知**：读取 `.agent/refs/shadow-state.md`，检查目标是否存在于 Active 节：

- **若在 Active 中**：进入 Shadow 恢复模式（见 Step 2A）
- **若不在 Active 中**：正常实现模式（见 Step 2B）

## Step 2A：Shadow 恢复模式（目标当前为 stub）

从 shadow-state.md 的 Active 记录中读取：

- `SHADOW_ID`
- 功能描述路径：`.agent/shadows/{SHADOW_ID}/description.md`

**读取 description.md 作为首要需求输入**（不读取 `original_*` 备份代码）：

- 理解方法/类的整体职责、输入输出语义、执行流程、副作用、异常场景

然后补充以下上下文：

1. **接口/抽象类契约**：从 symbols.md 定位目标实现的接口，读取当前签名和文档注释
2. **调用方（call sites）**：用 Grep 找 2~3 处典型调用点，理解返回值使用方式和异常预期
3. **同类实现**：若有其他同接口的实现类，读取 1 个理解当前惯用结构
4. **相关测试**：用 Glob/Grep 查找对应测试文件，补充边界条件理解

目标写入位置：当前文件中含 `SHADOWED:` 标记的区域（方法体 stub 的范围）。

## Step 2B：正常实现模式（目标为新实现）

根据 symbols.md 中的路径，读取相关文件：

- 若是方法实现：读取该方法所在的类文件，理解现有上下文
- 若是新类：读取对应接口/抽象类，理解需实现的契约
- 额外读取 1~2 个同类型的现有实现，作为风格参考

## Step 3：生成实现

按以下要求生成代码：

1. **严格遵循** conventions.md 中的命名规范和错误处理模式
2. **参照** patterns.md 中对应的扩展步骤
3. 若是新类，还需完成注册/工厂/DI 配置（patterns.md 中有步骤）
4. 在关键决策处添加注释，标注依据（如 `// ref: conventions.md#error-handling`）
5. Shadow 恢复模式下：注释中同时标注需求来源（如 `// from description: 校验 username 唯一性`）
6. 不添加超出当前任务范围的额外功能

## Step 4：写入代码

- **Shadow 恢复模式**：用 Edit 精确替换 `SHADOWED:` 标记范围内的 stub，不影响文件其他部分
- **方法实现（正常模式）**：用 Edit 精确插入到目标位置
- **新类（正常模式）**：用 Write 创建新文件，路径遵循 conventions.md 中的目录规则；若需注册到工厂/容器，用 Edit 修改对应注册文件

## Step 5：自验证

1. **语法检查**：运行编译命令（从 CODEBASE.md 获取），若失败进入重试流程
2. **接口契约**：确认实现了接口/抽象类的所有方法
3. **依赖完整**：确认所有 import/依赖都已正确引入

## 重试流程

若编译失败：

1. 解析错误类型（ImportNotFound / TypeMismatch / MissingMethod 等）
2. 查阅 `failures.md`，是否有已知解法
3. 若有：直接应用修正
4. 若无：分析根因，更新 `failures.md`，若涉及 conventions.md 或 patterns.md 有误则同步修正
5. 修正后重新生成，最多重试 3 次

3 次失败后，输出错误类型、根因分析、已尝试的修正方式，请用户介入。
Shadow 恢复模式下，3 次失败时 stub 保持原样（不破坏现有状态）。

## Step 6：更新 shadow-state.md（仅 Shadow 恢复模式）

编译验证通过后，将 Active 节中的记录**移动**到 Restored 节：

```markdown
### {SHADOW_ID}（已恢复）
- ...（原有字段保留）
- **恢复时间**: {今天日期}
- **恢复方式**: extend（shadow 恢复模式）
- **推断依据**: description.md + 接口契约 / 调用方 / 同类实现（列出实际用到的）
- **不确定项**: （若有无法确认的逻辑，明确列出）
```

## 完成输出

- 生成/修改的文件列表（路径 + 修改描述）
- 若是 Shadow 恢复模式：说明推断依据来源和不确定项（若有），提示备份仍保留在 `.agent/shadows/{SHADOW_ID}/`
- 若新增了注册步骤，说明在哪里注册
- 建议下一步（如：可运行 `/test-gen <文件路径> 80` 验证实现正确性）
