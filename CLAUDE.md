# Codebase Agent 行为规则

## 前置检查

执行任何代码任务前，必须：

1. 检查 `.agent/refs/CODEBASE.md` 是否存在
2. 若存在，先读取它，了解架构和扩展点，再开始工作
3. 若不存在，停止并提示用户先运行 `/bootstrap`
4. 若任务涉及已知可能被遮蔽的目标，顺带检查 `.agent/refs/shadow-state.md` 中的 Active 节，避免对 stub 代码进行误操作

## 知识库维护规则

- `.agent/refs/` 下的文件只记录**观察到的结论和规律**，不粘贴原始代码
- 每条记录注明来源文件路径（如 `src/core/UserService.java:42`）
- **路径只写相对于所在代码库根目录的相对路径**，不写任何本机绝对路径（如 `/home/user/...`、`/Users/...`、`C:\...`）；跨库引用时用别名代替绝对路径（如 `deps/codebase-a/` 而非 `/home/user/projects/codebase-a/`）
- 发现 refs 内容有误或遗漏时，**当场修正**，不等到失败才改
- `failures.md` 按错误类型归类，同类错误合并，不按时间堆积

## refs 文件拆分规则

单个 refs 文件超过 200 行时，按主题拆分为多个子文件：

**命名规则**：`{原文件名}-{主题}.md`，主题用英文 kebab-case，例如：
- `symbols.md` 过长 → `symbols-service.md`、`symbols-repository.md`、`symbols-domain.md`
- `conventions.md` 过长 → `conventions-naming.md`、`conventions-error-handling.md`
- `patterns.md` 过长 → `patterns-extension.md`、`patterns-testing.md`

**原文件变为索引**，只保留各子文件的主题说明和链接：

```markdown
# symbols 索引
> 文件过长已按主题拆分，按需读取对应子文件：
- [symbols-service.md](symbols-service.md) — Service 层公共类和方法
- [symbols-repository.md](symbols-repository.md) — Repository/DAO 层接口
- [symbols-domain.md](symbols-domain.md) — 领域模型和值对象
```

**查找规则**：执行任务时，先读索引文件确定主题分布，再按需加载相关子文件，不需要全量读取所有子文件。

## 依赖符号查找规则

在 `symbols.md` 中找不到某个符号时，按以下顺序继续查找：

1. 检查 `.agent/refs/deps.md` 是否有注册的依赖库
2. 若有，依次查阅 `.agent/refs/deps/{name}/symbols.md`
3. 若在 deps 中找到：使用该符号描述，并在生成的代码注释中标注来源（如 `// ref: deps/codebase-a/symbols.md`）
4. 若 deps 中也找不到：提示用户该符号可能来自未导入的依赖，建议运行 `/import-dep`

## 参考库查阅规则

实现新功能时，若当前库的 patterns.md 中没有对应模式，检查是否有参考库可以借鉴：

1. 读取 `.agent/refs/refs-registry.md`，查看是否有已注册的参考库
2. 若有，查阅 `.agent/refs/refs/{name}/patterns.md`，寻找与当前任务相关的模式
3. 若找到相关模式：将其作为实现思路的参考，在代码注释中标注来源（如 `// ref: refs/codebase-a/patterns.md#缓存装饰器`）
4. 参考库的模式是**灵感来源**，不是强制约束；若当前库技术栈不同，需要适配后使用

## 重试规则

任务失败时（编译错误、测试失败、工具报错）：

1. 解析错误信息，判断错误类型
2. 查阅 `.agent/refs/failures.md`，是否有已知解法
3. 有已知解法 → 直接应用
4. 无已知解法 → 推断根因，更新 failures.md，再重试
5. 每次重试前必须更新 refs（缺什么补什么）
6. 最多重试 **3 次**；3 次失败后，报告根因并请求用户介入

## 输出规范

- 生成代码时，注释中标注参考了哪条 refs 规则（如 `// ref: conventions.md#error-handling`）
- 静态调用链追踪结果以树形展示，每层标注文件路径和行号
- 存在动态调用（接口多态、反射）时，在对应节点标注 `[dynamic]` 风险
- 覆盖率不足时，明确列出哪些分支/路径未被覆盖
