# Phase 1: 配置项清单

> 扫描时间：2026-04-02
> 扫描范围：`src/og_memory/`、`openclaw_context_engine_plugin/`、`scripts/`、`examples/`、`tests/`

---

## 1. 环境变量完整清单

### Python 代码中的环境变量

| # | 变量名 | 文件:行号 | 默认值 | 用途 | 必填 |
|---|--------|-----------|--------|------|------|
| 1 | `OGMEM_HTTP_PORT` | `server/app.py:139` | `8090` | HTTP 服务监听端口 | 否 |
| 2 | `AGFS_BASE_URL` | `server/memory_service.py:103` | `http://localhost:1833` | AGFS 服务地址 | 否 |
| 3 | `AGFS_DEFAULT_ACCOUNT_ID` | `server/memory_service.py:106` | 无 | 默认 account_id | 否 |
| 4 | `AGFS_DEFAULT_USER_ID` | `server/memory_service.py:109` | 无 | 默认 user_id | 否 |
| 5 | `CONTEXTENGINE_PROVIDER` | `providers/config.py:71` | `mock` | 提供者类型 (mock/openai/openai-cached) | 否 |
| 6 | `OPENAI_API_KEY` | `providers/config.py:72`, `openai_llm.py:64`, `openai_embedder.py:64` | 无 | OpenAI API 密钥 | openai 模式必填 |
| 7 | `OPENAI_BASE_URL` | `providers/config.py:65` | `https://api.openai.com/v1` | OpenAI API 基础 URL | 否 |
| 8 | `OPENAI_EMBEDDING_MODEL` | `providers/config.py:74` | `text-embedding-ada-002` | Embedding 模型名 | 否 |
| 9 | `OPENAI_LLM_MODEL` | `providers/config.py:75` | `gpt-4o-mini` | LLM 模型名 | 否 |
| 10 | `VECTOR_DB_TYPE` | `providers/config.py:76` | `memory` | 向量数据库类型 (memory/opengauss) | 否 |
| 11 | `OPENGUASS_CONNECTION_STRING` | `providers/config.py:77` | 无 | openGauss 连接串 | opengauss 模式必填 |
| 12 | `OPENGUASS_DIMENSION` | `providers/config.py:78` | `1536` | 向量维度 | 否 |
| 13 | `OPENGUASS_TABLE_NAME` | `providers/config.py:79` | `vector_index` | 向量索引表名 | 否 |
| 14 | `OPENGUASS_POOL_SIZE` | `providers/config.py:80` | `5` | 连接池大小 | 否 |

### JavaScript 代码中的环境变量

| # | 变量名 | 文件:行号 | 默认值 | 用途 | 必填 |
|---|--------|-----------|--------|------|------|
| 15 | `OGMEM_API_URL` | `openclaw_context_engine_plugin/index.js:69` | 无 | 插件 HTTP 模式 URL | 否 |
| 16 | `OG_PYTHON_BIN` | `openclaw_context_engine_plugin/index.js:71` | `python3` | 插件 subprocess 模式 Python 路径 | 否 |

### 脚本/测试中的环境变量

| # | 变量名 | 文件:行号 | 默认值 | 用途 | 必填 |
|---|--------|-----------|--------|------|------|
| 17 | `AGFS_BASE_URL` | `tests/conftest.py:30` | `http://localhost:1833` | 测试用 AGFS 地址 | 否 |
| 18 | `AGFS_DATA_DIR` | `tests/conftest.py:31` | `../.agfs_test_data` | 测试用 AGFS 数据目录 | 否 |

---

## 2. 硬编码可配置项清单

| # | 类型 | 当前硬编码值 | 文件:行号 | 建议 config 字段 |
|---|------|-------------|-----------|-----------------|
| 1 | 端口 | `8090` | `server/app.py:139` | `server.port` |
| 2 | 主机 | `0.0.0.0` | `server/app.py:141` | `server.host` |
| 3 | AGFS URL | `http://localhost:1833` | `server/memory_service.py:103`, `fs/agfs_context_fs.py:258` | `agfs.base_url` |
| 4 | AGFS 端口 | `1833` | AGFS config yaml | `agfs.port` |
| 5 | AGFS 数据目录 | `/data/agfs` | `docker/agfs-config.yaml` | `agfs.data_dir` |
| 6 | AGFS 二进制路径 | `agfs-server` (PATH 查找) | — | `agfs.bin_path` |
| 7 | LLM 模型 | `gpt-4o-mini` | `providers/config.py:75`, `openai_llm.py:39` | `llm.model` |
| 8 | Embedding 模型 | `text-embedding-ada-002` | `providers/config.py:74` | `llm.embedding_model` |
| 9 | Embedding 维度 | `1536` | `vector_index/opengauss_index.py:88`, `embedder/openai_embedder.py:25` | `llm.dimension` |
| 10 | 向量表名 | `vector_index` | `providers/config.py:79` | `opengauss.table_name` |
| 11 | 连接池大小 | `5` | `providers/config.py:80` | `opengauss.pool_size` |
| 12 | Provider 类型 | `mock` | `providers/config.py:71` | `provider` |
| 13 | 日志级别 | `INFO` (basicConfig) | `server/app.py:25` | `server.log_level` |
| 14 | Flask threaded | `True` | `server/app.py:141` | 不需配置 |
| 15 | openGauss 端口 | `8888` (宿主机映射) | `.env` | `opengauss.port` |

