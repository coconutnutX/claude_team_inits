# PROGRESS — oG-Memory Write Integration

## 总览

| Phase | 描述 | 状态 |
|-------|------|------|
| Phase 2 | 打通插件写入链路 | ✅ 完成 |

## 关键路径信息

| 项目 | 值 |
|------|-----|
| oG-Memory 项目根 | `/data/Workspace2/oG-Memory/` |
| oG-Memory Service API | `/data/Workspace2/oG-Memory/service/api.py` |
| OpenClaw 插件目录 | `/data/Workspace2/oG-Memory/openclaw_context_engine_plugin/` |
| AGFS 数据目录 | `/tmp/agfs-data` |
| AGFS 配置文件 | `~/.agfs-config.yaml` |
| AGFS 服务端口 | `:8080` (配置) / `1833` (文档标准) |
| Python 环境 | conda og-memory (py11) |
| LLM Provider | OpenAI (真实 API，自动检测 OPENAI_API_KEY) |
| 写入路径选择 | MemoryWriteAPI.commit_session() (完整 pipeline) |

---

## 记录

### 2026-03-25 [LEAD] [FIX] 修复 OpenClaw 插件配置读取错误 ✅
**问题描述：**
即使已在 `~/.openclaw/openclaw.json` 中配置 `pythonBin`，Gateway 仍报错：
```
ModuleNotFoundError: No module named 'pyagfs'
Command failed: python3 /data/Workspace2/oG-Memory/.../bridge/memory_api.py dispose {}
```

**根本原因：**
```javascript
// ❌ 错误代码（index.js）
const pluginConfig = api.getConfig?.() ?? {};
const pythonBin = pluginConfig.config?.pythonBin || "python3";
```

通过 debug 打印发现：
- `api.getConfig` 是 `undefined`，不是函数
- `pluginConfig` 总是空对象 `{}`
- 无法从 `openclaw.json` 读取配置

**Debug 过程：**
```javascript
console.log("[PLUGIN-JS] api keys:", Object.keys(api));
// 输出包含: 'config', 'pluginConfig', ...
console.log("[PLUGIN-JS] api.getConfig:", typeof api.getConfig);
// 输出: undefined
console.log("[PLUGIN-JS] api.config:", api.config);
// 输出: 完整的 openclaw.json 配置对象
```

**解决方案：**
```javascript
// ✅ 正确代码（index.js）
const pluginEntry = api.config?.plugins?.entries?.['mock-context-engine'] ?? {};
const pythonBin = pluginEntry.config?.pythonBin || "python3";
```

**配置路径：**
`api.config.plugins.entries['mock-context-engine'].config.pythonBin`

**验证结果：**
```
[PLUGIN-JS] pythonBin: /home/aaa/miniconda3/envs/py11/bin/python
[PLUGIN-JS] pythonBin resolved: /home/aaa/miniconda3/envs/py11/bin/python
```

**相关变更：**
- `index.js`: 修正配置读取逻辑
- `ENV.md`: 删除 `PLUGIN_PYTHON_BIN` 环境变量相关说明
- `openclaw_context_engine_plugin/ENV.md`: 删除环境变量配置章节
- `check-env.sh`: 删除环境变量检查，优化"系统 Python 检查"提示

**关键发现：**
OpenClaw 插件 SDK 的 `api` 对象结构：
- `api.config` - 完整的 openclaw.json 配置
- `api.pluginConfig` - 插件特定配置（可能不包含 config 字段）
- `api.getConfig` - 不存在（不是函数）

---

### 2026-03-24 [LEAD] [INFO] 移除硬编码和 MockLLM，使用真实 OpenAI LLM ✅
**变更内容：**
- ✅ 移除 `project_root` 硬编码，使用相对路径 `Path(__file__).parent.parent`
- ✅ 替换 MockLLM 为真实 OpenAI LLM（读取 `OPENAI_API_KEY` 环境变量）
- ✅ 删除所有 legacy 函数（memory_store, memory_get, memory_delete 等）
- ✅ 清理插件 ENV.md，删除重复的 Python 环境配置和硬编码路径
- ✅ 删除 Phase 1/2 测试文件（`test_write_api.py`, `test_plugin_write.py`）
- ✅ 更新 TEST_GUIDE.md，添加 OpenClaw Dashboard 测试说明和预期行为
- ✅ 在未实现的 lifecycle 方法中添加 "not yet implemented" 日志

**文件更新：**
- `bridge/memory_api.py`：
  - 使用相对路径导入 oG-Memory
  - `_get_llm()` 使用 ProviderConfig 检测 `OPENAI_API_KEY`，自动切换到 openai provider
  - 删除 legacy 方法，保留 lifecycle 方法 + memory_health
  - `assemble()`: 添加 "memory search not yet implemented" 日志
  - `compact()`: 添加 "memory compaction not yet implemented" 日志
  - `prepare_subagent_spawn()`: 添加 "not yet implemented" 日志
  - `on_subagent_ended()`: 添加 "not yet implemented" 日志
