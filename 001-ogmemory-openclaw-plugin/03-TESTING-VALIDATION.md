# Phase 3: 端到端验证 — @tester

## 你的角色

你是测试验证者。你的任务是确认整条链路完整跑通。

## ⚠️ PROGRESS.md 规则

在执行本阶段时，你必须维护 `.claude-team/PROGRESS.md`：

- **开始时**：更新总览表 Phase 3 状态为 🔄，追加 `[STARTED]` 记录
- **每个测试完成时**：追加 `[NOTE]` 记录，写明测试结果
- **发现 bug 时**：追加 `[BLOCKED]` 记录，描述复现步骤
- **全部通过时**：更新总览表为 ✅，追加 `[DONE]` 记录

---

## 前置条件

- Phase 2 已完成
- **先查 PROGRESS.md 的"关键路径信息"获取 agfs 存储路径等信息**

---

## 测试计划

### Test 1: 插件加载

```bash
openclaw plugins list
openclaw plugins info og-memory
openclaw plugins doctor
```

预期：插件 enabled，doctor 无错误。

### Test 2: 单次写入

```bash
openclaw agent --local \
  -m "Please remember that my favorite programming language is Rust" \
  --session-id test-write-001
```

观察 agent 是否调用了 `og_memory_write` tool 且返回成功。

如果 agent 没有自动调用：
```bash
openclaw agent --local \
  -m "Use the og_memory_write tool to store this: I prefer dark mode" \
  --session-id test-write-002
```

### Test 3: agfs 路径验证 ⭐关键

```bash
# 创建时间标记
touch /tmp/agfs-test-marker

# 执行写入（通过 agent 或手动调 Python 桥接都行）

# 查找新文件（用 PROGRESS.md 中记录的 agfs 存储路径）
find <agfs存储路径> -newer /tmp/agfs-test-marker -type f -ls 2>/dev/null
```

**必须在 agfs 路径下找到写入的数据，这是 MVP 的核心验收标准。**

### Test 4: 错误处理

```bash
openclaw agent --local \
  -m "Use og_memory_write with empty content" \
  --session-id test-error-001
```

预期：返回友好的错误信息，不崩溃。

---

## 调试指南

**agent 没调 tool** → `openclaw plugins list` 检查插件状态

**Python 桥接报错** → 直接用 PROGRESS.md 中记录的 Python 路径手动测试：
```bash
echo '{"content": "test", "tags": []}' | \
  OG_MEMORY_ROOT=<项目路径> \
  <python路径> <项目路径>/openclaw-plugin/bridge/write_memory.py
```

**agfs 没数据** → 检查 agfs 进程状态，检查路径权限

---

## 完成动作

更新 PROGRESS.md，示例：

```markdown
### [DONE] @tester — Phase 3 端到端验证通过

Test 1 插件加载: ✅
Test 2 单次写入: ✅ agent 调用 og_memory_write 并返回成功
Test 3 agfs 验证: ✅ 在 ~/.agfs/data/memories/ 找到新写入的 JSON 文件
Test 4 错误处理: ✅ 空内容时返回友好错误

🎉 MVP 链路跑通。
```
