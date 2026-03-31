# oG-Memory 快速启动指南 (v2)

> 简化、优雅、稳定的 Docker 部署方案

---

## 🚀 快速开始

### 前置要求

- Docker >= 20.10
- Docker Compose >= 2.0
- 至少 4GB 内存

### 首次部署

```bash
# 1. 配置环境变量
cp .env.example .env
nano .env  # 必填: OPENAI_API_KEY

# 2. 启动服务
./scripts/deploy.sh start
```

### 日常使用

| 命令 | 作用 |
|------|------|
| `./scripts/deploy.sh status` | 查看状态 |
| `./scripts/deploy.sh logs` | 查看日志 |
| `./scripts/deploy.sh stop` | 停止服务 |
| `./scripts/deploy.sh restart` | 重启服务 |
| `./scripts/deploy.sh test` | 运行测试 |

### 服务地址

| 服务 | 端口 | 地址 |
|------|------|------|
| oG-Memory API | 8090 | http://localhost:8090 |
| OpenClaw Dashboard | 18789 | http://localhost:18789 |
| openGauss | 8888 | localhost:8888 |
| AGFS | 1833 | localhost:1833 |

---

## 🧪 测试验证

### API 测试

```bash
# 健康检查
curl http://localhost:8090/api/v1/health

# Bootstrap
curl -X POST http://localhost:8090/api/v1/bootstrap \
  -H "Content-Type: application/json" \
  -d '{"agentId":"test","sessionId":"test","userId":"test","accountId":"aaa"}'

# 写入记忆
curl -X POST http://localhost:8090/api/v1/after_turn \
  -H "Content-Type: application/json" \
  -d '{"agentId":"test","sessionId":"test","userId":"test","accountId":"aaa","messages":[{"role":"user","content":"我是张三"}]}'

# 检索记忆
curl -X POST http://localhost:8090/api/v1/assemble \
  -H "Content-Type: application/json" \
  -d '{"agentId":"test","sessionId":"test","userId":"test","accountId":"aaa","messages":[],"prompt":"张三是谁？"}'
```

### OpenClaw 集成

```bash
openclaw plugin install ./openclaw_context_engine_plugin
openclaw gateway
openclaw chat "测试记忆功能"
```

---

## 🐛 故障排查

| 问题 | 解决方法 |
|------|---------|
| 服务启动失败 | `./scripts/deploy.sh logs` 查看错误 |
| API 无法访问 | `curl http://localhost:8090/api/v1/health` |
| 端口冲突 | `netstat -tuln \| grep -E "8090|8888|1833"` |
| 索引不更新 | `docker logs ogmem-server \| grep IndexService` |

---

## 📝 配置说明

### 必填配置（.env）
- `OPENAI_API_KEY`: OpenAI API 密钥

### 可选配置
- `OPENAI_BASE_URL`: API 基础地址
- `OPENAI_EMBEDDING_MODEL`: Embedding 模型
- `OPENAI_LLM_MODEL`: LLM 模型
- `OG_ACCOUNT_ID`: 账户 ID
- `INDEX_INTERVAL`: 索引间隔（秒）

---

## 📁 文件说明

| 文件 | 作用 |
|------|------|
| `docker-compose.yml` | 服务编排 |
| `Dockerfile.standalone` | 主服务镜像 |
| `Dockerfile.agfs` | AGFS 服务镜像 |
| `scripts/deploy.sh` | 部署脚本 |
| `scripts/verify-deployment.sh` | 验证脚本 |
| `.env.example` | 配置模板 |

---

## 🆘 遇到问题？

1. 查看 `./scripts/deploy.sh logs`
2. 运行 `./scripts/deploy.sh test`
3. 查看容器状态: `docker-compose ps`
4. 重置环境: `./scripts/deploy.sh reset`