# oG-Memory 部署分析报告

> 生成时间：2026-04-01
> 分析目标：理解当前部署方式，设计一键部署方案

---

## 一、当前部署架构

```
┌─────────────────────────────────────────────────────────────────┐
│                    部署架构总览                               │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌─────────────┐    ┌─────────────┐    ┌─────────────┐        │
│  │ OpenClaw    │    │ oG-Memory   │    │ openGauss   │        │
│  │ (Web UI)    │    │ (API)       │    │ (Vector DB) │        │
│  │ :18789      │    │ :8090       │    │ :8888/5432  │        │
│  └──────┬──────┘    └──────┬──────┘    └──────┬──────┘        │
│         │                  │                   │                 │
│         └──────────────────┼───────────────────┘                 │
│                            │                                    │
│                    ┌───────▼───────┐                           │
│                    │     AGFS      │                           │
│                    │  :1833        │                           │
│                    │  (File Store) │                           │
│                    └───────────────┘                           │
└─────────────────────────────────────────────────────────────────┘
```

---

## 二、各组件部署详情

### 2.1 openGauss 数据库

**当前启动方式：**
```bash
docker run --name opengauss --privileged=true -d \
  -e GS_PASSWORD=Gauss@1234 \
  -e GS_USERNAME=aaa \
  -e GS_USER_PASSWORD=aaa@123123 \
  -p 8888:5432 \
  -v /data/opengauss_data:/var/lib/opengauss \
  opengauss:6.0.3
```

**关键参数：**
| 参数 | 值 | 说明 |
|------|-----|------|
| 镜像 | `opengauss:6.0.3` | openGauss 6.0.3 版本 |
| 端口 | `8888:5432` | 宿主机 8888 映射到容器内 5432 |
| 数据卷 | `/data/opengauss_data` | 数据持久化 |
| 特权模式 | `--privileged=true` | 需要 |

**数据库表结构：**
表 `vector_index` 在 oG-Memory 首次启动时自动创建：
```sql
CREATE TABLE IF NOT EXISTS vector_index (
    id VARCHAR(16) PRIMARY KEY,
    uri VARCHAR(512) NOT NULL,
    level INTEGER NOT NULL,
    text TEXT NOT NULL,
    embedding vector(1536) NOT NULL,
    filters JSONB NOT NULL,
    metadata JSONB NOT NULL DEFAULT '{}',
    created_at TIMESTAMPTZ DEFAULT NOW(),
    updated_at TIMESTAMPTZ DEFAULT NOW()
);
```

**连接字符串格式：**
```
host=127.0.0.1 port=8888 dbname=postgres user=aaa password=aaa@123123
```

---

### 2.2 AGFS (Adaptive Globally-addressable File System)

**作用：**
- 为 oG-Memory 提供文件存储层
- 通过 HTTP API 提供文件读写服务
- 支持多租户隔离

**当前安装方式：**

**方式 1：官方脚本（推荐）**
```bash
curl -fsSL https://raw.githubusercontent.com/c4pt0r/agfs/master/install.sh | sh
```

**方式 2：源码编译**
```bash
go env -w GOPROXY=https://goproxy.cn,direct
git clone https://github.com/c4pt0r/agfs.git /tmp/agfs-build
cd /tmp/agfs-build/agfs-server
go build -o ~/.local/bin/agfs-server .
```

**配置文件：**
```yaml
# ~/.agfs-config.yaml 或 docker/agfs-config.yaml
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

**启动命令：**
```bash
agfs-server -c ~/.agfs-config.yaml &
```

**验证启动：**
```bash
curl http://localhost:1833/api/v1/health
# 预期: {"status":"healthy","version":"..."}
```

---

### 2.3 oG-Memory

**作用：**
- 提供长期记忆 API 服务
- 处理记忆提取、检索、存储
- 通过 AGFS 存储文件，通过 openGauss 存储向量索引

**当前 Dockerfile：**

**Standalone 模式 (`Dockerfile.standalone`)**
```dockerfile
FROM python:3.11-slim-bookworm

RUN pip install --no-cache-dir \
    flask gunicorn openai psycopg2-binary pyagfs rich requests apscheduler

# 复制源码
COPY core/       /opt/ogmem/core/
COPY retrieval/  /opt/ogmem/retrieval/
COPY extraction/ /opt/ogmem/extraction/
COPY commit/     /opt/ogmem/commit/
COPY index/      /opt/ogmem/index/
COPY providers/  /opt/ogmem/providers/
COPY service/    /opt/ogmem/service/
COPY fs/         /opt/ogmem/fs/
COPY server/     /opt/ogmem/server/
COPY scripts/    /opt/ogmem/scripts/

# 复制 AGFS（需要预先编译）
COPY agfs/agfs-server          /opt/agfs/agfs-server
COPY docker/agfs-config.yaml   /opt/agfs/config.yaml

COPY docker/entrypoint-standalone.sh /opt/entrypoint.sh
RUN chmod +x /opt/entrypoint.sh /opt/agfs/agfs-server

ENV PYTHONPATH=/opt/ogmem
ENV AGFS_BASE_URL=http://127.0.0.1:1833
ENV VECTOR_DB_TYPE=opengauss
ENV OPENAI_BASE_URL=https://dashscope.aliyuncs.com/compatible-mode/v1
ENV OGMEM_HTTP_PORT=8090

USER ogmem
EXPOSE 8090
ENTRYPOINT ["/opt/entrypoint.sh"]
```

**启动方式：**
```bash
docker run -d \
  --name ogmem-server \
  --network host \
  -v /data/agfs-data:/data/agfs \
  -e OPENAI_API_KEY="sk-xxx" \
  -e "OPENGUASS_CONNECTION_STRING=host=127.0.0.1 port=8888 dbname=postgres user=aaa password=aaa@123123" \
  ogmem-server:v3
