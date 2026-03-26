# 03-E2E-TESTING — Phase 3：端到端测试（OpenClaw CLI）

## 目标

通过 OpenClaw CLI 进行端到端测试，验证完整链路：

```
用户发消息 → OpenClaw CLI → 插件 after_turn → oG-Memory 写入 → AGFS + OpenGauss
用户发问题 → OpenClaw CLI → 插件 assemble → oG-Memory search → OpenGauss 向量检索 → 注入 systemPrompt → agent 回复中体现记忆
```

**验收三件事：**
1. ✅ 写入成功：AGFS 中有 ContextNode 文件
2. ✅ 写入成功：OpenGauss 中有对应数据
3. ✅ 召回成功：agent 回复中体现了之前的记忆

**前置：Phase 2 必须完成。**

---

## 测试准备

### P3.1 [RUNNER] 确认所有服务在运行

```bash
# 1. AGFS 服务
curl -s http://localhost:1833/health || echo "❌ AGFS 未启动"

# 2. OpenGauss
# 根据 Phase 1 确认的连接方式验证
# 例如：
psql -h <host> -p <port> -U <user> -d <database> -c "SELECT 1" || echo "❌ OpenGauss 未连接"

# 3. OpenClaw gateway
openclaw doctor || echo "❌ OpenClaw 未启动"

# 4. 确认插件已加载
# openclaw doctor 输出中应该有 og-memory-context-engine 插件
```

### P3.2 [RUNNER] 清理测试环境（可选）

```bash
# 如果需要干净的测试环境：

# 备份并清空 AGFS 数据
# cp -r /tmp/agfs-data /tmp/agfs-data-backup-$(date +%Y%m%d)
# rm -rf /tmp/agfs-data/*

# 清空 OpenGauss 中的测试数据（根据实际表名）
# psql ... -c "DELETE FROM memory_vectors WHERE account_id = 'default';"

# 重启 gateway
openclaw restart
```

---

## 测试执行

### T3.1 [RUNNER] 写入测试 — 通过 CLI 发送消息建立记忆

```bash
# 使用 openclaw CLI 发送几条消息让 after_turn 写入记忆
# 注意：需要连续对话，让 after_turn 有足够的消息去抽取知识

# 方式 1：交互式 CLI
openclaw chat

# 然后依次发送：
# "你好，我叫张三，是一名后端工程师，主要用 Python 和 Go 开发微服务"
# 等 agent 回复
# "我最近在做一个数据库迁移项目，从 PostgreSQL 迁移到 OpenGauss"
# 等 agent 回复
# "项目deadline是下个月底，目前进度大概60%"
# 等 agent 回复

# 方式 2：如果 CLI 支持单次发送
# openclaw send "你好，我叫张三，是一名后端工程师"
# openclaw send "我最近在做数据库迁移项目"
```

**发送后等待 10-30 秒**，让 after_turn 完成写入 pipeline（涉及 LLM 调用，可能较慢）。

### T3.2 [RUNNER] 验证写入 — AGFS

```bash
# 检查 AGFS 中是否有新的 ContextNode 文件
find /tmp/agfs-data -name "content.md" -newer /tmp/agfs-data -mmin -5 | head -10

# 如果上面没结果，查看所有 content.md
find /tmp/agfs-data -name "content.md" | head -10

# 查看其中一个文件的内容，确认是从对话中抽取的知识
cat $(find /tmp/agfs-data -name "content.md" | head -1)

# 查看 abstract
find /tmp/agfs-data -name ".abstract.md" | head -5
cat $(find /tmp/agfs-data -name ".abstract.md" | head -1)
```

**预期结果：**
- 至少有 1 个 content.md 文件
- 内容应包含从对话中抽取的知识（如"张三"、"后端工程师"、"Python"等）

### T3.3 [RUNNER] 验证写入 — OpenGauss

```bash
# 连接 OpenGauss 查看数据
# 根据 Phase 1 确认的实际表名和字段

# 示例（以实际表名为准）：
psql -h <host> -p <port> -U <user> -d <database> <<EOF
-- 查看向量索引表中的数据
SELECT * FROM memory_vectors ORDER BY created_at DESC LIMIT 5;

-- 或者查看 outbox 表
SELECT * FROM outbox_events ORDER BY created_at DESC LIMIT 5;

-- 或者搜索包含测试数据的记录
SELECT * FROM memory_vectors WHERE abstract LIKE '%张三%' OR abstract LIKE '%后端%';
EOF
```

**预期结果：**
- OpenGauss 中有对应的记录
- 记录内容与 AGFS 中的数据一致
- **如果 OpenGauss 为空但 AGFS 有数据，说明 outbox_store 仍未正确接入 — 回到 Phase 2**

