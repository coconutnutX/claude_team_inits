# oG-Memory 本地 HTTP 模式部署总结

**部署日期**: 2026-03-28

---

## 概述

由于 Docker Hub 连接超时无法构建镜像，采用本地 HTTP 模式部署。该模式在功能上与 Docker 模式等效，使用本地 Python 进程运行 oG-Memory HTTP API。

### 部署模式对比

| 对比项 | Docker 镜像模式 | 本地 HTTP 模式（当前） |
|--------|----------------|---------------------|
| **AGFS** | 容器内运行 | 本地进程 |
| **HTTP API** | 容器内运行 | 本地进程 |
| **IndexService** | 容器内运行 | 本地进程 |
| **OpenClaw** | 本地安装 | 本地安装 |
| **通信方式** | HTTP | HTTP |
| **环境变量** | Docker `-e` 参数 | 环境变量 |

---

## 运行中的服务

### 服务状态总览

| 服务 | 状态 | 端口 | PID | 说明 |
|------|------|------|-----|------|
| **AGFS Server** | ✅ Running | 1833 | 186214 | 文件存储 |
| **oG-Memory HTTP API** | ✅ Running | 8090 | 187044 | REST API 服务 |
| **IndexService** | ✅ Running | - | 186408 | 异步索引服务（30秒轮询） |
| **OpenClaw Gateway** | ✅ Running | 18789 | 189837 | AI Agent 网关 |

### Docker 容器状态

| 容器名 | 镜像 | 端口映射 | 状态 |
|--------|------|----------|------|
| **opengauss** | opengauss:6.0.3 | 0.0.0.0:8888->5432/tcp | Up 44 hours |

**说明**: 只有 openGauss 使用 Docker 部署，oG-Memory 相关服务作为本地进程运行。

---

## 服务架构

```
┌─────────────────────────────────┐
│   OpenClaw Gateway          │
│   (本地 npm 安装)           │
│   http://localhost:18789     │
└──────────┬────────────────────┘
           │ HTTP POST
           ▼
┌─────────────────────────────────┐
│   oG-Memory HTTP API        │
│   python3 server/app.py      │
│   http://localhost:8090     │
└──────────┬────────────────────┘
           │
           ├─▶ IndexService (轮询 outbox，30秒间隔)
           │       └─▶ Embedder (OpenAI) → 向量化
           │
           └─▶ AGFS Server (agfs-server)
                   │
                   └─▶ 本地文件系统 (/tmp/agfs-data)
                           │
                           ▼
                   ┌─────────────────────────┐
                   │   openGauss 容器      │
                   │   opengauss:6.0.3     │
                   │   localhost:8888        │
                   │   vector_index 表       │
                   └─────────────────────────┘
```

---

## 环境变量配置

### 当前生效的环境变量

```bash
# AGFS 配置
export AGFS_BASE_URL=http://localhost:1833
export AGFS_MOUNT_PREFIX=/local

# OpenAI 配置
export OPENAI_API_KEY=xxx
export OPENAI_BASE_URL=xxx
export OPENAI_EMBEDDING_MODEL=text-embedding-3-small
export OPENAI_LLM_MODEL=gpt-4o-mini

# 向量数据库配置
export VECTOR_DB_TYPE=opengauss
export OPENGUASS_CONNECTION_STRING="postgresql://aaa:aaa%40123123@localhost:8888/aaa"
export OPENGUASS_DIMENSION=1536
export OPENGUASS_TABLE_NAME="vector_index"
export OPENGUASS_POOL_SIZE=5

# 账户配置
export OG_ACCOUNT_ID=aaa
export OG_USER_ID=u-default
export OG_AGENT_ID=main
```

### 注意事项

- **当前会话环境变量**: 这些环境变量只在当前 shell 会话中有效
- **持久化配置**: 如需在重启后保持，请添加到 `~/.bashrc`
- **代理设置**: 已在当前会话中清除 `HTTP_PROXY` 和 `HTTPS_PROXY` 以避免本地连接问题

---

## OpenClaw 插件配置

### 插件安装信息

- **插件 ID**: og-memory-context-engine
- **插件类型**: context-engine
- **通信模式**: HTTP（非 subprocess）

### openclaw.json 配置片段

```json
{
  "plugins": {
    "enabled": true,
    "slots": {
      "contextEngine": "og-memory-context-engine"
    },
    "entries": {
      "og-memory-context-engine": {
        "enabled": true,
        "config": {
          "memoryApiBaseUrl": "http://127.0.0.1:8090"
        }
      }
    }
  }
}
```

### 配置文件位置

