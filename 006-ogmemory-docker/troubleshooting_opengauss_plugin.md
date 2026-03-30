# oG-Memory OpenClaw Plugin 问题排查记录

**日期**: 2026-03-28

## 当前问题

### 主要问题：数据没有写入 aaa 账户到 openGauss

与 OpenClaw Gateway 对话后，数据应该写入 openGauss 向量数据库，但查询结果显示 `aaa` 账户没有记录。

## 已完成的修改

### 1. 环境配置

**文件**: `/data/Workspace2/oG-Memory/.env`

```bash
# 新增配置
OG_ACCOUNT_ID=aaa
OG_USER_ID=u-default
OG_AGENT_ID=main
```

### 2. 插件代码修改

#### 2.1 自动加载 .env 文件

**文件**: `openclaw_context_engine_plugin/bridge/memory_api.py`

**修改**: 在模块加载时自动加载 `.env` 文件

```python
# 新增代码（在 import sys 之后）
# Load .env file if exists
env_file = project_root / ".env"
if env_file.exists():
    for line in env_file.read_text().split('\n'):
        line = line.strip()
        if line and not line.startswith('#') and '=' in line:
            key, value = line.split('=', 1)
            # Only set if not already set
            if key not in os.environ:
                os.environ[key] = value
```

**问题**: 即使加载了 `.env`，插件仍在使用旧的静态值 `DEFAULT_ACCOUNT_ID`。

#### 2.2 动态读取账户配置

**文件**: `openclaw_context_engine_plugin/bridge/memory_api.py`

**修改**: 将静态 `DEFAULT_ACCOUNT_ID` 改为动态读取函数

```python
# 修改前
DEFAULT_ACCOUNT_ID = os.environ.get("OG_ACCOUNT_ID", "acct-demo")

# 修改后
def _get_account_config():
    """Dynamic read of account configuration from environment."""
    return {
        'account_id': os.environ.get("OG_ACCOUNT_ID", "acct-demo"),
        'user_id': os.environ.get("OG_USER_ID", "u-alice"),
        'agent_id': os.environ.get("OG_AGENT_ID", "main"),
    }

def _build_request_context(session_id: str) -> RequestContext:
    config = _get_account_config()
    return RequestContext(
        account_id=config['account_id'],
        user_id=config['user_id'],
        agent_id=config['agent_id'],
        ...
    )
```

### 3. 数据库日志增强

**文件**: `providers/vector_index/opengauss_index.py`

**修改**: 在 `upsert` 方法中添加详细日志

```python
# 新增日志
logger.info(f"[UPSERT] #{i+1}/{len(records)} | account={account_id} | table={self._table_name} | uri={uri} | level={level} | text={text_preview}...")
```

### 4. AGFS 写入日志

**文件**: `fs/agfs_adapter/agfs_context_fs.py`

**修改**: 在 `write_node` 方法开头添加日志

```python
# 新增日志
logger.info(f"[WRITE_NODE] account={ctx.account_id} user={ctx.user_id} uri={node.uri} category={node.category}")
```

### 5. Index Service 配置修复

**文件**: `examples/run_index_service.py`

**修改**: 传递 `fs` 和 `llm` 参数到 `IndexService`

```python
# 修改前
return IndexService(
    outbox_store=outbox_store,
    embedder=embedder,
    vector_index=vector_index,
    get_account_ids=get_account_ids,
    interval_seconds=interval_seconds,
    worker_count=worker_count,
)

# 修改后 - 添加 fs 和 llm 参数
llm_for_directory = openai_embedder_cls(...)
return IndexService(
    ...
    fs=fs,
    llm=llm_for_directory,
    ...
)
```

### 6. 索引模块修复

**文件**: `index/index_record_builder.py`

**修改**: 添加 `Final` 类型导入

```python
# 修改前
from typing import Optional, List
from core.models import ContextNode, IndexRecord

_REQUIRED_FILTER_KEYS: Final[tuple[str, ...]] = ("account_id", "owner_space")

# 修改后
from typing import Optional, List
from typing_extensions import Final
from core.models import ContextNode, IndexRecord

_REQUIRED_FILTER_KEYS: Final[tuple[str, ...]] = ("account_id", "owner_space")
```

**文件**: `index/__init__.py`

