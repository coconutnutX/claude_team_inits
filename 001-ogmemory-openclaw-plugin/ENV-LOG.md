# 环境变更日志

> 每次对环境做出修改（安装软件、创建目录/文件、修改配置、设置环境变量等），
> 都必须在这里追加一条记录。格式：时间 + 做了什么 + 路径/命令。

---

### 2026-03-19 19:40 — 项目启动
- 操作：创建 .claude-team/ 目录及协调文件
- 路径：/data/Workspace2/oG-Memory/.claude-team/

### 2026-03-19 19:45 — 清理旧插件残留
- 操作：删除旧的 memory-og 插件
- 命令：rm -rf ~/.openclaw/extensions/memory-og
- 清理 OpenClaw 配置中的 memory-og 和 memory-openviking 条目

### 2026-03-19 19:50 — 创建错误的 localfs_adapter（已删除）
- 操作：创建了 fs/localfs_adapter/（错误做法，用户指出不应修改 oG-Memory）
- 路径：/data/Workspace2/oG-Memory/fs/localfs_adapter/
- 状态：已删除

### 2026-03-19 19:55 — 创建插件脚手架
- 操作：创建 openclaw-plugin/ 目录及文件
- 路径：/data/Workspace2/oG-Memory/openclaw-plugin/
- 文件：
  - package.json
  - openclaw.plugin.json
  - index.ts
  - tsconfig.json

### 2026-03-19 19:56 — 插件依赖处理
- 操作：创建 node_modules/@sinclair 符号链接
- 命令：ln -s ~/.npm-global/lib/node_modules/openclaw/node_modules/@sinclair
- 路径：/data/Workspace2/oG-Memory/openclaw-plugin/node_modules/@sinclair

### 2026-03-19 19:57 — 清理 OpenClaw 配置
- 操作：修改 ~/.openclaw/openclaw.json，删除 memory slot 配置和旧插件条目
- 路径：~/.openclaw/openclaw.json

### 2026-03-19 19:58 — 设置插件 allow list
- 操作：设置 plugins.allow 为 ["og-memory","memory-openviking"]
- 命令：openclaw config set plugins.allow '["og-memory","memory-openviking"]'

### 2026-03-19 20:09 — AGFS 源码克隆
- 操作：克隆 AGFS 仓库到临时目录
- 命令：git clone https://github.com/c4pt0r/agfs.git /tmp/agfs-build
- 路径：/tmp/agfs-build/

### 2026-03-19 20:10 — AGFS 编译
- 操作：使用 Go 编译 agfs-server
- 命令：cd /tmp/agfs-build/agfs-server && go build -o ~/.local/bin/agfs-server ./cmd/server
- 输出：~/.local/bin/agfs-server

### 2026-03-19 20:11 — AGFS 编译成功
- 操作：Go 编译 agfs-server
- 命令：cd /tmp/agfs-build/agfs-server && go build -o ~/.local/bin/agfs-server ./cmd/server
- 输出：~/.local/bin/agfs-server (36MB)

### 2026-03-19 20:14 — AGFS 配置和启动
- 操作：创建 AGFS 配置文件并启动服务器
- 配置：/tmp/agfs-config.yaml（使用 localfs 挂载到 /tmp/agfs-data）
- 命令：~/.local/bin/agfs-server -c /tmp/agfs-config.yaml &
- PID: 283128
- 状态：运行中，监听 :8080
- 挂载点：/dev, /local, /memfs

### 2026-03-19 20:16 — 创建 Python 桥接脚本
- 操作：创建 bridge/write_memory.py
- 路径：/data/Workspace2/oG-Memory/openclaw-plugin/bridge/write_memory.py
- 功能：JSON stdin → oG-Memory API → AGFS → JSON stdout

### 2026-03-19 20:17 — 更新 TypeScript 桥接代码
- 操作：修改 index.ts，将 stub 替换为真实 Python 桥接调用
- 路径：/data/Workspace2/oG-Memory/openclaw-plugin/index.ts
- 使用：child_process.execFile 调用 Python 脚本

### 2026-03-19 20:18 — 配置 OpenClaw 插件
- 操作：设置 pythonPath 和 projectRoot
- 命令：
  - openclaw config set plugins.entries.og-memory.config.pythonPath "/home/aaa/miniconda3/envs/py11/bin/python"
  - openclaw config set plugins.entries.og-memory.config.projectRoot "/data/Workspace2/oG-Memory"

### 2026-03-19 20:19 — AGFS 配置问题诊断
- 操作：诊断 AGFS memfs/localfs 后端问题
- 发现：memfs 可写但不可读不存在的文件（返回 502）
- 问题：oG-Memory exists() 依赖 404 响应

---

<!-- APPEND HERE -->

<!-- APPEND HERE -->
### 2026-03-19 20:34 — AGFS 代理问题修复
- 操作：在 bridge/write_memory.py 中禁用代理环境变量
- 原因：requests 库使用代理导致 AGFS 500 响应转换为 502 错误
- 修复：添加 `os.environ.pop('http_proxy', None)` 等
- 结果：Python 桥接脚本成功写入记忆到 AGFS

### 2026-03-19 20:34 — Phase 2 完成
- 状态：桥接层功能正常
- 验证：成功写入并验证文件创建于 /tmp/agfs-data/
- 测试命令：`echo '{"content": "I like coffee", ...}' | python openclaw-plugin/bridge/write_memory.py`

### 2026-03-19 20:44 — Phase 3 完成
- 状态：端到端测试通过
- 配置变更：
  - tools.profile: coding → full
  - 添加 apiKey 到插件配置
  - 增强日志输出
- 测试命令：`openclaw agent --agent main --local --message "..."`
- 验证：文件正确创建于 /tmp/agfs-data/accounts/...

### 2026-03-19 20:44 — 插件配置更新
- 操作：更新 openclaw.plugin.json 添加 apiKey 字段
- 路径：/data/Workspace2/oG-Memory/openclaw-plugin/openclaw.plugin.json
- 原因：Python 桥接需要 OPENAI_API_KEY 环境变量

### 2026-03-19 20:45 — 项目完成，文档整理
- 创建 README.md：用户文档、安装配置、使用方法
- 创建 DEVELOPMENT.md：开发日志、问题分析、经验教训
- 项目状态：Phase 0-3 全部完成 ✅
- 总耗时：约 1 小时

### 2026-03-19 20:46 — 项目归档
- 所有文档已整理
- 代码已测试验证
- 插件已可用

**项目总结**：
oG-Memory OpenClaw 插件开发成功，实现了完整的 Agent → TypeScript → Python → AGFS 记忆写入链路。解决了代理禁用、API Key 配置、工具可见性等关键技术问题。

