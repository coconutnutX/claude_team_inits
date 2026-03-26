# IndexService 部署指南

## 架构说明

IndexService 是 oG-Memory 的异步索引同步服务，负责：
1. 定期扫描 AGFS 中的 `.outbox/` 目录
2. 处理待处理的 OutboxEvent
3. 调用 Embedder 生成向量
4. 调用 VectorIndex.upsert 写入向量索引（OpenGauss）

## 插件与 IndexService 的关系

```
OpenClaw Plugin (bridge/memory_api.py)
     ↓ 写入事件到 AGFS
AGFS: .outbox/{event_id}.json (PENDING 状态)
     ↑ 异步处理
IndexService (独立后台进程)
     ↓ 写入向量索引
OpenGauss: vector_index 表
```

**重要**：IndexService 是独立的后台服务，不应该在插件的生命周期中管理。

## 部署步骤

### 1. 启动 IndexService

```bash
# 方法1：使用启动脚本（推荐）
cd /data/Workspace2/oG-Memory
source ~/.bashrc  # 确保环境变量已加载
./scripts/start_index_service.sh

# 方法2：直接运行
cd /data/Workspace2/oG-Memory
export VECTOR_DB_TYPE=opengauss
python examples/run_index_service_opengauss.py --workers 3 --interval 30
```

### 2. 验证 IndexService 运行状态

```bash
# 检查进程
ps aux | grep index_service_opengauss

# 查看日志（如果有）
tail -f /tmp/index_service.log  # 如果启动脚本配置了日志

# 检查 AGFS 中的处理状态
find /tmp/agfs-data -name "*.processing" | head -5
```

### 3. 插件启动

```bash
# OpenClaw 插件会在每次启动时：
# 1. 设置 VECTOR_DB_TYPE=opengauss
# 2. 创建 OutboxStore
# 3. 依赖 IndexService 处理异步事件
```

## 环境变量

```bash
# 必需的环境变量
export AGFS_BASE_URL=http://localhost:1833
export AGFS_MOUNT_PREFIX=/local
export VECTOR_DB_TYPE=opengauss
export OPENGUASS_CONNECTION_STRING="postgresql://aaa:aaa@123123@localhost:8888/aaa"
export OPENAI_API_KEY=your_openai_api_key

# 可选的 IndexService 配置
export INDEX_SERVICE_WORKERS=3
export INDEX_SERVICE_INTERVAL=30
```

## 故障排查

### 问题：写入到 AGFS 成功，但 OpenGauss 中没有数据

**原因**：IndexService 没有运行或处理失败

**解决**：
1. 检查 IndexService 是否运行：`ps aux | grep run_index_service.py`
2. 查看 IndexService 日志
3. 确认环境变量设置：`echo $VECTOR_DB_TYPE`

### 问题：IndexService 启动失败

**可能原因**：
1. AGFS 未启动：`curl http://localhost:1833/health`
2. OpenGauss 连接失败：检查连接参数
3. OpenAI API Key 未设置：`echo $OPENAI_API_KEY`

**解决**：
1. 确保所有依赖服务正常运行
2. 检查环境变量是否正确设置
3. 查看详细的错误日志

### 日志位置

- IndexService 日志：控制台输出（可通过脚本重定向到文件）
- 插件日志：OpenClaw gateway 的 stderr
- AGFS 日志：`/tmp/agfs-data/` 目录下的日志文件

## 停止 IndexService

```bash
# 方法1：使用 PID 文件
kill $(cat /tmp/index_service.pid)

# 方法2：查找进程并杀掉
pkill -f index_service_opengauss

# 方法3：优雅停止（如果支持）
kill -SIGTERM $(pgrep -f index_service_opengauss)
```

## 监控指标

IndexService 提供以下统计信息：
- 处理的事件数量
- 成功率/失败率
- 平均处理时间
- 待处理事件队列长度

可以通过修改 `run_index_service_opengauss.py` 添加监控端点。