**修改**: 移除不存在的 `build_record_id` 导入

```python
# 移除的导入
"build_record_id",

# 移除的导出
"build_record_id",
```

**文件**: `index/outbox_worker.py`

**修改**: 使用 `IndexRecord.generate_id()` 而非存在的 `build_record_id`

```python
# 修改前
from index.index_record_builder import build_record_id

ids_to_delete = [
    build_record_id(event.uri, level)
    for level in [0, 1, 2]
]

# 修改后
ids_to_delete = [
    IndexRecord.generate_id(event.uri, level)
    for level in [0, 1, 2]
]
```

### 7. 配置路径修复

**文件**: `openclaw_context_engine_plugin/bridge/memory_api.py`

**修改**: 修正 `AGFS_MOUNT_PREFIX` 路径

```python
# 修改前
AGFS_MOUNT_PREFIX = "/local/plugin"

# 修改后
AGFS_MOUNT_PREFIX = "/local"
```

## 修改的文件统计

```
修改的文件（7个）：
  1. .env
  2. openclaw_context_engine_plugin/bridge/memory_api.py
  3. providers/vector_index/opengauss_index.py
  4. fs/agfs_adapter/agfs_context_fs.py
  5. index/index_record_builder.py
  6. index/__init__.py
  7. index/outbox_worker.py
  8. examples/run_index_service.py

总行数变化：208 insertions(+), 135 deletions(-)
```

## 遇到的问题

### 问题 1: Index Service 未启动

**症状**: 数据写入 AGFS 但 openGauss 数据库为空

**原因**: `examples/run_index_service.py` 需要单独启动作为后台进程

**解决**: 启动 Index Service
```bash
python examples/run_index_service.py --interval 10 --workers 1 > /tmp/index_service.log 2>&1 &
```

### 问题 2: Gateway 进程没有环境变量

**症状**: AGFS 中只有 `acct-test` 和 `default` 账户，没有 `aaa` 账户

**原因**: Gateway 进程（Node.js）没有继承启动 shell 的环境变量

**尝试的解决方法**:
1. 在插件中自动加载 `.env` 文件 ✅
2. 使用 `env` 命令传递环境变量到后台进程
3. 创建启动脚本 `/tmp/start_openclaw_with_env.sh`

**状态**: ⚠️ **未完全解决** - Gateway 进程的环境变量仍然为空

### 问题 3: 插件账户 ID 配置问题

**症状**: 即使加载了 `.env`，插件仍可能使用旧的默认值

**原因**: 静态变量在模块加载时设置，环境变量在模块加载后设置

**状态**: 部分解决 - 添加了动态读取函数 `_get_account_config()`

### 问题 4: opengauss/dev 分支拉取失败

**错误**: `fatal: couldn't find remote ref opengauss/dev`

**原因**: 分支名称可能在不同位置

**状态**: 未解决 - 分支已存在但拉取失败

### 问题 5: Index Service 日志问题

**症状**: Index Service 运行但没有输出

**原因**: 可能没有待处理的 outbox 事件

**状态**: 待观察

## 待验证

1. **Docker 部署方式**: 用户提到有人合入了通过 docker 部署 ogmem 的方式，需要了解部署配置

2. **环境变量传递**: 需要找到正确的方式让 Gateway 进程继承环境变量

## Git 操作

**当前分支**: opengauss/dev

**未提交的修改**: 已通过 `git stash` 暂存

**远程分支**: `opengauss/dev` 存在 `https://gitcode.com/opengauss/oG-Memory.git`

## 建议下一步

1. **了解 Docker 部署配置**: 查看 Docker Compose 或 docker run 命令中如何设置环境变量

2. **验证 Gateway 环境变量**: 在 Docker 环境中通过 `-e` 或 `--env-file` 设置

3. **测试数据写入**: 使用新的部署方式测试数据流是否正常

---

## Docker 架构变更说明

### 背景

2026-03-28 之后，oG-Memory 项目进行了重大架构改造，支持 Docker 容器化部署。这个变化彻底解决了本地开发模式下的环境变量传递和进程管理问题。

### 最近 3 个 Commit 变更

