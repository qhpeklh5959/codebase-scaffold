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
- 发现 refs 内容有误或遗漏时，**当场修正**，不等到失败才改
- `failures.md` 按错误类型归类，同类错误合并，不按时间堆积

## 依赖符号查找规则

在 `symbols.md` 中找不到某个符号时，按以下顺序继续查找：

1. 检查 `.agent/refs/deps.md` 是否有注册的依赖库
2. 若有，依次查阅 `.agent/refs/deps/{name}/symbols.md`
3. 若在 deps 中找到：使用该符号描述，并在生成的代码注释中标注来源（如 `// ref: deps/codebase-a/symbols.md`）
4. 若 deps 中也找不到：提示用户该符号可能来自未导入的依赖，建议运行 `/import-dep`

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
