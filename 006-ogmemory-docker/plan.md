# Docker Compose 迁移计划

## 1. 分析总结

### 现有架构
- **单容器模式**: 通过 `docker run` 启动一个容器 (`ogmem-openclaw:latest`)
- **容器内进程编排**: entrypoint.sh 启动三个进程
  - AGFS Server (:1833, 后台)
  - IndexService (后台守护进程)
  - OpenClaw Gateway (:18789, 前台主进程)
- **外部依赖**: openGauss 数据库（独立运行，需开启 vector 扩展）
- **网络模式**: `--network host`
- **现有文件**:
  - `.env.example` 已存在，包含环境变量模板
  - `Dockerfile` 已存在
  - `docker/` 目录下的配置文件已存在

## 2. Docker Compose 转换设计

### 2.1 服务定义原则
**不改变容器数量、用途，完全复用现有文件**
- 单一 service: `ogmem`
- 镜像: `ogmem-openclaw:latest` (需先构建)
- 保持所有现有环境变量默认值
- **不修改** docker_deployment_guide.md

### 2.2 新建文件
```
oG-Memory/
├── docker-compose.yml          # 新建
└── .env.example                # 复用（不修改）
```

### 2.3 docker-compose.yml 设计

```yaml
version: "3.8"

services:
  ogmem:
    image: ogmem-openclaw:latest
    container_name: ogmem
    network_mode: host
    restart: unless-stopped

    volumes:
      - ${OGMEM_AGFS_DATA_DIR:-./data/agfs}:/data/agfs
      - ${OGMEM_OPENCLAW_STATE_DIR:-./data/openclaw-state}:/home/node/.openclaw

    environment:
      # 必填参数
      - OPENAI_API_KEY=${OPENAI_API_KEY}
      - OPENAI_LLM_MODEL=${OPENAI_LLM_MODEL}
      - OPENGUASS_CONNECTION_STRING=${OPENGUASS_CONNECTION_STRING}

      # 可选参数
      - OPENAI_BASE_URL=${OPENAI_BASE_URL:-https://dashscope.aliyuncs.com/compatible-mode/v1}
      - OPENAI_EMBEDDING_MODEL=${OPENAI_EMBEDDING_MODEL:-text-embedding-v2}
      - OPENCLAW_GATEWAY_TOKEN=${OPENCLAW_GATEWAY_TOKEN:-test-token-123}
      - OG_ACCOUNT_ID=${OG_ACCOUNT_ID:-acct-demo}
      - OG_USER_ID=${OG_USER_ID:-u-alice}
      - OG_AGENT_ID=${OG_AGENT_ID:-main}
      - INDEX_INTERVAL=${INDEX_INTERVAL:-15}
```

## 3. 执行步骤

### Step 1: 创建 docker-compose.yml
- 位置: `/data/Workspace2/oG-Memory/docker-compose.yml`

### Step 2: 验证部署

#### 3.1 网页访问 OpenClaw（需要人类协助）
```bash
# 启动服务
docker-compose up -d

# 查看日志
docker-compose logs -f
```

**人类验证步骤**:
1. 浏览器访问 `http://<服务器IP>:18789`
2. 输入 Gateway Token (默认 `test-token-123`)
3. 确认前端能正常加载、连接成功

#### 3.2 验证 AGFS 写入
```bash
# 查看 AGFS 数据目录
ls -la ./data/agfs/

# 查看记忆文件结构
find ./data/agfs/ -type f -name "*.md" 2>/dev/null | head -20
```

**预期结果**: 应该能看到 `/accounts/{account_id}/.../memories/` 目录结构

#### 3.3 验证 openGauss 写入（提供 SQL 给人类）

```sql
-- 1. 连接到 openGauss
gsql -h 127.0.0.1 -p 8799 -d postgres -U <username>

-- 2. 检查 vector_index 表是否存在
\d vector_index;

-- 3. 查看索引记录数
SELECT COUNT(*) FROM vector_index;

-- 4. 查看最近的索引记录
SELECT
    id,
    uri,
    level,
    substring(text, 1, 100) as text_preview,
    created_at
FROM vector_index
ORDER BY created_at DESC
LIMIT 10;

-- 5. 检查特定 account 的记录
SELECT COUNT(*), category
FROM vector_index,
     jsonb_to_recordset(filters) as f(account_id text, owner_space text)
WHERE f.account_id = 'acct-demo'
GROUP BY category;

-- 6. 验证向量索引是否创建成功
\d idx_vector_index_embedding_hnsw;
```

### Step 3: 验证通过后更新文档

#### 3.1 删除 ENV.md
```bash
rm /data/Workspace2/oG-Memory/ENV.md
```

#### 3.2 更新 README.md（精简、开发者友好）

新的 README.md 结构：
```markdown
# oG-Memory

面向 AI Agent 的长期记忆基础设施。

## 快速部署

```bash
# 1. 配置环境变量
cp .env.example .env
# 编辑 .env，填入必填项：OPENAI_API_KEY, OPENAI_LLM_MODEL, OPENGUASS_CONNECTION_STRING

# 2. 构建镜像
docker build --network host -t ogmem-openclaw:latest .

# 3. 启动服务
docker-compose up -d

# 4. 访问前端
# 浏览器打开 http://<服务器IP>:18789
# Gateway Token: 默认 test-token-123（可在 .env 中修改）
```

## 验证检查

```bash
# 查看 AGFS 写入的数据/
ls -la ./data/agfs/

# 进入数据库验证（需要手动执行上面的 SQL）
```

## 文档

- [CLAUDE.md](CLAUDE.md) - 完整技术文档
- [architecture/docker_deployment_guide.md](architecture/docker_deployment_guide.md) - Docker 部署详细指南
```

## 4. 不变项清单

| 项目 | 状态 |
|------|------|
| Dockerfile | 不修改 |
| docker/entrypoint.sh | 不修改 |
| docker/openclaw.json | 不修改 |
| docker/agfs-config.yaml | 不修改 |
| .env.example | 不修改 |
| architecture/docker_deployment_guide.md | 不修改 |
| 容器内进程编排逻辑 | 不修改 |
| 网络模式 (host) | 保持不变 |
| 端口 (18789, 1833) | 保持不变 |
| 数据卷路径 | 保持不变 |
| 环境变量名称和默认值 | 保持不变 |

## 5. 待确认事项

- [ ] 验证通过后再删除 ENV.md 和更新 README.md
- [ ] 确认 openGauss 连接信息（由人类提供）