| Commit ID | 描述 | 主要影响 |
|-----------|------|---------|
| `8c03fab` | merge dev-326 into dev | 合并最新开发分支 |
| `8a9f339` | **改造ogmem以http形式提供api，构建ogmem单独镜像** | **核心变更**：HTTP API + 独立镜像 |
| `750116f` | add openclaw + oG-mem image | 添加 Docker 镜像支持 |

### 架构对比

#### 旧架构（本地开发模式）

```
┌─────────────────────┐
│  OpenClaw (本地)   │  ← npm 本地安装
│  - openclaw daemon │
└──────────┬──────────┘
           │ subprocess (execFileSync)
           ▼
┌─────────────────────┐
│  memory_api.py     │  ← CLI 桥接
│  (Python CLI)      │
└──────────┬──────────┘
           │
           ├─▶ AGFS Server (独立进程 :1833)  ← 手动启动
           ├─▶ IndexService (独立进程)        ← 手动启动
           └─▶ openGauss

**问题**：
- 需要手动启动 AGFS、IndexService 多个后台进程
- 环境变量管理复杂，Gateway 进程无法正确继承
- 插件和 oG-Memory 代码耦合紧密
```

#### 新架构（Docker 镜像模式）

```
┌─────────────────────┐
│  OpenClaw (本地)   │  ← npm 本地安装（可选 Docker）
│  - openclaw daemon │
└──────────┬──────────┘
           │ HTTP POST
           ▼
┌─────────────────────┐
│  oG-Memory 容器    │  ← Docker 容器
│  - HTTP API (:8090)│
│  - AGFS (:1833)    │  ← 打包在一起，自动启动
│  - IndexService     │
└──────────┬──────────┘
           │
           └─▶ openGauss

**优势**：
- oG-Memory 作为一个独立容器运行，内嵌 AGFS
- OpenClaw 通过 HTTP API 调用，无 subprocess 依赖
- 环境变量通过 Docker `-e` 传递，更清晰
- 支持多 OpenClaw 实例共享一个 oG-Memory
```

### 核心变化对比表

| 对比项 | 旧模式（本地） | 新模式（Docker） |
|--------|----------------|-----------------|
| **OpenClaw** | 本地 npm 安装 | 本地 npm 安装（可选 Docker） |
| **AGFS** | 单独进程，手动启动 | 打包在容器内，自动启动 |
| **IndexService** | 单独 Python 进程，手动启动 | 打包在容器内，自动启动 |
| **HTTP API** | 无，纯 CLI 调用 | Flask/Gunicorn (:8090) |
| **通信方式** | subprocess (execFileSync) | HTTP POST |
| **环境变量** | `.bashrc` + `.env` 文件 | Docker `-e` 参数 |
| **启动方式** | 手动启动多个进程 | 一个 `docker run` 命令 |
| **配置文件** | `Dockerfile.standalone` | 入口脚本 `entrypoint-standalone.sh` |

### Docker 部署文件

| 文件 | 说明 |
|------|------|
| `Dockerfile.standalone` | 独立镜像构建文件 |
| `docker/entrypoint-standalone.sh` | 容器入口脚本，依次启动 AGFS、IndexService、HTTP API |
| `docker/agfs-config.yaml` | 容器内 AGFS 配置 |
| `server/app.py` | HTTP API 服务（新增） |
| `server/memory_service.py` | 记忆服务核心（从 memory_api.py 提取） |
| `scripts/run_index_service.py` | 独立 IndexService 启动脚本 |
| `architecture/deployment_operations_guide.md` | 完整部署运维指南 |

### 环境变量映射

| 旧配置（本地模式） | 新配置（Docker 模式） | 说明 |
|-------------------|---------------------|------|
| `OG_ACCOUNT_ID` | `OG_ACCOUNT_ID` | 直接映射 |
| `OPENAI_API_KEY` | `OPENAI_API_KEY` | 直接映射 |
| `OPENAI_BASE_URL` | `OPENAI_BASE_URL` | 直接映射 |
| `OPENAI_EMBEDDING_MODEL` | `OPENAI_EMBEDDING_MODEL` | 直接映射 |
| `OPENGUASS_CONNECTION_STRING` | `OPENGUASS_CONNECTION_STRING` | 直接映射 |
| `AGFS_MOUNT_PREFIX` | `AGFS_MOUNT_PREFIX` | 容器内默认 `/local/plugin` |
| 手动启动 `agfs-server` | 容器内自动启动 | 入口脚本管理 |
| 手动启动 `IndexService` | 容器内自动启动 | 入口脚本管理 |

