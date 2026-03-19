# 项目进度追踪板

> **规则**：所有角色在以下情况必须往本文件追加记录：
> - 开始一个阶段
> - 完成一个阶段
> - 遇到阻塞需要讨论
> - 需要 human 介入
> - 做出了重要决策
> 
> **格式**：每条记录用 `###` 标题，包含时间、角色、类型标签。
> **追加位置**：始终添加到文件末尾 `<!-- APPEND HERE -->` 标记之前。

---

## 项目状态总览

| 阶段 | 状态 | 负责 | 备注 |
|------|------|------|------|
| Phase 0: 环境准备 | ✅ 完成 | @env | 已完成环境清理，创建了LocalFS适配器 |
| Phase 1: 插件脚手架 | ✅ 完成 | @plugin-dev | 插件已创建并成功加载 |
| Phase 2: 桥接集成 | ✅ 完成 | @integrator | 已修复代理问题，成功写入AGFS |
| Phase 3: 端到端验证 | ✅ 完成 | @tester | 通过OpenClaw测试完整流程 |

> 状态图例：⬜ 未开始 | 🔄 进行中 | ✅ 完成 | ❌ 阻塞 | ⏸️ 暂停

---

## 关键路径信息

> 由各阶段执行者填入，后续阶段依赖这些信息。

| 信息项 | 值 | 由谁填 |
|--------|-----|--------|
| oG-Memory 项目路径 | `/data/Workspace2/oG-Memory` | @env |
| Python (py11) 可执行文件路径 | `/home/aaa/miniconda3/envs/py11/bin/python` | @env |
| Node 版本 | v22.22.1 | @env |
| OpenClaw 版本 | 2026.3.13 | @env |
| Go 版本 | 1.26.1 | @env |
| agfs 存储路径 | `~/.og-memory/data` (LocalFS) | @env |
| agfs 是否需要后台进程 | ❌ 不需要 (使用 LocalFS 适配器) | @env |
| oG-Memory 写入 API 函数签名 | `MemoryWriteAPI.write_memory(candidate, ctx)` / `commit_session(messages, ctx)` | @env (从代码读取) |
| 插件安装方式 | `openclaw plugins install -l <path>` | @env (从 SHARED-KNOWLEDGE 读取) |
| 需要的环境变量 | (LocalFS 无需特殊环境变量，Provider 需 OPENAI_API_KEY) | @env |

---

## 记录区

### [INIT] @lead — 项目启动

项目开始。按 Phase 0 → 1 → 2 → 3 顺序推进。

### [STARTED] @integrator — Step 1: 理解 oG-Memory 写入接口

**发现的核心代码**：

1. **高级写入API** (service/api.py):
```python
from service.api import MemoryWriteAPI
from fs.agfs_adapter import AGFSContextFS
from pyagfs import AGFSClient
from providers.config import ProviderConfig

# 初始化
client = AGFSClient(api_base_url="http://localhost:8080")
fs = AGFSContextFS(client)
config = ProviderConfig.from_env()  # 从环境变量读取
llm = config.create_llm()
api = MemoryWriteAPI(fs=fs, llm=llm)

# 写入候选记忆
api.write_memory(candidate, ctx)

# 从对话消息抽取并写入
api.commit_session(messages, ctx)
```

2. **AGFS 需要** - 从 verify_agfs_real.py:
```bash
cd agfs/agfs-server
./build/agfs-server -c config.yaml
```
默认 URL: `http://localhost:8080`

3. **环境变量配置** (providers/config.py):
```bash
export CONTEXTENGINE_PROVIDER=openai
export OPENAI_API_KEY="sk-xxx"
export OPENAI_BASE_URL="https://chatapi.littlewheat.com/v1"
# 可选: EMBEDDING_MODEL, EMBEDDING_DIMENSION
```

### [DISCOVERY] @integrator — AGFS 来源确认

**Git 历史分析**：
- `agfs/` 曾经被 git 跟踪
- Commit `1b80650`: "chore: remove agfs/ from git tracking (per .gitignore)"
- 原因：AGFS 是外部依赖，有自己的独立仓库

