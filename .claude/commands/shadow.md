将一个方法或类替换为 stub，保留原始代码备份，记录遮蔽状态。用法：`/shadow <目标，如 UserService.createUser 或 UserService> [--mode no-op|throw|log] [--reason "原因"]`

---

## Step 1：加载上下文

读取以下 refs 文件（若 CODEBASE.md 不存在则停止，提示用户先运行 `/bootstrap`）：

1. `.agent/refs/CODEBASE.md` — 了解技术栈和目录结构
2. `.agent/refs/conventions.md` — 确定语言，生成语法正确的 stub
3. `.agent/refs/shadow-state.md` — 检查目标是否已处于遮蔽状态
4. `.agent/refs/failures.md` — 避免已知错误

解析 `$ARGUMENTS`：

- `TARGET`：方法（`ClassName.methodName`）或类（`ClassName`）
- `--mode`：stub 模式，默认 `no-op`
  - `no-op`：方法体为空/返回零值，静默丢弃调用
  - `throw`：抛出异常/返回错误，明确标记不可用
  - `log`：打印警告日志后返回零值，便于追踪是否被调用
- `--reason`：遮蔽原因，可选，记录在状态文件中

## Step 2：前置检查

**检查是否已遮蔽**：读取 `shadow-state.md`，若 Active 中已有该目标：
- 输出警告：当前已遮蔽，备份位于 `.agent/shadows/{SHADOW_ID}/`
- 询问用户是否覆盖，未明确确认则停止

**定位目标**：

- 从 `symbols.md` 中查找目标的文件路径和行号
- 若 symbols.md 中找不到，使用 Grep 搜索类/方法定义
- 若仍找不到，报告并停止

## Step 3：备份原始代码 + 生成功能描述

生成 SHADOW_ID：格式为 `{ClassName}_{methodName}` 或 `{ClassName}`（全小写+下划线），例如 `userservice_createuser`。

创建备份目录 `.agent/shadows/{SHADOW_ID}/`，写入两个文件：

```
.agent/shadows/userservice_createuser/
├── original_UserService.java     ← 完整文件副本（restore 时不可见）
└── description.md                ← 功能描述（restore 时可读）
```

**备份**：完整复制原始文件，确保恢复时有 ground truth。

**功能描述**：在写入 stub 之前，阅读原始实现，将其行为提炼为自然语言，写入 `description.md`。描述要求：

- **不包含任何代码**，只描述行为意图
- 方法遮蔽：描述该方法做什么、输入输出语义、副作用（如写 DB、发消息）、异常场景
- 类遮蔽：先写类的整体职责，再逐方法描述
- 格式示例：

```markdown
# UserService.createUser 功能描述

## 整体职责
接收用户注册信息，完成持久化并触发欢迎流程。

## 输入语义
- username: 唯一用户名，不允许重复
- email: 用于发送欢迎邮件，需通过格式校验
- password: 明文，方法内部负责加密后存储

## 执行流程
1. 校验 username 唯一性，重复时抛出 DuplicateUserException
2. 对 password 执行哈希加密
3. 将用户记录持久化到 users 表
4. 异步发送欢迎邮件（失败不影响主流程，仅记录日志）

## 返回值
创建成功的 User 对象，含自增 id

## 异常
- DuplicateUserException：用户名已存在
- ValidationException：邮箱格式不合法
```

## Step 4：生成 stub 代码

根据 `conventions.md` 确定语言，按以下规则生成 stub：

### 方法遮蔽

替换方法体，保留完整方法签名（注解、访问修饰符、参数列表不变）：

**Java / Kotlin（no-op）**
```java
// SHADOWED: UserService.createUser [shadow-id: userservice_createuser]
// Original backed up at .agent/shadows/userservice_createuser/
return null; // or 0 / false / void 根据返回类型
```

**Python（throw）**
```python
# SHADOWED: UserService.create_user [shadow-id: userservice_create_user]
raise NotImplementedError("SHADOWED: userservice_create_user")
```

**Go（log）**
```go
// SHADOWED: UserService.CreateUser [shadow-id: userservice_createuser]
log.Printf("[SHADOW] UserService.CreateUser called but shadowed")
return nil, nil
```

**TypeScript/JavaScript**
```typescript
// SHADOWED: UserService.createUser [shadow-id: userservice_createuser]
console.warn("[SHADOW] UserService.createUser called but shadowed");
return undefined;
```

stub 注释中必须包含 `SHADOWED:` 标记和 shadow-id，便于后续识别。

### 类遮蔽

对类中所有**公共方法**逐一应用方法遮蔽规则，保留类结构、字段声明、构造函数签名不变（构造函数体清空）。

## Step 5：写入 stub

使用 Edit 精确替换目标方法体（或目标类的所有方法体）：

- 记录替换前后的行号范围
- 若是类遮蔽，逐方法替换，每次记录行号

## Step 6：更新 shadow-state.md

在 `Active` 节中追加记录（Edit 追加，不覆盖现有记录）：

```markdown
### {SHADOW_ID}
- **目标**: {TARGET}
- **类型**: method | class
- **原始文件**: {相对路径}
- **备份路径**: .agent/shadows/{SHADOW_ID}/original_{filename}（restore 时不可读）
- **功能描述**: .agent/shadows/{SHADOW_ID}/description.md（restore 时可读）
- **遮蔽模式**: {mode}
- **遮蔽时间**: {今天日期}
- **遮蔽原因**: {reason 或"未指定"}
- **stub 行范围**: {start}-{end}（方法遮蔽时填写）
```

## Step 7：更新 failures.md（积累经验）

若执行过程中遇到任何问题（stub 语法错误、行号偏移、找不到方法等）：

- 分析根因
- 追加到 `failures.md` 对应类型节，或新增类型节
- 同步修正 `conventions.md` 中如有遗漏的语言特性描述

## 完成输出

- SHADOW_ID
- 备份位置
- 修改的文件路径 + 修改行范围
- stub 模式
- 下一步提示：`可运行 /restore {TARGET} 恢复原始实现`

## 重试规则

若写入 stub 后编译失败（可运行编译命令验证）：

1. 解析错误，判断是 stub 语法问题还是签名改变问题
2. 查阅 failures.md
3. 修正 stub，重新写入，最多 3 次
4. 3 次失败后，自动运行 `/restore` 恢复原始代码，并报告失败原因
