# PROGRESS — 005 oG-Memory 插件接入 OpenGauss

## 总览

| Phase | 状态 | 内容 |
|-------|------|------|
| Phase 1 | ⬜ | 代码仓分析：index_service / outbox_store / 写入链路 |
| Phase 2 | ⬜ | 插件修复接入 OpenGauss |
| Phase 3 | ⬜ | E2E 测试（OpenClaw CLI） |

## 关键发现

（Phase 1 完成后填充）

### OpenGauss 连接参数

| 参数 | 值 |
|------|-----|
| host | ??? |
| port | ??? |
| user | ??? |
| password | ??? |
| database | ??? |

### 写入链路完整调用图

```
after_turn(params)
  → bridge/memory_api.py::after_turn()
    → service/api.py::???()
      → ???
      → ???
      → ??? → AGFS
      → ??? → OpenGauss
```

### index_service 用途

- 定义位置：???
- 功能：???
- 被谁调用：???
- 和 VectorIndex 的关系：???
- 和 outbox_store 的关系：???

### outbox_store 用途和加载方式

- 定义位置：???
- 功能：???
- 正确加载方式：???
- 当前插件是否使用：???
- 不使用导致的后果：???

### 当前插件问题清单

1. ???
2. ???
3. ???

---

## 记录区

（按时间顺序追加）
