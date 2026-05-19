将备份的原始代码直接覆写回代码库，绕过 agent 重写。适用于 extend 恢复结果不满意或需要紧急恢复的场景。

用法：`/rollback <目标，如 UserService.createUser、UserService 或 com.example.service> | --list`

> **纯机械操作**：读取 `/shadow` 保存的备份文件覆写当前代码。无论目标处于 Active（stub/unload）还是 Restored（extend 重写），均可回滚。若 shadow 时处理了依赖方（strip-callers 或 unload），依赖方文件也会一并回滚。

---

## Step 1：解析参数

- `--list`：列出所有可回滚项（Active + Restored 中备份未清理的记录）
- `<TARGET>`：回滚指定目标

## Step 2：读取 shadow-state.md

在 Active 和 Restored 两节中查找目标记录。

**若是 `--list` 模式**，按类型分组输出：

```
可回滚项（共 N 项）：

[method] userservice_createuser  [Active - stub/no-op]
  目标: UserService.createUser
  文件: src/service/UserService.java
  依赖方处理: 无

[package] pkg_com_example_service  [Active - unload]
  目标: com.example.service（3 个文件，已删除）
  依赖方处理: unload-dependents（2 个文件已修改）

[class]  paymentservice  [Restored - extend 重写]
  目标: PaymentService
  文件: src/service/PaymentService.java
  依赖方处理: 无
```

**若是回滚模式**，查找记录；找不到则报告并停止。读取：遮蔽模式、类型、包含文件、依赖方处理方式。

## Step 3：验证备份完整性

检查 `.agent/shadows/{SHADOW_ID}/` 存在，且备份文件与 shadow-state.md 记录一致：
- 若备份目录不存在：报告备份丢失，停止
- 若部分文件缺失：列出缺失项，询问用户是否继续回滚剩余文件

## Step 4：输出回滚预览并确认

**Stub 模式（method/class/package）**：
```
即将回滚：{TARGET} [{SHADOW_ID}]
当前状态：Active（stub/{mode}）
将覆写目标文件（N 个）：
  src/service/UserService.java
将回滚调用方文件（M 个）：（若有 strip-callers）
  src/controller/UserController.java

确认继续？(yes/no)
```

**Unload 模式**：
```
即将回滚：{TARGET} [{SHADOW_ID}]
当前状态：Active（unload）
将重建目标文件（N 个，当前已不存在）：
  src/service/UserService.java
  src/service/OrderService.java
将还原依赖方文件（M 个，覆盖当前修改）：
  src/controller/UserController.java
  ⚠ 还原将覆盖 shadow 期间对依赖方的所有其他修改

确认继续？(yes/no)
```

用户未确认则停止。

## Step 5：回滚目标文件

**Stub 模式**：从备份读取文件内容，Write 覆写目标文件。

**Unload 模式**：从备份读取文件内容，Write 重建目标文件（文件已被删除，重新创建）；若所在目录不存在，先创建目录。

子包：遍历"包含文件"列表，逐一还原，文件名通过 sanitize 规则反推路径（`_` → `/`，保留扩展名）。

## Step 6：还原依赖方文件（若有）

**Stub + strip-callers**：从 `callers/` 目录逐一还原，Write 覆写各调用方文件。

**Unload**：从 `dependents/` 目录逐一还原，Write 覆写各依赖方文件。读取 `dependents/change-log.md` 确认所有记录的依赖方均已处理。

若某个依赖方备份缺失：记录警告，继续处理其余文件，最后汇总缺失项。

## Step 7：编译验证

运行编译命令：

- **成功**：继续
- **失败**：代码库在 shadow 期间发生了不兼容变更
  - 输出错误详情，**不撤销本次回滚**
  - 更新 failures.md
  - 提示用户手动处理冲突，或使用 `/extend` 让 agent 重写以适配当前代码库

## Step 8：更新 shadow-state.md

移动记录到 Rolled Back 节：

```markdown
### {SHADOW_ID}（已回滚）
- ...（原有字段保留）
- **回滚时间**: {今天日期}
- **回滚前状态**: Active（stub/{mode}）| Active（unload）| Restored（extend 重写）
- **依赖方已回滚**: 是（N 个文件）| 否 | 不适用
- **编译验证**: 通过 | 失败（见 failures.md）
```

## Step 9：更新 failures.md（积累经验）

若编译失败，追加：

```
## [RollbackConflict]
- 现象：回滚后编译失败
- 根因：代码库在 shadow 期间发生了不兼容变更
- 修正方式：手动解决冲突，或使用 /extend 重写以适配当前代码库
- 相关 refs 修正：
```

## 完成输出

- 回滚的目标（遮蔽模式 + 文件列表）
- 是否还原了依赖方文件（列出）
- 编译验证结果
- 若编译失败，列出冲突点和建议处理方式
