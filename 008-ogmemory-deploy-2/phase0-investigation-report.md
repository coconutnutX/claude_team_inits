# Context Engine 环境调查报告

> 调查时间：2026-04-01
> 调查环境：Windows + WSL2, Python 3.11 (conda), Docker

---

## 1. AGFS

### 1.1 二进制

- **路径**：`/home/aaa/.local/bin/agfs-server`
- **大小**：36,323,200 bytes (34.6 MB)
- **版本**：1.4.0
- **安装方式**：独立二进制，已通过官方 install.sh 安装到 `~/.local/bin/`

**`agfs-server --help` 完整输出**：
```
Usage of agfs-server:
  -addr string
    	Server listen address (will override addr in config file)
  -c string
    	Path to configuration file (default "config.yaml")
  -print-sample-config
    	Print a sample configuration file and exit
  -version
    	Print version information and exit
```

### 1.2 启动

- **本地启动命令**：`agfs-server -c <config_path>`
- **Docker 内启动命令**：`/opt/agfs/agfs-server -c /opt/agfs/config.yaml &`
- **当前运行状态**：运行中 (PID 532470)，命令为 `agfs-server -c agfs-config.yaml`
- **配置文件路径**：`/data/Workspace2/oG-Memory/docker/agfs-config.yaml`（后续 CLI 版本将放在项目根目录）
- **配置文件格式**：YAML

**配置文件完整内容**（原样粘贴，来自 `docker/agfs-config.yaml`）：
```yaml
server:
  address: ":1833"
  log_level: info

plugins:
  serverinfofs:
    enabled: true
    path: /serverinfo
    config:
      version: "1.0.0"

  localfs:
    enabled: true
    path: /local
    config:
      local_dir: /data/agfs
```

**配置字段说明**：
| 字段 | 含义 |
|------|------|
| `server.address` | 监听地址，`:1833` 表示所有接口的 1833 端口 |
| `server.log_level` | 日志级别 |
| `plugins.localfs.path` | AGFS 挂载路径前缀 `/local` |
| `plugins.localfs.config.local_dir` | 物理存储目录 `/data/agfs` |

### 1.3 网络

- **监听地址**：`0.0.0.0:1833`（所有接口）
- **监听端口**：1833
- **健康检查**：`GET http://localhost:1833/`
  - 返回：`{"service":"AGFS Server","status":"running","version":"1.4.0"}`
- **版本端点**：`GET http://localhost:1833/serverinfo/version`

### 1.4 存储

- **数据目录**：`/data/agfs`（由配置文件 `local_dir` 字段指定）
- **本地开发默认**：`/data/agfs`（当前运行实例使用）
- **startup.sh 默认**：`$PROJECT_ROOT/.agfs_data`（通过 `AGFS_DATA_DIR` 环境变量）
- **check-env.sh 默认**：`/tmp/agfs-data`
- **查看方式**：直接 `ls /data/agfs/` 或通过 `pyagfs` Python 客户端

### 1.5 代码中的引用

| 文件:行号 | 引用内容 | 硬编码/环境变量 |
|-----------|----------|----------------|
| `server/memory_service.py:103` | `os.environ.get("AGFS_BASE_URL", "http://localhost:1833")` | 环境变量，默认 `http://localhost:1833` |
| `server/memory_service.py:105` | `agfs_mount_prefix` 参数，默认 `"/local/plugin"` | 构造参数，硬编码默认值 |
| `Dockerfile:53` | `ENV AGFS_BASE_URL=http://127.0.0.1:1833` | Docker 环境变量 |
| `Dockerfile:54` | `ENV AGFS_MOUNT_PREFIX=/local/plugin` | Docker 环境变量 |
| `docker/entrypoint.sh:25` | `/opt/agfs/agfs-server -c /opt/agfs/config.yaml &` | 硬编码路径 |
| `scripts/startup.sh:46` | `AGFS_BASE_URL="${AGFS_BASE_URL:-http://localhost:1833}"` | 环境变量 |
| `scripts/deploy.sh:34` | `AGFS_BASE_URL` 未设置时默认 `http://localhost:1833` | 环境变量 |
| `tests/conftest.py:30` | `AGFS_BASE_URL = os.environ.get("AGFS_BASE_URL", "http://localhost:1833")` | 环境变量 |
| `examples/basic_usage.py:20` | `os.environ.get("AGFS_BASE_URL", "http://localhost:1833")` | 环境变量 |
| `.env` | `AGFS_BASE_URL=http://localhost:1833` | 环境变量 |

