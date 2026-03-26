# SHARED-KNOWLEDGE — 项目背景

## 我们在做什么

oG-Memory 是面向 AI Agent 的长期记忆基础设施。我们已将它接入 OpenClaw（一个开源 AI 助手网关）作为 context engine 插件。**写入和检索链路已在之前的任务中跑通**（使用 mock VectorIndex），现在主线合入了 OpenGauss 支持，需要将插件从 mock 切换到 OpenGauss 作为真实的向量存储后端。

## 当前问题（人类已观察到）

1. **数据"幽灵"问题**：通过 OpenClaw 测试可以存入记忆并召回，但 OpenGauss 数据库中没有数据 — 说明写入链路可能走了 mock/内存路径而非真实 DB
2. **`index_service` 来历不明**：`service/` 下新增了 `index_service`（或类似模块），不清楚它的用途和调用方式
3. **`outbox_store` 未被使用**：插件硬编码了写入逻辑，没有使用 `outbox_store`，而 `outbox_store` 应该是负责将数据持久化到数据库的组件
4. **配置加载问题**：`outbox_store` 本应按照整体配置来加载（可能是通过某种 DI/config 机制），但插件硬编码绕过了它
5. **整体代码混乱**：主线合了很多新代码，层次关系和调用链路不清晰

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
      ├─ 写入：messages → CandidateExtractor(LLM) → MergePolicy → ContextWriter → AGFS + ???
      ├─ 检索：search_memory → VectorIndex → SeedHit[]
      ├─ outbox_store → ??? → OpenGauss（持久化）
      └─ index_service → ???（新增，待调研）
```

**本次任务的核心疑问：**
- `outbox_store` 在写入链路中的位置是什么？
- `index_service` 和 `outbox_store` 是什么关系？
- 正确的配置加载方式是什么（不应硬编码）？
- 写入数据最终应如何流入 OpenGauss？

## 架构分层（已确立）

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
  │
  ├─ outbox_store → ???（应写入 OpenGauss）
  └─ index_service → ???（新增，待调研）
```

## 已完成的工作（前几轮任务）

| 任务 | 状态 | 内容 |
|------|------|------|
| 001 | ✅ | 初始插件脚手架（mock 阶段） |
| 002 | ✅ | Rebase 修复 |
| 003 | ✅ | 真实写入链路接通（after_turn → service/api.py → AGFS） |
| 004 | ✅ | 检索链路接通（assemble → search_memory → systemPromptAddition） |
| 005 | 🆕 | **本次任务：从 mock VectorIndex 切换到 OpenGauss** |

## 插件 lifecycle hooks 状态

| Hook | 状态 | 当前行为 |
|------|------|---------|
| `bootstrap` | ✅ 每次 turn 调用 | 轻量初始化 |
| `assemble` | ✅ 每次 turn 调用 | 调用 search_memory 注入记忆到 systemPrompt |
| `after_turn` | ✅ 每次 turn 调用 | 调用写入 pipeline |
| `dispose` | ✅ 调用正常 | 释放资源 |
| `compact` | ⚠️ 待验证 | 未实现 |
| `ingest` | ⚠️ 未触发 | 已知问题，不影响本次任务 |

## 关键文件位置

| 内容 | 路径 |
|------|------|
| oG-Memory 项目根 | `/data/Workspace2/oG-Memory/` |
| oG-Memory 核心文档 | `/data/Workspace2/oG-Memory/CLAUDE.md` |
| oG-Memory Service API | `/data/Workspace2/oG-Memory/service/api.py` |
| oG-Memory index_service | `/data/Workspace2/oG-Memory/service/index_service.py`（待确认） |
| OpenClaw 插件目录 | `/data/Workspace2/oG-Memory/openclaw_context_engine_plugin/` |
| 插件 bridge 层 | `.../openclaw_context_engine_plugin/bridge/memory_api.py` |
| 插件 JS 入口 | `.../openclaw_context_engine_plugin/index.js` |
| AGFS 数据目录 | `/tmp/agfs-data` |
| AGFS 配置文件 | `~/.agfs-config.yaml` |
| OpenGauss 连接信息 | `~/.bashrc`（末尾的环境变量） |

## 环境信息

| 项目 | 值 |
|------|-----|
| OS | Windows WSL |
| Python 环境 | conda py11 |
| LLM Provider | OpenAI（`OPENAI_API_KEY` 环境变量） |
| AGFS 服务端口 | `1833` |
| OpenGauss | 已下载并启动，连接信息见 `~/.bashrc` 末尾 |
| OpenClaw 插件配置 | `~/.openclaw/openclaw.json` |

## 不可违反的规则

1. **不要修改 oG-Memory 的 `core/` 包**
2. **`service/api.py` 保持通用**，不添加任何 OpenClaw 特有逻辑
3. OpenClaw 特有的适配全部在 `bridge/memory_api.py` 中完成
4. oG-Memory 的写入涉及 LLM 调用 — 确保环境中有可用的 API key
5. **outbox_store 应按照 oG-Memory 整体配置机制加载**，不应在插件中硬编码
6. 测试文件放在 `tests/`（unit + integration），不放在插件目录中
7. 独立脚本放在 `scripts/`，不混在 `tests/` 中
8. 团队文件统一放 `.claude-team/`
