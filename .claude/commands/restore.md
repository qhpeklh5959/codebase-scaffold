根据当前代码库上下文，重新实现被 `/shadow` 遮蔽的方法或类。用法：`/restore <目标，如 UserService.createUser 或 UserService> | --list`

> 重写过程中**不读取备份**，备份对 agent 完全不可见，仅作为用户自行比对的安全网保留在磁盘上。Agent 通过接口契约、调用方、类型签名等上下文信息推断实现。

---

## Step 1：解析参数

从 `$ARGUMENTS` 中解析：

- `--list`：列出所有当前遮蔽状态，不执行恢复
- `<TARGET>`：恢复指定目标（方法或类）

## Step 2：读取 shadow-state.md

读取 `.agent/refs/shadow-state.md`：

**若是 `--list` 模式**，输出 Active 节中的所有记录：

```
当前遮蔽状态（共 N 项）：

[1] userservice_createuser
    目标: UserService.createUser
    备份: .agent/shadows/userservice_createuser/（请勿在 restore 前读取）
    遮蔽时间: YYYY-MM-DD  原因: ...

可运行 /restore <目标> 重新实现某一项
```

**若是恢复模式**，在 Active 节中查找目标对应的记录：

- 若找不到：报告"该目标未处于遮蔽状态"，列出当前所有 Active 项，停止
- 若找到：读取 SHADOW_ID 和原始文件路径（**不读取备份目录内容**）

## Step 3：从当前代码库推断实现意图

**不读取任何备份代码文件**（`original_*` 对 agent 不可见）。按优先级依次读取以下上下文：

**1. 功能描述（首要输入）**

读取 `.agent/shadows/{SHADOW_ID}/description.md`，这是 shadow 时提炼的行为意图描述：
- 方法的整体职责、输入输出语义、执行流程、副作用、异常场景
- 以此作为重写的**核心需求依据**，补充步骤 2~4 的上下文推断

**2. 接口 / 抽象类契约**

从 `.agent/refs/symbols.md` 定位目标所实现的接口或基类，读取：
- 方法签名（参数类型、返回类型）
- 接口上的文档注释（语义约束、前置/后置条件）

**3. 调用方（call sites）**

用 Grep 搜索目标方法名的调用点，读取 2~3 处典型调用方代码，理解：
- 调用方对返回值的使用方式
- 调用方对异常/错误的处理预期

**4. 同类其他实现**

若存在同一接口/基类的其他实现类，读取 1 个，理解当前代码库的惯用实现结构。

**5. 相关测试（若存在）**

用 Glob/Grep 查找目标对应的测试文件，读取测试用例，补充对边界条件和错误路径的理解。

**6. 当前规范**

- `.agent/refs/conventions.md` — 命名、错误处理、日志风格
- `.agent/refs/patterns.md` — 扩展模式、标准结构

## Step 4：重新实现

基于 Step 3 推断出的行为意图，生成新实现：

1. **契约完整**：覆盖接口/基类要求的所有方法，满足调用方的使用预期
2. **设计语言一致**：命名、错误处理、注释风格完全遵循当前 conventions.md 和同类实现的惯用写法
3. **不过度发明**：无法从上下文推断的逻辑，保守实现（返回合理默认值或抛出明确的未实现异常），并在注释中标注推断依据
4. 关键决策处注释标注来源，例如：
   - `// inferred from call site: OrderController.java:87`
   - `// ref: conventions.md#error-handling`
   - `// consistent with: EmailNotifier.send()`

## Step 5：写入代码

在当前文件中定位 stub（通过 `SHADOWED:` 标记确定范围），用 Edit 精确替换为新实现，不影响文件其他部分。

## Step 6：验证

运行编译命令（从 `.agent/refs/CODEBASE.md` 获取）：

- **成功**：继续下一步
- **失败**：进入重试流程
  1. 解析编译错误（依赖缺失、类型不匹配、签名错误等）
  2. 查阅 `failures.md` 是否有已知解法
  3. 修正实现，重新写入，最多重试 3 次
  4. 每次重试将错误和修正记录到 failures.md
  5. 3 次失败后：stub 保持原样，报告推断失败的具体点，请用户介入或补充上下文

## Step 7：更新 shadow-state.md

将 Active 节中对应的记录**移动**到 Restored 节：

```markdown
### {SHADOW_ID}（已恢复）
- **目标**: ...（原有字段保留）
- **恢复时间**: {今天日期}
- **恢复方式**: agent 重写（未读取备份）
- **推断依据**: 接口契约 / 调用方 / 同类实现 / 测试用例（列出实际用到的）
- **不确定项**: （若有无法确认的逻辑，明确列出）
```

## Step 8：保留备份

备份目录 `.agent/shadows/{SHADOW_ID}/` **始终保留**，供用户在 restore 完成后自行与新实现比对。

若用户在 $ARGUMENTS 中包含 `--clean`，删除备份目录，并在 Restored 记录中注明"备份已清理"。

## Step 9：更新 failures.md（积累经验）

若重写过程中遇到推断困难（接口文档缺失、无调用方、无同类实现可参考等），记录到 failures.md：

```
## [错误类型，如：InsufficientContext / NoCallSiteFound / AmbiguousContract]
- 现象：
- 根因：
- 修正方式：（如：补充接口文档、增加同类实现、在 symbols.md 中补充行为描述）
- 相关 refs 修正：
```

## 完成输出

- 重新实现的目标（文件路径 + 方法/类名）
- 推断依据来源（用到了哪些上下文信号）
- 不确定项（若有无法从上下文推断的逻辑，明确列出）
- 编译验证结果
- 备份保留路径（提示用户可自行比对）
- 建议下一步：`/test-gen <文件路径> 80` 验证新实现正确性
