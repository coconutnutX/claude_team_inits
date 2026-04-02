# oG-Memory 操作指南

> 基于：Phase 3 端到端验证结果（2026-04-01）
> 环境：Windows + WSL2, Python 3.11+ (conda), Docker

---

## 前置条件

1. **openGauss Docker 容器运行中**
   ```bash
   # 检查
   docker ps --filter name=opengauss

   # 如未启动（alias: ogon）
   docker run --name opengauss --privileged=true -d \
     -e GS_PASSWORD=Gauss@1234 \
     -e GS_USERNAME=aaa \
     -e GS_USER_PASSWORD=aaa@123123 \
     -p 8888:5432 \
     -v /data/opengauss_data:/var/lib/opengauss \
     opengauss:6.0.3
   ```

2. **AGFS 已安装**
   ```bash
   # 检查
   agfs-server -version
   # 预期输出: agfs-server v1.4.0

   # 如未安装
   curl -fsSL https://raw.githubusercontent.com/c4pt0r/agfs/master/install.sh | sh
   ```

3. **Python 包已安装**
   ```bash
   cd /data/Workspace2/oG-Memory
   pip install -e .
   ```

---

## 快速启动（3 步）

### 步骤 1：配置环境

```bash
cd /data/Workspace2/oG-Memory

# 首次使用：复制模板并编辑
cp .env.example .env
# 编辑 .env，至少填写 OPENAI_API_KEY

# 或使用交互式向导
ogc setup
```

**必须确认 .env 中这些值正确**：
- `OPENGUASS_CONNECTION_STRING` — openGauss 连接串（注意引号包裹）
- `OPENAI_API_KEY` — 你的 OpenAI API 密钥
- `OPENAI_BASE_URL` — API 基础 URL

### 步骤 2：健康检查

```bash
ogc health-check
```

预期输出：
```
[AGFS] Binary: /home/aaa/.local/bin/agfs-server
[AGFS] Health: OK
[openGauss] Connection: host=127.0.0.1 port=8888 ...password=***
[openGauss] Status: OK
[ENV] OPENAI_API_KEY=sk-sxWGh...
[ENV] OPENAI_BASE_URL=https://chatapi.littlewheat.com/v1
[ENV] CONTEXTENGINE_PROVIDER=openai
[ENV] VECTOR_DB_TYPE=opengauss

All checks passed.
```

> 如果 AGFS Health 显示 NOT RESPONDING，先启动 AGFS：`ogc serve` 会自动启动。

### 步骤 3：启动服务

```bash
ogc serve
```

输出：
```
[AGFS] Started (PID xxxxx)
[AGFS] Ready
[API] Started on :8090 (PID xxxxx)

Service running. Ctrl+C to stop.
```

**另开终端**，启动 IndexService（负责将新记忆同步到向量数据库）：

```bash
cd /data/Workspace2/oG-Memory
set -a && source .env && set +a
python3 scripts/run_index_service.py
```

> 注意：`set -a && source .env && set +a` 是必须的，否则环境变量不会正确导出。

---

## 使用 API

### 写入记忆

```bash
curl -s http://localhost:8090/api/v1/after_turn \
  -H "Content-Type: application/json" \
  -d '{
    "account_id": "acct-demo",
    "user_id": "u-alice",
    "agent_id": "main",
    "session_id": "s-001",
    "trace_id": "t-001",
    "messages": [
      {"role": "user", "content": "I prefer using Python for writing code, especially async programming."},
      {"role": "assistant", "content": "Got it, I'\''ll remember your Python async preference."}
    ]
  }' | python3 -m json.tool
```

预期输出：
```json
{
  "ok": true,
  "candidates": 1,
  "writes": {
    "completed": 1,
    "failed": 0,
    "skipped": 0
  }
}
```

### 等待索引同步

IndexService 每 30 秒轮询一次 outbox。等待约 30 秒后，记忆会被嵌入到向量数据库。

检查是否已同步（vector_index 表有数据即表示已处理）：
```bash
set -a && source .env && set +a
python3 -c "
import psycopg2
conn = psycopg2.connect('$OPENGUASS_CONNECTION_STRING')
cur = conn.cursor()
cur.execute('SELECT count(*) FROM vector_index')
print('向量索引条数:', cur.fetchone()[0])
conn.close()
"
```

### 查询记忆

```bash
curl -s http://localhost:8090/api/v1/assemble \
  -H "Content-Type: application/json" \
  -d '{
    "account_id": "acct-demo",
    "user_id": "u-alice",
    "agent_id": "main",
    "session_id": "s-002",
    "trace_id": "t-002",
    "prompt": "What is the coding style preference?"
  }' | python3 -m json.tool
```

预期输出：
```json
{
  "estimatedTokens": 0,
  "messages": [],
  "systemPromptAddition": "[oG-Memory 长期记忆系统] 以下是从向量数据库中语义检索到的用户相关记忆... - [preference] 用户表示喜欢使用 Python 编写代码，尤其偏好异步编程。(相关度: 64%)"
}
```

> 注意：assemble 使用 `prompt` 字段（不是 `query`）。

### 健康检查端点

```bash
curl http://localhost:8090/api/v1/health
```

---

## 停止服务

```bash
# 在 ogc serve 终端按 Ctrl+C

# 或从另一个终端
ogc stop

# 手动停止 IndexService
kill $(ps aux | grep run_index_service | grep -v grep | awk '{print $2}')
```

---

## OpenClaw 插件安装（可选）

```bash
# 安装插件到 ~/.openclaw/extensions/
ogc plugin install

# 查看状态
ogc plugin status

# 卸载
ogc plugin uninstall
```

安装后插件会自动配置 HTTP 模式，连接 `http://127.0.0.1:8090`。

---

## 故障排查

| 问题 | 原因 | 解决方法 |
|------|------|---------|
| `OPENGUASS_CONNECTION_STRING` 只有 `host=127.0.0.1` | .env 值未用引号包裹 | 确保 `OPENGUASS_CONNECTION_STRING="host=127.0.0.1 port=8888 ..."` |
| IndexService 报 `OpenAI API key is required` | 环境变量未导出 | 使用 `set -a && source .env && set +a` |
| assemble 返回 "No context available" | IndexService 未运行或未同步 | 启动 IndexService 并等待 30 秒轮询 |
| assemble 用 `query` 字段查询无结果 | 字段名错误 | 使用 `prompt` 字段，不是 `query` |
| AGFS 连接失败 | agfs-server 未运行 | `ogc serve` 会自动启动 AGFS |
| openGauss 连接失败 | Docker 容器未启动 | `docker start opengauss` 或使用 `ogon` alias |

---

## API 端点清单

| 端点 | 方法 | 说明 |
|------|------|------|
| `/api/v1/health` | GET | 健康检查 |
| `/api/v1/assemble` | POST | 语义检索记忆（字段: `prompt`） |
| `/api/v1/after_turn` | POST | 写入记忆 |
| `/api/v1/ingest` | POST | 单条消息摄入 |
| `/api/v1/ingest_batch` | POST | 批量消息摄入 |
