分析代码库并初始化知识库。用法：`/bootstrap [代码库路径，默认为当前目录]`

---

## Step 1：确定分析目标

- 若 `$ARGUMENTS` 非空，将其作为代码库根目录
- 否则使用当前工作目录
- 读取根目录下的配置文件（pom.xml / package.json / go.mod / requirements.txt / Cargo.toml 等），确定技术栈和项目名称

## Step 2：扫描目录结构

使用 Glob 扫描文件树，完成以下识别：

- **目录分层**：哪些目录对应 core/domain/service/api/test/config 等职责
- **入口文件**：main、index、app、server 等启动点
- **测试目录**：测试文件的命名规律和存放位置
- **构建产物目录**：需要排除的 node_modules/target/build/dist

## Step 3：提取关键符号

扫描核心源码目录（非测试、非构建产物），提取：

- **扩展点**：接口（interface）、抽象类（abstract class）的名称、方法签名、文件路径
- **核心类**：关键业务类的公共方法签名（只记录签名，不复制方法体）
- **注册/工厂机制**：Factory、Registry、Provider、Container 等名称的类及其注册方式
- **依赖注入配置**：注解、配置文件、Wire/DI 容器

每个符号记录格式：`SymbolName: 一句话描述 | path/to/file.ext:行号`

## Step 4：归纳编码规范

阅读 5~10 个典型的非测试源文件，归纳：

- **命名风格**：类名（PascalCase/snake_case）、方法名、常量、私有成员前缀等
- **错误处理模式**：异常类型、错误码枚举、是否有统一错误类、返回值约定
- **包/模块组织**：同类文件放在哪个目录，是按层级还是按功能分包
- **注释风格**：JSDoc / Javadoc / godoc 等，必填字段
- **测试规范**：测试类命名（XxxTest/test_xxx）、Mock 框架、断言风格

## Step 5：识别扩展模式

重点分析：

- **添加新实现类**：找到已有的一个实现类作为示例，归纳步骤（实现接口 → 注册到工厂/容器 → 可能需要修改的配置）
- **添加新路由/Handler**：找到已有路由的注册方式
- **标准测试结构**：找到一个完整的单元测试文件，作为测试模板的参考

## Step 6：生成 refs 文件

在 `.agent/refs/` 目录下创建以下文件（若已存在则覆盖）：

### CODEBASE.md（入口文档，≤ 200 行）

```
# {项目名} 代码库概览
## 技术栈
## 目录结构及职责
## 关键入口文件（路径 + 一句话描述）
## 扩展点索引（→ 详见 symbols.md）
## 测试框架和运行命令
```

### symbols.md（关键符号索引）

```
# 关键符号索引
## 接口 / 抽象类（扩展点）
- InterfaceName: 职责 | path:行号
  - methodSignature(): 描述

## 核心类
- ClassName: 职责 | path:行号
  - publicMethod(): 描述 | 行号
```

### conventions.md（编码规范）

```
# 编码规范
## 命名规范
## 包/模块组织规则
## 错误处理规范
## 测试规范
```

### patterns.md（扩展模式）

```
# 扩展模式
## 添加新实现类
步骤 1：...
步骤 2：...
示例参考：path/to/ExistingImpl.ext

## 添加新路由
## 标准单元测试结构
```

### failures.md（初始化为空模板）

```
# 失败记录
<!-- 新增格式：
## [错误类型，如：ImportNotFound / TypeMismatch / CoverageInsufficient]
- 现象：
- 根因：
- 修正方式：
- 相关 refs 修正：（修改了哪个 refs 文件的哪条规则）
-->
```

## Step 7：分析并建议定制 commands

基于已完成的代码库分析，识别适合为该代码库定制的 commands：

从以下维度各找 1~3 个机会：
- **技术栈特定工作流**：该栈下高频但多步骤的任务（如 Spring Boot 的三层脚手架、Django 的 migration 流程）
- **扩展模式封装**：patterns.md 中步骤超过 3 步的模式，值得封装为 command
- **测试特化需求**：通用 `/test-gen` 无法覆盖的测试场景（集成测试、E2E、性能测试）
- **构建运维脚本**：常用的多步操作序列

将建议写入 `.agent/refs/command-suggestions.md`（若文件已有内容则追加，不覆盖）：

```markdown
## {name}（bootstrap 分析）
- **触发场景**: ...
- **解决的问题**: ...
- **建议的步骤摘要**: 1. ... 2. ...
- **依赖的 refs**: ...
- **状态**: 待处理
- **建议时间**: {今天日期}
```

## Step 8：输出摘要

完成后输出：
- 已生成的 refs 文件列表
- 识别到的扩展点数量（接口/抽象类）
- 识别到的核心类数量
- 测试运行命令
- 建议的定制 commands 列表（名称 + 一句话描述）
- 下一步建议（如：运行 `/gen-command --create <name>` 生成某个定制 command）