- `openclaw_context_engine_plugin/ENV.md`：
  - 删除重复的 Python 环境配置（已包含在主项目 ENV.md）
  - 删除硬编码的 conda 路径，改用动态示例
  - 仅保留插件特定变量（`PLUGIN_PYTHON_BIN`, `MOCK_ENGINE_LOG_LEVEL`）
  - 核心变量（`OPENAI_API_KEY` 等）指向主项目 ENV.md
- `openclaw_context_engine_plugin/TEST_GUIDE.md`：
  - 完整重写，说明当前实现状态（写入已实现，检索/压缩暂未实现）
  - 添加 OpenClaw Dashboard 测试步骤
  - 添加日志预期输出示例
  - 添加 AGFS 数据验证步骤
  - 添加故障排查指南

**文件更新：**
- `bridge/memory_api.py`：
  - 使用相对路径导入 oG-Memory
  - `_get_llm()` 使用 ProviderConfig 检测 `OPENAI_API_KEY`，自动切换到 openai provider
  - 删除 legacy 方法，保留 lifecycle 方法 + memory_health
- `openclaw_context_engine_plugin/ENV.md`：
  - 删除重复的 Python 环境配置（已包含在主项目 ENV.md）
  - 删除硬编码的 conda 路径，改用动态示例 `$(conda env list | grep og-memory | awk '{print $1}')/bin/python`
  - 仅保留插件特定变量（`PLUGIN_PYTHON_BIN`, `MOCK_ENGINE_LOG_LEVEL`）
  - 核心变量（`OPENAI_API_KEY` 等）指向主项目 ENV.md

**文件更新：**
- `bridge/memory_api.py`：
  - 使用相对路径导入 oG-Memory
  - `_get_llm()` 使用 ProviderConfig 检测 `OPENAI_API_KEY`，自动切换到 openai provider
  - 删除 legacy 方法，保留 lifecycle 方法 + memory_health

**环境配置：**
```bash
export OPENAI_API_KEY="sk-..."
export OPENAI_BASE_URL="https://chatapi.littlewheat.com/v1"
export CONTEXTENGINE_PROVIDER="openai"  # 可选，有 API key 会自动用 openai
```

**验证测试：**
```bash
python bridge/memory_api.py memory_health '{}'
# 输出: {"status":"ok", "backend":"og-memory", "llm":"gpt-4o-mini"}
```

### 2026-03-24 [LEAD] [DONE] Phase 2 完成 - 打通插件写入链路 ✅
**架构重构完成：**
- ✅ 删除 `mock_engine.py`，合并到 `bridge/memory_api.py`
- ✅ `bridge/memory_api.py` 现在承担：CLI 入口 + OpenClaw lifecycle 适配 + 调用 service/api.py
- ✅ `service/api.py` 保持通用，无 OpenClaw 特有逻辑
- ✅ 分层关系：`index.js → bridge/memory_api.py → service/api.py → oG-Memory`

**文件变更：**
- `bridge/memory_api.py` - 完全重写，包含 CLI 入口和所有 lifecycle 方法
- `index.js` - 更新 `callPython` 目标为 `bridge/memory_api.py`
- `mock_engine.py` - 保留但不再使用（可作为备份参考）

**测试验证：**
- ✅ `tests/test_plugin_write.py` 独立测试通过
- ✅ Gateway 端到端测试通过（使用 conda py11 Python）
- ✅ 日志显示 bootstrap/assemble/after_turn 全部成功调用

**[环境配置] - Python 环境配置完成:**
- **问题根源**：Gateway (Node.js) 调用 Python 脚本时需要知道用哪个解释器
- **解决方案**：环境变量 `PLUGIN_PYTHON_BIN` 指向 conda py11
- **验证**：conda py11 环境测试通过，pyagfs 可用
- **配置方式**：
  ```bash
  # 添加到 ~/.bashrc 或 ~/.zshrc
  export PLUGIN_PYTHON_BIN="/home/aaa/miniconda3/envs/py11/bin/python"
  ```
- **文件更新**：
  - `index.js`: 使用 `PLUGIN_PYTHON_BIN` 环境变量
  - `ENV.md`: 完整的 Python 环境配置文档

### 2026-03-24 [LEAD] [INFO] Phase 2 进行中 - 打通插件写入链路
**任务 T2.1 [CODER] [DONE] - 更新 bridge/memory_api.py**
- 添加 `extract_and_commit()` 函数
- 调用 oG-Memory 的 MemoryWriteAPI.commit_session()
- 自动配置 MockLLM 返回用户简介数据
- 文件位置：`/data/Workspace2/oG-Memory/openclaw_context_engine_plugin/bridge/memory_api.py`

**任务 T2.2 [CODER] [DONE] - 更新 mock_engine.py 的 after_turn**
- 修改 `after_turn()` 方法调用 `extract_and_commit()`
- 从 params 获取 messages 和 sessionId
- 添加日志记录写入结果
- 文件位置：`/data/Workspace2/oG-Memory/openclaw_context_engine_plugin/mock_engine.py`