```
~/.openclaw/
├── openclaw.json          # 主配置文件
├── openclaw.json.bak      # 备份文件
└── extensions/
    └── og-memory-context-engine/
        ├── index.js         # 插件入口
        ├── package.json
        └── bridge/
            └── memory_api.py
```

---

## 服务日志

### 日志文件位置

| 服务 | 日志文件 | 说明 |
|------|---------|------|
| AGFS | `/tmp/agfs.log` | 文件系统服务日志 |
| HTTP API | `/tmp/ogmem_http.log` | API 服务日志 |
| IndexService | `/tmp/index_service.log` | 索引服务日志 |
| OpenClaw | `/tmp/openclaw.log` | Gateway 日志 |

### 查看日志命令

```bash
# 查看所有日志（多窗口）
tail -f /tmp/agfs.log
tail -f /tmp/ogmem_http.log
tail -f /tmp/index_service.log
tail -f /tmp/openclaw.log

# 查看最近日志
tail -20 /tmp/index_service.log
```

---

## 健康检查命令

### 服务健康检查

```bash
# AGFS Server
curl -s --noproxy localhost http://localhost:1833/
# 预期: {"service":"AGFS Server","status":"running","version":"1.4.0"}

# oG-Memory HTTP API
curl -s http://localhost:8090/api/v1/health
# 预期: {"agfs":true,"agfs_url":"http://localhost:1833","backend":"og-memory","llm":"gpt-4o-mini","status":"ok"}

# OpenClaw Gateway
curl -s http://localhost:18789/health
# 预期: {"ok":true,"status":"live"}
```

### 数据库检查

```bash
# 检查 vector_index 表记录数
docker exec opengauss sh -c "LD_LIBRARY_PATH=/usr/local/opengauss/lib /usr/local/opengauss/bin/gsql -d postgres -U aaa -W aaa@123123 -p 5432 -c 'SELECT count(*) FROM vector_index;'"

# 查看最近的记录
docker exec opengauss sh -c "LD_LIBRARY_PATH=/usr/local/opengauss/lib /usr/local/opengauss/bin/gsql -d postgres -U aaa -W aaa@123123 -p 5432 -c 'SELECT * FROM vector_index ORDER BY created_at DESC LIMIT 10;'"
```

---

## 启动/停止脚本

### 完整启动流程

```bash
# 1. 设置环境变量
export HTTP_PROXY=""
export HTTPS_PROXY=""
export AGFS_BASE_URL=http://localhost:1833
export AGFS_MOUNT_PREFIX=/local
export CONTEXTENGINE_PROVIDER=openai
export OPENAI_API_KEY=xxx
export OPENAI_BASE_URL=xxx
export OPENAI_EMBEDDING_MODEL="text-embedding-3-small"
export OPENAI_LLM_MODEL="gpt-4o-mini"
export VECTOR_DB_TYPE=opengauss
export OPENGUASS_CONNECTION_STRING="postgresql://aaa:aaa%40123123@localhost:8888/aaa"
export OG_ACCOUNT_ID=aaa

# 2. 启动 AGFS
agfs-server -c ~/.agfs-config.yaml > /tmp/agfs.log 2>&1 &

# 3. 启动 IndexService
cd /data/Workspace2/oG-Memory
python3 scripts/run_index_service.py > /tmp/index_service.log 2>&1 &

# 4. 启动 HTTP API
python3 server/app.py > /tmp/ogmem_http.log 2>&1 &

# 5. 启动 OpenClaw Gateway
openclaw daemon > /tmp/openclaw.log 2>&1 &
```

### 停止所有服务

```bash
# 停止 oG-Memory 相关进程
pkill -f agfs-server
pkill -f "server/app.py"
pkill -f "run_index_service"

# 停止 OpenClaw Gateway
openclaw daemon stop

# 或强制停止
pkill -9 -f openclaw-gateway
```

---

## 依赖包版本

### Python 依赖

```bash
# 新增安装的包（为 HTTP API 服务）
flask==3.1.3
gunicorn==25.3.0
blinker==1.9.0
itsdangerous==2.2.0
werkzeug==3.1.7
```

### Node.js 依赖

```bash
# OpenClaw 版本
openclaw@2026.3.24 (cff6dc9)

# 新增安装的包（修复依赖问题）
@sinclair/typebox@latest
```

---

## 数据流说明

### 对话数据写入流程

```
用户输入
    ↓
OpenClaw Gateway (localhost:18789)
    │
    │ HTTP POST /api/v1/after_turn
    ▼
oG-Memory HTTP API (localhost:8090)
    │
    ├─▶ MemoryService → OutboxStore → AGFS
    │       │                           │
    │       │                           ▼
    │       │                   .outbox/{event_id}.json
    │       │
    └─▶ 响应 {"writes_completed": N}
            │
            ▼
    IndexService (30秒轮询)
            │
            ├─▶ Embedder (OpenAI) → 向量化
            │
            └─▶ openGauss vector_index 表
```

