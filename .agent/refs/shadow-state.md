# Shadow 状态记录

本文件由以下 skill 共同维护，是 shadow/extend/rollback 之间唯一共享的状态：

- `/shadow` 写入 Active
- `/extend`（Shadow 恢复模式）将 Active 移动到 Restored
- `/rollback` 将 Active 或 Restored 移动到 Rolled Back

**请勿手动修改。**

---

## Active（当前遮蔽中）

<!-- 格式：
### SHADOW_ID
- **目标**: ClassName.methodName | ClassName | com.example.pkg（或 src/pkg/）
- **类型**: method | class | package
- **遮蔽模式**: no-op | throw | log
- **遮蔽时间**: YYYY-MM-DD
- **遮蔽原因**: 用户提供的原因（可选）
- **备份目录**: .agent/shadows/SHADOW_ID/（extend 只可读 description.md，不可读原始代码）
- **功能描述**: .agent/shadows/SHADOW_ID/description.md
- **包含文件**:（类型为 package 时列出每个文件及其 stub 行范围）
  - src/service/UserService.java → stub 行范围 12-45
  - src/service/OrderService.java → stub 行范围 8-92
- **stub 行范围**: start-end（类型为 method 或 class 时填写）
- **调用方中性化**: 是 | 否
- **已处理调用方**:（调用方中性化为"是"时列出）
  - src/controller/UserController.java
  - src/controller/OrderController.java
  - 详见 .agent/shadows/SHADOW_ID/callers/strip-description.md
-->

## Restored（extend 重写后）

<!-- extend 完成后从 Active 移动到此，rollback 可继续从此回滚
### SHADOW_ID（已恢复）
- ...原有字段保留...
- **恢复时间**: YYYY-MM-DD
- **恢复方式**: extend（shadow 恢复模式）
- **推断依据**: description.md + 接口契约 / 调用方 / 同类实现
- **不确定项**: （若有）
-->

## Rolled Back（已回滚至原始代码）

<!-- rollback 完成后从 Active 或 Restored 移动到此
### SHADOW_ID（已回滚）
- ...原有字段保留...
- **回滚时间**: YYYY-MM-DD
- **回滚前状态**: Active（stub）| Restored（extend 重写）
- **调用方已回滚**: 是 | 否 | 不适用
- **编译验证**: 通过 | 失败（见 failures.md）
-->
