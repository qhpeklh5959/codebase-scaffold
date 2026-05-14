# Shadow 状态记录

本文件由以下 skill 共同维护，是 shadow/restore/rollback 之间唯一共享的状态：

- `/shadow` 写入 Active
- `/restore` 将 Active 移动到 Restored
- `/rollback` 将 Active 或 Restored 移动到 Rolled Back

**请勿手动修改。**

---

## Active（当前遮蔽中）

<!-- 格式：
### SHADOW_ID
- **目标**: ClassName.methodName 或 ClassName（整个类）
- **类型**: method | class
- **原始文件**: 相对于代码库根目录的路径
- **备份路径**: .agent/shadows/SHADOW_ID/original_filename（rollback 可读；restore 不可读）
- **功能描述**: .agent/shadows/SHADOW_ID/description.md（restore/rollback 均可读）
- **遮蔽模式**: no-op | throw | log-and-return-null
- **遮蔽时间**: YYYY-MM-DD
- **遮蔽原因**: 用户提供的原因（可选）
- **stub 行范围**: 行号start-行号end（方法遮蔽时记录）
-->

## Restored（agent 重写后）

<!-- restore 完成后从 Active 移动到此，rollback 可继续从此回滚
### SHADOW_ID（已恢复）
- ...原有字段保留...
- **恢复时间**: YYYY-MM-DD
- **恢复方式**: agent 重写（未读取备份）
- **推断依据**: 接口契约 / 调用方 / 同类实现 / 测试用例
- **不确定项**: （若有）
-->

## Rolled Back（已回滚至原始代码）

<!-- rollback 完成后从 Active 或 Restored 移动到此
### SHADOW_ID（已回滚）
- ...原有字段保留...
- **回滚时间**: YYYY-MM-DD
- **回滚前状态**: Active（stub）| Restored（agent 重写）
- **编译验证**: 通过 | 失败（见 failures.md）
-->
