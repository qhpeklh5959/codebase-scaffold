# 关键符号索引

## Codex skill 扩展点

- `bootstrap`：分析仓库并创建或更新 `.agent/refs/`。`.codex/skills/bootstrap/SKILL.md:2-13`
- `companion`：以计划与 checklist 陪跑实现，并同步已验证的知识库。`.codex/skills/companion/SKILL.md:2-11`
- `extend`：依据知识库实现方法或类，并支持遮蔽目标恢复。`.codex/skills/extend/SKILL.md:2-8`
- `test-gen`：生成并运行测试，遵循三次失败上限。`.codex/skills/test-gen/SKILL.md:2-8`
- `trace`：静态追踪方法调用与可选调用方；不执行动态分析。`.codex/skills/trace/SKILL.md:2-8`
- `gen-command`：创建仓库专属 skill，产物位于 `.codex/skills/{kebab-case-name}/SKILL.md`。`.codex/skills/gen-command/SKILL.md:2-14`
- `import-dep`：将实际使用的依赖库公开 API 导入 `.agent/refs/deps/`。`.codex/skills/import-dep/SKILL.md:2-8`
- `ref-from`：从无调用关系的参考仓库提炼实现模式。`.codex/skills/ref-from/SKILL.md:2-8`
- `shadow`：显式请求时将方法、类或包替换为 stub 或卸载，并保存备份。`.codex/skills/shadow/SKILL.md:2-8`
- `rollback`：显式请求时恢复遮蔽目标的原始备份；覆盖前需要确认。`.codex/skills/rollback/SKILL.md:2-8`
- `reflect`：复盘近期工作并维护知识库与定制 skill 建议。`.codex/skills/reflect/SKILL.md:2-8`

## 核心配置文件

- `AGENTS.md`：定义所有任务的知识库前置检查、refs 维护规则和失败处理上限。`AGENTS.md:5-31`
- `shadow-state.md`：`shadow`、`extend`、`rollback` 的唯一共享状态文件。`.agent/refs/shadow-state.md:1-9`

## 业务符号

- 尚未发现业务源码、接口、抽象类或可执行入口；后续 `$bootstrap` 应在引入业务代码后补充本节。`README.md:1-216`
