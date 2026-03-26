# 启动操作手册（直接照做）

## 第一步：放文件

```bash
cd /data/Workspace2/oG-Memory
git checkout dev  # 或当前主分支

# 把下载的文件放到 .claude-team 目录
mkdir -p .claude-team
# 把这个文件夹里除 STARTUP-GUIDE.md 外的所有 .md 文件复制进去
# cp ~/Downloads/005-ogmemory-plugin-opengauss/*.md .claude-team/
```

最终结构：

```
oG-Memory/
├── .claude-team/
│   ├── PROGRESS.md              ← 你看这个追踪进度
│   ├── SHARED-KNOWLEDGE.md
│   ├── 00-LEAD-COORDINATION.md
│   ├── 01-CODEBASE-ANALYSIS.md
│   ├── 02-PLUGIN-FIX-OPENGAUSS.md
│   └── 03-E2E-TESTING.md
├── service/
│   ├── api.py                   ← 写入/检索 API
│   └── index_service.py         ← 新增模块（待分析）
├── openclaw_context_engine_plugin/
│   ├── bridge/
│   │   └── memory_api.py        ← 要改的文件
│   └── index.js
├── tests/                       ← 测试放这里
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

# 确认 OpenGauss 连接信息
grep -i "gauss\|OG_" ~/.bashrc | tail -10

# 确认 OpenGauss 服务在运行（根据实际方式）
# psql -h <host> -p <port> -U <user> -c "SELECT 1" || echo "OpenGauss 未连接"
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
- 根据每个 Phase 的需要 spawn 对应的 agent（investigator / coder / runner）
- 你自己负责：读文档、制定计划、审查结果、更新 PROGRESS.md、做 go/no-go 决策
- 你不写代码，不跑命令
- agent 负责：执行具体的编码、运行测试、排查问题
- 每个 Phase 完成后，你更新 PROGRESS.md 总览表，再 spawn 下一个 Phase 的 agent

推进顺序：Phase 1 → Phase 2 → Phase 3（严格按序，不跳步）

核心规则：
- 每个阶段开始和结束时，你必须更新 .claude-team/PROGRESS.md
- 遇到不确定的问题，在 PROGRESS.md 写 [ASK-HUMAN] 然后停下来问我
- 有任何路径、版本、接口签名等信息，都同步填入 PROGRESS.md 的"关键发现"区域
- Phase 1 的目的是分析，不改代码；Phase 2 才改代码；Phase 3 做端到端测试

背景：
- 之前的任务 001-004 已完成插件的写入和检索链路（使用 mock VectorIndex）
- 主线合了新代码包括 OpenGauss 支持，但插件没有正确对接
- 当前问题：用 OpenClaw 测试可以存和召回记忆，但 OpenGauss 数据库里没有数据
- 插件硬编码了初始化逻辑，没有使用 outbox_store
- service/ 下新增了 index_service，不清楚用途
- 本次任务：先分析代码 → 修复插件 → E2E 测试

经验教训（从上一轮任务学到的）：
- 测试文件放 tests/（unit + integration），不放插件目录
- 用 pytest 标准格式，assert 验证，不要返回 True/False
- 独立脚本放 scripts/，不放 tests/
- 团队文件放 .claude-team/
- search 失败要降级，不阻塞对话
- OpenClaw 的 content 字段可能是字符串或列表，需要处理
- PROGRESS.md 放 .claude-team/，不放项目根目录

现在开始 Phase 1：
1. 读 .claude-team/01-CODEBASE-ANALYSIS.md
2. spawn 一个 investigator agent 去分析代码仓
3. 重点搞清楚：index_service 是什么、outbox_store 怎么用、写入链路从哪到哪
4. 将所有发现汇总到 PROGRESS.md 的"关键发现"区域
5. 如果分析结果不够明确，写 [ASK-HUMAN] 让我确认

我的环境：Windows WSL，conda 环境 py11。
oG-Memory 路径：/data/Workspace2/oG-Memory
OpenClaw 插件路径：/data/Workspace2/oG-Memory/openclaw_context_engine_plugin
OpenGauss 连接信息在 ~/.bashrc 末尾
```

---

## 你怎么追踪进度

**随时查看 `.claude-team/PROGRESS.md`**。

这个文件里有：
1. **顶部总览表** — 一眼看到每个阶段是 ⬜/🔄/✅/❌
2. **关键发现区域** — Phase 1 的核心输出，后续 Phase 依赖这些信息
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
我们在给 oG-Memory 的 OpenClaw 插件接入 OpenGauss。
请先读 .claude-team/PROGRESS.md 了解当前进度。
然后读 .claude-team/SHARED-KNOWLEDGE.md 了解背景。
你是 Team Lead，进入 delegate mode，从 PROGRESS.md 中最后一条记录的位置继续推进。
你不写代码不跑命令，只协调 agent、审查结果、更新 PROGRESS.md。
```

## 关键检查点

| 检查点 | 你要做什么 |
|--------|-----------|
| Phase 1 完成后 | 审核"关键发现"，确认理解正确 |
| Phase 2 T2.1 方案确认后 | 审核修改方案的 [DECISION] |
| Phase 2 完成后 | 确认修改合理 |
| Phase 3 完成后 | 验收三件事：AGFS 有数据 + OpenGauss 有数据 + 召回成功 |
