# LESSONS LEARNED — OpenClaw Plugin Integration & Testing

## 任务背景
将 oG-Memory 接入 OpenClaw 作为 context engine 插件，包括写入和检索两条链路的端到端测试。

## 核心经验总结

### 1. 进度文件位置规范

**正确位置**：`.claude-team/PROGRESS.md`
- 团队协调文件应统一放在 `.claude-team/` 目录下
- 项目根目录的 `PROGRESS.md` 应删除或移动
- 这样保持项目结构清晰，团队文件集中管理

**错误做法**：
```bash
PROGRESS.md  # ❌ 放在项目根目录
```

**正确做法**：
```bash
.claude-team/PROGRESS.md  # ✅ 集中管理
```

---

### 2. 测试文件位置规范

**核心原则**：插件目录是发布包，不应包含测试代码

```
openclaw_context_engine_plugin/
├── index.js           # ✅ 插件入口
├── bridge/            # ✅ Python bridge 层
├── package.json       # ✅ 插件配置
└── tests/             # ❌ 不要在这里放测试！
```

**正确位置**：
```
tests/
├── unit/service/      # ✅ 单元测试（service 层 API）
└── integration/       # ✅ 集成测试（包括 plugin 测试）
```

**原因**：
- 插件目录会被打包安装，测试代码不应包含在内
- 测试代码应在项目统一的测试目录下管理
- 方便 CI/CD 和测试覆盖率统计

---

### 3. 测试文件的正确设计模式

#### 重要澄清

**测试文件应该只支持 pytest 执行，不需要 standalone 模式！**

| 类型 | 位置 | 执行方式 | 函数命名 |
|------|------|---------|---------|
| **Pytest 测试** | `tests/unit/`<br>`tests/integration/` | `pytest tests/...` | `test_xxx()` |
| **独立脚本/工具** | `scripts/`<br>或项目根目录 | `python script.py` | `run_xxx()` / `main()` |

**错误观念**：认为测试文件需要同时支持 `pytest` 和 `python xxx.py` 两种执行方式
- 这会导致过度设计
- 实际上测试就用 pytest，脚本就用 python，各司其职

#### Pytest 测试的正确写法

```python
# ✅ 正确：标准的 pytest 测试
def test_extract_query():
    """Test _extract_query() extracts queries from messages."""
    messages = [
        {"role": "user", "content": "Hello"},
        {"role": "user", "content": "How are you?"},
    ]
    query = _extract_query(messages)
    expected = "Hello\nHow are you?"

    assert query == expected, f"Expected {repr(expected)}, got {repr(query)}"
```

**规则**：
- ✅ 函数名以 `test_` 开头
- ✅ 使用 `assert` 验证结果
- ✅ 不返回任何值（或返回 `None`）
- ✅ 不需要 `if __name__ == "__main__"`
- ❌ 不要用 `print` 判断成功失败（pytest 只看 assert）
- ❌ 不要返回 `True/False`

#### 独立脚本的正确写法

如果确实需要 standalone 脚本（不是测试），应该放在 `scripts/` 目录：

```python
# scripts/benchmark_search.py  ✅ 正确位置
#!/usr/bin/env python3
"""Benchmark search performance - standalone tool, not a test."""

def run_benchmark():
    """Run benchmark (not a test function)."""
    # ... benchmark logic ...
    print(f"Duration: {duration}ms")

if __name__ == "__main__":
    run_benchmark()
```

**规则**：
- ✅ 放在 `scripts/` 目录，不在 `tests/`
- ✅ 函数名**不以** `test_` 开头（如 `run_xxx`, `benchmark_xxx`）
- ✅ 使用 `print` 输出结果
- ✅ 用 `if __name__ == "__main__"` 作为入口
- ❌ 不要放在 `tests/` 下（避免被 pytest 误收集）

#### 常见坑点

**坑 1：`test_` 函数被 pytest 自动收集**

如果你在 `tests/` 目录下有一个独立脚本（不是测试），pytest 会自动收集所有 `test_` 开头的函数。

```python
# ❌ 错误：在 tests/ 下但不是 pytest test
def test_search_performance():
    """这是一个性能测试脚本，不是单元测试"""
    results = run_search_benchmark()
    print(f"Results: {results}")  # 只 print，不 assert
```

