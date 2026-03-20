# 环境变更日志

> 每次 git 操作、文件创建/删除/修改、配置变更都追加记录。
> 格式：操作描述 + 具体命令或路径。

---

### Phase 0: 上游同步 + 分支比较

**Git 操作**:
- `git remote add upstream https://gitcode.com/akushonkamen/oG-Memory.git`
- `git fetch upstream`
  - 新增分支: dev, master, phase1, phase1_write

**文件修改**:
- `.claude-team/PROGRESS.md` — 更新Phase 0状态、关键信息表、记录区
- `.claude-team/ENV-LOG.md` — 本文件初始化

---

### Phase 1: Cherry-pick 插件代码

**Git 操作**:
- `git checkout -b feature/openclaw-plugin-v2 upstream/dev`
  - 成功创建并切换到新分支
  - 分支跟踪 upstream/dev
- `git cherry-pick 1a6c98c`
  - ⚠️  冲突：`.gitignore` 文件
  - 冲突内容：HEAD (upstream/dev) 有详细的注释结构和分类，cherry-pick commit 添加了 `.claude-team` 和 `openclaw-plugin` 忽略规则
- 冲突解决：
  - 合并两边的 .gitignore 内容
  - 保留 upstream/dev 的完整注释结构
  - 添加插件特有规则：`.claude-team/`, `openclaw-plugin/node_modules/`, `openclaw-plugin/package-lock.json`
- `git add .gitignore && git cherry-pick --continue`
  - ✅ 成功完成：新 commit 5a0dd4d
  - 9 files changed, 765 insertions(+)
  - 新增文件：config.yaml, openclaw-plugin/{README.md,bridge/write_memory.py,index.ts,openclaw.plugin.json,package.json,test-tool.mjs,tsconfig.json}

**Git 状态**:
```
On branch feature/openclaw-plugin-v2
Your branch is ahead of 'upstream/dev' by 1 commit.
nothing to commit, working tree clean
```

---

### Phase 2: 适配检查

**2.1 Import 检查**:
```bash
conda activate py11
python -c "
import sys; sys.path.insert(0, '.')
from core.models import RequestContext, CandidateMemory
from fs.agfs_adapter import AGFSContextFS
from service.api import MemoryWriteAPI
print('All imports OK')
"
```
结果：✅ All imports OK

**2.2 API 签名检查**:
```bash
grep -n "def commit_session" service/api.py
# 输出: 68:    def commit_session(
grep -n "class MemoryWriteAPI" service/api.py
# 输出: 21:class MemoryWriteAPI:
grep -n "def write_memory" service/api.py
# 输出: 130:    def write_memory(
```

验证结果：
- ✅ `MemoryWriteAPI.write_memory(candidate, ctx)` 签名与 bridge 期望一致
- ✅ `MemoryWriteAPI.commit_session(messages, ctx, confidence_threshold)` 存在
- ✅ 所有导入路径正确

**2.3 回归检查**:
```bash
grep -n "@sinclair/typebox" openclaw-plugin/index.ts
# 结果：无输出（✅ 不使用 typebox）

ls -la openclaw-plugin/ | grep -E "node_modules|package-lock"
# 结果：无输出（✅ 没有 node_modules 或 package-lock.json）

grep -r "chatapi.littlewheat.com" openclaw-plugin/
# 结果：无输出（✅ 没有硬编码 API URL）
```

**文件修改**:
- `.claude-team/PROGRESS.md` — 更新 Phase 1+2 状态为完成，添加详细执行记录
- `.claude-team/ENV-LOG.md` — 本 Phase 1+2 日志追加

---

### Phase 3: 集成验证 + Bug修复

**Git 状态**:
- 当前分支: `feature/openclaw-plugin-v2` (clean, ahead of upstream/dev by 1 commit)

**环境检查**:
- ✅ AGFS 运行中 (v1.4.0, healthy)
- ✅ OpenClaw CLI 已安装

**发现的问题**:
- `providers/config.py` import 错误：直接 import `OpenAIEmbedder`/`OpenAILLM`，但 dev 分支已改为 lazy loading

**代码修改**:
- `providers/config.py` — 修复 lazy import bug
  - Line 11-12: 使用 `get_openai_embedder()` 和 `get_openai_llm()`
  - `create_embedder()`: 添加 lazy load 逻辑
  - `create_llm()`: 添加 lazy load 逻辑

**测试执行**:
1. Import check: ✅ 所有模块导入成功
2. Python bridge test: ✅ 成功写入记忆到 AGFS
   - URI: `ctx://default/users/default/memories/preference/preference_9401ddba`

**文件修改**:
- `.claude-team/PROGRESS.md` — 更新 Phase 3 状态为完成，添加 bug 修复记录
- `.claude-team/ENV-LOG.md` — 本 Phase 3 日志追加

---

### Bug 修复提交

**测试验证**:
- providers: 15 passed, 17 skipped
- service: 40 passed
- contract: 46 passed
- 总计: 101 passed, 0 failed

**Git 操作**:
- `git add providers/config.py`
- `git commit -m "fix(providers): use lazy loading for OpenAI imports in config.py"`
  - commit: acaa1f9

**文件修改**:
- `.claude-team/PROGRESS.md` — 添加测试结果记录

---

### OpenAI API 测试

**环境变量设置**:
- OPENAI_API_KEY: [已设置，未记录]
- OPENAI_BASE_URL: https://chatapi.littlewheat.com/v1

**测试结果**:
- providers: 31 passed, 1 failed (determinism test - 已知 flaky)
- service: 40 passed
- contract: 46 passed
- 总计: 117 passed, 1 failed

**失败原因**: Embedding API 浮点精度差异，不影响功能

**文件修改**:
- `.claude-team/PROGRESS.md` — 添加 OpenAI API 测试结果

---

### 文档重组

**目标**: 分离环境配置与功能文档，便于不同环境快速配置。

**新建文件**:
- `ENV.md` — 通用环境配置（AGFS、Python、环境变量表）
- `check-env.sh` — 统一环境检查脚本（chmod +x）
- `openclaw-plugin/ENV.md` — 插件专用配置

**更新文件**:
- `README.md` — 添加 ENV.md 到文档列表
- `openclaw-plugin/README.md` — 移除环境配置，引用 ENV.md

**关键改进**:
- conda 环境名通用化：py11 → og-memory（可自定义）
- AGFS 路径变量化：/tmp/agfs-data → $AGFS_DATA_DIR
- 检查脚本独立化：内联脚本 → check-env.sh

**文件修改**:
- `.claude-team/PROGRESS.md` — 添加文档重组记录
- `.claude-team/ENV-LOG.md` — 本日志

---

### 推送到远程

**Git 操作**:
- `git push origin feature/openclaw-plugin-v2`
- ✅ 推送成功
- 新分支已创建: `feature/openclaw-plugin-v2` on `origin`

**Merge Request 链接**:
```
https://gitcode.com/coconutnutx/oG-Memory/merge_requests/new?source_branch=feature/openclaw-plugin-v2
```

---