**OpenViking AGFS 仓库**：
- 主仓库：https://github.com/volcengine/OpenViking
- AGFS 是 OpenViking 项目的一部分（Go 编写）
- 官方文档：https://volcengine-openviking.mintlify.app/
- 最新版本：v0.2.1

**重要发现**：
根据 GitHub 讨论 #293，**AGFS 支持 binding client 模式**，无需启动独立的 HTTP 服务！
> "agfs支持binding client运行，本地不用额外启动agfs-server"

这意味着 `pyagfs` 可能已经支持直接调用，不需要服务器进程。

### [ASK-HUMAN] @integrator — AGFS 集成方式确认

基于以上发现，请确认：

1. **是否需要克隆 AGFS 仓库？**
   - 如果是，应该克隆到 `agfs/` 目录吗？
   - 还是用 pyagfs 的 binding client 模式？

2. **当前 pyagfs 版本是否支持 binding client？**
   - 已安装版本：pyagfs 1.4.0
   - 是否需要更新？

3. **MVP 开发阶段的最简方案是什么？**
   - 选项A：克隆完整 AGFS 并运行服务器
   - 选项B：使用 pyagfs binding client（如果支持）
   - 选项C：使用 mock 存储暂时绕过 AGFS

请告诉我应该采用哪种方案，我会据此创建桥接脚本。

