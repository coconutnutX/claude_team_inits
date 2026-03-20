# 启动操作手册

## 1. 放文件

```bash
cd /path/to/oG-Memory
# 把这几个文件放到 .claude-team/ 目录下
# 如果目录已存在，替换 PROGRESS.md 和 ENV-LOG.md（旧的可以备份）
# TEAM-INIT-V2.md 是新的
mkdir -p .claude-team
cp TEAM-INIT-V2.md PROGRESS.md ENV-LOG.md .claude-team/
```

## 2. 启动 Claude Code

```bash
cd /path/to/oG-Memory
claude
```

## 3. 初始指令

粘贴以下内容：

```
请阅读 .claude-team/TEAM-INIT-V2.md，这是你的完整工作指令。

角色分工：
- 你（lead）负责 Phase 0（上游同步、分支比较）和最终验收
- Phase 1（cherry-pick）和 Phase 2（适配）交给 coder 执行
- 你消化文档后给 coder 精确指令，不要让 coder 读全文

工作方式：
1. 先自己执行 Phase 0，把差异分析写入 PROGRESS.md
2. 完成后整理 Phase 1 + 2 的指令交给 coder
3. Coder 完成后你检查，确认没问题进 Phase 3 验证
4. 全程维护 PROGRESS.md，coder 的操作记录到 ENV-LOG.md
5. 遇到不确定的写 [ASK-HUMAN] 然后停下来问我

从 Phase 0 开始。
```

## 4. 追踪进度

随时查看 `.claude-team/PROGRESS.md`：
- 顶部总览表看整体状态
- `[ASK-HUMAN]` 标签 = 需要你介入
- `[BLOCKED]` 标签 = 卡住了

在另一个终端：
```bash
cat /path/to/oG-Memory/.claude-team/PROGRESS.md
```
