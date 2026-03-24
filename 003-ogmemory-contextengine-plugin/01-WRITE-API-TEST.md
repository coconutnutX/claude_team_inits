# 01-WRITE-API-TEST — Phase 1：独立测试 oG-Memory Write API

## 目标

不经过 OpenClaw 插件，直接调用 oG-Memory 的写入 API，验证消息能正确写入 AGFS 目录。

---

## 前置准备

1. 阅读 oG-Memory 的 `CLAUDE.md`，重点关注：
   - §3 核心对象模型（RequestContext, ContextNode）
   - §4 核心接口规范（ContextFS, CandidateExtractor）
   - §2 AGFS 目录 Spec（写入顺序和文件结构）
2. 确认可用的写入路径：
   - 方案 A：完整 pipeline（CandidateExtractor → MergePolicy → ContextWriter）
   - 方案 B：直接 ContextFS.write_node()（如果 pipeline 未就绪）
3. 确认 AGFS 数据目录位置（记录到 PROGRESS.md 关键路径表）

---

## 任务清单

### T1.1 [CODER] 编写 tests/test_write_api.py

创建一个独立可运行的测试脚本：

```python
"""
独立测试 oG-Memory write API。
运行：python tests/test_write_api.py
"""

# 1. 构造测试消息（模拟 OpenClaw 格式）
test_messages = [
    {"id": "m001", "role": "user", "content": "我想把数据库从 PostgreSQL 迁移到 SQLite"},
    {"id": "m002", "role": "assistant", "content": "好的，SQLite 适合嵌入式场景..."},
    {"id": "m003", "role": "user", "content": "我偏好用 Python 的 sqlite3 标准库"},
]

# 2. 构造 RequestContext（用测试值）
ctx = RequestContext(
    account_id="test-account",
    user_id="test-user",
    agent_id="test-agent",
    session_id="test-session-001",
    trace_id="trace-001",
)

# 3. 调用写入（方案 A 或方案 B，根据可用性选择）
# ...

# 4. 验证
# - 检查 AGFS 目录是否出现 content.md, .abstract.md, .overview.md, .meta.json
# - 打印文件路径和内容
# - 打印 SUCCESS 或 FAILED
```

**要求：**
- 根据实际 oG-Memory 代码调整 import 和调用方式
- 如果某个组件未实现，在代码注释中说明，降级到可用路径
- 必须能独立运行：`python tests/test_write_api.py`
- 打印清晰的 SUCCESS / FAILED 和写入路径
- 完成后更新 PROGRESS.md

### T1.2 [RUNNER] 运行测试

```bash
cd /data/Workspace2/oG-Memory
python tests/test_write_api.py
```

### T1.3 [RUNNER] 检查 AGFS 输出

```bash
# 查找写入的文件（路径需根据 T1.1 确认）
find <AGFS数据目录> -name "content.md" -o -name ".meta.json" | head -20

# 查看目录结构，预期：
# <AGFS根>/accounts/test-account/users/test-user/memories/entities/sqlite/
#   ├── content.md          ← 完整内容
#   ├── .abstract.md        ← ≤100字摘要
#   ├── .overview.md        ← 结构化概述
#   ├── .relations.json     ← 关系边
#   └── .meta.json          ← status 应为 ACTIVE

# 检查 .meta.json
cat <路径>/.meta.json | python3 -m json.tool

# 检查 content.md 有内容
cat <路径>/content.md
```

### T1.4 [RUNNER] 记录结果到 PROGRESS.md

记录：
- 脚本完整输出
- AGFS 目录 tree 结构
- 各文件内容摘要
- ✅ 通过 或 ❌ 失败（含原因）

---

## 完成标准

全部满足后在 PROGRESS.md 标记 Phase 1 为 ✅：

- [ ] test_write_api.py 运行成功（exit code 0）
- [ ] AGFS 目录中出现预期文件
- [ ] content.md 有内容
- [ ] .meta.json 中 status=ACTIVE
- [ ] 结果已记录到 PROGRESS.md
