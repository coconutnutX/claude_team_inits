# Phase 0: 环境诊断与准备 — @env

## 你的角色

你是环境工程师。你的任务是确保整个开发环境干净可用，让后续插件开发能顺利进行。用户之前尝试过开发这个插件，可能把环境搞乱了，所以需要格外仔细。

## ⚠️ PROGRESS.md 规则

在执行本阶段时，你必须维护 `.claude-team/PROGRESS.md`：

- **开始时**：更新总览表 Phase 0 状态为 🔄，追加 `[STARTED]` 记录
- **发现问题时**：追加 `[NOTE]` 或 `[BLOCKED]` 记录
- **需要 human 操作时**：追加 `[ASK-HUMAN]` 记录，写清楚需要做什么
- **完成时**：更新总览表为 ✅，填写"关键路径信息"表格，追加 `[DONE]` 记录

---

## 环境背景

- **OS**: Windows WSL (Ubuntu)
- **Python**: conda 环境 `py11`（先 `conda activate py11`）
- **Node/npm**: 用于 OpenClaw（`npm install -g openclaw@latest` 已安装过）
- **Go**: 已安装，用于 agfs
- **项目**: oG-Memory 已 clone 到本地

---

## 诊断清单（按顺序执行）

### 1. 基础工具版本

```bash
node --version        # 需要 22+
npm --version
go version            # agfs 需要 go
conda activate py11
python --version
which python
which node
which go
```

**把每个版本号填入 PROGRESS.md 的"关键路径信息"表格。**

### 2. OpenClaw 状态检查

```bash
which openclaw
openclaw --version

# 检查是否有残留的旧插件配置
openclaw plugins list

# 检查配置文件
cat ~/.openclaw/openclaw.json 2>/dev/null || echo "无配置文件"

# 检查是否有残留的 extensions
ls -la ~/.openclaw/extensions/ 2>/dev/null || echo "无 extensions 目录"
```

**关键**：如果看到之前尝试的 og-memory 相关残留，需要清理：
```bash
openclaw plugins disable og-memory 2>/dev/null
rm -rf ~/.openclaw/extensions/og-memory 2>/dev/null
```

如果不确定是否该删某个东西，在 PROGRESS.md 写一条 `[ASK-HUMAN]`，描述你看到了什么，问要不要删。

### 3. 检查 OpenClaw 配置污染

```bash
cat ~/.openclaw/openclaw.json | grep -A 5 "plugins" 2>/dev/null
```

如果配置中有 `plugins.entries.og-memory` 或 `plugins.slots.memory` 指向旧插件，需要删除这些条目。**不要删除整个配置文件**，只清理 og-memory 相关的部分。

### 4. agfs 验证

agfs 是 oG-Memory 的核心存储层。需要确认它能正常工作。

```bash
cd <项目根目录>
ls -la
find . -name "*.go" -o -name "agfs*" | head -20
cat go.mod 2>/dev/null
go build ./... 2>&1 | head -20
```

**重点验证**：
- agfs 二进制是否能编译成功
- 预期的存储路径是什么
- 是否需要特定的环境变量
- 是否需要后台进程才能运行

**把 agfs 存储路径和是否需要后台进程填入 PROGRESS.md。**

如果编译失败，在 PROGRESS.md 追加一条 `[BLOCKED]`，贴上错误信息，这是一级阻塞。

### 5. Python 依赖

```bash
conda activate py11
cd <项目根目录>
cat requirements.txt 2>/dev/null
cat pyproject.toml 2>/dev/null
pip install -r requirements.txt 2>/dev/null
# 或
pip install -e . 2>/dev/null
```

### 6. 创建开发分支

```bash
git status
git branch -a
git checkout phase1_write
git pull origin phase1_write 2>/dev/null
git checkout -b feature/openclaw-plugin
```

如果分支已存在（之前创建过），在 PROGRESS.md 写 `[ASK-HUMAN]` 问是要继续用还是重建。

---

## 完成动作

所有检查通过后，更新 PROGRESS.md：

1. 更新总览表 Phase 0 → ✅
2. 确保"关键路径信息"表格全部填完
3. 追加 `[DONE]` 记录，附上简要总结

示例：

```markdown
### [DONE] @env — Phase 0 环境诊断完成

所有工具版本满足要求。已清理旧的 og-memory 插件残留。
agfs 可编译，存储路径为 ~/.agfs/data。无需后台进程。
Python 依赖已安装。开发分支 feature/openclaw-plugin 已创建。
```