---

## 3. AGFS 配置项清单

### AGFS 服务端配置（通过 config.yaml）

| 配置项 | 当前值 | 来源 | 说明 |
|--------|--------|------|------|
| `server.address` | `:1833` | `docker/agfs-config.yaml` | 监听地址 |
| `server.log_level` | `info` | `docker/agfs-config.yaml` | 日志级别 |
| `plugins.localfs.enabled` | `true` | `docker/agfs-config.yaml` | localfs 插件 |
| `plugins.localfs.path` | `/local` | `docker/agfs-config.yaml` | 挂载路径前缀 |
| `plugins.localfs.config.local_dir` | `/data/agfs` | `docker/agfs-config.yaml` | 物理存储目录 |

### AGFS 命令行参数

```
agfs-server -c <config_path>       # 配置文件路径
agfs-server -addr <address>        # 覆盖配置文件中的地址
agfs-server -version               # 显示版本
agfs-server -print-sample-config   # 打印示例配置
```

### 启动方式（env-investigation-report 记录）

- 本地：`agfs-server -c agfs-config.yaml`
- Docker：`/opt/agfs/agfs-server -c /opt/agfs/config.yaml &`
- 二进制路径：`/home/aaa/.local/bin/agfs-server` (v1.4.0)

---

## 4. openGauss 配置项清单

### 连接参数

| 参数 | 当前值 | 来源 | 说明 |
|------|--------|------|------|
| host | `127.0.0.1` | `.env` | 宿主机地址 |
| port | `8888` | `.env` | 宿主机映射端口 (容器内 5432) |
| dbname | `postgres` | `.env` | 数据库名 |
| user | `aaa` | `.env` | 用户名 |
| password | `aaa@123123` | `.env` | 密码 |

### Docker 启动参数

```bash
docker run --name opengauss --privileged=true -d \
  -e GS_PASSWORD=Gauss@1234 \
  -e GS_USERNAME=aaa \
  -e GS_USER_PASSWORD=aaa@123123 \
  -p 8888:5432 \
  -v /data/opengauss_data:/var/lib/opengauss \
  opengauss:6.0.3
```

### psycopg2 连接方式

psycopg2 支持纯参数传入，**不依赖 PG* 环境变量**。可通过关键字参数直接传参：
```python
psycopg2.connect(host=..., port=..., dbname=..., user=..., password=...)
```

---

## 5. .env.example 与代码一致性检查

### 问题清单

| # | 问题 | 严重度 | 说明 |
|---|------|--------|------|
| 1 | `.env.example` 包含**真实 API Key** | **严重** | 文件中有 `sk-sxWGh4h...` 真实密钥，应替换为占位符 |
| 2 | `.env.example` 缺少多个环境变量 | 中 | 缺少 `OGMEM_HTTP_PORT`、`AGFS_DEFAULT_ACCOUNT_ID`、`AGFS_DEFAULT_USER_ID`、`AGFS_DATA_DIR`、`VECTOR_DB_TYPE`、`CONTEXTENGINE_PROVIDER` |
| 3 | 变量名拼写不一致 | 低 | 代码中 `OPENGUASS` (少一个 A)，标准拼写应为 `OPENGAUSS`。本次保持现状不改名 |
| 4 | `.env.example` 包含 Markdown 格式 | 中 | 文件内容是 Markdown 而非纯 shell 变量格式，不能直接 `source` |
| 5 | `providers/config.py` 中 `OPENGUASS_CONNECTION_STRING` 与 `.env` 中 `OPENGUASS_CONNECTION_STRING` 拼写一致 | — | OK |

### .env.example 中有但代码未使用的变量

无。所有变量均有对应代码引用。

---

## 6. 建议 config.yaml 结构草案

基于以上发现，建议 config.yaml 结构如下：

```yaml
# config.example.yaml
# cp config.example.yaml config.yaml

ogmem-server:
  host: "0.0.0.0"           # oG-Memory HTTP API 监听地址
  port: 8090                 # oG-Memory HTTP API 监听端口
  log_level: "info"          # debug / info / warning / error

agfs:
  base_url: "http://localhost:1833"  # AGFS 服务地址
  bin_path: ""               # agfs-server 二进制路径，空则自动查找
  config_path: "agfs-config.yaml"    # AGFS 配置文件路径
  data_dir: "/data/agfs"     # AGFS 数据存储目录
  # external_url: ""          # 取消注释则使用外部 AGFS，跳过本地进程管理

opengauss:
  host: "127.0.0.1"
  port: 8888
  dbname: "postgres"
  user: "aaa"
  password: ""               # [必填]
  table_name: "vector_index"
  pool_size: 5
  dimension: 1536

llm:
  provider: "mock"           # mock / openai / openai-cached
  api_key: ""                # [openai 模式必填]
  base_url: "https://api.openai.com/v1"
  model: "gpt-4o-mini"
  embedding_model: "text-embedding-ada-002"

# 默认身份（用于无 RequestContext 的场景）
defaults:
  account_id: ""
  user_id: ""
```
