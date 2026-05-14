静态追踪一个方法的完整调用链（不执行代码）。用法：`/trace <方法全限定名，如 UserService.createUser 或 createUser>`

---

## Step 1：解析目标

从 `$ARGUMENTS` 中解析目标方法名，可能的格式：

- `ClassName.methodName` — 指定类和方法
- `methodName` — 仅方法名，需在 symbols.md 中查找所属类
- `path/to/File.ext:行号` — 精确定位

从 `.agent/refs/symbols.md` 中查找目标方法的文件路径和行号。若找不到，使用 Grep 在代码库中搜索方法定义。

## Step 2：加载上下文

读取：

1. `.agent/refs/CODEBASE.md` — 了解模块边界，辅助判断调用方向
2. `.agent/refs/symbols.md` — 快速定位被调用方法的位置

## Step 3：追踪调用链（向下：callee 方向）

从目标方法出发，递归追踪它调用了哪些方法：

1. 读取目标方法所在文件，解析方法体内的所有方法调用
2. 对每个调用：
   - 先在 symbols.md 中查找定义位置
   - 若未找到，检查 `.agent/refs/deps/` 下各依赖的 symbols.md（调用可能跨库）
   - 若在 deps 中找到：在树节点标注 `[dep: {name}]`，展示 deps/symbols.md 中的描述，不继续递归进入依赖库内部
   - 若仍未找到，使用 Grep 在本库代码中搜索 `def methodName` / `func methodName` / `methodName(` 等模式
   - 读取被调用方法，继续递归
3. **深度限制**：默认最多追踪 5 层，超过时标注 `[depth limit]`
4. **递归检测**：检测到循环调用时，标注 `[recursive]` 并停止该分支

### 动态调用识别

遇到以下情况时，标注 `[dynamic]` 并说明原因，不继续追踪该分支：

- 接口调用（调用的是接口方法，存在多个实现）
- 反射调用（`Class.forName` / `getattr` / `reflect.Value` 等）
- 回调/函数变量（`callback()` / `fn()` / `handler()` 等）
- 动态代理（`Proxy.newProxyInstance` / `@Transactional` 等 AOP 场景）

## Step 4：追踪调用链（向上：caller 方向，可选）

若用户在 $ARGUMENTS 中包含 `--callers` 标志，额外追踪谁调用了该方法：

使用 Grep 搜索目标方法名的调用点，列出所有调用方（不递归，只列一层）。

## Step 5：输出调用树

以树形结构输出，每节点格式：

```
方法名 (文件路径:行号)
├── 调用的方法A (文件路径:行号)
│   ├── 方法A1 (文件路径:行号)
│   └── [dynamic] 接口方法B → 可能实现: ImplX, ImplY (symbols.md)
├── 调用的方法C (文件路径:行号)
│   └── [recursive] → 回到 方法名
└── 调用的方法D (文件路径:行号)
    └── [depth limit] 超过追踪深度
```

## Step 6：输出摘要

- 调用链总深度
- 经过的模块边界列表（跨文件/跨包的位置）
- `[dynamic]` 风险点列表（需要人工确认的动态调用）
- 建议关注的关键路径（如：涉及 IO、外部服务、锁等的节点）

## 注意事项

- 这是**纯静态分析**，结果基于代码文本，不保证运行时完全一致
- 接口多态场景下，静态分析只能确定接口类型，无法确定运行时实现
- 所有 `[dynamic]` 标注处建议结合运行时日志或断点调试确认
