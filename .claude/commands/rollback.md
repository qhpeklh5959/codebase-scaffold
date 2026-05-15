将备份的原始代码直接覆写回代码库，绕过 agent 重写。适用于 extend 恢复结果不满意或需要紧急恢复的场景。

用法：`/rollback <目标，如 UserService.createUser、UserService 或 com.example.service> | --list`

> **纯机械操作**：直接读取 `/shadow` 保存的备份文件覆写当前代码。无论目标处于 Active（stub）还是 Restored（extend 重写），均可回滚。若 shadow 时启用了 `--strip-callers`，调用方文件也会一并回滚。

---

## Step 1：解析参数

- `--list`：列出所有可回滚项（Active + Restored 中备份未清理的记录），不执行回滚
- `<TARGET>`：回滚指定目标

## Step 2：读取 shadow-state.md

在 Active 和 Restored 两节中查找目标记录。

**若是 `--list` 模式**，按类型分组输出：

```
可回滚项（共 N 项）：

[method] userservice_createuser  [Active - stub]
  目标: UserService.createUser
  文件: src/service/UserService.java
  调用方已中性化: 否

[class]  paymentservice  [Restored - extend 重写]
  目标: PaymentService
  文件: src/service/PaymentService.java
  调用方已中性化: 否

[package] pkg_com_example_service  [Active - stub]
  目标: com.example.service（3 个文件）
  调用方已中性化: 是（2 个文件）
```

**若是回滚模式**，查找记录：
- 找不到：报告"无可用回滚记录"，列出所有可回滚项，停止
- 找到：读取类型、备份目录、包含文件列表、调用方中性化情况

## Step 3：验证备份完整性

检查备份目录 `.agent/shadows/{SHADOW_ID}/` 是否存在：
- 不存在（已被 `--clean` 清理）：报告备份丢失，停止
- 存在：确认备份文件与 shadow-state.md 记录的文件列表一致

## Step 4：输出回滚预览并确认

根据 shadow 类型，输出不同的确认提示：

**方法 / 类**：

```
即将回滚：UserService.createUser [userservice_createuser]
当前状态：Active（stub）
将覆写：src/service/UserService.java
调用方变更：无

确认继续？(yes/no)
```

**子包**：

```
即将回滚：com.example.service [pkg_com_example_service]
当前状态：Active（stub）
将覆写（3 个文件）：
  src/service/UserService.java
  src/service/OrderService.java
  src/service/EmailService.java
调用方变更：无

确认继续？(yes/no)
```

**含调用方中性化**：

```
即将回滚：com.example.service [pkg_com_example_service]
当前状态：Active（stub）
将覆写目标文件（3 个）：
  src/service/UserService.java
  src/service/OrderService.java
  src/service/EmailService.java
将回滚调用方文件（2 个）：
  src/controller/UserController.java
  src/controller/OrderController.java
  ⚠ 调用方回滚将覆盖 shadow 期间对这些文件的所有修改

确认继续？(yes/no)
```

用户未确认则停止。

## Step 5：覆写目标文件

按 shadow 类型执行：

**方法 / 类**：从 `.agent/shadows/{SHADOW_ID}/` 读取对应备份文件，Write 覆写目标文件。

**子包**：遍历 shadow-state.md 中"包含文件"列表，逐一从备份目录读取对应文件，Write 覆写各目标文件。文件名与备份的对应关系通过 sanitize 规则还原（`_` → `/`，保留扩展名）。

## Step 6：回滚调用方文件（若调用方已中性化）

读取 `.agent/shadows/{SHADOW_ID}/callers/` 目录，逐一还原各调用方文件：

1. 从 `callers/{sanitized_filename}` 读取备份内容
2. Write 覆写对应的调用方原始文件
3. 读取 `callers/strip-description.md` 确认所有记录的调用方文件均已处理

若某个调用方备份文件缺失，记录警告但继续处理其余文件，最后汇总缺失项报告。

## Step 7：编译验证

运行编译命令（从 CODEBASE.md 获取）：

- **成功**：继续下一步
- **失败**：代码库在 shadow 期间发生了不兼容变更
  - 输出编译错误详情，**不撤销本次覆写**
  - 更新 failures.md
  - 提示用户手动处理冲突，或使用 `/extend` 让 agent 重写以适配当前代码库

## Step 8：更新 shadow-state.md

将记录从所在节移动到 Rolled Back 节：

```markdown
### {SHADOW_ID}（已回滚）
- ...（原有字段保留）
- **回滚时间**: {今天日期}
- **回滚前状态**: Active（stub）| Restored（extend 重写）
- **调用方已回滚**: 是 | 否 | 不适用
- **编译验证**: 通过 | 失败（见 failures.md）
```

## Step 9：更新 failures.md（积累经验）

若编译失败，追加到 failures.md：

```
## [RollbackConflict]
- 现象：原始代码回滚后编译失败
- 根因：代码库在 shadow 期间发生了不兼容变更（接口签名变更、类被删除等）
- 修正方式：手动解决冲突，或使用 /extend 让 agent 重写以适配当前代码库
- 相关 refs 修正：
```

## 完成输出

- 回滚的目标（类型 + 文件列表）
- 是否回滚了调用方文件（列出）
- 编译验证结果
- 若编译失败，列出冲突点和建议处理方式
