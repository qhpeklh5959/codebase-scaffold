将一个方法、类或子包替换为 stub 或从代码库中完全卸载，可选同时处理调用方依赖。保留原始代码备份，记录遮蔽状态。

用法：
- `/shadow <ClassName.methodName>` — 遮蔽单个方法
- `/shadow <ClassName>` — 遮蔽整个类
- `/shadow <包名或目录路径>` — 遮蔽整个子包（如 `com.example.service` 或 `src/services/`）

选项：
- `--mode no-op|throw|log` — stub 模式（默认 `no-op`）：保留文件结构，替换方法体为 stub
- `--mode unload` — 卸载模式：删除目标文件，修改所有依赖方使代码库可编译
- `--strip-callers` — 仅 stub 模式有效；unload 模式下默认处理所有依赖方
- `--reason "原因"` — 记录遮蔽原因

---

## Step 1：加载上下文

读取（若 CODEBASE.md 不存在则停止，提示用户先运行 `/bootstrap`）：

1. `.agent/refs/CODEBASE.md` — 技术栈、目录结构
2. `.agent/refs/conventions.md` — 确定语言
3. `.agent/refs/shadow-state.md` — 检查目标是否已处于遮蔽状态
4. `.agent/refs/failures.md` — 避免已知错误

解析 `$ARGUMENTS`，确定 TARGET、MODE（默认 `no-op`）、STRIP_CALLERS、REASON。

**Target 类型识别**：
- `ClassName.methodName`（最后段首字母大写且含 `.`）→ **方法**
- `ClassName`（首字母大写，无路径分隔符）→ **类**
- 含 `/` 或全小写点分格式 → **子包/目录**

## Step 2：前置检查

**已遮蔽检查**：在 shadow-state.md Active 节查找目标，若已存在则询问是否覆盖，未确认则停止。

**定位目标文件列表**：
- 方法/类：从 symbols.md 查找；找不到则 Grep；仍找不到则报告并停止
- 子包：Glob 扫描目标目录，得到所有源文件；超过 10 个先列出请用户确认

## Step 3：备份原始代码 + 生成功能描述

生成 SHADOW_ID（规则：方法 `{class}_{method}`；类 `{classname}`；包 `pkg_{sanitized}`）。

创建 `.agent/shadows/{SHADOW_ID}/`，写入：

```
.agent/shadows/{SHADOW_ID}/
├── description.md               ← 功能描述（extend 恢复时可读；不含代码）
├── {sanitized_path_file1}       ← 目标文件完整备份（extend 不可读）
├── {sanitized_path_file2}
└── dependents/                  ← unload 模式下：被修改的依赖方文件备份
    ├── change-log.md            ← 每个文件的修改记录
    └── {sanitized_dep_file}     ← 依赖方原始文件备份
```

**生成 description.md**：读取目标文件，提炼为自然语言，不含任何代码。方法/类描述职责+契约；子包额外描述包的整体边界和各文件分工。

---

## Step 4A：Stub 模式（`--mode no-op | throw | log`）

对所有目标文件逐一替换方法体，保留文件结构（类签名、字段、构造函数签名不变）：

**no-op**（默认）：返回语言对应的零值，静默丢弃
```java
// SHADOWED: UserService.createUser [shadow-id: userservice_createuser]
return null;
```

**throw**：明确标记不可用
```java
// SHADOWED: UserService.createUser [shadow-id: userservice_createuser]
throw new UnsupportedOperationException("SHADOWED: userservice_createuser");
```

**log**：打印警告后返回零值
```java
// SHADOWED: UserService.createUser [shadow-id: userservice_createuser]
log.warn("[SHADOW] UserService.createUser called but shadowed");
return null;
```

类遮蔽时，对所有公共方法逐一应用；私有方法不处理。子包遮蔽时，对包内每个文件应用类遮蔽。

用 Edit 精确替换，记录每个文件的 stub 行范围。

### Stub 模式的调用方处理（`--strip-callers` 时）

用 Grep 查找所有调用方文件，备份到 `callers/` 目录（同原有逻辑），再逐一中性化调用：

- 方法调用 → 注释原调用，插入 null 赋值或空语句 + `[CALLER-STRIPPED]` 标记
- 实例化 → 替换为 null + `[CALLER-STRIPPED]` 标记
- import → 保留但加注释标记

记录到 `callers/strip-description.md`。

---

