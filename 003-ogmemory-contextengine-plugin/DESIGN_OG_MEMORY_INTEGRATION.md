# oG-Memory × OpenClaw Context Engine 集成设计文档

> **状态：** Draft v4
> **日期：** 2026-03-24
> **相关文档：**
> - [OpenClaw Context Engine 概念文档](https://docs.openclaw.ai/concepts/context-engine)
> - [OpenClaw Session Management 文档](https://docs.openclaw.ai/concepts/session)
> - [OpenClaw Plugin Manifest 规范](https://docs.openclaw.ai/plugins/manifest)
> - [OpenClaw Compaction 文档](https://docs.openclaw.ai/concepts/compaction)
> - oG-Memory CLAUDE.md（ContextEngine v3.1）

---

## 1. 背景

当前已完成 OpenClaw context engine 插件的接口打通（mock 阶段），验证了以下生命周期 hook 可正常触发：

| Hook | 状态 | 说明 |
|------|------|------|
| `bootstrap` | ✅ 已验证 | **每次 turn 都会调用**（非文档描述的"仅首次"） |
| `assemble` | ✅ 已验证 | 每次模型调用前构建 context |
| `after_turn` | ✅ 已验证 | 模型回复完成后调用 |
| `dispose` | ✅ 已验证 | 每次 turn 结束 / gateway 关闭时调用 |
| `compact` | ⚠️ 待验证 | context 溢出时自动触发 |
| `ingest` | ⚠️ 未触发 | 设计上按文档作为消息接收入口，实测未触发列入已知问题（见附录 A） |

下一步需要将 mock 实现替换为 oG-Memory 的真实后端。本文档定义 plugin 与 oG-Memory 之间的集成方案。

---

## 2. 架构总览

### 2.1 两个系统的职责边界

oG-Memory **不是简单的消息存储**。它存储的是从消息中**提取出的知识**（Profile、Preference、Entity、Event、Case、Pattern、Skill 七类 ContextNode），写入路径是一条完整的 pipeline：

```
OpenClaw Gateway
  │
  ├─ Session Manager（OpenClaw 内置）
  │   ├─ 路由：消息来源 → sessionKey → sessionId
  │   ├─ 持久化：原始消息写入 .jsonl 文件
  │   └─ 加载：读取 .jsonl 构建 messages 数组传给 context engine
  │
  ├─ Context Engine Plugin（本插件，bridge 层）
  │   ├─ bootstrap()  ──► 轻量初始化
  │   ├─ ingest()     ──► 接收消息（缓存，不立即触发 extract）
  │   ├─ assemble()   ──► 调 oG-Memory search_memory(L0) → 注入 systemPromptAddition
  │   ├─ compact()    ──► 触发知识抽取 pipeline（待讨论，见第 7 章）
  │   ├─ after_turn() ──► 触发知识抽取 pipeline（待讨论，见第 7 章）
  │   └─ dispose()    ──► 释放资源
  │         │
  │         ▼
  │
  └─ oG-Memory（知识基础设施）
      │
      ├─ 写入 pipeline（重量级，涉及 LLM 调用）：
      │   messages → CandidateExtractor（LLM 抽取 7 类知识）
      │           → MergePolicy（create / merge / append / skip）
      │           → ContextWriter（写入 AGFS：content.md + .abstract.md + .overview.md + .meta.json）
      │           → OutboxEvent → 异步 Embedder + VectorIndex upsert
      │
      ├─ 检索（轻量级）：
      │   search_memory(query) → VectorIndex L0 召回 → SeedHit[]
      │   read_memory(uri, L1) → 读 .overview.md
      │   read_memory(uri, L2) → 读 content.md
      │
      └─ 隔离：RequestContext(account_id, user_id, agent_id, session_id)
```

**核心区别：** lossless-claw 把每条原始消息存进 SQLite，检索时召回原始消息。oG-Memory 对消息做**知识抽取**后存储结构化知识节点，检索时召回的是知识摘要而非原始消息。因此写入路径涉及 LLM 调用，不适合在每条消息到达时同步执行。

### 2.2 OpenClaw Session Manager 与 sessionId 机制

Session Manager 是 OpenClaw gateway 进程内的模块，负责消息的路由、持久化和加载。

**存储结构：**

```
~/.openclaw/agents/<agentId>/sessions/
├── sessions.json              # SessionStore：sessionKey → SessionEntry 的映射表
├── <sessionId>.jsonl           # 对话记录（每条消息一行 JSON）
└── ...
```

**sessionKey 路由机制：** 消息进入 gateway 时，Session Manager 根据来源信息（channel + sender + group 等）确定性地生成 sessionKey，再查 `sessions.json` 得到 sessionId：

```
WhatsApp DM from +1234567890
  → sessionKey = "agent:main:whatsapp:dm:+1234567890"  (dmScope=per-channel-peer)
  → sessionId  = "9c23e8b6-4477-4311-a40a-4e7d32bbc763"

Discord #general
  → sessionKey = "agent:main:discord:channel:123456"
  → sessionId  = "f133247e-dcce-4b31-9f4b-21466f176806"
```

**用户隔离：** 通过 `session.dmScope` 配置控制：

| dmScope | 隔离粒度 | 适用场景 |
|---------|---------|---------|
| `main`（默认） | 所有 DM 共享一个 session | 单用户个人助手 |
| `per-peer` | 按发送者隔离 | 多用户但不区分 channel |
| `per-channel-peer` | 按 channel + 发送者隔离 | 推荐的多用户配置 |
| `per-account-channel-peer` | 按账号 + channel + 发送者隔离 | 多账号多用户 |

### 2.3 sessionId → RequestContext 映射

oG-Memory 使用 `RequestContext(account_id, user_id, agent_id, session_id, trace_id)` 做多租户隔离。plugin 收到的 OpenClaw `sessionId` 需要映射到 RequestContext。

映射规则待讨论（见第 7 章 D1）。

### 2.4 关于 `ingest` 的设计

按照 OpenClaw 文档，`ingest` 是 context engine 接收新消息的入口。但由于 oG-Memory 的写入涉及 LLM 抽取，不应在 `ingest` 中同步执行完整 pipeline。`ingest` 仅做消息缓存，知识抽取在其他时机触发（见第 7 章 D2）。

> ⚠️ **已知问题：** 当前 mock 阶段实测中 `ingest` 未被 runtime 调用（详见附录 A）。

### 2.5 关于 `bootstrap` 每次 turn 都调用的说明

文档描述 bootstrap "called once when the engine first sees a session"，但实测中 **每次 turn 都会调用**。因此 bootstrap 中不应做重量级操作，只做轻量的状态标记。

---

## 3. Plugin 与 oG-Memory 的接口契约

Plugin 通过 bridge 层调用 oG-Memory 的 service 层接口。与之前版本不同，**plugin 不自己定义 write/search 抽象，而是直接对接 oG-Memory 已有的接口**。

### 3.1 写入：oG-Memory 的完整 pipeline

Plugin 将一批消息交给 oG-Memory 的 service 层，由 oG-Memory 内部完成：

```
messages (OpenClaw 格式)
  → CandidateExtractor.extract(messages, ctx)    # LLM 抽取 → CandidateMemory[]
  → MergePolicy.plan(candidate, ctx)             # 决定 create/merge/append/skip → WritePlan
  → ContextWriter.write(plan, ctx)               # 写入 AGFS（content.md + .abstract.md + .overview.md + .meta.json）
  → OutboxEvent                                  # 异步 → Embedder + VectorIndex upsert
```

**plugin 侧的调用：**

```python
# bridge/memory_api.py

def extract_and_commit(
    messages: list[dict],
    ctx: RequestContext,
) -> dict:
    """
    将 OpenClaw 消息交给 oG-Memory 的写入 pipeline。
    oG-Memory 内部完成 extract → plan → write → outbox 全流程。
    """
    # 调用 oG-Memory service 层
    ...
```

**性能特征：** 涉及 LLM 调用（CandidateExtractor），单次耗时可能在秒级。不适合同步阻塞。

### 3.2 检索：search_memory + read_memory

oG-Memory 已有两个检索接口，plugin 直接调用：

```python
# oG-Memory service 层已有接口

search_memory(
    query: str,
    top_k: int = 10,
    category: list[str] | None = None,
) -> list[SeedHit]
# SeedHit 包含：uri, score, abstract(≤100字), has_overview, has_content
# service 层自动注入 account_id + owner_space，调用方不可覆盖

read_memory(
    uri: str,
    level: Literal["L0", "L1", "L2"],
) -> str
# L0 → .abstract.md，L1 → .overview.md，L2 → content.md
# service 层校验 uri 的 account 前缀
```

**plugin 侧的使用：**

| 场景 | 调用 | 返回 | 性能要求 |
|------|------|------|---------|
| `assemble` 自动注入 | `search_memory(query)` | SeedHit[]（uri + abstract） | **< 100ms**，每次 turn 都跑 |
| agent search tool（标准） | `search_memory(query)` → `read_memory(uri, L1)` | overview 文本 | 按需，宽松 |
| agent search tool（深度） | `search_memory(query)` → `read_memory(uri, L2)` | 完整 content | 按需，宽松 |

### 3.3 健康检查

```python
def memory_health() -> dict
```

**调用时机：** `openclaw doctor` 诊断时。不在 `bootstrap` 中调用。

---

## 4. 各生命周期阶段的集成方案

### 4.1 `bootstrap` — 轻量初始化

> ⚠️ **实测：每次 turn 都会调用。** 此处只做极轻量的操作。

```
bootstrap(sessionId)
  └─ 在内存中标记 sessionId 为 "active"
     不做任何 I/O 操作
```

### 4.2 `ingest` — 消息缓存（不触发 extract）

```
ingest(sessionId, message, isHeartbeat)
  │
  ├─ 如果 isHeartbeat=true → 跳过
  │
  └─ 将 message 追加到内存缓冲区（按 sessionId 分组）
     不调用 oG-Memory，不触发 LLM
```

**设计说明：** oG-Memory 的写入 pipeline 涉及 LLM 调用，不适合在每条消息到达时同步执行。`ingest` 仅做缓存，知识抽取的触发时机见第 7 章 D2。

> ⚠️ **已知问题：** 当前 ingest 未被 runtime 调用（见附录 A）。

### 4.3 `assemble` — 构建模型 context（核心，轻量级）

```
assemble(sessionId, messages[], tokenBudget)
  │
  ├─ 1. 构建 RequestContext（sessionId → account_id 映射，见第 7 章 D1）
  │
  ├─ 2. 从 messages 提取最近 N 条 user 消息拼接为 query
  │
  ├─ 3. search_memory(query, top_k=5)
  │   └─ 返回 SeedHit[]：每条含 uri + abstract(≤100字) + score
  │
  ├─ 4. 构建 context：
  │   ├─ 保留最近 K 条消息（短期记忆，直接从 messages 取）
  │   ├─ 将 SeedHit 的 abstract 拼成 systemPromptAddition（长期记忆提示）
  │   │   示例：
  │   │   "[oG-Memory] 相关记忆：
  │   │    - [profile] 用户是一名后端工程师，偏好 Python
  │   │    - [entity] SQLite：用户正在将数据库从 PostgreSQL 迁移到 SQLite
  │   │    - [preference] 代码风格：偏好函数式编程
  │   │    如需详细信息，可使用 search_memory / read_memory 工具。"
  │   └─ 确保总 token 不超过 budget
  │
  └─ 5. 返回 AssembleResult
```

**关键：assemble 只做检索，不做写入。** `search_memory` 是纯粹的向量查询 + 读 `.abstract.md`，延迟可控。

### 4.4 `compact` — 压缩旧 context + 可能触发知识抽取

```
compact(sessionId, force)
  │
  ├─ 方案 A（推荐初期）：ownsCompaction=false
  │   └─ 调用 delegateCompactionToRuntime() 复用 OpenClaw 内置 compact
  │
  └─ 方案 B（后续迭代）：ownsCompaction=true
      ├─ 将被压缩的旧消息交给 oG-Memory extract_and_commit()
      │  （compact 本身就是处理旧消息的时机，适合做知识抽取）
      └─ 返回 {ok: true, compacted: true}
```

compact 作为知识抽取触发点的可能性见第 7 章 D2。

### 4.5 `after_turn` — 后处理

```
after_turn(sessionId)
  │
  ├─ 1. 检查内存缓冲区中是否有未处理的消息
  │
  └─ 2. 触发知识抽取（待讨论，见第 7 章 D2）：
      ├─ 方案 A：在此处异步触发 extract_and_commit()
      ├─ 方案 B：不在此处触发，留给 compact
      └─ 方案 C：标记为"待处理"，由独立后台任务消费
```

### 4.6 `dispose` — 释放资源

```
dispose()
  └─ 清理内存缓冲区、session 状态标记、连接池（如有）
```

---

## 5. Agent 侧的 Search Tool

除了 plugin 在 `assemble` 中自动注入 L0（abstract）外，还应给 agent 提供工具，按需获取更详细的记忆。

oG-Memory 已有两个 service 层接口，直接暴露为 agent tool：

```
工具 1：search_memory(query, top_k, category)
  → 返回 SeedHit[]（uri + abstract + score + has_overview + has_content）
  → 等同于 L0 召回，但 agent 可以看到 uri 和 has_overview/has_content 标记

工具 2：read_memory(uri, level)
  → level=L1 → 读 .overview.md（主题 + 范围 + 适用边界）
  → level=L2 → 读 content.md（完整正文）
```

**协作模式：**

```
用户："之前我们讨论的数据库迁移方案是什么？"

┌─ assemble 自动注入 L0：
│  "[oG-Memory] 相关记忆：
│   - [entity] SQLite：用户正在将数据库从 PostgreSQL 迁移到 SQLite"
│
├─ agent 看到提示，决定需要更多细节
│  → search_memory("数据库迁移") → 拿到 uri
│  → read_memory(uri, L1) → 拿到 overview（迁移原因、方案对比、决策结论）
│
└─ 如果还需要原始细节：
   → read_memory(uri, L2) → 拿到完整 content
```

**与 lossless-claw 工具的对比：**

| lossless-claw | oG-Memory | 差异 |
|---------------|-----------|------|
| `lcm_grep`（全文搜索原始消息） | `search_memory`（向量搜索知识节点） | oG-Memory 搜索的是提取后的知识，非原始消息 |
| `lcm_describe`（按 ID 查看 summary） | `read_memory(uri, L1)` | 类似，查看概述 |
| `lcm_expand_query`（子 agent 展开 DAG） | `read_memory(uri, L2)` | oG-Memory 直接读 content，不需要 DAG 展开 |

---

## 6. oG-Memory 需要对齐的能力

oG-Memory 现有的 service 层接口（`search_memory`、`read_memory`、CandidateExtractor pipeline）已基本满足需求。bridge 层做参数格式适配即可。

| 对接点 | oG-Memory 侧 | Plugin 侧（bridge） | 需要做的事 |
|--------|-------------|---------------------|-----------|
| 写入 | service 层接收 messages + RequestContext → 完整 pipeline | 将 OpenClaw 消息格式转为 oG-Memory 期望的 messages 格式 | 确认 messages 格式兼容性 |
| L0 检索 | `search_memory(query)` → SeedHit[] | assemble 中调用，将 abstract 拼成 systemPromptAddition | 确认 L0 延迟满足 < 100ms |
| L1/L2 读取 | `read_memory(uri, level)` → str | 暴露为 agent tool | 直接对接 |
| 隔离 | RequestContext 做多租户隔离 | sessionId → RequestContext 映射 | **需要定义映射规则（见第 7 章 D1）** |

---

## 7. 关键设计决策（待讨论）

### D1：sessionId → RequestContext 的映射规则

oG-Memory 使用 `RequestContext(account_id, user_id, agent_id, session_id, trace_id)` 做多租户隔离和数据归属。OpenClaw 传给 plugin 的只有一个 `sessionId`（UUID 格式）。

**需要回答：**
- 一个 OpenClaw session 对应 oG-Memory 的哪个 account / user / agent？
- 是否需要在 plugin 配置中静态指定 account_id / user_id / agent_id？
- 还是从 OpenClaw 的 sessionKey（如 `agent:main:whatsapp:dm:+1234567890`）中解析出来？
- 单用户部署场景下能否简化为固定值？

**可能的方案：**

| 方案 | 做法 | 优点 | 缺点 |
|------|------|------|------|
| A. 静态配置 | plugin config 中写死 account_id / user_id / agent_id | 简单，单用户场景够用 | 多用户时不隔离 |
| B. 从 sessionKey 解析 | 解析 `agent:<agentId>:<channel>:dm:<peerId>` 中的各字段 | 自动多用户隔离 | 需要 OpenClaw 暴露 sessionKey（当前 plugin 只收到 sessionId） |
| C. 查询 OpenClaw gateway | 通过 `sessions.get` API 查 sessionId 对应的 sessionKey 和元数据 | 最完整 | 额外 API 调用，增加延迟 |

### D2：知识抽取（extract pipeline）在哪个生命周期触发

oG-Memory 的写入 pipeline 涉及 LLM 调用（CandidateExtractor），单次耗时可能在秒级甚至更长。需要选择合适的触发时机。

**各 hook 的特征：**

| Hook | 调用频率 | 是否可以慢 | 适合做 extract？ |
|------|---------|-----------|----------------|
| `ingest` | 每条消息 | ❌ 不应阻塞消息接收 | ❌ 不适合 |
| `assemble` | 每次 turn | ❌ 阻塞模型调用 | ❌ 不适合 |
| `after_turn` | 每次 turn 结束后 | ⚠️ 不阻塞用户，但每次都跑可能太频繁 | ⚠️ 可能 |
| `compact` | context 溢出时 | ✅ 本身就是处理旧消息的时机 | ✅ 适合 |
| 独立后台任务 | 异步 | ✅ 完全解耦 | ✅ 适合 |

**可能的方案：**

| 方案 | 做法 | 优点 | 缺点 |
|------|------|------|------|
| A. after_turn 异步触发 | 每次 turn 结束后，异步启动 extract（不等待完成） | 及时性好，新知识很快可检索 | 每条消息都触发 LLM，成本高；after_turn 也是每次 turn 都调用 |
| B. compact 时批量处理 | compact 被触发时，将被压缩的旧消息批量交给 extract | 天然批量，成本可控；compact 本身就在处理旧消息 | 及时性差，新知识要等 compact 才可检索 |
| C. 混合：after_turn 标记 + 后台消费 | after_turn 把消息标记为"待处理"，独立后台 worker 定期消费 | 完全解耦，可控频率和批量大小 | 需要额外的后台任务调度机制 |
| D. 按消息数量触发 | 累积 N 条消息后触发一次 extract | 平衡及时性和成本 | 需要维护计数器状态 |

### D3：assemble 中 L0 search 的范围和策略

**需要回答：**
- `search_memory` 的 category 过滤：是否默认搜索所有类型（profile + preference + entity + event + ...），还是只搜特定类型？
- 跨 session 范围：oG-Memory 的隔离是按 account/user 的，同一用户不同 session 的记忆天然共享。这是期望的行为吗？
- top_k 设多少？SeedHit 的 abstract（≤100 字）× top_k 条 ≈ systemPromptAddition 的 token 开销。top_k=5 约 500 字 ≈ 125 token，可接受。
- 是否需要 score 阈值？低相关度的记忆注入反而是噪音。

### D4：其他待讨论项

1. **`ownsCompaction` 设为 `true` 还是 `false`？**
   - `true` 可以在 compact 中触发知识抽取（与 D2 方案 B 配合）
   - `false` 初期更省事，但 compact 时机不受控

2. **search tool 由谁注册？**
   - 方案 A：由本 plugin 通过 `api.registerTool()` 注册
   - 方案 B：oG-Memory 作为独立的 OpenClaw memory plugin 提供 search tool

---

## 8. 接入步骤建议

```
Phase 1 — 写入打通（oG-Memory 已实现 write，尚未实现 read）
├─ 确定 sessionId → RequestContext 映射规则（D1，初期可用静态配置）
├─ 独立测试 oG-Memory write API（不经过插件，直接调用验证写入文件）
├─ bridge 层对接 extract_and_commit pipeline
├─ 在选定的 hook 中触发写入（D2，初期可用 after_turn 异步）
├─ 排查 ingest 未触发的问题（见附录 A）
├─ 验证：对话消息 → extract → AGFS 目录出现对应 ContextNode 文件
└─ 预期：对话产生的新知识自动沉淀到 oG-Memory

Phase 2 — 检索打通（待 oG-Memory 实现 read / search_memory）
├─ bridge 层对接 search_memory + read_memory
├─ assemble 中调用 L0 search，注入 systemPromptAddition
├─ 注册 agent search tool（search_memory + read_memory）
└─ 预期：agent 能通过 assemble 提示和 tool 检索到沉淀的知识

Phase 3 — 压缩与优化
├─ 决定 ownsCompaction 策略
├─ L0 search 延迟优化（缓存、category 过滤等）
└─ 预期：长对话不丢失重要信息，检索延迟可控

Phase 4 — 扩展
├─ 跨 session 全局记忆策略
├─ 多 agent namespace 隔离
└─ 知识抽取频率与成本优化
```

---

## 附录 A：已知问题

### A.1 `ingest` 未被 runtime 调用

**现象：** 在 mock 阶段实测中，无论通过 Dashboard 还是 CLI 发送消息，gateway 日志中始终没有 `ingest()` 的输出。已确认 `ingestBatch` 已注释掉。

**对比参考：** lossless-claw 插件的 `ingest` 能正常触发，并将其作为核心写入路径。

**可能原因：**
- 插件注册方式差异（JS 入口 → Python 子进程的桥接方式可能影响 runtime 的 hook 检测）
- `ownsCompaction` 或 `info` 中其他字段的配置影响了 runtime 对 hook 的调用策略
- OpenClaw 版本差异（context engine 是 v2026.3.7 新增功能，runtime 行为可能在后续版本调整）

**排查方向：**
1. 在 `ingest` 方法中加纯 JS `console.log`，定位是 runtime 没调还是 Python 桥接问题
2. 对比 lossless-claw 的 `index.ts` 注册方式与我们的差异
3. 检查 OpenClaw runtime 源码中 `ingest` 的调用条件

**当前应对：** 设计上保留 `ingest` 作为消息接收入口（缓存用），知识抽取在其他 hook 触发。即使 `ingest` 始终不被调用，`after_turn` 可从 `assemble` 收到的 messages 数组中获取消息。

### A.2 `bootstrap` 每次 turn 都调用

**现象：** 文档描述 "called once when the engine first sees a session"，但实测中每次 turn 都调用。

**影响：** bootstrap 中不能做 health check、DB 查询等 I/O 操作，只能做内存状态标记。

### A.3 `/compact` 在 Dashboard 中被当作 bash 命令

**现象：** 在 Dashboard 对话框中输入 `/compact` 后，agent 将其交给 exec tool 当作 shell 命令执行。

**规避：** 通过连续对话撑满 context 触发自动 compact，或通过 CLI 指定 session-id 发送。
