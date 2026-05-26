# PaddleFleet 共享规范（与 PaddleFormers 交互部分）

## 共享约定

### 错误处理
- 前置条件检查使用 `assert`（如 `assert config is not None`）
- 运行时错误使用 `ValueError` / `RuntimeError`
- 日志使用 `logger = logging.getLogger(__name__)`

### 类型注解
- 文件头部加 `from __future__ import annotations`
- 使用 `TYPE_CHECKING` 块做仅类型检查的导入，避免循环依赖

### 导入顺序
- 标准库 → 第三方（paddle）→ paddlefleet
- PaddleFormers 对 paddlefleet 的所有导入均为条件导入（`try/except ImportError`）

### 版权头
- Apache 2.0 License，含 PaddlePaddle 和 NVIDIA 双重版权（部分文件）
