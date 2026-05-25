将另一个代码库的公共接口知识导入为本代码库的依赖描述，供 extend / trace / test-gen 使用。用法：`/import-dep <依赖代码库路径> [--name 别名] [--refresh]`

> 不合并两个知识库，只提取当前代码库**实际用到的**那部分：公共 API 符号 + 可选的共享规范。导入结果存入 `.agent/refs/deps/{name}/`，与本库 refs 隔离。

---

## Step 1：解析参数

从 `$ARGUMENTS` 中解析：

- `DEP_PATH`：依赖代码库的根目录路径（绝对路径或相对路径）
- `--name`：本库对该依赖的命名别名，默认取目录名
- `--refresh`：强制重新导入，覆盖已有的依赖描述

读取本库的 `.agent/refs/deps.md`，检查该依赖是否已导入：

- 若已存在且未指定 `--refresh`：输出当前导入信息，提示使用 `--refresh` 更新，停止
- 若已存在且指定 `--refresh`：继续，覆盖已有内容

## Step 2：扫描依赖方使用情况

在**当前代码库**（非依赖库）中，扫描对该依赖的实际使用：

用 Grep 搜索依赖库名称相关的 import / require / use 语句，找出：

- 当前库中**哪些文件**依赖了它
- 实际使用了依赖库中的**哪些类/方法**（调用点）

将这份"实际使用清单"作为后续导入的过滤依据，只导入用到的部分，不导入全部。

## Step 3：读取依赖库的知识

**首先**：检查 `{DEP_PATH}/.agent/refs/` 目录是否存在并包含已生成的知识库文件。优先使用已有知识库，避免重复分析源码。

### 情况 A：依赖库已有 refs（优先路径）

按以下顺序读取：

1. `{DEP_PATH}/.agent/refs/CODEBASE.md` — **必读**，了解依赖库的整体架构和模块划分，辅助定位符号位置
2. `{DEP_PATH}/.agent/refs/symbols.md` — **必读**，公共符号表，直接从中过滤当前库实际用到的符号

根据 Step 2 的"实际使用清单"，从 symbols.md 中**过滤**出当前库实际用到的符号，丢弃无关内容。

**读完 refs 文件后，不再扫描依赖库源码**，除非：
- symbols.md 中找不到 Step 2 发现的某个实际使用的符号（需补充查找）
- 需要确认某个符号的详细签名或行号

### 情况 B：依赖库尚无 refs

直接分析依赖库的公共 API：

1. 用 Glob 扫描依赖库源码目录
2. 提取对外暴露的接口/类/方法签名（public / export / exported 等）
3. 重点关注 Step 2 中实际被使用的符号，优先处理这部分
4. 不做完整 bootstrap，只提取接口层

## Step 4：生成依赖描述文件

在 `.agent/refs/deps/{name}/` 下创建：

### `symbols.md`（必须）

只记录当前库实际用到的符号，格式与本库 symbols.md 一致：

```markdown
# {dep-name} 公共 API（当前库使用的部分）

## 类 / 接口
- ClassName: 职责描述 | {dep-path}/src/...:行号
  - methodName(params): ReturnType — 描述 | 行号

## 注意事项
- （与本库交互时的已知约束，如：线程安全、版本要求等）
```

### `conventions.md`（可选，仅在共享规范时创建）

记录两库之间需要保持一致的约定（如：共同的错误类型、共同的日志格式），格式同本库 conventions.md。

**不导入**以下内容（与本库无关）：

- 依赖库的 patterns.md（如何扩展依赖库，本库不需要知道）
- 依赖库的 failures.md（依赖库内部的坑，本库不关心）
- 依赖库的 shadow-state.md（完全无关）

## Step 5：更新 deps.md

在本库的 `.agent/refs/deps.md` 中追加（或更新）记录：

```markdown
## {name}
- **路径**: {DEP_PATH}
- **refs 来源**: {有 refs / 从源码提取}
- **导入时间**: {今天日期}
- **当前库中的使用方**: （列出依赖它的本库文件，来自 Step 2）
- **已导入符号数**: N 个类/接口
- **共享规范**: 是 / 否
```

## Step 6：更新 failures.md（积累经验）

若导入过程中遇到问题（依赖库无 refs 且源码难以解析、符号找不到等），记录到 failures.md：

```
## [DepImportFailed / DepSymbolNotFound]
- 现象：
- 根因：
- 修正方式：
- 相关 refs 修正：
```

## 完成输出

- 导入的依赖名称和路径
- 导入了哪些符号（类/接口数量，列出名称）
- 是否导入了共享规范
- 当前库中依赖它的文件列表
- 提示：后续 `/extend`、`/trace`、`/test-gen` 遇到该依赖的符号时将自动查阅 `.agent/refs/deps/{name}/`