### Docker 启动命令示例

```bash
# 1. 启动 openGauss（如未运行）
docker run --name opengauss --privileged=true -d \
  -e GS_PASSWORD=Gauss@1234 \
  -e GS_USERNAME=aaa \
  -e GS_USER_PASSWORD=aaa@123123 \
  -p 8888:5432 \
  -v /data/opengauss_data:/var/lib/opengauss \
  opengauss:6.0.3

# 2. 构建 oG-Memory 镜像
docker build --network host -f Dockerfile.standalone -t ogmem-server:v3 .

# 3. 启动 oG-Memory 容器
docker run -d \
  --name ogmem-server \
  --network host \
  -v /data/ogmem-data:/data/agfs \
  -e OPENAI_API_KEY="sk-your-key" \
  -e OPENAI_BASE_URL="https://chatapi.littlewheat.com/v1" \
  -e OPENAI_EMBEDDING_MODEL="text-embedding-3-small" \
  -e "OPENGUASS_CONNECTION_STRING=postgresql://aaa:aaa%40123123@127.0.0.1:8888/aaa" \
  -e OG_ACCOUNT_ID="aaa" \
  -e INDEX_INTERVAL="15" \
  ogmem-server:v3

# 4. 健康检查
curl http://127.0.0.1:8090/api/v1/health
```

### OpenClaw 插件配置变化

#### 旧模式（subprocess）

```json
{
  "plugins": {
    "entries": {
      "og-memory-context-engine": {
        "config": {
          "pythonBin": "/path/to/python"
        }
      }
    }
  }
}
```

#### 新模式（HTTP fetch）

```json
{
  "plugins": {
    "entries": {
      "og-memory-context-engine": {
        "config": {
          "memoryApiBaseUrl": "http://127.0.0.1:8090",
          "accountMapping": {
            "default": "aaa"
          }
        }
      }
    }
  }
}
```

---

## 环境清理记录

### 清理日期：2026-03-28

### 已完成的清理

1. **停止后台进程**
   - agfs-server (PID 10812) ✓
   - openclaw-gateway (PID 170769, 178985) ✓

2. **移除 OpenClaw 插件**
   - `~/.openclaw/extensions/openclaw_context_engine_plugin` ✓
   - `~/.openclaw/extensions/og-memory` ✓
   - `~/.openclaw/extensions/mock-context-engine` ✓
   - 清理 `openclaw.json` 插件配置 ✓

3. **清理环境变量**
   - `.bashrc` 中的环境变量已注释
   - 当前会话中的变量在重新加载 `.bashrc` 后清除

4. **清理项目文件**
   - 删除 `/data/Workspace2/oG-Memory/.env` ✓
   - 保留 `.env.example` 作为参考

### 创建的配置文件

| 文件 | 说明 |
|------|------|
| `.claude-team/docker_setup_guide.md` | Docker 部署完整指南 |
| `.claude-team/cleanup_env.sh` | 环境清理脚本 |
| `.claude-team/start-docker-deployment.sh` | Docker 部署快速启动脚本 |
| `.claude-team/README.md` | 配置目录索引和状态 |

---

## 迁移建议

### 从本地模式迁移到 Docker 模式

1. **清理旧环境**
   ```bash
   bash .claude-team/cleanup_env.sh
   ```

2. **重新加载 bashrc**
   ```bash
   source ~/.bashrc
   ```

3. **启动 Docker 部署**
   ```bash
   bash .claude-team/start-docker-deployment.sh
   ```

4. **重新配置 OpenClaw 插件**
   ```bash
   openclaw plugin install /data/Workspace2/oG-Memory/openclaw_context_engine_plugin
   # 编辑 ~/.openclaw/openclaw.json，添加 memoryApiBaseUrl 配置
   ```

### 数据迁移（如有需要）

```bash
# 停止旧服务
pkill -f agfs-server
pkill -f index_service

# 复制数据
cp -r /tmp/agfs-data/accounts/* /data/ogmem-data/

# 启动新容器
docker run -d ...
```