**问题**：pytest 会收集并执行这个函数，如果没有 `assert` 就永远 PASS

**解决方案**：

1. **如果是真正的 pytest 测试** → 添加 `assert`
```python
# ✅ 正确：添加断言
def test_search_performance():
    results = run_search_benchmark()
    assert results["duration_ms"] < 1000, "Search should complete within 1s"
```

2. **如果是独立脚本** → 移到 `scripts/` 目录
```python
# ✅ 正确：放在 scripts/，改名为 benchmark_search.py
def run_benchmark():
    results = run_search_benchmark()
    print(f"Results: {results}")
```

**坑 2：`PytestReturnNotNoneWarning`**
```python
# ❌ 错误：pytest 不希望测试函数有返回值
def test_extract_query():
    if query == expected:
        print_success("OK")
        return True   # pytest 警告：should return None
    else:
        print_error("FAIL")
        return False
```

**解决方案 2**：使用 `assert` 语句
```python
# ✅ 正确：使用 assert 让 pytest 捕获失败
def test_extract_query():
    if query == expected:
        print_success("OK")
    else:
        print_error("FAIL")

    assert query == expected, f"Expected {expected}, got {query}"
```

**坑 3：只 print 错误，pytest 认为通过**
```python
# ❌ 错误：print 了错误但 pytest 不知道
def test_assemble_normal():
    try:
        result = assemble(params)
        if not result:
            print_error("assemble() returned None")
            return None  # pytest 仍然认为通过！
    except Exception as e:
        print_error(f"Exception: {e}")
```

**解决方案 3**：异常 + assert 双保险
```python
# ✅ 正确：assert 让 pytest 能检测失败
def test_assemble_normal():
    result = assemble(params)
    assert result is not None, "assemble() returned None"
    assert "messages" in result, f"Missing 'messages' field: {result.keys()}"
    print_success("assemble() returned with correct structure")
```

#### 最佳实践：标准 Pytest 测试模板

```python
#!/usr/bin/env python3
"""
Pytest test file - use 'pytest tests/integration/test_xxx.py' to run.
"""

def test_feature_normal_case():
    """Test feature with normal input."""
    result = function_under_test(normal_input)

    assert result is not None, "Function should return a result"
    assert result.status == "success", f"Expected success, got {result.status}"
    assert result.value > 0, f"Value should be positive, got {result.value}"


def test_feature_edge_case():
    """Test feature with edge case (empty input)."""
    result = function_under_test("")

    assert result == "", "Empty input should return empty output"


def test_feature_error_handling():
    """Test feature handles errors gracefully."""
    with pytest.raises(ValueError, match="invalid input"):
        function_under_test("invalid")

# ❌ 不要写 main() 函数 - 用 pytest 运行，不需要自己实现 runner
# ❌ 不要用 print 判断成功失败 - pytest 只看 assert
# ❌ 不要返回 True/False - 返回 None 即可
```

**关键要点**：
- 测试文件 = pytest 跑，不需要支持 `python xxx.py`
- 独立脚本 = 放在 `scripts/`，不在 `tests/` 下
- 各司其职，不要混在一起


---

### 4. Git Rebase 冲突处理经验

#### 场景 1：Modify/Delete 冲突
```
CONFLICT (modify/delete): mock_engine.py deleted in HEAD and modified in dev_context_engine_plugin
```

**分析**：
- 你的分支删除了文件（代码已合并到 memory_api.py）
- dev 分支修改了文件
- 应该保留删除，接受你的版本

**解决方案**：
```bash
git rm mock_engine.py  # 删除文件，接受你的分支
```

#### 场景 2：内容冲突
```
CONFLICT (content): Merge conflict in memory_api.py
```

**分析**：
- dev 分支更新了某些内容
- 你的分支也修改了同一文件
- 需要查看冲突内容决定

**解决方案**：
```bash
# 如果你的分支更新更完整
git checkout --theirs memory_api.py

# 或者手动编辑解决冲突
vim memory_api.py
git add memory_api.py
```

**经验**：
- rebase 前先确保本地 commit 清晰，便于解决冲突
- 冲突解决后用 `git rebase --continue` 继续
- 不要害怕冲突，理解原因后很容易解决

---

### 5. 插件重命名清单

重命名插件需要修改的地方（mock-context-engine → og-memory-context-engine）：