## Step 4B：Unload 模式（`--mode unload`）

### Step 4B-1：扫描所有依赖方

用 Grep 全面搜索以下引用，建立**依赖方文件列表**：

- import 语句中引用目标包/类的文件
- 字段声明使用目标类型的文件（`private UserService`、`val service: UserService` 等）
- 方法参数或返回类型使用目标类型的文件
- 直接方法调用（`userService.xxx()`）
- 实例化（`new UserService()`、`UserService()`）

排除目标文件自身，去重后得到依赖方列表。若列表非空，先输出依赖方文件列表供用户了解影响范围，不阻断流程。

### Step 4B-2：备份依赖方文件

将所有依赖方文件完整备份到 `.agent/shadows/{SHADOW_ID}/dependents/`，初始化 `change-log.md`。

### Step 4B-3：删除目标文件

直接删除所有目标文件（已备份在 Step 3）。子包遮蔽时删除包内所有源文件；若目录因此变空，也删除空目录。

### Step 4B-4：修改依赖方使代码库可编译

对每个依赖方文件，按以下顺序处理，目标是**让该文件编译通过**，不留对已删除代码的引用：

**移除 import 语句**：删除引用已删除类/包的 import 行。

**移除字段声明**：若字段类型是已删除的类，删除该字段声明。

**修改构造函数/方法签名**：若参数类型是已删除的类：
- 若该参数在方法体中有使用，删除参数并替换方法体中的用法（以 null 或默认值替代，加注释 `// [UNLOADED: {SHADOW_ID}]`）
- 若该参数在方法体中未使用，直接删除参数

**移除方法调用**：
- 有返回值且被使用：`User u = service.createUser(...)` → `User u = null; // [UNLOADED: {SHADOW_ID}]`
- 有返回值但未使用：`service.createUser(...)` → 删除整行，加注释 `// [UNLOADED: {SHADOW_ID}] was: service.createUser(...)`
- void 调用：删除整行，加注释

**移除类型注解/泛型参数**中对已删除类的引用（替换为 Object 或父类型，加注释标记）。

每处修改记录到 `change-log.md`：

```markdown
## src/controller/UserController.java
- 第 3 行：移除 import com.example.service.UserService
- 第 15 行：移除字段 private UserService userService
- 第 28 行：方法调用 userService.createUser(...) → null [UNLOADED]
```

### Step 4B-5：编译验证与修复

运行编译命令，检查是否有残留编译错误：

- **无错误**：继续
- **有错误**：分析每个错误，判断是否由卸载操作引起的遗漏引用
  - 若是遗漏引用：补充处理（重复 Step 4B-4 的对应动作）
  - 若是其他错误：记录到 failures.md，报告给用户
  - 最多修复循环 3 次；仍有错误则报告未解决项，提示用户手动处理

---

## Step 5：更新 shadow-state.md

在 Active 节追加记录：

```markdown
### {SHADOW_ID}
- **目标**: {TARGET}
- **类型**: method | class | package
- **遮蔽模式**: no-op | throw | log | unload
- **遮蔽时间**: {今天日期}
- **遮蔽原因**: {reason 或"未指定"}
- **备份目录**: .agent/shadows/{SHADOW_ID}/（extend 只可读 description.md）
- **功能描述**: .agent/shadows/{SHADOW_ID}/description.md
- **包含文件**:（package 时列出）
  - src/service/UserService.java
  - src/service/OrderService.java
- **stub 行范围**: start-end（method/class stub 时填写）
- **调用方处理**: 未处理 | strip-callers（stub）| unload-dependents（unload）
- **已处理依赖方**:（unload 或 strip-callers 时列出文件，详见 dependents/change-log.md 或 callers/strip-description.md）
```

## Step 6：更新 failures.md（积累经验）

遇到问题（stub 语法错误、依赖方修改遗漏、编译循环失败等）追加到 failures.md，同步修正 conventions.md。

## 完成输出

- SHADOW_ID 和遮蔽模式
- 目标类型及文件数量
- **Stub 模式**：已 stub 的文件和行范围；若 strip-callers 则列出处理的调用方
- **Unload 模式**：已删除的文件列表；已修改的依赖方文件列表；编译验证结果；若有未解决的编译错误，列出残留问题
- 提示：运行 `/extend 实现 {TARGET}` 可重新实现；运行 `/rollback {TARGET}` 可一键还原所有修改