### 记忆检索流程

```
用户查询
    ↓
OpenClaw Gateway
    │
    │ HTTP POST /api/v1/assemble
    ▼
oG-Memory HTTP API
    │
    ├─▶ QueryPlanner (解析查询)
    │
    ├─▶ VectorIndex.search_by_vector (OpenGauss)
    │
    └─▶ 返回相关记忆片段
            │
            ▼
    用户收到增强后的回复
```

---

## 与 Docker 模式的差异

### 为什么使用本地模式而非 Docker？

**原因**: Docker Hub 连接超时，无法拉取 `python:3.11-slim-bookworm` 基础镜像

**影响**: 需要手动管理多个本地进程，而不是使用单个 Docker 容器

**解决方案**: 可以通过以下方式切换到 Docker 模式
1. 配置 Docker 代理（需要 sudo 权限）
2. 或使用国内 Docker 镜像源
3. 或手动预拉取基础镜像

### 迁移到 Docker 模式

当 Docker Hub 连接问题解决后，可以通过以下命令切换：

```bash
# 1. 停止本地服务
pkill -f agfs-server
pkill -f "server/app.py"
pkill -f "run_index_service"
openclaw daemon stop

# 2. 构建 Docker 镜像
cd /data/Workspace2/oG-Memory
docker build --network host -f Dockerfile.standalone -t ogmem-server:v3 .

# 3. 启动 Docker 容器
docker run -d \
  --name ogmem-server \
  --network host \
  -v /data/ogmem-data:/data/agfs \
  -e OPENAI_API_KEY="sk-..." \
  -e OPENAI_BASE_URL="..." \
  -e OPENAI_EMBEDDING_MODEL="..." \
  -e "OPENGUASS_CONNECTION_STRING=..." \
  -e OG_ACCOUNT_ID="aaa" \
  ogmem-server:v3
```

---

## 故障排查

### 问题 1: 服务无法启动

**症状**: `curl http://localhost:8090/api/v1/health` 返回连接被拒绝

**排查**:
```bash
# 检查端口占用
netstat -tlnp | grep -E "8090|1833|18789"

# 检查进程状态
ps aux | grep -E "agfs|server/app.py|openclaw" | grep -v grep

# 查看日志
cat /tmp/ogmem_http.log
```

### 问题 2: 数据未写入 openGauss

**症状**: `SELECT * FROM vector_index` 返回 0 行

**排查**:
```bash
# 1. 检查 IndexService 日志
tail -50 /tmp/index_service.log | grep "processed\|succeeded"

# 2. 检查 AGFS outbox 目录
ls -la /tmp/agfs-data/accounts/*/memories/**/.outbox/ 2>/dev/null

# 3. 检查账户 ID 是否正确
grep "account=" /tmp/ogmem_http.log
```

### 问题 3: OpenClaw 插件无法加载

**症状**: OpenClaw Gateway 日志显示插件加载失败

**排查**:
```bash
# 1. 检查插件目录
ls -la ~/.openclaw/extensions/og-memory-context-engine/

# 2. 检查 openclaw.json
cat ~/.openclaw/openclaw.json | python3 -m json.tool

# 3. 查看插件日志
tail -50 /tmp/openclaw.log
```

---

## 配置文件汇总

### 主要配置文件

| 文件路径 | 用途 |
|---------|------|
| `~/.agfs-config.yaml` | AGFS 配置 |
| `~/.openclaw/openclaw.json` | OpenClaw 主配置 |
| `/data/Workspace2/oG-Memory/.env.example` | 环境变量模板 |
| `/data/Workspace2/oG-Memory/server/app.py` | HTTP API 入口 |
| `/data/Workspace2/oG-Memory/scripts/run_index_service.py` | IndexService 启动脚本 |

### 环境配置文档

| 文件 | 说明 |
|------|------|
| `.claude-team/docker_setup_guide.md` | Docker 部署完整指南 |
| `.claude-team/cleanup_env.sh` | 环境清理脚本 |
| `.claude-team/start-docker-deployment.sh` | Docker 部署启动脚本 |
| `.claude-team/local_http_deployment.md` | 本地 HTTP 模式部署（本文档） |

---

## 下一步建议

1. **测试数据流**: 通过 OpenClaw 对话，验证数据写入 openGauss
2. **监控日志**: 定期检查 IndexService 日志，确保索引正常工作
3. **数据验证**: 定期查询 vector_index 表，确认数据持续增长
4. **Docker 部署**: 当 Docker Hub 连接问题解决后，迁移到 Docker 模式