### T3.4 [RUNNER] 召回测试 — 通过 CLI 验证记忆检索

在**同一个 session** 中继续对话（或开新 session 如果记忆是跨 session 共享的）：

```bash
# 继续之前的 CLI 会话，发送与之前对话相关的问题
# "你还记得我是做什么的吗？"
# "我之前提到的数据库迁移项目进展如何？"
```

**检查 agent 回复：**
- agent 是否提到了"后端工程师"、"Python"、"Go"等之前的信息？
- agent 是否提到了"数据库迁移"、"OpenGauss"等？
- 如果 agent 没有体现记忆 → 检查 assemble 日志

### T3.5 [RUNNER] 日志验证

```bash
# 查看 gateway 日志中的 assemble 和 after_turn 输出
# 方式取决于 OpenClaw 的日志方式

# 方式 1：journalctl
journalctl --user -u openclaw-gateway -f | grep -E "assemble|search|memory|oG-Memory|outbox"

# 方式 2：直接日志文件
tail -100 ~/.openclaw/logs/gateway.log | grep -E "assemble|search|memory|outbox"

# 方式 3：查看插件的 stderr 输出
# bridge/memory_api.py 的 _log() 输出通常到 stderr
```

**日志中应该看到：**
- `after_turn` 被调用，写入 pipeline 执行
- `assemble` 中 `search_memory` 被调用，返回了 N 条 SeedHit
- `systemPromptAddition` 非空
- 没有异常或降级信息

### T3.6 [RUNNER] 边界情况测试

```bash
# 1. 空消息测试
# 发送一条空消息或只有空格的消息
# 预期：不崩溃，graceful degradation

# 2. 长消息测试
# 发送一条很长的消息（> 1000 字）
# 预期：正常处理

# 3. 无匹配记忆测试
# 发送一条与之前对话完全无关的消息
# "今天天气怎么样？"
# 预期：search 返回空或低分结果，agent 正常回复（不注入不相关记忆）
```

---

## 结果记录模板

### T3.7 [LEAD] 记录测试结果到 PROGRESS.md

```markdown
### [时间] [RUNNER] [DONE/FAILED] E2E 测试结果

#### 写入验证 — AGFS
- 状态：✅/❌
- ContextNode 文件数量：N
- 文件路径示例：/tmp/agfs-data/...
- 内容摘要：...

#### 写入验证 — OpenGauss
- 状态：✅/❌
- 数据库记录数量：N
- 查询结果摘要：...
- （如果 ❌）问题描述：...

#### 召回验证
- 状态：✅/❌
- 发送的问题："你还记得我是做什么的吗？"
- agent 回复摘要：...
- 回复是否体现了之前的记忆：是/否
- search_memory 返回条数：N
- systemPromptAddition 内容：...

#### 日志检查
- after_turn 正常触发：是/否
- search_memory 正常调用：是/否
- 有无异常/降级：...

#### 边界情况
- 空消息：✅/❌
- 长消息：✅/❌
- 无匹配：✅/❌
```

---

## 完成标准

全部满足后在 PROGRESS.md 标记 Phase 3 为 ✅：

- [ ] AGFS 中有写入的 ContextNode 文件
- [ ] OpenGauss 中有对应的数据
- [ ] agent 回复中体现了记忆召回
- [ ] 日志中 after_turn 和 assemble 正常工作
- [ ] 边界情况不崩溃
- [ ] 所有结果已记录到 PROGRESS.md

---

## 常见问题

**Q: OpenClaw CLI 怎么用？**
```bash
# 查看帮助
openclaw --help
openclaw chat --help

# 如果 CLI 不可用，用 Dashboard
openclaw dashboard
```

**Q: 写入到 AGFS 成功但 OpenGauss 没有？**
- 核心问题：outbox_store 没有被正确调用或配置
- 检查 after_turn 的日志，看 outbox_store 是否报错
- 检查 OpenGauss 连接是否还活着
- 回到 Phase 2 排查

**Q: search_memory 返回空？**
- 检查写入是否成功（AGFS + OpenGauss 都有数据？）
- 检查向量索引是否已构建（outbox_store → Embedder → VectorIndex upsert）
- 检查 RequestContext 的 account_id 是否一致（写入和检索要用同一个）
- 可能有异步延迟，等 30 秒再试

**Q: agent 回复没有体现记忆？**
- 检查 systemPromptAddition 是否非空
- 检查 OpenClaw 是否正确消费了 systemPromptAddition
- 检查注入的文本格式是否清晰
- 可能 agent 选择忽略了记忆提示 — 调整 _format_memory_addition 的措辞

**Q: 测试过程中 gateway 崩溃？**
- 检查崩溃日志
- 确认是插件问题还是 gateway 问题
- 在 PROGRESS.md 写 [ASK-HUMAN]
