# 启动操作手册（直接照做）

## 第一步：放文件

在 WSL 终端中执行：

```bash
# 进入 oG-Memory 项目目录
cd /path/to/oG-Memory

# 确保在正确分支
git checkout phase1_write
git checkout -b feature/openclaw-plugin

# 把 team-docs 文件夹放到项目根目录下，命名为 .claude-team
mkdir -p .claude-team
# 然后把下载的 team-docs 文件夹里的所有 .md 文件复制进去
# 例如：cp ~/Downloads/team-docs/* .claude-team/
```

最终结构：
```
oG-Memory/
├── .claude-team/
│   ├── PROGRESS.md            ← 你看这个追踪进度
│   ├── README.md
│   ├── SHARED-KNOWLEDGE.md
│   ├── 00-LEAD-COORDINATION.md
│   ├── 00-ENV-SETUP.md
│   ├── 01-PLUGIN-SCAFFOLD.md
│   ├── 02-INTEGRATION-BRIDGE.md
│   └── 03-TESTING-VALIDATION.md
├── ... (oG-Memory 现有文件)
```

## 第二步：确认环境

```bash
conda activate py11
node --version
which openclaw
```

## 第三步：启动 Claude Code

```bash
cd /path/to/oG-Memory
claude
```

## 第四步：给 Lead 的初始指令

进入 Claude Code 后，粘贴以下内容：

---

```
你是这个项目的 Lead Coordinator。

请先按顺序阅读以下文件：
1. .claude-team/SHARED-KNOWLEDGE.md
2. .claude-team/00-LEAD-COORDINATION.md

核心规则：
- 你按 Phase 0 → 1 → 2 → 3 顺序推进
- 因为只有一个会话，你自己扮演各阶段角色——但严格按对应文档操作
- 每个阶段开始和结束时，你必须更新 .claude-team/PROGRESS.md
- 遇到不确定的问题，在 PROGRESS.md 写 [ASK-HUMAN] 然后停下来问我
- 不要写复杂的 shell 脚本
- 有任何路径、版本等信息，都同步填入 PROGRESS.md 的"关键路径信息"表

现在开始 Phase 0：
1. 读 .claude-team/00-ENV-SETUP.md
2. 按诊断清单逐项检查
3. 把结果写入 PROGRESS.md
4. 如果有问题告诉我怎么修，或者在 PROGRESS.md 写 [ASK-HUMAN]
5. 全部通过后标记 Phase 0 完成，再进 Phase 1

我的环境：Windows WSL，conda 环境 py11，之前尝试过开发可能有残留。
```

---

## 你怎么追踪进度

**随时查看 `.claude-team/PROGRESS.md`**。

这个文件里有：

1. **顶部总览表** — 一眼看到每个阶段是 ⬜/🔄/✅/❌
2. **关键路径信息表** — 所有路径、版本号集中在这里
3. **记录区** — 每条记录带角色和类型标签，按时间顺序排列

你特别需要关注的标签：
- `[ASK-HUMAN]` — **需要你操作或回答**，看到这个就介入
- `[BLOCKED]` — 卡住了，可能需要你帮忙判断
- `[DECISION]` — Claude 做了一个设计选择，你可以审核

### 在 Claude Code 中随时查看

直接问 Claude：
```
读一下 .claude-team/PROGRESS.md，总结一下当前状态
```

或者在另一个终端窗口：
```bash
cat /path/to/oG-Memory/.claude-team/PROGRESS.md
```

### 如果你想介入

直接在 Claude Code 中说：
```
我看了 PROGRESS.md。关于 [某个 ASK-HUMAN]，答案是 [你的回答]。
请更新 PROGRESS.md 然后继续。
```

## 如果上下文太长需要新会话

在新的 Claude Code 会话中说：
```
我们在开发 oG-Memory 的 OpenClaw 插件。
请先读 .claude-team/PROGRESS.md 了解当前进度。
然后读 .claude-team/README.md 了解项目全貌。
从 PROGRESS.md 中最后一条记录的位置继续。
```
