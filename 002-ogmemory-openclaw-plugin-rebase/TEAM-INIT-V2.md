# oG-Memory 插件迁移 — 工作指令

## 背景

oG-Memory 的上游仓库 `akushonkamen/oG-Memory` 发布了 `dev` 分支（基于 `phase1_write` 有修改）。我们之前在 `feature/openclaw-plugin` 分支上完成了 OpenClaw 插件开发。现在需要将插件工作迁移到基于 `dev` 的新分支上。

---

## 角色分工

| 角色 | 职责 |
|------|------|
| **@lead**（你） | Phase 0 执行、任务分配、结果验收、PROGRESS.md 维护 |
| **@coder** | Phase 1 (cherry-pick) + Phase 2 (适配修改)，由 lead 分配精确指令 |

Lead 消化本文档后给 coder 精确指令，不要让 coder 自己读本文档全文。

---

## ⚠️ 追踪规则

**`.claude-team/PROGRESS.md`** — 进度追踪板：
- 每个阶段开始/结束时更新总览表 + 追加记录
- human 在对话中给的信息 → `[HUMAN-INPUT]`
- 需要 human 介入 → `[ASK-HUMAN]`，写清楚需要做什么，然后停下来等
- 不确定的事先问，不要猜

**`.claude-team/ENV-LOG.md`** — 环境变更日志：
- 每次 git 操作（checkout、cherry-pick、merge）都记录
- 文件创建/删除/修改都记录
- coder 执行操作时也要让它记录

记录追加到 `<!-- APPEND HERE -->` 标记之前。

---

## 阶段计划

### Phase 0: 上游同步 + 分支比较 → @lead 自己执行

```bash
# 1. 添加上游 remote（如果还没有）
git remote -v  # 先看看有没有 upstream
git remote add upstream https://gitcode.com/akushonkamen/oG-Memory.git
git fetch upstream

# 2. 查看 dev 分支
git log upstream/dev --oneline -20

# 3. 比较 dev 与 phase1_write
git log upstream/dev --oneline --not origin/phase1_write
git diff origin/phase1_write upstream/dev --stat
```

**输出要求**（写入 PROGRESS.md）：
- diff --stat 结果
- 2-3 句话总结 dev 改了什么
- 如果改动涉及插件直接依赖的文件（`service/api.py`、`fs/agfs_adapter.py`、`core/models.py`、`providers/config.py`），特别标注

### Phase 1: 新建分支 + Cherry-pick → 分配给 @coder

Lead 准备好以下信息后分配给 coder：

```bash
# 先找到插件 commit hash
git log feature/openclaw-plugin --oneline -10
# 插件相关的 commit 在 phase1_write 之后
```

**给 coder 的指令模板**：

```
执行以下 git 操作，每步结果记录到 .claude-team/ENV-LOG.md：

1. git checkout -b feature/openclaw-plugin-v2 upstream/dev
2. git cherry-pick <commit-hash>   # lead 填入具体 hash
3. 如果有冲突：
   - git status 列出冲突文件
   - 把冲突文件列表写入 PROGRESS.md [BLOCKED] 记录
   - 不要自己 resolve，等 lead 指示
4. 如果没有冲突：报告成功，进 Phase 2
```

### Phase 2: 适配检查 → 分配给 @coder

**给 coder 的指令模板**：

```
Cherry-pick 完成，现在检查插件代码是否兼容 dev 分支。逐项执行并报告：

1. Import 检查：
   conda activate py11
   python -c "
   import sys; sys.path.insert(0, '.')
   from core.models import RequestContext, CandidateMemory
   from fs.agfs_adapter import AGFSContextFS
   from service.api import MemoryWriteAPI
   from providers.config import ProviderConfig
   print('All imports OK')
   "

2. API 签名检查：
   grep -n "def write_memory" service/api.py
   grep -n "class CandidateMemory" core/models.py
   对比 bridge/write_memory.py 中的调用方式是否匹配。
   如果签名变了，说明哪里变了，写 [ASK-HUMAN]。

3. 确认这些已修复项没有回退：
   - index.ts 不 import @sinclair/typebox（用原生 JSON Schema）
   - openclaw-plugin/ 下没有 node_modules/ 或 package-lock.json
   - 没有硬编码的 API URL（如 chatapi.littlewheat.com）

每项检查结果写入 PROGRESS.md。需要改代码的地方说明改了什么。
```

### Phase 3: 集成验证 → @lead 执行或指挥 @coder

1. **确认 AGFS**
   ```bash
   curl -s http://localhost:8080/api/v1/health || echo "AGFS 未启动"
   # 如果没运行：agfs-server &
   ```

2. **安装插件**
   ```bash
   openclaw plugins install -l openclaw-plugin
   openclaw plugins list
   ```

3. **Python 桥接测试**
   ```bash
   conda activate py11
   echo '{"content": "test v2 migration", "category": "preference", "tags": ["test"]}' | \
     OG_MEMORY_ROOT=$(pwd) \
     AGFS_BASE_URL=http://localhost:8080 \
     CONTEXTENGINE_PROVIDER=openai \
     OPENAI_API_KEY=$OPENAI_API_KEY \
     python openclaw-plugin/bridge/write_memory.py
   ```

4. **端到端测试**
   ```bash
   openclaw agent --local \
     -m "Please remember that I am testing the v2 branch" \
     --session-id test-v2-001
   ```

5. **AGFS 数据验证**
   ```bash
   curl "http://localhost:8080/api/v1/files?path=/local/"
   ```

每步结果写入 PROGRESS.md。全部通过 → Phase 3 ✅。

---

## 注意事项

- **用 cherry-pick，不要 rebase**，保持 dev 历史干净
- **不要修改 dev 原有代码**（除非冲突必须）
- 插件代码在 `openclaw-plugin/` 目录下，和 dev 改动大概率不冲突
- 如果 dev 改了插件依赖的 API，适配我们的代码去匹配 dev
- AGFS 环境之前配好了，先检查再说，不要重装
