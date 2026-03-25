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

## 推进顺序

| Phase | 文档 | spawn agent | 目标 |
|-------|------|------------|------|
| Phase 0 | `00-REBASE-FIX.md` | investigator | 跑 oG-Memory/tests 下所有测试，暴露 rebase 问题，[ASK-HUMAN] 等人类修 |
| Phase 1 | `01-SEARCH-API-TEST.md` | investigator + runner | 独立测试 oG-Memory search API，确认检索可用 |
| Phase 2 | `02-PLUGIN-SEARCH.md` | coder + runner | 将 search 接入插件 assemble hook，实现自动记忆注入 |

**严格按序推进，每个 Phase 完成后才进下一个。**

## 每个 Phase 的工作流

```
1. 读对应的阶段文档
2. spawn 合适的 agent，给它任务清单
3. 审查 agent 返回的结果
4. 更新 PROGRESS.md
5. 遇到问题 → PROGRESS.md 写 [ASK-HUMAN] → 停下来等人类
6. 全部通过 → 更新 PROGRESS.md 总览表为 ✅ → 进下一个 Phase
```

## PROGRESS.md 更新规范

每条记录的格式：

```markdown
### [时间] [角色] [类型] 标题
内容...
```

角色：`LEAD` / `INVESTIGATOR` / `CODER` / `RUNNER`
类型：`INFO` / `DONE` / `FAILED` / `DECISION` / `ASK-HUMAN` / `BLOCKED`

示例：

```markdown
### 14:00 [INVESTIGATOR] [FAILED] oG-Memory 测试失败报告
- 通过: 12 / 失败: 3 / 错误: 1
- test_search.py::test_vector_search — ImportError: No module named 'xxx'
- test_write.py::test_merge_policy — AssertionError: expected 2 got 0

### 14:05 [LEAD] [ASK-HUMAN] 测试失败需要人类确认
- 3 个失败疑似 rebase 导致，请确认修复方式
- 1 个错误是环境依赖（需要 AGFS 服务），可跳过

### 15:00 [INVESTIGATOR] [DONE] 修复后重跑测试全部通过
- 通过: 16 / 失败: 0

### 15:30 [CODER] [DONE] test_search_api.py 编写完成
- 调用 search_memory() 接口，返回 SeedHit[]
- 文件位置：tests/test_search_api.py
```
