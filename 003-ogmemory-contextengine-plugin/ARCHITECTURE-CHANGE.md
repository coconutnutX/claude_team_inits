# 方案变更通知

## 变更内容

### 1. 删除 mock_engine.py，合并到 bridge/memory_api.py

`mock_engine.py` 和 `bridge/memory_api.py` 合并为一个文件 `bridge/memory_api.py`。它同时承担：
- CLI 入口（从 index.js 接收 `method` + `json_params`，分发调用）
- OpenClaw lifecycle 适配（after_turn / assemble / bootstrap / compact / dispose）
- 调用 oG-Memory 的 `service/api.py`

合并后 `index.js` 中 `callPython` 的目标改为 `bridge/memory_api.py`。

### 2. oG-Memory 的 service/api.py 保持通用，不加 OpenClaw 特有逻辑

oG-Memory 是通用记忆基础设施，不应耦合到 OpenClaw。`service/api.py` 只暴露通用方法：

```python
# service/api.py —— 通用接口，不知道 OpenClaw 的存在
commit_messages(messages, ctx)      # 写入
search_memory(query, ctx)           # 检索（future）
read_memory(uri, level, ctx)        # 读取（future）
```

### 3. OpenClaw 特有的适配全部在 bridge/memory_api.py 中完成

```python
# bridge/memory_api.py —— 知道 OpenClaw 也知道 oG-Memory

def main():
    # CLI 入口：sys.argv 拿 method + json_params → 分发
    ...

def after_turn(params):
    session_id = params.get("sessionId")
    messages = params.get("messages", [])
    ctx = _build_request_context(session_id)
    return api.commit_messages(messages, ctx)

def assemble(params):
    messages = params.get("messages", [])
    token_budget = params.get("tokenBudget", 128000)
    # 未来接 search_memory，当前先透传
    return {"messages": messages, "estimatedTokens": len(messages) * 200, ...}

def bootstrap(params): ...
def compact(params): ...
def dispose(params): ...

def _build_request_context(session_id):
    """sessionId → RequestContext（初期用静态配置）"""
    return RequestContext(
        account_id="default",
        user_id="default-user",
        agent_id="main",
        session_id=session_id,
        trace_id=str(uuid4()),
    )
```

### 4. 分层关系总结

```
index.js（Node.js，OpenClaw plugin 入口）
  │  execFileSync → python3 bridge/memory_api.py <method> <json_params>
  ▼
bridge/memory_api.py（Python，OpenClaw 适配层）
  │  - CLI 分发（替代原 mock_engine.py）
  │  - lifecycle 方法实现
  │  - RequestContext 构造
  │  - 调用 oG-Memory service 层
  ▼
service/api.py（Python，oG-Memory 通用 API）
  │  - commit_messages / search_memory / read_memory
  │  - 不知道 OpenClaw 的存在
  ▼
oG-Memory 内部（extract → plan → write → AGFS）
```

### 5. conda 环境

Python 路径需要用 conda py11 环境。请执行以下命令确认路径：

```bash
conda activate py11 && which python
```

## 要求

1. 按上述分层重构，删除 `mock_engine.py`
2. `service/api.py` 不添加任何 OpenClaw 特有逻辑
3. 重构后重新运行 Phase 2 的测试验证
4. 更新 PROGRESS.md
