将备份的原始代码直接覆写回代码库，绕过 agent 重写，适用于 `/restore` 结果不满意或需要紧急恢复的场景。用法：`/rollback <目标，如 UserService.createUser 或 UserService> | --list`

> 这是一个**纯机械操作**：直接读取 `/shadow` 保存的原始代码备份，覆写当前文件。不经过任何 agent 推断或重写。
> 无论目标当前处于 Active（仍是 stub）还是 Restored（已被 agent 重写），均可回滚。

---

## Step 1：解析参数

从 `$ARGUMENTS` 中解析：

- `--list`：列出所有可回滚的目标（Active + Restored 中备份未被清理的记录），不执行回滚
- `<TARGET>`：回滚指定目标

## Step 2：读取 shadow-state.md

读取 `.agent/refs/shadow-state.md`，在 Active 和 Restored 两节中查找目标记录：

**若是 `--list` 模式**，输出所有可回滚项：

```
可回滚项（共 N 项）：

[1] userservice_createuser  [状态: Active - 当前为 stub]
    目标: UserService.createUser
    原始文件: src/service/UserService.java
    备份: .agent/shadows/userservice_createuser/original_UserService.java

[2] paymentservice  [状态: Restored - 已被 agent 重写]
    目标: PaymentService
    原始文件: src/service/PaymentService.java
    备份: .agent/shadows/paymentservice/original_PaymentService.java

可运行 /rollback <目标> 直接恢复原始代码
```

**若是回滚模式**，查找目标记录：

- 若找不到：报告"该目标无可用的回滚记录"，列出所有可回滚项，停止
- 若找到：读取备份路径和原始文件路径

## Step 3：验证备份存在

检查备份文件 `.agent/shadows/{SHADOW_ID}/original_{filename}` 是否存在：

- 若不存在（可能已被 `--clean` 清理）：报告备份已丢失，无法回滚，停止
- 若存在：继续

## Step 4：确认当前状态

读取目标文件当前内容，判断当前状态，在回滚前输出提示，要求用户确认：

**若当前为 stub（Active）**：
```
即将回滚：UserService.createUser
当前状态：stub（由 /shadow 写入）
操作：将原始代码覆写回 src/service/UserService.java
注意：当前 stub 将被丢弃

确认继续？(yes/no)
```

**若当前为 agent 重写（Restored）**：
```
即将回滚：UserService.createUser
当前状态：已由 /restore 重写
操作：将原始代码覆写回 src/service/UserService.java
注意：/restore 生成的新实现将被丢弃，若需保留请先手动备份

确认继续？(yes/no)
```

用户未确认则停止。

## Step 5：覆写原始代码

直接将备份文件内容写入原始文件路径（Write 覆写），不做任何修改。

## Step 6：验证编译

运行编译命令（从 `.agent/refs/CODEBASE.md` 获取）：

- **成功**：继续下一步
- **失败**：说明原始代码与当前代码库其他部分存在不兼容（代码库在 shadow 期间有其他变更）
  - 输出编译错误详情
  - **不回滚此次覆写**（覆写已完成，原始代码已在文件中）
  - 更新 `failures.md`，记录该不兼容场景
  - 提示用户手动处理冲突

## Step 7：更新 shadow-state.md

将目标记录从所在节（Active 或 Restored）**移动**到 Rolled Back 节：

```markdown
### {SHADOW_ID}（已回滚）
- **目标**: ...（原有字段保留）
- **回滚时间**: {今天日期}
- **回滚前状态**: Active（stub）| Restored（agent 重写）
- **编译验证**: 通过 | 失败（见 failures.md）
```

## Step 8：更新 failures.md（积累经验）

若编译失败，追加到 failures.md：

```
## [RollbackConflict]
- 现象：原始代码回滚后编译失败
- 根因：代码库在 shadow 期间发生了不兼容变更（如：依赖接口签名变更、类被删除等）
- 修正方式：手动解决冲突，或使用 /restore 让 agent 重写以适配当前代码库
- 相关 refs 修正：
```

## 完成输出

- 回滚的目标（文件路径）
- 回滚前的状态（stub / agent 重写）
- 编译验证结果
- 若编译失败，列出冲突点并建议处理方式
