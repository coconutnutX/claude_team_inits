# 01-SEARCH-API-TEST — Phase 1：独立测试 oG-Memory Search API

## 目标

不经过 OpenClaw 插件，直接调用 oG-Memory 的 search API，验证检索功能可用。

**前置条件：**
- Phase 0 已完成（现有测试通过或问题已确认）
- AGFS 中应已有之前写入的测试数据。如果没有，需要先写入测试数据再检索。

---

## 前置准备

1. **阅读 `service/api.py` 中新合入的 search 逻辑**，重点关注：
   - `search_memory()` 的函数签名（参数名、类型、默认值）
   - 返回类型（`SeedHit` 或其他结构的字段定义）
   - 内部调用链路（VectorIndex？直接读 AGFS？）
   - 是否需要 RequestContext
   - 是否有 `read_memory(uri, level)` 接口
2. **确认 AGFS 中是否有可检索的数据**：
   ```bash
   find /tmp/agfs-data -name "content.md" | head -10
   find /tmp/agfs-data -name ".abstract.md" | head -10
   ```
   如果没有数据，Phase 1 需要先写入再检索。
3. **确认向量索引状态**：search 依赖 VectorIndex。检查：
   - 向量索引是否需要单独初始化/构建？
   - 写入时 OutboxEvent 是否已触发 Embedder + VectorIndex upsert？
   - 如果向量索引未就绪，search 是否有降级路径（如直接搜 AGFS 文件）？

---

## 任务清单

### T1.1 [INVESTIGATOR] 调研 search_memory 实际接口

在编写测试前，先确认实际代码中的接口：

```bash
cd /data/Workspace2/oG-Memory

# 查看 search_memory 的定义
grep -n "def search_memory\|def search\|class.*Search" service/api.py

# 查看 SeedHit 或返回类型的定义
grep -rn "class SeedHit\|class SearchResult\|class MemoryHit" service/ core/

# 查看 read_memory 的定义
grep -n "def read_memory\|def read" service/api.py
```

将发现的实际接口签名记录到 PROGRESS.md。如果与 SHARED-KNOWLEDGE.md 中描述的不一致，以代码为准。

### T1.2 [CODER] 编写 tests/test_search_api.py

创建一个独立可运行的测试脚本：

```python
"""
独立测试 oG-Memory search API。
运行：cd /data/Workspace2/oG-Memory && python tests/test_search_api.py

步骤：
1. 确认 AGFS 中有数据（如果没有，先写入测试数据）
2. 调用 search_memory() 检索
3. 如果有 read_memory()，测试 L0/L1/L2 读取
4. 验证结果
"""

# 1. 确认/准备测试数据
#    检查 AGFS 是否有可检索的内容
#    如果没有，先调用 commit_messages 写入一批测试消息

# 2. 构造 RequestContext
ctx = RequestContext(
    account_id="test-account",   # 或 "default"，与之前写入时一致
    user_id="test-user",         # 或 "default-user"
    agent_id="test-agent",       # 或 "main"
    session_id="test-search-session",
    trace_id="trace-search-001",
)

# 3. 调用 search_memory
#    根据 T1.1 确认的实际签名调用
results = search_memory(query="数据库迁移", top_k=5, ctx=ctx)

# 4. 验证结果
# - 是否返回了结果（results 不为空）
# - 每条结果是否有 uri / score / abstract 等字段
# - score 是否合理（0~1 范围）
# - abstract 内容是否与之前写入的数据相关

# 5. 如果有 read_memory，测试不同 level
# - L0 → abstract
# - L1 → overview
# - L2 → content
```

**要求：**
- 根据 T1.1 调研结果调整 import 和调用方式
- 如果 AGFS 中没有数据，脚本自动先写入测试数据再检索
- 必须能独立运行：`python tests/test_search_api.py`
- 打印清晰的 SUCCESS / FAILED 和检索结果
- 如果 search 依赖向量索引且未就绪，在 PROGRESS.md 中记录 [ASK-HUMAN]
- 完成后更新 PROGRESS.md

### T1.3 [RUNNER] 运行测试

```bash
cd /data/Workspace2/oG-Memory
conda activate py11
python tests/test_search_api.py
```

### T1.4 [RUNNER] 验证结果

检查以下几点：
1. **search_memory 返回了结果** — 非空列表
2. **结果字段完整** — 每条结果有 uri、score、abstract（或等效字段）
3. **结果相关性** — abstract 内容与查询相关
4. **read_memory 可用**（如果有） — 不同 level 返回不同粒度的内容
5. **性能** — search 耗时（目标 < 100ms，因为 assemble 中每次 turn 都要调）

### T1.5 [LEAD] 记录结果到 PROGRESS.md

记录：
- search_memory 的实际接口签名（与设计文档的差异）
- 脚本完整输出
- 检索到的结果数量和内容摘要
- 各字段的实际类型和值
- 耗时信息
- ✅ 通过 或 ❌ 失败（含原因）

---

## 完成标准

全部满足后在 PROGRESS.md 标记 Phase 1 为 ✅：

- [ ] 确认 search_memory 的实际接口签名并记录
- [ ] test_search_api.py 运行成功（exit code 0）
- [ ] search_memory 返回了非空结果
- [ ] 结果与查询相关
- [ ] 结果已记录到 PROGRESS.md

---

## 常见问题

**Q: AGFS 中没有可检索的数据？**
- 先运行写入：可以复用之前的 test_write_api.py，或在 test_search_api.py 中先写入再检索
- 写入时用的 account_id/user_id 要和检索时一致

**Q: search_memory 返回空结果但 AGFS 中有数据？**
- 可能是向量索引未构建 — 检查 OutboxEvent / Embedder 是否已运行
- 可能是 RequestContext 的 account_id/user_id 不匹配 — 检索时用的要和写入时一致
- 可能是向量索引需要手动触发构建 — 查看 oG-Memory 文档中关于索引的说明

**Q: search_memory 接口签名与设计文档不一致？**
- 以代码为准，更新 PROGRESS.md 中的"实际接口签名"
- 后续 Phase 2 中 bridge 层按实际签名调用

**Q: read_memory 不存在或未实现？**
- 本次 Phase 1 重点是 search_memory
- read_memory 如果不可用，记录到 PROGRESS.md，Phase 2 中暂不接入