**参考链接**：
- [OpenViking GitHub](https://github.com/volcengine/OpenViking)
- [GitHub 讨论 #293](https://github.com/volcengine/OpenViking/discussions/293)

---

代码显示 AGFS 需要运行在 `http://localhost:8080`。

请问：
1. AGFS 服务器当前是否已运行？
2. 如果未运行，需要如何启动？
3. agfs/ 目录是空的，是否需要编译或下载 AGFS？

请确认 AGFS 状态，我再继续创建桥接脚本。

---

开始执行环境诊断清单。用户环境：Windows WSL, conda py11, 之前有开发残留。

### [NOTE] @env — 环境诊断结果汇总

#### ✅ 工具版本检查
- Node: v22.22.1 (要求 22+, 满足)
- npm: 10.9.4
- Go: 1.26.1
- Python: 3.11.15 (conda py11)
- Python路径: `/home/aaa/miniconda3/envs/py11/bin/python`

#### ✅ OpenClaw 状态
- OpenClaw: 2026.3.13 (最新)
- 已安装，位于: `/home/aaa/.npm-global/bin/openclaw`

#### ⚠️ 发现现有插件残留
发现已存在 `memory-og` 插件位于 `~/.openclaw/extensions/memory-og/`，包含：
- 完整的 TypeScript 插件代码 (index.ts, bridge-client.ts)
- Python bridge server (bridge/server.py, bridge/memory_bridge.py)
- package.json, openclaw.plugin.json 等

**问题**: 该插件的 bridge server 期望 oG-Memory 项目位于 `~/.openclaw/extensions/`（代码中使用相对路径 `"../.."` 引用项目根目录），但实际项目位于 `/data/Workspace2/oG-Memory/`。路径不匹配！

#### ✅ Git 状态
- 当前分支: `feature/openclaw-plugin` (已存在)
- 有 `phase1_write` 分支可拉取
- untracked: `.claude-team/` (新增的团队协调文件)

#### ❓ agfs 状态
- agfs/ 目录存在但为空
- 无 go.mod 文件
- 需要确认 agfs 是否需要单独编译/运行

#### 📋 待确认关键路径信息

| 信息项 | 值 | 由谁填 |
|--------|-----|--------|
| oG-Memory 项目路径 | `/data/Workspace2/oG-Memory` | @env |
| Python (py11) 可执行文件路径 | `/home/aaa/miniconda3/envs/py11/bin/python` | @env |
| Node 版本 | v22.22.1 | @env |
| OpenClaw 版本 | 2026.3.13 | @env |
| Go 版本 | 1.26.1 | @env |
| agfs 存储路径 | _待确认_ | @env |
| agfs 是否需要后台进程 | _待确认_ | @env |

### [ASK-HUMAN] @env — 现有插件处理方案决策

发现之前开发的 `memory-og` 插件位于 `~/.openclaw/extensions/memory-og/`，但该插件与当前 oG-Memory 项目路径不匹配。

**请选择处理方案**：

1. **复用现有插件** - 修改现有插件的路径配置，使其指向 `/data/Workspace2/oG-Memory`
2. **重建插件** - 在 oG-Memory 项目内创建新的插件目录，删除或禁用旧插件
3. **其他方案** - 如果你有什么想法

另外，关于 agfs：
- agfs/ 目录存在但为空，是否需要单独初始化或编译？
- agfs 是否需要后台进程运行？

请告诉我你的选择，我再继续 Phase 0。

### [DECISION] @env — 采用 LocalFS 适配器而非 AGFS 服务器

用户要求更简单直接的方案。AGFS 服务器未安装且 agfs/ 目录为空。

**解决方案**：
- 创建 `fs/localfs_adapter/LocalContextFS` - 直接使用本地文件系统
- 存储路径：`~/.og-memory/data/accounts/...`
- 无需后台进程，无需编译 Go 代码
- 目录结构与 AGFS 一致，便于将来迁移

### [DONE] @env — Phase 0 环境诊断完成

**清理工作**：
- ✅ 删除旧插件残留 `~/.openclaw/extensions/memory-og/`
- ✅ 创建 LocalFS 适配器 (`fs/localfs_adapter/`)

**环境状态**：
- Node: v22.22.1 ✓
- Go: 1.26.1 ✓ (暂时不需要)
- Python: 3.11.15 (conda py11) ✓
- OpenClaw: 2026.3.13 ✓
- Git: `feature/openclaw-plugin` 分支 ✓

**关键决策**：
- 使用 LocalFS 适配器代替 AGFS 服务器
- 存储路径：`~/.og-memory/data`
- 无需后台进程

可以进入 Phase 1。

### [DONE] @plugin-dev — Phase 1 脚手架完成

**插件创建**：
- 目录：`/data/Workspace2/oG-Memory/openclaw-plugin/`
- 文件：package.json, openclaw.plugin.json, index.ts, tsconfig.json

**插件状态**：
- `openclaw plugins info og-memory`: ✅ loaded
- Tools: `og_memory_write`

**依赖处理**：
- 创建符号链接到OpenClaw的@sinclair/typebox（避免重复安装）
- 保持插件目录轻量

**下一步**：Phase 2 需要实现Python桥接层，让工具真正写入AGFS存储。

### [HUMAN-INPUT] — AGFS 安装方式确认

用户说明：
- AGFS 是 Go 编写的 HTTP 服务器，监听 localhost:8080
- pyagfs 只是 HTTP 客户端，不支持 binding client 模式
- 不用 Docker，从源码编译：git clone + go build
- 如果网络问题：设置 GOPROXY=https://goproxy.cn,direct

### [DONE] @integrator — AGFS 安装和启动成功

**安装方式**：从源码编译
- 仓库：https://github.com/c4pt0r/agfs
- 克隆到：/tmp/agfs-build/
- 编译命令：`go build -o ~/.local/bin/agfs-server ./cmd/server`
- 二进制位置：~/.local/bin/agfs-server (36MB)

**启动方式**：
- 配置文件：/tmp/agfs-config.yaml
- 命令：`~/.local/bin/agfs-server -c /tmp/agfs-config.yaml &`
- PID: 283128
- 监听：:8080

**挂载点**：
- /local → /tmp/agfs-data (localfs)
- /memfs → 内存文件系统
- /dev → DevFS

**验证**：
```bash
http_proxy= https_proxy= curl http://localhost:8080/api/v1/health
# 返回: {"status":"healthy","version":"1.4.0"}
```

**下一步**：创建 Python 桥接脚本

---

**问题描述**：
- AGFS 服务器运行正常（health check 通过）
- memfs 挂载点可用（可以创建文件）
- 但读取不存在的文件时返回 502 而不是 404
- oG-Memory 的 `exists()` 方法调用 `read()` 导致 502 错误

**错误栈**：
```
AGFSClientError: Bad Gateway - backend service unavailable
URL: http://localhost:8080/api/v1/files?path=/memfs/accounts/default/...
```

**可能原因**：
1. AGFS memfs 后端在读取不存在的文件时返回 502
2. AGFS 配置问题（backend 未正确初始化）

**需要决策**：
1. 尝试使用空的 mount_prefix（根挂载）？
2. 修复 AGFS 配置？
3. 在 oG-Memory 代码中处理 502 作为 "文件不存在"？

### [HUMAN-INPUT] — AGFS 502 错误处理

用户需要确认如何处理 AGFS 的 502 错误问题。

---

### [DIAGNOSIS] @integrator — AGFS 后端诊断结果

**诊断命令和输出**：

1. 根目录挂载点：
```bash
curl http://localhost:8080/api/v1/files?path=/
# 返回: {"error": "read: /: not found"}
```

2. memfs 挂载和写入：
```bash
curl -X PUT "http://localhost:8080/api/v1/files?path=/memfs/test" -d "hello"
# 返回: {"message":"Written 5 bytes"}  ✅ 写入成功！

curl http://localhost:8080/api/v1/files?path=/memfs
# 返回: {"error": "is a directory: /"}  ⚠️ 目录列出失败
```

3. kvfs 状态：
```bash
# /api/v1/plugins 显示 kvfs 未挂载（mounted_paths: null）
# 尝试读写 kvfs 返回 "not found"
```

**发现**：
- memfs **可以写入**但**不能列出目录**
- 问题可能在于 AGFS 目录列表 API 或 memfs 实现
- oG-Memory 的 `exists()` 方法调用 `read()` 可能不会触发这个问题

**完整写入流程测试**：
```bash
echo '{"content": "test memory - I like coffee", ...}' | python bridge/write_memory.py
# 错误：AGFSClientError: Bad Gateway - backend service unavailable
# 触发点：exists() → read() → .meta.json (不存在文件) → 502 而不是 404
```

**根本原因**：
- AGFS memfs 在读取不存在的文件时返回 502 而不是 404
- oG-Memory 依赖正确的 404 响应来判断文件不存在
- 这是 AGFS 的 bug，不是 oG-Memory 的问题

### [DISCOVERY] @integrator — mount_prefix 定义位置

**代码位置**：`fs/agfs_adapter/agfs_context_fs.py:218`

```python
def __init__(self, client: AGFSClient, mount_prefix: str = "/local"):
    """Initialize AGFS context filesystem adapter.

    Args:
        client: AGFS Python SDK client instance
        mount_prefix: AGFS mount point prefix (default: "/local")
                     Use "" for root mount, "/local" for localfs, etc.
    """
    self._client = client
    self._mount_prefix = mount_prefix.rstrip("/")
```

**关键发现**：
- `mount_prefix` 是**构造函数参数**，不是硬编码
- 默认值是 `"/local"`
- 在桥接脚本中我设置成了 `mount_prefix="/memfs"`，然后又改成 `mount_prefix=""`
- 这是正确的使用方式，但 memfs 后端本身有问题

**问题链条**：
1. 桥接脚本: `fs = AGFSContextFS(client, mount_prefix="/memfs")`
2. AGFS 配置: memfs 挂载在 `/memfs`
3. 问题: memfs 无法正确处理文件读取（返回 502）
4. 结果: oG-Memory exists() → read() → 502 错误

### [HUMAN-INPUT] — AGFS 后端问题决策

AGFS memfs 后端存在问题：读取不存在文件返回 502 而不是 404。

请选择处理方案：

**选项 A**: 继续诊断 AGFS
- 尝试其他 AGFS 后端（localfs 配置正确路径？）
- 检查是否需要更新 AGFS 版本
- 可能需要更多时间调试

**选项 B**: 使用 sqlfs（SQLite 后端）
- AGFS 内置、稳定支持
- 持久化到本地文件
- 配置相对简单

**选项 C**: MVP 期间使用 mock 存储
- 快速完成桥接功能验证
- 暂时不依赖 AGFS
- Phase 3 再集成真实 AGFS

请告诉我应该采用哪个选项。

### [TEST RESULTS] @integrator — AGFS 后端测试

**测试 1: /local (默认后端)**
```bash
curl -X PUT "http://localhost:8080/api/v1/files?path=/local/test.txt" -d "hello"
# 返回: {"message":"Written 5 bytes"} HTTP_CODE:200 ✅

curl "http://localhost:8080/api/v1/files?path=/local/test.txt"
# 返回: hello HTTP_CODE:200 ✅

curl "http://localhost:8080/api/v1/files?path=/local/not-exist.txt"
# 返回: {"error":"no such file: /not-exist.txt"} HTTP_CODE:500 ❌ (期望404)
```

**测试 2: /kvfs**
```bash
curl -X PUT "http://localhost:8080/api/v1/files?path=/kvfs/keys/test" -d "hello"
# 返回: {"error":"write: /kvfs/keys/test: not found"} HTTP_CODE:404 ❌

# /api/v1/plugins 显示: kvfs mounted_paths: null (未挂载)
```

**结论**: AGFS 后端配置存在根本问题，无法提供正确的文件系统语义。


### [DECISION] @integrator — 使用 /local 后端 + 禁用代理

用户决策：
- **不换后端**：/local 后端读写功能正常，只是状态码非标准（500 而非 404）
- **在适配层处理**：AGFSContextFS 的 `_read_file` 已经检查 "no such file" 消息，能正确处理
- **禁用代理**：根本问题是 `requests` 库使用代理导致 500 → 502 转换
- **使用 mount_prefix="/local"**：默认值，无需修改

**修复方案**：
在 `bridge/write_memory.py` 开头添加：
```python
# Disable proxy for local AGFS requests
os.environ.pop('http_proxy', None)
os.environ.pop('https_proxy', None)
os.environ.pop('HTTP_PROXY', None)
os.environ.pop('HTTPS_PROXY', None)
```

### [DONE] @integrator — Phase 2 桥接完成

**修复内容**：
1. 禁用代理环境变量（解决 502 错误）
2. 使用 `/local` 后端（mount_prefix="/local"）

**测试结果**：
```bash
echo '{"content": "I like coffee", "category": "preference", "tags": []}' | \
  /home/aaa/miniconda3/envs/py11/bin/python openclaw-plugin/bridge/write_memory.py

# 返回: {"ok": true, "message": "Memory written successfully",
#        "uri": "ctx://default/users/default/memories/preference/preference_749d4ca0",
#        "action": "create"}
```

**验证文件创建**：
```bash
ls /tmp/agfs-data/accounts/default/users/default/memories/preference/preference_749d4ca0/
# .abstract.md  .meta.json  .overview.md  .relations.json  content.md
```

**下一步**：Phase 3 端到端测试（通过 OpenClaw 调用插件）

### [DONE] @tester — Phase 3 端到端验证完成

**修复的问题**：
1. **OpenClaw 工具配置**：设置 `tools.profile=full` 以包含插件工具
2. **API Key 配置**：
   - 在 `openclaw.plugin.json` 中添加 `apiKey` 字段
   - 从 Agent 模型配置中提取 API key
   - 在 Python 桥接进程环境中设置 `OPENAI_API_KEY`
3. **增强日志**：在 index.ts 中添加详细的错误和调试日志

**测试结果**：
```bash
openclaw agent --agent main --local --message "Save to oG-Memory: I drink green tea"

# ✅ 成功：Memory written successfully
# ✅ 文件创建：/tmp/agfs-data/.../preference_869ae79c/
# ✅ 内容正确：I drink green tea in the afternoon
```

**中英文测试**：
- 英文："Save to oG-Memory: I drink green tea" ✅
- 中文："记住：我喜欢深烘焙咖啡" ✅

**验证清单**：
- ✅ OpenClaw 加载插件（`og-memory` loaded）
- ✅ Agent 可以调用 `og_memory_write` 工具
- ✅ TypeScript → Python 桥接正常
- ✅ Python 写入 AGFS 成功
- ✅ 文件正确创建在 `/tmp/agfs-data/`
- ✅ 中英文内容都支持

**项目状态**：Phase 0-3 全部完成 🎉

