# 02-PLUGIN-SEARCH — Phase 2：将 Search 接入插件 assemble hook

## 目标

在 OpenClaw context engine 插件的 `assemble` hook 中调用 oG-Memory 的 `search_memory`，实现 agent 对话时自动召回相关记忆并注入 system context。

**前置：Phase 1 必须完成。** search_memory 的实际接口签名已确认，独立测试已通过。

---

## 设计回顾

根据设计文档（DESIGN_OG_MEMORY_INTEGRATION.md §4.3），assemble 的检索流程：

```
assemble(sessionId, messages[], tokenBudget)
  │
  ├─ 1. 构建 RequestContext（sessionId → account_id 映射）
  │
  ├─ 2. 从 messages 提取最近 N 条 user 消息拼接为 query
  │
  ├─ 3. search_memory(query, top_k=5)
  │   └─ 返回 SeedHit[]：每条含 uri + abstract(≤100字) + score
  │
  ├─ 4. 构建 systemPromptAddition：
  │   │   "[oG-Memory] 相关记忆：
  │   │    - [profile] 用户是一名后端工程师，偏好 Python
  │   │    - [entity] SQLite：用户正在将数据库从 PostgreSQL 迁移到 SQLite
  │   │    如需详细信息，可使用 search_memory / read_memory 工具。"
  │   └─ 确保总 token 不超过 budget
  │
  └─ 5. 返回 AssembleResult
```

**关键原则：assemble 只做检索，不做写入。** search_memory 是轻量级查询，延迟目标 < 100ms。

---

## 任务清单

### T2.1 [CODER] 更新 bridge/memory_api.py — 添加 search 调用

位置：`/data/Workspace2/oG-Memory/openclaw_context_engine_plugin/bridge/memory_api.py`

当前 `assemble()` 方法只透传 messages 并打印 "memory search not yet implemented"。需要替换为真实的 search 调用：

```python
def assemble(params: dict) -> dict:
    """
    每次模型调用前构建 context。
    从 messages 中提取 query → search_memory → 拼成 systemPromptAddition。
    """
    session_id = params.get("sessionId", "unknown")
    messages = params.get("messages", [])
    token_budget = params.get("tokenBudget", 128000)

    ctx = _build_request_context(session_id)

    # 1. 从最近的 user 消息中提取 query
    query = _extract_query(messages)

    if not query:
        # 无用户消息，跳过检索
        return _passthrough_assemble(messages, token_budget)

    # 2. 调用 search_memory
    #    ⚠️ 根据 Phase 1 确认的实际签名调用
    try:
        hits = search_memory(query=query, top_k=5, ctx=ctx)
    except Exception as e:
        _log("error", f"assemble search_memory failed: {e}")
        # 降级：search 失败不阻塞对话
        return _passthrough_assemble(messages, token_budget)

    # 3. 格式化为 systemPromptAddition
    if hits:
        addition = _format_memory_addition(hits)
    else:
        addition = ""

    # 4. 返回 AssembleResult
    return {
        "messages": messages,
        "systemPromptAddition": addition,
        "estimatedTokens": _estimate_tokens(messages, addition),
        # ... 其他字段按 OpenClaw AssembleResult 规范填充
    }


def _extract_query(messages: list[dict]) -> str:
    """
    从最近 N 条 user 消息中提取 search query。
    策略：取最近 3 条 user 消息的 content 拼接。
    """
    user_msgs = [m["content"] for m in messages if m.get("role") == "user" and m.get("content")]
    recent = user_msgs[-3:]  # 最近 3 条
    return " ".join(recent) if recent else ""


def _format_memory_addition(hits: list) -> str:
    """
    将 SeedHit[] 格式化为注入 system prompt 的文本。
    """
    lines = ["[oG-Memory] 相关记忆："]
    for hit in hits:
        # ⚠️ 字段名根据 Phase 1 确认的实际结构调整
        category = getattr(hit, "category", "memory")
        abstract = getattr(hit, "abstract", str(hit))
        score = getattr(hit, "score", 0)
        lines.append(f"  - [{category}] {abstract} (相关度: {score:.2f})")

    lines.append("如需详细信息，可使用 search_memory / read_memory 工具。")
    return "\n".join(lines)


def _passthrough_assemble(messages: list[dict], token_budget: int) -> dict:
    """search 不可用时的降级透传。"""
    return {
        "messages": messages,
        "systemPromptAddition": "",
        "estimatedTokens": len(messages) * 200,
    }
```

**关键要求：**
- `search_memory` 的调用签名必须与 Phase 1 中确认的实际接口一致
- SeedHit 的字段访问方式（属性 vs 字典）以实际代码为准
- **search 失败时必须降级**，不能阻塞对话（try/except + passthrough）
- import 路径与 Phase 1 的 test_search_api.py 保持一致
- 完成后更新 PROGRESS.md

### T2.2 [CODER] 确认 assemble 返回值符合 OpenClaw 规范

检查 `assemble` 返回的字段是否满足 OpenClaw ContextEngine 的 `AssembleResult` 接口：

```bash
# 查看之前 assemble 的返回格式（从日志或代码中确认）
grep -A 20 "def assemble" bridge/memory_api.py

# 查看 OpenClaw 对 AssembleResult 的期望
# 参考之前 mock 阶段的日志
```

确保返回值中包含 OpenClaw 需要的所有字段。特别关注 `systemPromptAddition` 字段是否被 OpenClaw runtime 正确消费。

