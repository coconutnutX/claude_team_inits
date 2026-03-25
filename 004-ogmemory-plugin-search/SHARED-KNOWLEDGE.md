# SHARED-KNOWLEDGE — 项目背景

## 我们在做什么

oG-Memory 是面向 AI Agent 的长期记忆基础设施。我们已将它接入 OpenClaw（一个开源 AI 助手网关）作为 context engine 插件。**写入链路已跑通**，现在要接入 **search（检索）**，让 agent 对话时能自动召回相关记忆。

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
      ├─ 写入：messages → CandidateExtractor(LLM) → MergePolicy → ContextWriter → AGFS  ← ✅ 已完成
      └─ 检索：search_memory → VectorIndex → SeedHit[]  ← 🆕 本次任务
```

**关键理解：** oG-Memory 不是存原始消息，而是从消息中**提取知识**（Profile/Preference/Entity/Event/Case/Pattern/Skill 七类 ContextNode），存到 AGFS 文件系统。检索时返回的是知识摘要，非原始消息。

## 当前已完成的工作

### 架构分层（已确立）

```
index.js（Node.js，OpenClaw plugin 入口）
  │  execFileSync → python3 bridge/memory_api.py <method> <json_params>
  ▼
bridge/memory_api.py（Python，OpenClaw 适配层）
  │  - CLI 分发
  │  - lifecycle 方法实现（bootstrap/assemble/after_turn/compact/dispose）
  │  - RequestContext 构造
  │  - 调用 oG-Memory service 层
  ▼
service/api.py（Python，oG-Memory 通用 API）
  │  - commit_messages / search_memory / read_memory
  │  - 不知道 OpenClaw 的存在
  ▼
oG-Memory 内部（extract → plan → write → AGFS）
```

### 写入链路（✅ 已跑通）

- `after_turn` hook 中调用 `service/api.py` 的写入接口
- 对话消息 → CandidateExtractor → AGFS 目录出现 ContextNode 文件
- 使用真实 OpenAI LLM（通过 `OPENAI_API_KEY` 环境变量）
- Gateway 端到端测试通过

### 插件 lifecycle hooks 状态

| Hook | 状态 | 当前行为 |
|------|------|---------|
| `bootstrap` | ✅ 每次 turn 调用 | 轻量初始化 |
| `assemble` | ✅ 每次 turn 调用 | **当前只透传 messages，search 未接入** ← 本次要改 |
| `after_turn` | ✅ 每次 turn 调用 | 调用写入 pipeline |
| `dispose` | ✅ 调用正常 | 释放资源 |
| `compact` | ⚠️ 待验证 | 未实现 |
| `ingest` | ⚠️ 未触发 | 已知问题，不影响本次任务 |

### Search 在 service/api.py 中的状态

**主线已合入 search 逻辑。** `service/api.py` 中已有 `search_memory` 方法，但尚未被 `bridge/memory_api.py` 调用。

oG-Memory 检索接口（来自设计文档）：

```python
search_memory(
    query: str,
    top_k: int = 10,
    category: list[str] | None = None,
) -> list[SeedHit]
# SeedHit 包含：uri, score, abstract(≤100字), has_overview, has_content
# service 层自动注入 account_id + owner_space

read_memory(
    uri: str,
    level: Literal["L0", "L1", "L2"],
) -> str
# L0 → .abstract.md，L1 → .overview.md，L2 → content.md
```

## 关键文件位置

| 内容 | 路径 |
|------|------|
| oG-Memory 项目根 | `/data/Workspace2/oG-Memory/` |
| oG-Memory 核心文档 | `/data/Workspace2/oG-Memory/CLAUDE.md` |
| oG-Memory Service API | `/data/Workspace2/oG-Memory/service/api.py` |
| OpenClaw 插件目录 | `/data/Workspace2/oG-Memory/openclaw_context_engine_plugin/` |
| 插件 bridge 层 | `.../openclaw_context_engine_plugin/bridge/memory_api.py` |
| 插件 JS 入口 | `.../openclaw_context_engine_plugin/index.js` |
| AGFS 数据目录 | `/tmp/agfs-data` |
| AGFS 配置文件 | `~/.agfs-config.yaml` |

## 环境信息

| 项目 | 值 |
|------|-----|
| OS | Windows WSL |
| Python 环境 | conda py11 |
| LLM Provider | OpenAI（`OPENAI_API_KEY` 环境变量） |
| AGFS 服务端口 | `1833` |
| 写入路径 | `MemoryWriteAPI.commit_session()`（完整 pipeline） |
| OpenClaw 插件配置 | `api.config.plugins.entries['og-memory-context-engine'].config` |
| Python 路径配置 | `~/.openclaw/openclaw.json` 中 `pythonBin` 字段 |

## 不可违反的规则

1. **不要修改 oG-Memory 的 `core/` 包**
2. **`service/api.py` 保持通用**，不添加任何 OpenClaw 特有逻辑
3. OpenClaw 特有的适配全部在 `bridge/memory_api.py` 中完成
4. oG-Memory 的写入涉及 LLM 调用 — 确保环境中有可用的 API key