---

## 2. openGauss

### 2.1 启动

**实测可行的启动命令**（来自 `~/.bashrc` alias `ogon`）：
```bash
docker run --name opengauss --privileged=true -d \
  -e GS_PASSWORD=Gauss@1234 \
  -e GS_USERNAME=aaa \
  -e GS_USER_PASSWORD=aaa@123123 \
  -p 8888:5432 \
  -v /data/opengauss_data:/var/lib/opengauss \
  opengauss:6.0.3
```

- **镜像**：`opengauss:6.0.3`
- **容器名**：`opengauss`
- **查看日志**：`docker logs -f opengauss`（alias `oglog`）

### 2.2 网络

- **端口映射**：`8888:5432`（宿主机 8888 → 容器内 5432）
- **WSL2 访问方式**：`127.0.0.1:8888`（直接通过 localhost 访问映射端口）
- **容器内访问**：`localhost:5432`（容器内部进程使用）

### 2.3 认证

- **用户名**：`aaa`
- **密码**：`aaa@123123`
- **数据库**：`postgres`（openGauss 默认数据库）
- **管理员密码**：`Gauss@1234`（`GS_PASSWORD`，用于 `omm` 超级用户）

**连接串**（来自 `.env`）：
```
OPENGUASS_CONNECTION_STRING=host=127.0.0.1 port=8888 dbname=postgres user=aaa password=aaa@123123
```

### 2.4 表结构

**表创建在代码中完成**（`providers/vector_index/opengauss_index.py:126-154`），无独立 init.sql 文件。

**`vector_index` 表结构**：
```sql
CREATE TABLE IF NOT EXISTS vector_index (
    id          VARCHAR(16) PRIMARY KEY,      -- sha256(uri+level)[:16]
    uri         VARCHAR(512) NOT NULL,
    level       INTEGER NOT NULL,             -- 0=abstract, 1=overview, 2=content
    text        TEXT NOT NULL,
    embedding   vector(1536) NOT NULL,        -- 默认 1536 维，可通过 OPENGUASS_DIMENSION 配置
    filters     JSONB NOT NULL,               -- 含 account_id, owner_space, category, context_type
    metadata    JSONB NOT NULL DEFAULT '{}',  -- category, context_type, has_overview, has_content
    created_at  TIMESTAMPTZ DEFAULT NOW(),
    updated_at  TIMESTAMPTZ DEFAULT NOW()
);
```

**索引**：
```sql
CREATE INDEX idx_vector_index_account ON vector_index ((filters->>'account_id'));
CREATE INDEX idx_vector_index_level ON vector_index (level);
CREATE INDEX idx_vector_index_filters_gin ON vector_index USING GIN (filters);
CREATE INDEX idx_vector_index_embedding_hnsw ON vector_index USING hnsw (embedding vector_cosine_ops);
```

**需要安装的扩展**：无（openGauss 已内置向量能力，无需额外安装 pgvector）

### 2.5 连接方式

**进入容器**：
```bash
docker exec -it opengauss bash
```

**连接数据库**（容器内，注意 openGauss 的 gsql 路径可能不在标准 PATH 中）：
```bash
# gsql 可能需要用 su 切换到 omm 用户
docker exec -it opengauss su - omm -c "gsql -d postgres -p 5432"
```

**从 WSL2 通过 psycopg2 连接**：
```python
import psycopg2
conn = psycopg2.connect("host=127.0.0.1 port=8888 dbname=postgres user=aaa password=aaa@123123")
```

### 2.6 代码中的引用

| 文件:行号 | 引用内容 | 硬编码/环境变量 |
|-----------|----------|----------------|
| `providers/config.py:77` | `os.environ.get("OPENGUASS_CONNECTION_STRING")` | 环境变量，无默认值 |
| `providers/config.py:78` | `os.environ.get("OPENGUASS_DIMENSION", "1536")` | 环境变量，默认 `1536` |
| `providers/config.py:79` | `os.environ.get("OPENGUASS_TABLE_NAME", "vector_index")` | 环境变量，默认 `vector_index` |
| `providers/config.py:80` | `os.environ.get("OPENGUASS_POOL_SIZE", "5")` | 环境变量，默认 `5` |
| `providers/config.py:76` | `os.environ.get("VECTOR_DB_TYPE", "memory")` | 环境变量，默认 `memory`（非 opengauss！） |
| `.env` | `OPENGUASS_CONNECTION_STRING=host=127.0.0.1 port=8888 ...` | 环境变量 |

