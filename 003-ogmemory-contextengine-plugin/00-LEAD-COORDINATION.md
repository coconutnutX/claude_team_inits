# 00-LEAD-COORDINATION — 协调指引

## 你的角色

你是 Lead Coordinator。因为只有一个会话，你自己按阶段扮演 coder 和 runner 的角色。但请严格按对应阶段文档操作，不要跳步。

## 推进顺序

| Phase | 文档 | 角色 | 目标 |
|-------|------|------|------|
| Phase 1 | `01-WRITE-API-TEST.md` | coder + runner | 独立测试 oG-Memory write API，确认写入 AGFS 正常 |
| Phase 2 | `02-PLUGIN-WRITE.md` | coder + runner | 将写入接入 OpenClaw 插件的 after_turn hook |

## 每个 Phase 的工作流

```
1. 读对应的阶段文档
2. 按文档中的任务清单逐项执行
3. 每完成一项，更新 PROGRESS.md
4. 遇到问题 → PROGRESS.md 写 [ASK-HUMAN] → 停下来等人类
5. 全部通过 → 更新 PROGRESS.md 总览表为 ✅ → 进下一个 Phase
```

## PROGRESS.md 更新规范

每条记录的格式：

```markdown
### [时间] [角色] [类型] 标题
内容...
```

角色：`LEAD` / `CODER` / `RUNNER`
类型：`INFO` / `DONE` / `FAILED` / `DECISION` / `ASK-HUMAN` / `BLOCKED`

示例：

```markdown
### 14:30 [CODER] [DONE] test_write_api.py 编写完成
- 使用了 ContextFS.write_node() 路径（完整 pipeline 的 Extractor 依赖 LLM key）
- 文件位置：tests/test_write_api.py

### 14:35 [RUNNER] [DONE] 运行 test_write_api.py 通过
- AGFS 路径：/data/Workspace2/oG-Memory/data/accounts/test-account/...
- content.md 大小：342 bytes
- .meta.json status: ACTIVE ✅

### 14:40 [RUNNER] [ASK-HUMAN] LLM API key 未配置
- CandidateExtractor 需要 LLM 调用
- 环境变量 ANTHROPIC_API_KEY 未设置
- 请配置后告知
```