1. **插件元数据**：
   - `package.json` → `name` 字段
   - `openclaw.plugin.json` → `id` 字段
   - `index.js` → 插件标识符
   - `README.md` → 所有文档引用

2. **配置文件**：
   - `~/.openclaw/openclaw.json` → `plugins.allow[]`、`plugins.entries[]`、`plugins.slots`
   - 环境变量或脚本中的引用

3. **文档说明**：
   - `ENV.md` → 配置示例
   - 任何 README 或指南中的引用

**检查清单**：
```bash
grep -r "mock-context-engine" . --exclude-dir=.git
```

---

### 6. OpenClaw 结构化内容格式

**问题**：OpenClaw 消息的 `content` 字段不总是字符串

```python
# 有时是字符串
{"role": "user", "content": "Hello"}

# 有时是结构化列表
{"role": "user", "content": [
    {"type": "text", "text": "Hello"},
    {"type": "image_url", "image_url": {"url": "..."}}
]}
```

**解决方案**：添加内容提取辅助函数

```python
def _extract_content_text(content) -> str:
    """Extract text from message content (handles both formats)."""
    if isinstance(content, str):
        return content
    elif isinstance(content, list):
        text_parts = []
        for block in content:
            if isinstance(block, dict):
                text_parts.append(block.get("text", ""))
        return " ".join(text_parts).strip()
    return str(content)
```

---

### 7. Graceful Degradation 模式

在 `assemble()` hook 中，搜索失败不应阻塞对话：

```python
def assemble(params: dict) -> dict:
    try:
        result = read_api.search_memory(query, ctx, top_k=5)
        hits = result.hits
        system_prompt_addition = _format_memory_addition(hits)
    except Exception as e:
        # 降级：搜索失败不影响对话
        _log("warn", f"Memory search failed: {e}")
        system_prompt_addition = (
            "[oG-Memory] Memory search temporarily unavailable. "
            "Conversation continues normally."
        )

    return {
        "messages": messages,
        "systemPromptAddition": system_prompt_addition,
    }
```

**原因**：
- 记忆是辅助功能，不应阻断主流程
- 用户可以继续对话，只是没有上下文召回
- 生产环境中可用性更重要

---

## 工具与命令速查

### Git 操作
```bash
# Rebase 到目标分支
git rebase dev

# 解决冲突后继续
git rebase --continue

# 放弃 rebase
git rebase --abort

# 修改最后一个 commit（amend）
git commit --amend --no-edit  # 不改 message
git commit --amend -m "New message"  # 修改 message

# 撤销最后一个 commit（保留修改）
git reset --soft HEAD~1
```

### Pytest 操作
```bash
# 运行所有测试
pytest tests/

# 运行特定目录的测试
pytest tests/integration/

# 运行单个测试文件
pytest tests/integration/test_openclaw_plugin.py

# 运行单个测试函数
pytest tests/integration/test_openclaw_plugin.py::test_extract_query -v

# 检查哪些函数会被 pytest 收集
pytest --collect-only tests/integration/test_openclaw_plugin.py
```

**注意**：测试文件应该用 `pytest` 运行，不是 `python xxx.py`

### 配置检查
```bash
# 检查 OpenClaw 配置
cat ~/.openclaw/openclaw.json

# 检查 AGFS 服务
curl http://localhost:1833/api/v1/health

# 检查 Python 环境
which python
python --version
```

---

## 总结

1. **文件组织**：
   - 测试在 `tests/`（unit + integration）
   - 团队文件在 `.claude-team/`（PROGRESS.md, LESSONS-LEARNED.md）
   - 独立脚本在 `scripts/`
   - 插件目录保持干净（不放测试）

2. **测试设计**：
   - **pytest 测试**：用 `pytest tests/...` 运行，使用 `assert` 验证结果
   - **独立脚本**：用 `python scripts/xxx.py` 运行，使用 `print` 输出结果
   - 不要混在一起，各司其职

3. **命名约定**：
   - pytest 测试：函数名 `test_xxx()`
   - 独立脚本：函数名 `run_xxx()`, `benchmark_xxx()` 等（避免 `test_` 前缀）

4. **降级策略**：辅助功能失败不应阻塞主流程

5. **Git 卫生**：小步 commit，清晰 message，冲突不可怕

这些经验可用于后续的插件开发和测试工作。

