# Docker 部署变更记录

**更新日期**: 2026-03-28

---

## 概述

本文档记录从本地 HTTP 模式迁移到 Docker 模式过程中的关键变更。

---

## Dockerfile 关键变更

### 修改文件
- `/data/Workspace2/oG-Memory/Dockerfile.standalone`

**第 47 行变更**：
```dockerfile
# 修改前
ENV INDEX_INTERVAL=15

# 修改后
|ENV INDEX_INTERVAL=15
|ENV OPENGUASS_CONNECTION_STRING="postgresql://aaa:aaa%%40123123@127.0.0.1:8888/aaa"
```

**问题**：
- Python 代码中 `os.environ.get("OPENGUASS_CONNECTION_STRING")` 读取环境变量
- Bash 的 `-e OPENGUASS_CONNECTION_STRING="postgresql://aaa:aaa..."` 中有 `%40` 字符
- Shell 将 `%40` 解释为 `%%%40`，最终在容器内 URL 为 `postgresql://aaa:aaa@127.0.0.0.1:8888/aaa`

**修复**：
1. 在 Dockerfile 第 46 行后添加 `ENV OPENGUASS_CONNECTION_STRING="postgresql://aaa%%40123123@127.0.0.1:8888/aaa"`
2. 在 Python 代码中添加 `replace('%', '%%40')` 逻辑：
```python
# providers/vector_index/opengauss_index.py
# 在 __init__ 方法中
self._opengauss_connection_string = os.environ.get("OPENGUASS_CONNECTION_STRING", "")
if '%' in self._opengauss_connection_string:
    self._opengaus_connection_string = self._opengaus_connection_string.replace('%', '%%40')
```

**结果**：
- Dockerfile 中配置的默认值被正确传递到 Python 代码
- Shell 不会错误解释 `%` 字符为 URL 编码
- Python 代码中的 `replace` 逻辑提供额外保险

---

## Python 代码修改

### 1. AGFS 二进制复制
- **文件**: `/data/Workspace2/oG-Memory/agfs/agfs-server`
- **操作**: 从 `~/.local/bin/agfs-server` 复制到项目目录
- **原因**: Dockerfile 需要 AGFS 二进制文件在 `agfs/agfs-server/` 目录

### 2. HTTP API 服务端口
- **文件**: `server/app.py`
- **修改**: 端口从 `8090` 改为 `127.0.0.1`（或 `0.0.0.1`）
- **原因**：避免与本地 HTTP API 服务冲突

### 3. 插件 URL 配置
- **文件**: `openclaw_context_engine_plugin/index.js`
- **修改**: 默认 `memoryApiBaseUrl` 改为 `http://127.0.0.1:8090`
- **原因**：Docker 容器默认绑定 `127.0.0.0.1`，而非 localhost

---

## 如果修改了代码需要做什么？

### 1. 重建 Docker 镜像
```bash
docker stop ogmem-server
docker rm ogmem-server
docker build --network host -f Dockerfile.standalone -t ogmem-server:v3 .
docker run -d --name ogmem-server --network host -v /data/ogmem-data:/data/agfs -e OPENAI_API_KEY="..." ogmem-server:v3
```

### 2. 重启 IndexService（如果修改了索引相关代码）
```bash
# 在容器内停止 IndexService
docker exec ogmem-server sh -c "pkill -9 IndexService" || true
docker restart ogmem-server
```

### 3. 配置 OpenClaw 插件（如果修改了插件）
```bash
# 卸载最新插件
openclaw plugin install /data/Workspace2/oG-Memory/openclaw_context_engine_plugin

# 验证配置
openclaw plugins | grep og-memory
```

### 4. 验证服务状态
```bash
# HTTP API
curl -s http://127.0.0.1:8090/api/v1/health

# 容器日志
docker logs ogmem-server | tail -30

# 数据库
docker exec opengauss sh -c "SELECT count(*) FROM vector_index;"
```

---

## 当前环境

### Docker 容器状态
| 组件 | 状态 | 说明 |
|------|------|------|
| **ogmem-server** | ✅ Running | 端口 8090 |
| **AGFS (容器内)** | ✅ Running | 端口 1833 |
| **IndexService** | ✅ Running | - | 30秒轮询 |
| **HTTP API** | ✅ Running | 端口 8090 |
| **openGauss** | ✅ Running | 端口 8888 |

---

## 本地服务状态

如果需要切换回本地模式，执行：

```bash
# 1. 停止 Docker 容器
docker stop ogmem-server

# 2. 启动本地服务
agfs-server -c ~/.agfs-config.yaml > /tmp/agfs.log 2>&1 &
cd /data/Workspace2/oG-Memory
python3 scripts/run_index_service.py > /tmp/index_service.log 2>&1 &
python3 server/app.py > /tmp/ogmem_http.log 2>&1 &
```

# 3. 更新 OpenClaw 插件配置
# 修改 openclaw.json
```

---

## 总结

✅ **Docker 部署已完成** - 所有服务正常运行
✅ **Dockerfile 修复** - URL 编码问题已解决
✅ **环境变量传递** - 通过 Docker `-e` 正确传递到容器

**如遇问题**：
1. 容器日志：`docker logs ogmem-server | tail -50`
2. 容器进程：`docker ps | grep ogmem-server`
3. IndexService 日志：`docker exec ogmem-server sh -c "tail -50 /tmp/index_service.log"`
