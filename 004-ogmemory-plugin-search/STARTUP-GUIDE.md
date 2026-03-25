# 启动操作手册（直接照做）

## 第一步：放文件

```bash
cd /data/Workspace2/oG-Memory
git checkout dev

# 把下载的文件放到 .claude-team 目录
mkdir -p .claude-team
# 把这个文件夹里除 STARTUP-GUIDE.md 外的所有 .md 文件复制进去
# cp ~/Downloads/004-ogmemory-search-integration/*.md .claude-team/
```

最终结构：

```
oG-Memory/
├── .claude-team/
│   ├── PROGRESS.md              ← 你看这个追踪进度
│   ├── SHARED-KNOWLEDGE.md
│   ├── 00-LEAD-COORDINATION.md
│   ├── 00-REBASE-FIX.md
│   ├── 01-SEARCH-API-TEST.md
│   └── 02-PLUGIN-SEARCH.md
├── service/
│   └── api.py                   ← search 逻辑已在这里
├── openclaw_context_engine_plugin/
│   ├── bridge/
│   │   └── memory_api.py        ← 要改的文件
│   └── index.js
├── tests/                       ← Phase 0 先跑这里的测试
├── ... (oG-Memory 其他文件)
```

## 第二步：确认环境

```bash
conda activate py11
node --version
python --version
which openclaw

# 确认 AGFS 服务在运行
curl -s http://localhost:1833/health || echo "AGFS 未启动"

# 确认有可检索数据
find /tmp/agfs-data -name "content.md" | head -5
```

## 第三步：启动 tmux + Claude Code

```bash
# 新建 tmux 会话（或恢复已有的）
tmux new -s team
# 或
tmux attach -t team

# 在 tmux 中启动 Claude Code
claude
```

## 第四步：给 Team Lead 的初始指令

进入 Claude Code 后，粘贴以下内容：

---

```
你是这个项目的 Team Lead。

请先按顺序阅读以下文件：
1. .claude-team/SHARED-KNOWLEDGE.md
2. .claude-team/00-LEAD-COORDINATION.md

你的工作模式：
- 你进入 delegate mode，作为 Team Lead 协调工作
- 根据每个 Phase 的需要 spawn 对应的 agent（coder / runner / investigator）
- 你自己负责：读文档、制定计划、审查结果、更新 PROGRESS.md、做 go/no-go 决策
- agent 负责：执行具体的编码、运行测试、排查问题
- 每个 Phase 完成后，你更新 PROGRESS.md 总览表，再 spawn 下一个 Phase 的 agent

推进顺序：Phase 0 → Phase 1 → Phase 2（严格按序，不跳步）

核心规则：
- 每个阶段开始和结束时，你必须更新 .claude-team/PROGRESS.md
- 遇到不确定的问题，在 PROGRESS.md 写 [ASK-HUMAN] 然后停下来问我
- 有任何路径、版本、接口签名等信息，都同步填入 PROGRESS.md 的"关键路径信息"表
- Phase 0 的目的是先跑通现有测试，有一些因为 rebase 导致的问题需要先暴露出来让我修

背景：
- 写入链路已在上一轮工作中跑通（after_turn → service/api.py → AGFS）
- 主线已合入 search 逻辑（也在 service/api.py 里）
- 但 rebase 后可能有一些测试挂掉了
- 本次任务：先修 rebase 问题 → 独立验证 search API → 把 search 接到插件上 → 测试 → 修 bug

现在开始 Phase 0：
1. 读 .claude-team/00-REBASE-FIX.md
2. spawn 一个 investigator agent 去跑 oG-Memory/tests 下的所有测试
3. 收集失败结果，整理到 PROGRESS.md
4. 在 PROGRESS.md 写 [ASK-HUMAN]，等我确认并修复
5. 我确认后再进 Phase 1

我的环境：Windows WSL，conda 环境 py11。
oG-Memory 路径：/data/Workspace2/oG-Memory
OpenClaw 插件路径：/data/Workspace2/oG-Memory/openclaw_context_engine_plugin
```

---

## 你怎么追踪进度

**随时查看 `.claude-team/PROGRESS.md`**。

这个文件里有：
1. **顶部总览表** — 一眼看到每个阶段是 ⬜/🔄/✅/❌
2. **关键路径信息表** — 所有路径、接口签名集中在这里
3. **记录区** — 每条记录带角色和类型标签，按时间顺序排列

你特别需要关注的标签：
- `[ASK-HUMAN]` — **需要你操作或回答**，看到这个就介入
- `[BLOCKED]` — 卡住了，可能需要你帮忙判断
- `[DECISION]` — Claude 做了一个设计选择，你可以审核

### 在 Claude Code 中随时查看

直接问 Lead：

```
读一下 .claude-team/PROGRESS.md，总结一下当前状态
```

或者在另一个 tmux pane 中：

```bash
# 在 tmux 中分屏
# Ctrl+b % （左右分屏）或 Ctrl+b " （上下分屏）
cat /data/Workspace2/oG-Memory/.claude-team/PROGRESS.md
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
我们在给 oG-Memory 的 OpenClaw 插件接入 search 功能。
请先读 .claude-team/PROGRESS.md 了解当前进度。
然后读 .claude-team/SHARED-KNOWLEDGE.md 了解背景。
你是 Team Lead，进入 delegate mode，从 PROGRESS.md 中最后一条记录的位置继续推进。
```