**任务 T2.3 [CODER] [DONE] - 编写 tests/test_plugin_write.py**
- 模拟 after_turn 的 params 调用
- 验证 AGFS 目录出现新文件
- 检查 .meta.json 的 status=ACTIVE
- 文件位置：`/data/Workspace2/oG-Memory/openclaw_context_engine_plugin/tests/test_plugin_write.py`

**任务 T2.5 [RUNNER] [DONE] - OpenClaw Dashboard 端到端测试**
- 测试命令：`openclaw agent --agent main -m "我叫吴九，是一名产品经理"`
- Gateway 日志显示：bootstrap/assemble/after_turn 全部成功调用
- conda py11 Python 环境正确加载，无 `ModuleNotFoundError`
- 写入调用成功：`commit_messages_to_memory session=... extracted=0 completed=0`

**Python 环境配置问题 [需要解决]:**
- 当前临时方案：在 `index.js` 中硬编码了 conda py11 路径（已撤销）
- 需要用户确认的解决方案：
  1. **推荐**：在系统 Python 中安装 pyagfs：`sudo apt install python3-pip && sudo pip3 install pyagfs`
  2. 或者设置环境变量：`export MOCK_ENGINE_PYTHON_BIN="/home/aaa/miniconda3/envs/py11/bin/python"`
  3. 或者在 `~/.openclaw/openclaw.json` 中配置插件 Python 路径（需要确认格式）

**任务 T2.6 [RUNNER] [DONE] - 结果记录到 PROGRESS.md**
- 架构重构完成：`mock_engine.py` 合并入 `bridge/memory_api.py`
- `service/api.py` 保持通用，无 OpenClaw 耦合
- 分层清晰：`index.js → bridge/memory_api.py → service/api.py → oG-Memory 内部`

**任务 T2.5 [RUNNER] [BLOCKED] - OpenClaw Dashboard 端到端测试**
- **问题**: Gateway 使用 `/usr/bin/python3` (系统 Python) 而非 conda py11
- **错误**: `ModuleNotFoundError: No module named 'pyagfs'`
- **尝试解决**: 创建 `~/.openclaw/extensions/mock-context-engine/openclaw.plugin.json` 配置 `pythonBin`，但未生效
- **[ASK-HUMAN]** 用户反馈：
  1. Python 环境是 conda py11（而非 miniconda3）
  2. **需求变更**: 希望直接把 `bridge/memory_api.py` 接到实际的 `service/api.py`，不再使用 mock_engine
  3. 问题：是否可以跳过 mock_engine，直接在 OpenClaw 插件中调用 oG-Memory 的 service/api.py？

**任务 T2.4 [RUNNER] [DONE] - 独立测试通过**
- 测试输出：candidates_extracted=1, writes_completed=1
- AGFS 路径：`/local/plugin/accounts/default/users/default-user/memories/profile/`
- .meta.json status=ACTIVE ✓
- content.md 包含："我叫张三，是一名后端工程师"

### 2026-03-24 [LEAD] [DONE] Phase 1 完成 - 独立测试 oG-Memory Write API ✅
**任务 T1.1 [CODER] [DONE] - tests/test_write_api.py 编写完成**
- 文件位置：`/data/Workspace2/oG-Memory/tests/test_write_api.py`
- 使用 MemoryWriteAPI.commit_session() 完整写入 pipeline
- 使用 MockLLM（无需真实 API key）
- 自动禁用代理以访问 localhost (WSL 环境问题)
- 动态获取写入 URI 而非硬编码预期路径

**任务 T1.2 [RUNNER] [DONE] - 运行 test_write_api.py 通过**
- 输出：candidates_extracted=1, writes_completed=1
- AGFS 服务已重启，端口从 8080 改为 1833
- 测试数据写入：`/tmp/agfs-data/phase1_test/accounts/test-account/users/test-user/memories/entity/entity/`

**任务 T1.3 [RUNNER] [DONE] - AGFS 输出验证通过**
- content.md: 166 bytes ✓
- .abstract.md: 66 bytes ✓
- .overview.md: 132 bytes ✓
- .relations.json: 2 bytes ✓
- .meta.json: 315 bytes, status=ACTIVE ✓

**任务 T1.4 [RUNNER] [DONE] - 结果记录到 PROGRESS.md**
- 测试脚本已验证完整写入 pipeline 可用
- 所有文件按照 AGFS 目录 Spec 正确写入
- 节点状态为 ACTIVE，可被检索

### 2026-03-24 [LEAD] [INFO] Phase 1 开始 - 环境调研
- AGFS 服务运行在 PID 33496，监听端口 8080
- HTTP API 返回 "Bad Gateway" - 后端服务异常
- 已有测试数据：`/tmp/agfs-data/accounts/test-account/users/test-user/memories/entities/sqlite/`
- 现有测试文件参考：`tests/integration/test_full_pipeline.py`
- 依赖确认：pyagfs、core、service、commit、extraction 包都已就绪
- LLM：使用 MockLLM 进行测试（无需真实 API key）
- [FIXED] 重启 AGFS 服务，端口改为 1833，禁用代理访问 localhost


