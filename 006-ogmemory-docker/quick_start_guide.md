# oG-Memory 快速启动指南

> 用于快速拉起和验证 oG-Memory 服务的简易教程

---

## 目录

- [1. 环境准备](#1-环境准备)
- [2. 启动服务](#2-启动服务)
- [3. CLI 测试](#3-cli-测试)
- [4. OpenClaw 交互](#4-openclaw-交互)
- [5. Dashboard 交互](#5-dashboard-交互)
- [6. 查看日志](#6-查看日志)
- [7. 检查 openGauss](#7-检查-opengauss)
- [8. 常见问题](#8-常见问题)

---

## 1. 环境准备

### 1.1 检查 Docker

```bash
docker --version
```

### 1.2 检查端口占用

```bash
# 检查 8090 (oG-Memory API)
netstat -tuln | grep 8090

# 检查 8888 (openGauss)
netstat -tuln | grep 8888

# 检查 18789 (OpenClaw Gateway)
netstat -tuln | grep 18789
```

### 1.3 清理旧容器（如需）

```bash
# 停止并删除旧容器
docker stop opengauss ogmem-server 2>/dev/null
docker rm opengauss ogmem-server 2>/dev/null

# 查看当前容器状态
docker ps -a
```

---

## 2. 启动服务

### 2.1 方法一：使用启动脚本（推荐）

```bash
cd /data/Workspace2/oG-Memory

# 使用内置启动脚本
./.claude-team/start-docker-deployment.sh
```

### 2.2 方法二：手动启动

```bash
cd /data/Workspace2/oG-Memory

# 1. 启动 openGauss
docker run --name opengauss --privileged=true -d \
  -e GS_PASSWORD=Gauss@1234 \
  -e GS_USERNAME=aaa \
  -e GS_USER_PASSWORD=aaa@123123 \
  -p 8888:5432 \
  -v /data/opengauss_data:/var/lib/opengauss \
  opengauss:6.0.3

# 2. 等待 openGauss 启动（约 10-15 秒）
for i in {1..15}; do
  if docker exec opengauss sh -c "LD_LIBRARY_PATH=/usr/local/opengauss/lib /usr/local/opengauss/bin/gsql -d postgres -U aaa -W aaa@123123 -p 5432 -c 'SELECT 1'" 2>/dev/null | grep -q "1"; then
    echo "openGauss 就绪"
    break
  fi
  sleep 1
done

# 3. 检查/构建 oG-Memory 镜像
docker images | grep ogmem-server || docker build -f Dockerfile.standalone -t ogmem-server:v4 .

# 4. 启动 oG-Memory 容器
docker stop ogmem-server 2>/dev/null
docker rm ogmem-server 2>/dev/null

docker run -d \
  --name ogmem-server \
  --network host \
  -v /data/ogmem-data:/data/agfs \
  -e OPENAI_API_KEY="sk-sxWGh4hWeExbe8sqZEkgBi4E9l8E53oaAaoYEzjxbzR5IOgk" \
  -e OPENAI_BASE_URL="https://chatapi.littlewheat.com/v1" \
  -e OPENAI_EMBEDDING_MODEL="text-embedding-3-small" \
  -e OPENAI_LLM_MODEL="gpt-4o-mini" \
  -e 'OPENGUASS_CONNECTION_STRING=postgresql://aaa:aaa%40123123@127.0.0.1:8888/aaa' \
  -e OG_ACCOUNT_ID="aaa" \
  -e INDEX_INTERVAL="15" \
  -e AGFS_MOUNT_PREFIX="/local" \
  ogmem-server:v4

# 5. 等待服务启动（约 20 秒）
sleep 20

# 6. 健康检查
curl -s http://127.0.0.1:8090/api/v1/health | python3 -m json.tool
```

### 环境变量说明

| 变量 | 默认值 | 说明 |
|------|---------|------|
| `OPENAI_API_KEY` | - | OpenAI API 密钥（必填） |
| `OPENAI_BASE_URL` | `https://chatapi.littlewheat.com/v1` | API 基础 URL |
| `OPENAI_EMBEDDING_MODEL` | `text-embedding-3-small` | Embedding 模型 |
| `OPENAI_LLM_MODEL` | `gpt-4o-mini` | LLM 模型 |
| `OPENGUASS_CONNECTION_STRING` | - | openGauss 连接字符串 |
| `OG_ACCOUNT_ID` | `acct-demo` | 账户 ID |
| `AGFS_MOUNT_PREFIX` | `/local` | AGFS 挂载前缀 |
| `INDEX_INTERVAL` | `15` | 索引间隔（秒） |
| `OGMEM_HTTP_PORT` | `8090` | HTTP 服务端口 |

---

## 3. CLI 测试

### 3.1 测试 API 可用性

```bash
# 健康检查
curl http://127.0.0.1:8090/api/v1/health

# Bootstrap（初始化会话）
curl -X POST http://127.0.0.1:8090/api/v1/bootstrap \
  -H "Content-Type: application/json" \
  -d '{
    "agentId": "test-agent",
    "sessionId": "test-session",
    "userId": "test-user",
    "accountId": "aaa"
  }'
```

### 3.2 测试写入记忆

```bash
# 写入对话到长期记忆
curl -X POST http://127.0.0.1:8090/api/v1/after_turn \
  -H "Content-Type: application/json" \
  -d '{
    "agentId": "test-agent",
    "sessionId": "test-session",
    "userId": "test-user",
    "accountId": "aaa",
    "messages": [
      {"role": "user", "content": "我叫张三，是后端工程师，喜欢用 Python 开发"},
      {"role": "assistant", "content": "好的，我会记住你是后端工程师"}
      {"role": "user", "content": "我平时喜欢喝咖啡"}
    ]
  }' | python3 -m json.tool
```

### 3.3 测试检索记忆

```bash
# 检索记忆（通过 assemble 接口）
curl -X POST http://127.0.0.1:8090/api/v1/assemble \
  -H "Content-Type: application/json" \
  -d '{
    "agentId": "test-agent",
    "sessionId": "test-session",
    "userId": "test-user",
    "accountId": "aaa",
    "messages": [],
    "prompt": "张三喜欢什么？"
  }' | python3 -m json.tool
```

---

## 4. OpenClaw 交互

### 4.1 安装/配置 OpenClaw 插件

```bash
# 检查 OpenClaw 安装
which openclaw

# 配置插件（如需要）
openclaw plugin install /data/Workspace2/oG-Memory/openclaw_context_engine_plugin

# 检查插件状态
openclaw plugin list

# 查看插件配置
cat ~/.openclaw/openclaw.json | python3 -m json.tool
```

### 4.2 通过 OpenClaw CLI 测试

```bash
# 发送消息（会自动调用 oG-Memory）
openclaw chat "测试记忆：我是新用户，喜欢喝拿铁"

# 检查 OpenClaw 是否运行
openclaw gateway status
```

### 4.3 启动 OpenClaw Gateway

```bash
# 后台启动 Gateway
openclaw gateway &

# 查看进程
ps aux | grep -E "openclaw|gateway"

# 查看端口
netstat -tuln | grep 18789
```

---

## 5. Dashboard 交互

### 5.1 访问 Dashboard

```bash
# Dashboard 默认地址
# http://localhost:18789

# 或查看配置获取实际地址
cat ~/.openclaw/openclaw.json | python3 -m json.tool
```

### 5.2 在 Dashboard 中测试

1. 打开浏览器访问 `http://localhost:18789`
2. 在聊天框中输入测试消息
3. 查看返回内容，应包含 `[oG-Memory 长期记忆系统]` 相关提示

### 5.3 确认插件已启用

在 Dashboard 设置中确认：
- **Context Engine** 已启用
- 配置的 **Memory API Base URL** 为 `http://127.0.0.1:8090`

---

## 6. 查看日志

### 6.1 查看容器日志

```bash
# 实时查看日志
docker logs -f ogmem-server

# 查看最近 50 行
docker logs --tail 50 ogmem-server

# 搜索特定内容
docker logs ogmem-server 2>&1 | grep -E "after_turn|WriteAPI|ERROR|Exception"

# 查看索引服务日志
docker logs ogmem-server 2>&1 | grep -E "embed_texts|processed UPSERT|IndexService"
```

### 6.2 日志关键词说明

| 关键词 | 含义 |
|--------|------|
| `WriteAPI ready` | 写入 API 初始化成功 |
| `after_turn session=` | 会话提交处理 |
| `extracted=` | 提取的候选记忆数 |
| `completed=` | 写入成功的记忆数 |
| `embed_texts called` | 向量化处理中 |
| `processed UPSERT_CONTEXT` | 索引更新成功 |
| `AGFS server ready` | AGFS 文件服务器就绪 |

### 6.3 容器内查看日志

```bash
# 进入容器查看日志
docker exec -it ogmem-server sh

# 在容器内查看特定目录
docker exec ogmem-server ls -la /data/agfs/
docker exec ogmem-server ls -la /local/accounts/aaa/users/
```

---

## 7. 检查 openGauss

### 7.1 查看 AGFS 文件

```bash
# 查看 AGFS 中存储的原始文件
docker exec ogmem-server python3 -c "
import sys
sys.path.insert(0, '/opt/ogmem')
from pyagfs import AGFSClient

client = AGFSClient(api_base_url='http://127.0.0.1:1833')

def ls_recursive(path):
    try:
        items = client.ls(path)
        for item in items:
            full_path = f'{path}/{item[\"name\"]}' if path else item['name']
            print(full_path)
            if item.get('isDir'):
                ls_recursive(full_path)
    except:
        pass

ls_recursive('/local/accounts/aaa')
"
```

### 7.2 查看向量索引数据

```bash
# 查看所有记忆
docker exec opengauss sh -c "LD_LIBRARY_PATH=/usr/local/opengauss/lib /usr/local/opengauss/bin/gsql -d aaa -U aaa -W aaa@123123 -p 5432 -c \"SELECT id, uri, level, filters->>'category' as category, substring(text, 1, 60) as content FROM vector_index WHERE filters->>'account_id' = 'aaa' ORDER BY created_at DESC LIMIT 20;\""

# 查看特定用户的记忆
docker exec opengauss sh -c "LD_LIBRARY_PATH=/usr/local/opengauss/lib /usr/local/opengauss/bin/gsql -d aaa -U aaa -W aaa@123123 -p 5432 -c \"SELECT id, uri, level, filters->>'category' as category FROM vector_index WHERE filters->>'account_id' = 'aaa' AND uri LIKE '%users/test-user%' ORDER BY created_at DESC;\""

# 统计各类型记忆数量
docker exec opengauss sh -c "LD_LIBRARY_PATH=/usr/local/opengauss/lib /usr/local/opengauss/bin/gsql -d aaa -U aaa -W aaa@123123 -p 5432 -c \"SELECT filters->>'category' as category, COUNT(*) as count FROM vector_index WHERE filters->>'account_id' = 'aaa' GROUP BY filters->>'category' ORDER BY count DESC;\""

# 查看最新写入的 10 条记录
docker exec opengauss sh -c "LD_LIBRARY_PATH=/usr/local/opengauss/lib /usr/local/opengauss/bin/gsql -d aaa -U aaa -W aaa@123123 -p 5432 -c \"SELECT id, uri, level, filters->>'category' as category, substring(text, 1, 100) as text_preview FROM vector_index WHERE filters->>'account_id' = 'aaa' ORDER BY created_at DESC LIMIT 10;\""
```

### 7.3 向量搜索测试

```bash
# 计算查询向量（示例：测试文本的向量）
docker exec opengauss sh -c "LD_LIBRARY_PATH=/usr/local/opengauss/lib /usr/local/opengauss/bin/gsql -d aaa -U aaa -W aaa@123123 -p 5432 -c \"SELECT uri, embedding <-> '[0.1,0.2,...]' AS similarity FROM vector_index WHERE filters->>'account_id' = 'aaa' ORDER BY similarity DESC LIMIT 5;\""
```

---

## 8. 常见问题

### 8.1 oG-Memory API 返回 `{"ok": true}`

**原因**：可能是没有消息内容，或配置问题

**解决**：检查请求参数是否正确，查看日志确认是否调用

### 8.2 openGauss 数据没有更新

**原因**：索引服务没有处理，或数据没有写入 AGFS

**解决**：
```bash
# 查看索引日志
docker logs ogmem-server | grep -E "IndexService|processed|succeeded"

# 查看是否有 outbox 事件
docker exec ogmem-server python3 -c "
import sys
sys.path.insert(0, '/opt/ogmem')
from pyagfs import AGFSClient

client = AGFSClient(api_base_url='http://127.0.0.1:1833')
items = client.ls('/local/accounts/aaa/users/test-user/memories/profile/.outbox')
print('Outbox events:', items)
"
```

### 8.3 容器启动失败

**错误**：`Error response from daemon: Conflict. The container name is already in use`

**解决**：
```bash
# 停止旧容器
docker stop ogmem-server
docker rm ogmem-server

# 然后重新启动
docker run -d --name ogmem-server ...
```

### 8.4 AGFS 权限错误

**错误**：`failed to create directory: mkdir ... permission denied`

**解决**：
```bash
# 修复 AGFS 数据目录权限
docker exec -u root ogmem-server chown -R ogmem:ogmem /data/agfs

# 或在启动时使用正确的卷挂载
```

### 8.5 openGauss 连接失败

**错误**：连接超时或认证失败

**解决**：
```bash
# 检查 openGauss 是否运行
docker ps | grep opengauss

# 检查连接
docker exec opengauss sh -c "LD_LIBRARY_PATH=/usr/local/opengauss/lib /usr/local/opengauss/bin/gsql -d postgres -U aaa -W aaa@123123 -p 5432 -c 'SELECT 1'"

# 重启 openGauss
docker restart opengauss
```

---

## 快速验证检查清单

完成启动后，按以下顺序验证：

```bash
# 1. 容器状态
echo "=== 1. 容器状态 ==="
docker ps | grep -E "ogmem-server|opengauss"

# 2. 服务健康检查
echo -e "\n=== 2. 服务健康 ==="
curl -s http://127.0.0.1:8090/api/v1/health | python3 -m json.tool
curl -s http://127.0.0.1:18789/serverinfo/version

# 3. 写入测试
echo -e "\n=== 3. 写入测试 ==="
curl -X POST http://127.0.0.1:8090/api/v1/after_turn \
  -H "Content-Type: application/json" \
  -d '{"agentId": "qa", "sessionId": "qa", "userId": "qa", "accountId": "aaa", "messages": [{"role": "user", "content": "我是测试用户，用于验证记忆功能"}]}'

# 4. 检索测试
echo -e "\n=== 4. 检索测试 ==="
curl -X POST http://127.0.0.1:8090/api/v1/assemble \
  -H "Content-Type: application/json" \
  -d '{"agentId": "qa", "sessionId": "qa", "userId": "qa", "accountId": "aaa", "messages": [], "prompt": "测试用户是谁？"}'

# 5. 数据持久化检查（等待索引处理）
sleep 25
echo -e "\n=== 5. 数据持久化检查 ==="
docker exec opengauss sh -c "LD_LIBRARY_PATH=/usr/local/opengauss/lib /usr/local/opengauss/bin/gsql -d aaa -U aaa -W aaa@123123 -p 5432 -c \"SELECT COUNT(*) FROM vector_index WHERE filters->>'account_id' = 'aaa';\""

# 6. OpenClaw 测试
echo -e "\n=== 6. OpenClaw 测试 ==="
openclaw chat "QA验证：我刚才说我是什么？"
```

---

## 文件结构参考

### Docker 镜像构建
- **基础镜像**: `python:3.11-slim-bookworm`
- **工作目录**: `/opt/ogmem`
- **数据目录**: `/data/agfs` (AGFS 存储)
- **默认端口**: `8090`

### 关键路径
| 组件 | 路径 | 说明 |
|--------|------|------|
| oG-Memory 源码 | `/opt/ogmem/` | 容器内代码 |
| AGFS 配置 | `/opt/agfs/config.yaml` | AGFS 服务配置 |
| AGFS 数据 | `/data/agfs/` | 持久化文件存储 |
| HTTP API | `http://127.0.0.1:8090` | RESTful API |
| 向量索引 | openGauss 数据库 | 存储向量和元数据 |

---

## 版本信息

- **镜像版本**: `ogmem-server:v4`
- **修复内容**: `AGFS_MOUNT_PREFIX` 环境变量读取
- **修复文件**: `server/memory_service.py`