---

## 3. OpenClaw 插件

### 3.1 插件目录结构

**源码目录**（仓库内）：`/data/Workspace2/oG-Memory/openclaw_context_engine_plugin/`
```
openclaw_context_engine_plugin/
├── index.js               # 插件入口（ESM 模块）
├── package.json           # 插件元数据
├── openclaw.plugin.json   # OpenClaw 插件描述文件
└── bridge/
    ├── __init__.py
    └── memory_api.py      # Python CLI 桥接（subprocess 模式用）
```

**安装目录**：`~/.openclaw/extensions/og-memory-context-engine/`
- 安装时间：2026-03-28T02:01:24.542Z
- 版本：0.1.0

### 3.2 插件配置

**两种运行模式**：

| 模式 | 配置字段 | 说明 |
|------|---------|------|
| **HTTP** | `memoryApiBaseUrl` | 调用独立 oG-Memory HTTP 服务 |
| **subprocess** | `pythonBin` | 通过 Python 子进程调用 bridge/memory_api.py |

**地址读取优先级**：
1. `api.pluginConfig?.memoryApiBaseUrl`（openclaw.json 中的插件 config）
2. `process.env.OGMEM_API_URL`（环境变量）
3. 如果都为空 → 使用 subprocess 模式

**当前已安装的配置**（`~/.openclaw/openclaw.json` 中）：
```json
{
  "memoryApiBaseUrl": "http://127.0.0.1:8090"
}
```

**Docker 默认配置**（`docker/openclaw.json`）：
```json
{
  "pythonBin": "python3"
}
```
即 Docker 内默认使用 subprocess 模式。

### 3.3 安装行为

- 安装方式：`openclaw plugins install` 从本地路径安装
- 安装记录存储在 `~/.openclaw/openclaw.json` 的 `plugins.installs` 字段中
- 已安装插件来源路径：`/data/Workspace2/oG-Memory/openclaw_context_engine_plugin`

### 3.4 代码中的引用

| 文件:行号 | 引用内容 |
|-----------|----------|
| `openclaw_context_engine_plugin/index.js:69` | `api.pluginConfig?.memoryApiBaseUrl \|\| process.env.OGMEM_API_URL` |
| `openclaw_context_engine_plugin/index.js:71` | `api.pluginConfig?.pythonBin \|\| process.env.OG_PYTHON_BIN \|\| "python3"` |
| `openclaw_context_engine_plugin/bridge/memory_api.py:30` | `os.environ['no_proxy'] = "localhost,127.0.0.1"` |

---

## 4. 全局信息

### 4.1 目录结构

```
oG-Memory/
├── core/                    # 领域对象 + 接口
│   ├── models.py
│   ├── interfaces.py
│   ├── enums.py
│   ├── errors.py
│   ├── validation.py
│   └── logging_config.py
├── fs/                      # 文件系统适配
│   └── agfs_adapter/
│       └── agfs_context_fs.py
├── extraction/              # 候选抽取
│   └── tools.py
├── commit/                  # 写入编排
│   ├── context_writer.py
│   ├── candidate_pipeline.py
│   ├── merge_policies.py
│   ├── policy_router.py
│   ├── archive_builder.py
│   └── outbox_store.py
├── index/                   # 索引同步
│   ├── outbox_worker.py
│   ├── index_record_builder.py
│   ├── directory_event_handler.py
│   ├── directory_summarizer.py
│   ├── repair_job.py
│   └── scheduler.py
├── retrieval/               # 检索链路
│   ├── pipeline.py
│   ├── query_planner.py
│   ├── seed_retriever.py
│   ├── hierarchical_searcher.py
│   ├── assembly_service.py
│   ├── memory_read_service.py
│   ├── hotness.py
│   └── trace.py
├── providers/               # 外部能力适配
│   ├── config.py
│   ├── llm/
│   │   ├── openai_llm.py
│   │   └── mock_llm.py
│   ├── embedder/
│   │   ├── openai_embedder.py
│   │   └── mock_embedder.py
│   ├── vector_index/
│   │   ├── opengauss_index.py
│   │   └── in_memory_index.py
│   └── relation_store/
│       └── agfs_relation_store.py
├── service/                 # 服务层
│   ├── api.py
│   ├── index_service.py
│   └── memory_fs.py
├── server/                  # HTTP API
│   ├── app.py              # Flask 入口
│   └── memory_service.py   # 业务逻辑
├── openclaw_context_engine_plugin/  # OpenClaw 插件
│   ├── index.js
│   ├── package.json
│   ├── openclaw.plugin.json
│   └── bridge/memory_api.py
├── docker/                  # Docker 配置
│   ├── agfs-config.yaml
│   ├── entrypoint.sh
│   ├── entrypoint-standalone.sh
│   └── openclaw.json
├── scripts/                 # 辅助脚本
│   ├── startup.sh
│   ├── deploy.sh
│   ├── run_index_service.py
│   └── verify_*.py
├── tests/                   # 测试
│   ├── conftest.py
│   ├── contract/
│   ├── integration/
│   ├── unit/
│   └── fixtures/
├── examples/
│   ├── basic_usage.py
│   ├── run_index_service.py
│   └── claude_code_integration.py
├── Dockerfile               # OpenClaw 集成镜像
├── pyproject.toml           # Python 包配置
└── .env                     # 环境变量
```

