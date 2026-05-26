# 参考代码库注册表

本文件由 `/ref-from` 维护，记录当前代码库参考的外部代码库及其导入状态。

参考库与当前库**没有调用关系**，只提供实现模式和架构思路。若需要导入有调用关系的依赖库，使用 `/import-dep`。

每个参考库的详细模式存放在 `.agent/refs/refs/{name}/patterns.md`。

---

<!-- 新增格式：
## {name}
- **路径**: 参考库根目录路径
- **refs 来源**: 有 refs（直接读取）| 从源码提取
- **导入时间**: YYYY-MM-DD
- **聚焦领域**: 全量 | 缓存 / 认证 / 错误处理 等
- **已提取模式数**: N 条
- **有价值的领域**: （列出）
-->

## transformers
- **路径**: /root/paddlejob/share-storage/gpfs/system-public/qinhuapeng/transformers
- **refs 来源**: 从源码提取（无 .agent/refs/）
- **导入时间**: 2026-05-25
- **聚焦领域**: 具体模型实现 + 两边接口对应关系（用于模型迁移）
- **已提取模式数**: 7 条
- **有价值的领域**:
  - 模型目录结构（与 PaddleFormers 完全对应）
  - Config 字段与默认值（直接参考迁移）
  - 模型类层次结构差异（`PreTrainedModel` 三层 vs 两层）
  - torch→paddle 层级替换关系（nn.Linear → GeneralLinear 等）
  - Attention 实现接口差异（`get_interface` vs `[]` 访问）
  - Modular 文件（快速理解新模型架构来源）
  - chat_template 位置（tokenizer_config.json 中）
