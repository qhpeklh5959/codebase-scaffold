将一个方法、类或子包替换为 stub，可选同时中性化调用方依赖。保留原始代码备份，记录遮蔽状态。

用法：
- `/shadow <ClassName.methodName>` — 遮蔽单个方法
- `/shadow <ClassName>` — 遮蔽整个类
- `/shadow <包名或目录路径>` — 遮蔽整个子包（如 `com.example.service` 或 `src/services/`）
- 追加 `--mode no-op|throw|log` — stub 模式，默认 `no-op`
- 追加 `--strip-callers` — 同时中性化所有调用方的依赖调用
- 追加 `--reason "原因"` — 记录遮蔽原因

---

## Step 1：加载上下文

读取（若 CODEBASE.md 不存在则停止，提示用户先运行 `/bootstrap`）：

1. `.agent/refs/CODEBASE.md` — 技术栈、目录结构
2. `.agent/refs/conventions.md` — 确定语言，生成语法正确的 stub
3. `.agent/refs/shadow-state.md` — 检查目标是否已处于遮蔽状态
4. `.agent/refs/failures.md` — 避免已知错误

解析 `$ARGUMENTS`，确定：

- `TARGET`：方法 / 类 / 子包（见下方识别规则）
- `MODE`：`no-op`（默认）/ `throw` / `log`
- `STRIP_CALLERS`：是否指定了 `--strip-callers`
- `REASON`：遮蔽原因（可选）

**Target 类型识别规则**：

- 含 `.` 且最后一段首字母大写，如 `UserService.createUser` → **方法**
- 首字母大写，无路径分隔符，如 `UserService` → **类**
- 含路径分隔符（`/`）或全小写点分格式（`com.example.service`）→ **子包/目录**

## Step 2：前置检查

**已遮蔽检查**：在 shadow-state.md Active 节查找目标：
- 若已存在：输出警告，询问是否覆盖，未确认则停止

**定位目标文件列表**：

- **方法 / 类**：从 `symbols.md` 查找；找不到则 Grep 搜索；仍找不到则报告并停止
- **子包**：
  1. 若为路径格式（含 `/`），用 Glob 扫描该目录下所有源文件
  2. 若为包名格式（`com.example.service`），根据 CODEBASE.md 中的目录结构推断对应路径，再 Glob 扫描
  3. 列出将被遮蔽的所有文件，若超过 10 个，先输出文件列表请用户确认

## Step 3：生成 SHADOW_ID 并创建备份

**SHADOW_ID 规则**：

- 方法：`{classname}_{methodname}`，如 `userservice_createuser`
- 类：`{classname}`，如 `userservice`
- 子包：`pkg_{sanitized_package}`，如 `pkg_com_example_service` 或 `pkg_src_services`

创建备份目录 `.agent/shadows/{SHADOW_ID}/`，写入以下文件：

```
.agent/shadows/{SHADOW_ID}/
├── description.md               ← 功能描述（extend 恢复时可读）
├── {sanitized_path_file1}       ← 各源文件完整副本（extend 不可读）
├── {sanitized_path_file2}
└── callers/                     ← 仅 --strip-callers 时创建
    ├── strip-description.md     ← 记录每处调用方的修改内容
    ├── {sanitized_caller_file1} ← 调用方原始文件副本
    └── {sanitized_caller_file2}
```

**文件名sanitize规则**：将路径中的 `/` 替换为 `_`，保留扩展名，如 `src/service/UserService.java` → `src_service_UserService.java`。

**生成功能描述**（在写入 stub 之前）：

读取所有目标文件，将行为提炼为自然语言写入 `description.md`：
- **不包含任何代码**
- 方法/类：同原有规范
- 子包：先写包的整体职责和模块边界，再按文件逐一描述各类的职责

## Step 4：生成并写入 stub

根据 `conventions.md` 确定语言，对所有目标文件逐一处理：

### 方法 stub（替换方法体，保留签名）

**no-op**：返回零值，不抛出，不打印

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

### 类 stub（对类中所有公共方法逐一应用方法 stub）

- 保留类结构、字段声明、构造函数签名（构造函数体置空）
- 私有方法不处理（调用方不可见）