### T2.3 [CODER] 编写 tests/test_plugin_search.py

模拟 assemble 调用，验证端到端检索注入：

```python
"""
模拟 OpenClaw 调用 assemble，验证 search_memory 注入。
运行：cd /data/Workspace2/oG-Memory/openclaw_context_engine_plugin && python tests/test_plugin_search.py

步骤：
1. 确保 AGFS 中有可检索数据（如果没有先写入）
2. 模拟 assemble 的 params
3. 调用 assemble()
4. 验证 systemPromptAddition 中包含记忆内容
"""

# 1. 准备测试数据（如果需要）
# ...

# 2. 模拟 assemble params
params = {
    "sessionId": "test-search-session",
    "tokenBudget": 128000,
    "messages": [
        {"id": "s001", "role": "user", "content": "我之前提到过数据库迁移的事情，进展如何了？"},
        {"id": "s002", "role": "assistant", "content": "让我帮你回顾一下..."},
        {"id": "s003", "role": "user", "content": "对，就是从 PostgreSQL 迁移到 SQLite 那个"},
    ],
}

# 3. 调用 assemble
from bridge.memory_api import assemble
result = assemble(params)

# 4. 验证
# - result 不为 None
# - result["systemPromptAddition"] 非空
# - addition 中包含与查询相关的记忆内容
# - result["messages"] 与输入一致（未被修改）

# 5. 测试降级场景
# - 空 messages → 应返回空 addition
# - 无匹配结果 → 应返回空 addition，不报错
```

**要求：**
- 脚本必须能独立运行
- 覆盖正常检索和降级两种场景
- 打印 systemPromptAddition 的完整内容
- 完成后更新 PROGRESS.md

### T2.4 [RUNNER] 独立测试

```bash
cd /data/Workspace2/oG-Memory/openclaw_context_engine_plugin
conda activate py11
python tests/test_plugin_search.py
```

### T2.5 [RUNNER] OpenClaw Dashboard 端到端测试

**准备：**

```bash
# 确保有之前写入的测试数据
# 如果没有，先通过 Dashboard 对话几轮让 after_turn 写入

# 重启 gateway
openclaw restart

# 确认插件加载
openclaw doctor

# 打开日志监控（另一个 tmux pane）
# Ctrl+b % 分屏，然后：
journalctl --user -u openclaw-gateway -f | grep -E "assemble|search|memory|oG-Memory"
```

**执行：**

1. 打开 `openclaw dashboard`
2. 先确保有写入数据。如果之前已有对话，跳到步骤 3。如果没有：
   - 发送：`我叫张三，是一名后端工程师，主要用 Python 和 Go 开发`
   - 等待 agent 回复（after_turn 会写入）
   - 再发几条建立记忆基础
3. 测试检索效果 — 发送与之前对话相关的问题：
   - `你还记得我是做什么的吗？`
   - `我之前提到过什么编程语言？`

**检查：**

```bash
# 日志中是否有 assemble search 相关输出
# - "search_memory" 被调用
# - 返回了 N 条 SeedHit
# - systemPromptAddition 非空

# agent 的回复是否利用了召回的记忆
# - 如果 agent 能"记起"之前的对话内容，说明 search 注入成功
```

### T2.6 [LEAD] 记录结果到 PROGRESS.md

记录：
- gateway 日志中 assemble 的 search 输出
- systemPromptAddition 的实际内容
- agent 回复是否体现了记忆召回
- 耗时信息（assemble 整体耗时）
- ✅ 通过 或 ❌ 失败

---

## 完成标准

全部满足后在 PROGRESS.md 标记 Phase 2 为 ✅：

- [ ] bridge/memory_api.py 的 assemble 已调用 search_memory
- [ ] search 失败时有降级逻辑（不阻塞对话）
- [ ] test_plugin_search.py 运行成功
- [ ] Dashboard 对话中 agent 能召回之前的记忆
- [ ] 结果已记录到 PROGRESS.md

---

## 常见问题

**Q: assemble 中 search_memory 耗时过长（> 500ms）？**
- 检查向量索引是否在本地 — 远程服务会增加网络延迟
- 减小 top_k（从 5 降到 3）
- 考虑在 PROGRESS.md 中记录实际耗时，后续优化

**Q: systemPromptAddition 注入了但 agent 没有利用？**
- 检查注入的文本格式 — agent 可能没有理解注入的结构
- 调整 _format_memory_addition 的格式，更清晰地提示 agent
- 检查 tokenBudget — 如果预算不够，OpenClaw 可能截断了 addition

**Q: search 返回了不相关的结果？**
- 检查 score 阈值 — 可以在 _format_memory_addition 中过滤低分结果（如 score < 0.3）
- 检查 _extract_query 的逻辑 — 拼接 3 条 user 消息可能引入噪音
- 考虑只用最后 1 条 user 消息作为 query

**Q: assemble 返回值被 OpenClaw 拒绝？**
- 检查 AssembleResult 的必需字段 — 对比之前 mock 阶段的返回格式
- 在 PROGRESS.md 中记录 [ASK-HUMAN]

**Q: import service/api.py 的 search_memory 失败？**
- 确认 Python path 包含 oG-Memory 项目根目录
- 复用 Phase 1 中 test_search_api.py 的 import 方式
- 检查 bridge/memory_api.py 中已有的 oG-Memory import（写入时用的）
