在当前代码库中实现一个方法或类。用法：`/extend <描述，如 "实现 UserService.createUser 方法" 或 "实现 EmailNotifier 类">`

---

## Step 1：加载上下文

读取以下 refs 文件（若不存在则停止，提示用户先运行 `/bootstrap`）：

1. `.agent/refs/CODEBASE.md` — 了解整体架构和目录结构
2. `.agent/refs/symbols.md` — 定位目标类/方法所在的接口或基类
3. `.agent/refs/conventions.md` — 了解命名和错误处理规范
4. `.agent/refs/patterns.md` — 了解扩展步骤
5. `.agent/refs/failures.md` — 避免已知错误

解析 `$ARGUMENTS`，确定：

- 目标是**方法实现**还是**新类实现**
- 目标方法/类的名称
- 所属的接口或基类（若 $ARGUMENTS 未指定，从 symbols.md 中推断）

## Step 2：定位目标

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
5. 不添加超出当前任务范围的额外功能

## Step 4：写入代码

- 方法实现：使用 Edit 精确插入到目标位置
- 新类：使用 Write 创建新文件，路径遵循 conventions.md 中的目录规则
- 若需要注册到工厂/容器，用 Edit 修改对应的注册文件

## Step 5：自验证

验证以下几点：

1. **语法检查**：尝试运行编译命令（从 CODEBASE.md 中获取），若失败进入重试流程
2. **接口契约**：确认实现了接口/抽象类的所有方法
3. **依赖完整**：确认所有 import/依赖都已正确引入

## 重试流程

若编译失败：

1. 解析错误信息，判断错误类型（ImportNotFound / TypeMismatch / MissingMethod 等）
2. 查阅 `.agent/refs/failures.md`，是否有该类型的已知解法
3. 若有：直接应用修正
4. 若无：
   - 分析根因
   - 更新 `.agent/refs/failures.md`（追加该错误类型的记录）
   - 若根因涉及 conventions.md 或 patterns.md 有误，同步修正
5. 修正后重新生成，最多重试 3 次

3 次失败后，输出：错误类型、根因分析、已尝试的修正方式，请用户介入。

## 完成输出

- 生成/修改的文件列表（路径 + 修改描述）
- 若新增了注册步骤，说明在哪里注册
- 建议下一步（如：可运行 `/test-gen <新文件路径> 80` 为其生成测试）