### 4.2 环境变量完整清单

| 变量名 | 文件:行号 | 默认值 | 用途 |
|--------|-----------|--------|------|
| `OPENAI_API_KEY` | providers/config.py:72 | 无（必须设置） | OpenAI API 密钥 |
| `OPENAI_BASE_URL` | providers/config.py:65 | `https://api.openai.com/v1` | OpenAI API 基础 URL |
| `OPENAI_EMBEDDING_MODEL` | providers/config.py:74 | `text-embedding-ada-002` | Embedding 模型名 |
| `OPENAI_LLM_MODEL` | providers/config.py:75 | `gpt-4o-mini` | LLM 模型名 |
| `CONTEXTENGINE_PROVIDER` | providers/config.py:71 | `mock` | 提供者类型（mock/openai） |
| `VECTOR_DB_TYPE` | providers/config.py:76 | `memory` | 向量数据库类型（memory/opengauss） |
| `OPENGUASS_CONNECTION_STRING` | providers/config.py:77 | 无 | openGauss 连接串 |
| `OPENGUASS_DIMENSION` | providers/config.py:78 | `1536` | 向量维度 |
| `OPENGUASS_TABLE_NAME` | providers/config.py:79 | `vector_index` | 向量索引表名 |
| `OPENGUASS_POOL_SIZE` | providers/config.py:80 | `5` | 连接池大小 |
| `AGFS_BASE_URL` | server/memory_service.py:103 | `http://localhost:1833` | AGFS 服务地址 |
| `AGFS_DATA_DIR` | scripts/startup.sh:45 | `$PROJECT_ROOT/.agfs_data` | AGFS 数据目录 |
| `AGFS_MOUNT_PREFIX` | scripts/run_index_service.py:53 | `/local/plugin` | AGFS 挂载前缀 |
| `OGMEM_HTTP_PORT` | server/app.py:141 | `8090` | HTTP 服务监听端口 |
| `OG_ACCOUNT_ID` | scripts/run_index_service.py:54 | `acct-demo` | 默认账户 ID |
| `OG_USER_ID` | .env | `u-alice` | 默认用户 ID |
| `OG_AGENT_ID` | .env | `main` | 默认 Agent ID |
| `INDEX_INTERVAL` | scripts/run_index_service.py:55 | `30` | 索引同步间隔（秒） |
| `INDEX_WORKERS` | scripts/run_index_service.py:56 | `1` | 索引 worker 数量 |
| `OGMEM_API_URL` | openclaw_context_engine_plugin/index.js:69 | 无 | 插件 HTTP 模式 URL |
| `OG_PYTHON_BIN` | openclaw_context_engine_plugin/index.js:71 | `python3` | 插件 subprocess 模式 Python 路径 |

### 4.3 Python 依赖

**pyproject.toml 中的依赖**：
```
dependencies = [
    "pyagfs>=1.4.0",
    "openai>=1.0.0",
    "apscheduler>=3.10.0",
]
```

**Dockerfile 中额外安装的**：
```
openai psycopg2-binary pyagfs rich requests apscheduler
```

**注意**：`psycopg2-binary` 在 Dockerfile 中安装但不在 `pyproject.toml` 的 `dependencies` 中。代码 `providers/vector_index/opengauss_index.py` 使用 `import psycopg2`。

### 4.4 入口模块

- **HTTP API 入口**：`server/app.py`
  - 框架：Flask
  - 开发启动：`python server/app.py`（端口 8090）
  - 生产启动：`gunicorn -w 2 -b 0.0.0.0:8090 server.app:app`
  - 端口由 `OGMEM_HTTP_PORT` 环境变量控制

