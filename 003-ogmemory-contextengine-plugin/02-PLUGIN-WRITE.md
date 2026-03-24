# 02-PLUGIN-WRITE — Phase 2：打通插件写入链路

## 目标

在 OpenClaw context engine 插件的 `after_turn` hook 中调用 oG-Memory 写入，实现对话消息自动沉淀。

**前置：Phase 1 必须完成。**

---

## 任务清单

### T2.1 [CODER] 更新 bridge/memory_api.py

位置：`/data/Workspace2/oG-Memory/openclaw_context_engine_plugin/bridge/memory_api.py`

将 mock 的 `memory_store` 替换为真实的 oG-Memory 调用：

```python
from uuid import uuid4

def extract_and_commit(messages: list[dict], session_id: str) -> dict:
    """
    将 OpenClaw 消息交给 oG-Memory 写入。
    初期 RequestContext 使用静态配置。
    """
    ctx = RequestContext(
        account_id="default",       # 初期写死
        user_id="default-user",     # 初期写死
        agent_id="main",            # 初期写死
        session_id=session_id,      # 从 OpenClaw 传入
        trace_id=str(uuid4()),
    )

    # 调用 Phase 1 验证通过的写入路径
    # 方案 A（完整 pipeline）或 方案 B（直接 write_node）
    # ...

    return {"written": True, "count": len(messages)}
```

**关键：**
- import 路径要从 Phase 1 的 test_write_api.py 中复用
- RequestContext 的 account_id/user_id/agent_id 初期写死测试值
- session_id 从 OpenClaw 传入的参数获取
- 完成后更新 PROGRESS.md

### T2.2 [CODER] 更新 mock_engine.py 的 after_turn

位置：`/data/Workspace2/oG-Memory/openclaw_context_engine_plugin/mock_engine.py`

修改 `after_turn` 方法，调用 bridge 层的 `extract_and_commit`：

```python
def after_turn(params: dict) -> dict:
    _dump_params("after_turn", params)

    session_id = params.get("sessionId", "unknown")

    # 从 params 获取本轮消息
    # 注意：after_turn 的 params 结构需要从之前的日志中确认
    # 可能在 params["messages"] 或需要其他方式获取
    messages = params.get("messages", [])

    if messages:
        result = extract_and_commit(messages, session_id)
        _log("info", f"after_turn wrote {result.get('count', 0)} messages to oG-Memory")
    else:
        _log("warn", "after_turn: no messages in params, skipping write")

    return {"ok": True}
```

**注意：** after_turn 的 params 结构需要从之前的日志中确认。如果 params 中没有 messages，在 PROGRESS.md 中记录 [ASK-HUMAN]。

### T2.3 [CODER] 编写 tests/test_plugin_write.py

模拟 mock_engine.py 的 after_turn 调用，验证端到端写入：

```python
"""
模拟 OpenClaw 调用 after_turn，验证写入到 oG-Memory。
运行：python tests/test_plugin_write.py
"""

# 1. 模拟 after_turn 的 params
params = {
    "sessionId": "test-plugin-session",
    "messages": [
        {"id": "p001", "role": "user", "content": "我叫张三，是一名后端工程师"},
        {"id": "p002", "role": "assistant", "content": "你好张三..."},
    ],
}

# 2. 调用 after_turn
from mock_engine import after_turn
result = after_turn(params)

# 3. 验证 AGFS 目录出现新文件
# ...
```

### T2.4 [RUNNER] 独立测试

```bash
cd /data/Workspace2/oG-Memory/openclaw_context_engine_plugin
python tests/test_plugin_write.py
```

检查 AGFS 目录是否出现新文件，记录到 PROGRESS.md。

### T2.5 [RUNNER] OpenClaw Dashboard 端到端测试

**准备：**

```bash
# 重启 gateway
openclaw restart

# 确认插件加载
openclaw doctor

# 打开日志监控（另一个终端）
journalctl --user -u openclaw-gateway -f | grep -E "mock-engine|memory-api|extract|commit"
```

**执行：**

1. 打开 `openclaw dashboard`
2. 发送：`我叫张三，是一名后端工程师，主要用 Python 和 Go 开发`
3. 等待 agent 回复
4. 再发：`我最近在学 Rust，打算用它重写一个性能关键的服务`

**检查：**

```bash
# 日志中是否有 after_turn 写入相关输出
# AGFS 目录是否出现新文件
find <AGFS数据目录> -type f -newer /tmp/test_marker 2>/dev/null

# 新文件内容是否包含对话关键词
grep -r "张三\|Rust\|Python" <AGFS数据目录> 2>/dev/null
```

### T2.6 [RUNNER] 记录结果到 PROGRESS.md

记录：
- gateway 日志中 after_turn 的输出
- AGFS 新增文件的目录结构
- 文件内容
- ✅ 通过 或 ❌ 失败

---

## 完成标准

全部满足后在 PROGRESS.md 标记 Phase 2 为 ✅：

- [ ] bridge/memory_api.py 更新完成
- [ ] mock_engine.py after_turn 调用 bridge 层写入
- [ ] test_plugin_write.py 运行成功
- [ ] Dashboard 对话后 AGFS 目录出现新文件
- [ ] 文件内容与对话内容相关
- [ ] 结果已记录到 PROGRESS.md

---

## 常见问题

**Q: after_turn 的 params 里没有 messages？**
- 检查之前 mock 阶段的日志，确认 params 结构
- 如果确实没有，可能需要在 assemble 中缓存 messages，after_turn 中使用缓存
- 记录 [ASK-HUMAN] 等待确认

**Q: import oG-Memory 模块失败？**
- 确认 Python path 包含 oG-Memory 项目根目录
- 可能需要 `sys.path.insert(0, "/data/Workspace2/oG-Memory")`
- 或设置 PYTHONPATH 环境变量

**Q: LLM API 调用失败？**
- `env | grep -i "api_key\|openai\|anthropic"`
- 使用 Phase 1 中验证通过的降级路径（直接 write_node）