### 子包 stub

对包内每个文件，应用"类 stub"规则，逐文件逐方法处理。

用 Edit 精确替换目标方法体，记录每个文件的 stub 行范围。

## Step 5：中性化调用方（仅 `--strip-callers`）

### Step 5-1：查找调用方

用 Grep 搜索以下模式，找出所有调用方文件：
- 方法调用：`targetMethod(` 或 `target.method(`
- 类引用：import 语句、类型声明、实例化（`new ClassName`）
- 子包：搜索包内所有类名的引用

去重后得到调用方文件列表，排除目标文件本身。

### Step 5-2：备份调用方

将每个调用方文件完整复制到 `.agent/shadows/{SHADOW_ID}/callers/` 目录。

初始化 `callers/strip-description.md`，记录每个调用方文件将被修改的内容。

### Step 5-3：中性化调用方中的依赖

对每个调用方文件，逐一处理对遮蔽目标的引用：

**方法调用中性化**：

```java
// 原始
User user = userService.createUser(username, email, password);

// 中性化后（no-op 模式）
// [CALLER-STRIPPED: userservice_createuser] User user = userService.createUser(username, email, password);
User user = null; // stripped
```

```java
// 原始（void 调用）
userService.notifyUser(userId);

// 中性化后
// [CALLER-STRIPPED: userservice_createuser] userService.notifyUser(userId);
// stripped
```

**实例化中性化**：若调用方实例化了被遮蔽的类，替换为 null：

```java
// [CALLER-STRIPPED: userservice] UserService svc = new UserService(...);
UserService svc = null; // stripped
```

**导入语句**：保留 import（因为类型声明可能还在），在其上方加注释标记：

```java
// [CALLER-STRIPPED: userservice] import kept for type reference
import com.example.service.UserService;
```

**记录到 `strip-description.md`**，格式：

```markdown
## {调用方文件路径}
- 第 N 行：方法调用 `userService.createUser(...)` → 中性化为 null
- 第 M 行：实例化 `new UserService(...)` → 中性化为 null
```

### Step 5-4：写入修改后的调用方文件

使用 Edit 逐一应用中性化修改。

## Step 6：编译验证

运行编译命令（从 CODEBASE.md 获取）：

- **成功**：继续
- **失败**：进入重试流程
  1. 解析错误，判断是 stub 语法问题还是调用方中性化不完整（如遗漏了某处调用）
  2. 查阅 failures.md
  3. 修正后重试，最多 3 次
  4. 每次失败更新 failures.md
  5. 3 次失败后：回滚所有已修改的文件（目标文件 + 调用方），报告失败原因，停止

## Step 7：更新 shadow-state.md

在 Active 节中追加记录：

```markdown
### {SHADOW_ID}
- **目标**: {TARGET}
- **类型**: method | class | package
- **遮蔽模式**: {mode}
- **遮蔽时间**: {今天日期}
- **遮蔽原因**: {reason 或"未指定"}
- **包含文件**:（类型为 package 时列出）
  - src/service/UserService.java → stub 行范围 12-45
  - src/service/OrderService.java → stub 行范围 8-92
- **备份目录**: .agent/shadows/{SHADOW_ID}/（extend 只可读 description.md）
- **功能描述**: .agent/shadows/{SHADOW_ID}/description.md
- **调用方中性化**: 是 | 否
- **已处理调用方**:（--strip-callers 时列出）
  - src/controller/UserController.java（备份：callers/src_controller_UserController.java）
  - src/controller/OrderController.java（备份：callers/src_controller_OrderController.java）
  - 详见 .agent/shadows/{SHADOW_ID}/callers/strip-description.md
```

## Step 8：更新 failures.md（积累经验）

若遇到问题（stub 语法错误、调用方遗漏、编译失败等），追加到 failures.md，同步修正相关 conventions.md。

## 完成输出

- SHADOW_ID
- 遮蔽类型（方法 / 类 / 子包）及文件数量
- 备份位置
- 是否执行了调用方中性化，处理了哪些文件（列出）
- 编译验证结果
- 提示：运行 `/extend 实现 {TARGET}` 可重新实现；运行 `/rollback {TARGET}` 可直接恢复原始代码
