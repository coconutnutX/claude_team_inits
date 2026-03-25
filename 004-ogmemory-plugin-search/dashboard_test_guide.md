# OpenClaw Dashboard 端到端测试指南

## 准备工作

1. 确认 AGFS 服务运行中
   ```bash
   curl -s http://localhost:1833/health
   ```

2. 重启 OpenClaw Gateway（加载最新代码）
   ```bash
   openclaw restart
   ```

3. 确认插件加载
   ```bash
   openclaw doctor
   ```

4. 打开日志监控（另一个终端）
   ```bash
   journalctl --user -u openclaw-gateway -f | grep -E "assemble|search|memory|oG-Memory"
   ```

## 测试步骤

### 步骤 1：确保有写入数据

如果 AGFS 中没有足够的测试数据，先进行几轮对话：

**发送消息 A**：
```
我叫张三，是一名后端工程师，主要用 Python 和 Go 开发。我最近在做一个数据库迁移项目，从 PostgreSQL 迁移到 SQLite。
```

等待 agent 回复（这会触发 after_turn 写入 oG-Memory）

**发送消息 B**：
```
我的编程习惯是：喜欢用 vim 作为编辑器，重视代码可读性，经常写单元测试。
```

**发送消息 C**：
```
我对 Django 框架很熟悉，之前做过多个 Web 项目。
```

### 步骤 2：测试检索效果

发送与之前对话相关的问题：

**测试问题 1**：
```
你还记得我是做什么的吗？
```

**预期结果**：
- agent 应该能回答你是后端工程师
- 日志中应显示 assemble 调用了 search_memory
- 日志中应显示返回了相关记忆（如 profile）

**测试问题 2**：
```
我之前提到过什么编程语言？
```

**预期结果**：
- agent 应该能回答 Python 和 Go
- 日志中应显示相关记忆被召回

**测试问题 3**：
```
我最近在做什么项目？
```

**预期结果**：
- agent 应该能提到数据库迁移项目
- 相关 entity 记忆应被召回

## 检查点

### 日志检查
在日志中查找以下关键信息：
```
[memory-api ...] [INFO] assemble  sessionId=...
[memory-api ...] [INFO] assemble  memory search returned N results
```

### Agent 回复检查
- ✅ agent 能"记住"之前对话的内容
- ✅ agent 回复中包含之前提到的信息
- ✅ 如果搜索失败，agent 仍能正常对话（降级）

## 常见问题

**Q: agent 说"我不记得了"？**
- 检查 AGFS 中是否真的有数据：`find /tmp/agfs-data -name content.md | wc -l`
- 检查向量索引是否初始化（第一次 assemble 会初始化）

**Q: assemble 日志没有出现？**
- 检查 gateway 是否重启加载了最新代码
- 检查插件是否启用：`openclaw doctor`

**Q: search 返回 0 条结果？**
- 可能是向量索引中还没有数据（需要等 Outbox Worker 处理）
- 可以尝试多对话几轮，让系统积累更多数据
