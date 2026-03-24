# SHARED-KNOWLEDGE — 项目背景

## 我们在做什么

oG-Memory 是面向 AI Agent 的长期记忆基础设施。我们正在将它接入 OpenClaw（一个开源 AI 助手网关）作为 context engine 插件，让 OpenClaw 的对话消息能自动沉淀为结构化知识。

## 两个系统的关系

```
OpenClaw Gateway（对话层）
  │
  ├─ Session Manager：存原始消息（.jsonl 文件）
  │
  └─ Context Engine Plugin（我们写的桥接层）
        │
        ▼
    oG-Memory（知识层）
      ├─ 写入：messages → CandidateExtractor(LLM) → MergePolicy → ContextWriter → AGFS
      └─ 检索：search_memory → VectorIndex → SeedHit[]
```

**关键理解：** oG-Memory 不是存原始消息，而是从消息中**提取知识**（Profile/Preference/Entity/Event/Case/Pattern/Skill 七类 ContextNode），存到 AGFS 文件系统。

## 当前状态

- OpenClaw context engine 插件的 **mock 版本已完成并验证**
  - bootstrap / assemble / after_turn / dispose 四个 hook 已验证可触发
  - ingest 未触发（已知问题，不影响本次任务）
  - 插件位置：`/data/Workspace2/oG-Memory/openclaw_context_engine_plugin/`
- oG-Memory 的 **write pipeline 已实现**，read/search 尚未实现
- 本次任务目标：**验证 write API → 接入插件 after_turn**

## 关键文件位置

| 内容 | 路径 |
|------|------|
| oG-Memory 项目根 | `/data/Workspace2/oG-Memory/` |
| oG-Memory 核心文档 | `/data/Workspace2/oG-Memory/CLAUDE.md` |
| OpenClaw 插件 | `/data/Workspace2/oG-Memory/openclaw_context_engine_plugin/` |
| 插件 bridge 层 | `.../openclaw_context_engine_plugin/bridge/memory_api.py` |
| 插件引擎入口 | `.../openclaw_context_engine_plugin/mock_engine.py` |
| AGFS 数据目录 | 需要确认（检查 oG-Memory 配置） |

## 不可违反的规则

1. **不要修改 oG-Memory 的 `core/` 包** — Phase 0 已冻结
2. oG-Memory 的写入涉及 LLM 调用 — 确保环境中有可用的 API key
3. 如果完整 pipeline（CandidateExtractor）未就绪，降级到 ContextFS.write_node() 直接写入