```

---

### 2.4 openclaw_context_engine_plugin 插件

**作用：**
- 为 OpenClaw Gateway 提供长期记忆能力
- 通过 HTTP API 与 oG-Memory 通信

**插件注册：**

**插件清单 (`openclaw.plugin.json`)**
```json
{
  "id": "og-memory-context-engine",
  "name": "oG-Memory Context Engine",
  "version": "0.1.0",
  "kind": "context-engine",
  "configSchema": {
    "type": "object",
    "properties": {
      "pythonBin": {"type": "string"},
      "logLevel": {"type": "string"},
      "memoryApiBaseUrl": {"type": "string"}
    }
  }
}
```

**OpenClaw 配置 (`openclaw.json`)**
```json
{
  "plugins": {
    "slots": {
      "contextEngine": "og-memory-context-engine"
    },
    "entries": {
      "og-memory-context-engine": {
        "enabled": true,
        "config": {
          "pythonBin": "python3",
          "memoryApiBaseUrl": "http://127.0.0.1:8090"
        }
      },
      "memory-core": {
        "enabled": false
      }
    }
  }
}
```

**插件目录结构：**
```
openclaw_context_engine_plugin/
├── openclaw.plugin.json    # 插件清单
├── index.js                # 插件入口
├── package.json            # Node.js 依赖
└── bridge/
    └── memory_api.py       # Python 桥接层
```

---

## 三、当前部署存在的问题

| 问题 | 描述 | 影响 |
|------|------|------|
| **多步骤手动操作** | 需要手动启动 openGauss、安装 AGFS、构建镜像、启动容器 | 易出错，不友好 |
| **缺少 docker-compose** | 各组件独立启动，没有统一编排 | 管理复杂 |
| **AGFS 二进制缺失** | `agfs/` 目录为空，需要预先下载/编译 | 构建镜像会失败 |
| **无依赖检查** | 启动前不检查 openGauss 是否就绪 | 可能启动失败 |
| **配置分散** | openGauss 连接串、API Key 等分散在多个地方 | 配置困难 |
| **无一键脚本** | 没有统一的启动/停止/重启脚本 | 运维成本高 |

---

## 四、数据流向

```
用户消息 (OpenClaw UI)
    │
    ├── assemble() ──────────────────────────────────────────────┐
    │    │                                                     │
    │    ▼                                                     │
    │  ogmem:8090/api/v1/assemble                             │
    │    │                                                     │
    │    ├── Query from openGauss (vector search)                │
    │    │    └── host=127.0.0.1 port=8888 ...                │
    │    │                                                     │
    │    └── Fetch files from AGFS                             │
    │         └── http://127.0.0.1:1833/...                  │
    │                                                         │
    └── 返回相关记忆 ──────────────────────────────────────────┘
         │
         ▼
    LLM 生成回复 (带记忆上下文)
         │
    after_turn() ──────────────────────────────────────────────┐
    │    │                                                    │
    │    ▼                                                    │
    │  ogmem:8090/api/v1/after_turn                          │
    │    │                                                    │
    │    ├── 提取新记忆                                      │
    │    ├── 写入 AGFS (.md 文件) ─────────────────────┐     │
    │    │                                             │     │
    │    └── 创建 outbox 事件 ──────────────────────────┤     │
    │                                                  │     │
    │                                    IndexService   │     │
    │                                          ────────►▼     │
    │                                            生成 embedding │
    │                                                  │     │
    │                                    写入 openGauss ◄─────┘
    │                                    (vector_index 表)
```

---

## 五、关键文件清单

| 文件/目录 | 用途 |
|-----------|------|
| `Dockerfile` | OpenClaw 集成模式镜像 |
| `Dockerfile.standalone` | 独立 API 模式镜像 |
| `docker/entrypoint.sh` | 容器启动脚本 |
| `docker/entrypoint-standalone.sh` | 独立模式启动脚本 |
| `docker/agfs-config.yaml` | AGFS 配置 |
| `docker/openclaw.json` | OpenClaw 默认配置 |
| `openclaw_context_engine_plugin/` | OpenClaw 插件 |
| `providers/vector_index/opengauss_index.py` | openGauss 向量索引 |
| `scripts/startup.sh` | 本地启动脚本 |
| `scripts/deploy.sh` | 本地部署脚本 |
| `agfs/` | AGFS 二进制文件目录（**当前为空**） |

---

## 六、环境变量汇总

| 变量 | 必需 | 默认值 | 说明 |
|------|------|--------|------|
| `OPENAI_API_KEY` | ✅ | - | LLM/Embedding API 密钥 |
| `OPENGUASS_CONNECTION_STRING` | ✅ | - | openGauss 连接串 |
| `OPENAI_BASE_URL` | ❌ | `https://dashscope.aliyuncs.com/compatible-mode/v1` | LLM API 地址 |
| `OPENAI_EMBEDDING_MODEL` | ❌ | `text-embedding-v2` | Embedding 模型名 |
| `OPENAI_LLM_MODEL` | ❌ | - | LLM 模型名 |
| `OGMEM_HTTP_PORT` | ❌ | `8090` | HTTP API 端口 |
| `OG_ACCOUNT_ID` | ❌ | `acct-demo` | 默认账户 ID |
| `AGFS_BASE_URL` | ❌ | `http://127.0.0.1:1833` | AGFS 服务地址 |
| `VECTOR_DB_TYPE` | ❌ | `opengauss` | 向量数据库类型 |
| `OPENCLAW_GATEWAY_TOKEN` | ❌ | `test-token-123` | 前端访问 Token |

---

## 七、下一步

详见优化方案文档：[deployment_optimization_plan.md](./deployment_optimization_plan.md)
