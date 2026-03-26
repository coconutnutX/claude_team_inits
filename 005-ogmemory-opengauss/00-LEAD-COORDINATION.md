# 00-LEAD-COORDINATION — 协调指引

## 你的角色

你是 Team Lead。进入 delegate mode，根据每个 Phase 的需要 spawn 对应的 agent 来执行具体工作。

**你自己负责：**
- 读文档、制定计划
- 审查 agent 的执行结果
- 更新 PROGRESS.md
- 做 go/no-go 决策
- 遇到问题写 [ASK-HUMAN] 等待人类

**你 spawn 的 agent 负责：**
- 执行具体的编码、运行测试、排查问题
- 将结果汇报给你

**你不写代码，不跑命令。** 你只协调、审查、记录。

## 推进顺序

| Phase | 文档 | spawn agent | 目标 |
|-------|------|------------|------|
| Phase 1 | `01-CODEBASE-ANALYSIS.md` | investigator | 分析代码仓状态，搞清楚 index_service、outbox_store、写入链路的关系 |
| Phase 2 | `02-PLUGIN-FIX-OPENGAUSS.md` | coder + runner | 修复插件，接上 OpenGauss，按正确方式使用 outbox_store |
| Phase 3 | `03-E2E-TESTING.md` | runner | 通过 openclaw CLI 端到端测试：写入 → 召回 → 验证 AGFS 和 OpenGauss 都有数据 |

**严格按序推进，每个 Phase 完成后才进下一个。**

## Phase 间的依赖关系

```
Phase 1 输出：
  → 写入链路完整调用图（从 after_turn 到 OpenGauss 的每一步）
  → index_service 的用途和调用方式
  → outbox_store 的用途和正确加载方式
  → 当前插件的问题清单（哪些地方硬编码了、哪些该用配置加载）
  → OpenGauss 连接参数

Phase 2 依赖 Phase 1 的输出做修改

Phase 3 依赖 Phase 2 的修改做测试
```

## 每个 Phase 的工作流

```
1. 读对应的阶段文档
2. spawn 合适的 agent，给它任务清单 + Phase 1 输出的关键信息
3. 审查 agent 返回的结果
4. 更新 PROGRESS.md
5. 遇到问题 → PROGRESS.md 写 [ASK-HUMAN] → 停下来等人类
6. 全部通过 → 更新 PROGRESS.md 总览表为 ✅ → 进下一个 Phase
```

## PROGRESS.md 格式

### 总览表（文件顶部）

```markdown
## 总览

| Phase | 状态 | 内容 |
|-------|------|------|
| Phase 1 | ⬜/🔄/✅/❌ | 代码仓分析 |
| Phase 2 | ⬜/🔄/✅/❌ | 插件修复接入 OpenGauss |
| Phase 3 | ⬜/🔄/✅/❌ | E2E 测试 |

## 关键发现

（Phase 1 完成后填充）

### 写入链路调用图
...

### index_service 用途
...

### outbox_store 用途和加载方式
...

### OpenGauss 连接参数
...

### 当前插件问题清单
...
```

### 记录区格式

每条记录：

```markdown
### [时间] [角色] [类型] 标题
内容...
```

角色：`LEAD` / `INVESTIGATOR` / `CODER` / `RUNNER`
类型：`INFO` / `DONE` / `FAILED` / `DECISION` / `ASK-HUMAN` / `BLOCKED`

## 重要经验（来自上一轮任务）

### 文件组织
- 测试文件放 `tests/`（unit + integration），**不放插件目录**
- 独立脚本放 `scripts/`，**不放 tests/**
- 团队文件放 `.claude-team/`（PROGRESS.md 等）

### 测试设计
- **Pytest 测试**：函数名 `test_xxx()`，用 `assert` 验证，用 `pytest` 运行
- **独立脚本**：函数名 `run_xxx()` / `main()`，放 `scripts/`，用 `python` 运行
- **不要混在一起**，不要让 test 函数返回 True/False 或只 print 不 assert
- 不需要 `if __name__ == "__main__"` 在测试文件中

### 降级策略
- search 失败不阻塞对话（try/except + passthrough）
- 辅助功能失败不应阻断主流程

### OpenClaw 消息格式
- `content` 字段可能是字符串或结构化列表，需要用 `_extract_content_text()` 处理

### 进度文件
- **必须放在 `.claude-team/PROGRESS.md`**，不放项目根目录