- **索引服务入口**：`scripts/run_index_service.py`
  - 独立后台进程，轮询 outbox 并同步向量索引

- **Docker 入口**：`docker/entrypoint.sh`
  - 启动顺序：AGFS 后台 → IndexService 后台 → OpenClaw gateway 前台

- **pyproject.toml 中无 `[project.scripts]` 段**，即没有 CLI 入口点。这是本次项目的目标之一。

### 4.5 API 端点清单

| 端点 | 方法 | 说明 |
|------|------|------|
| `/api/v1/health` | GET | 健康检查 |
| `/api/v1/assemble` | POST | 查询记忆（返回 systemPromptAddition） |
| `/api/v1/after_turn` | POST | 写入记忆 |
| `/api/v1/ingest` | POST | 单条消息摄入 |
| `/api/v1/ingest_batch` | POST | 批量消息摄入 |
| `/api/v1/compact` | POST | 记忆压缩 |
| `/api/v1/bootstrap` | POST | 初始化 |
| `/api/v1/dispose` | POST | 清理 |
| `/api/v1/prepare_subagent_spawn` | POST | 子 Agent 准备 |
| `/api/v1/on_subagent_ended` | POST | 子 Agent 结束 |
| `/api/v1/call/<method>` | POST | 通用方法调用 |

### 4.6 .env 文件完整内容

```bash
# OpenGauss Database
OPENGUASS_CONNECTION_STRING=host=127.0.0.1 port=8888 dbname=postgres user=aaa password=aaa@123123

# OpenAI API
OPENAI_API_KEY=sk-sxWGh4hWeExbe8sqZEkgBi4E9l8E53oaAaoYEzjxbzR5IOgk
OPENAI_BASE_URL=https://chatapi.littlewheat.com/v1
OPENAI_EMBEDDING_MODEL=text-embedding-ada-002
OPENAI_LLM_MODEL=gpt-4o-mini

# oG-Memory Server
OGMEM_HTTP_PORT=8090
OG_ACCOUNT_ID=acct-demo
OG_USER_ID=u-alice
OG_AGENT_ID=main

# AGFS Configuration
AGFS_BASE_URL=http://localhost:1833
AGFS_MOUNT_PREFIX=/local/plugin

# Vector Database
VECTOR_DB_TYPE=opengauss
OPENGUASS_TABLE_NAME=vector_index
OPENGUASS_POOL_SIZE=5

# Provider Configuration
CONTEXTENGINE_PROVIDER=openai
enable_cache=true
cache_max_size=1000

# Model Parameters
llm_temperature=0.7
llm_max_tokens=4096
```

---

## 5. 发现的问题

### [ISSUE-01] AGFS 挂载路径不一致 → 已决策
- **决策**：统一使用 `/local`。代码中 `agfs_mount_prefix` 的 `/local/plugin` 是 bug，用一个单独 commit 修复，不在本次项目范围内。

### [ISSUE-02] pyproject.toml 缺少 psycopg2-binary 依赖
- **决策**：Phase 1 规格中添加到 dependencies。
- **状态**：待 Phase 1 修复

### [ISSUE-03] pyproject.toml 缺少 Flask 依赖
- **决策**：Phase 1 规格中添加到 dependencies。
- **状态**：待 Phase 1 修复

### [ISSUE-04] Dockerfile EXPOSE 端口与实际不符 → 不处理
- **决策**：后续不使用 Docker（openGauss 除外），Dockerfile 问题不在本次范围内。

### [ISSUE-05] VECTOR_DB_TYPE 默认值 → 已决策
- **决策**：默认使用 `opengauss`，在 `.env.example` 和 `ogc setup` 中体现。

### [ISSUE-06] .env 中包含明文 API Key → 已处理
- **决策**：`.env` 已在 `.gitignore` 中，无需额外处理。

### [ISSUE-07] openGauss 无 init.sql → 不需要
- **决策**：代码中 `ensure_table` 方法会自动建表，无需独立 SQL 文件。`ogc db init` 命令调用此方法即可。

---

## 6. 连通性验证结果（2026-04-01）

### AGFS 验证
```
GET http://localhost:1833/
→ {"service":"AGFS Server","status":"running","version":"1.4.0"}
```
**状态**：✅ 正常

### openGauss 验证
```
psycopg2.connect("host=127.0.0.1 port=8888 dbname=postgres user=aaa password=aaa@123123")
→ 连接成功，openGauss 6.0.3
→ 当前无表（待代码初始化，符合预期）
```
**状态**：✅ 正常
