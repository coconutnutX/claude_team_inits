# Context Engine 开发规格文档

> 基于：Phase 0 环境调查报告（2026-04-01）
> 约束：后续实施阶段的所有代码必须严格遵循此文档，不得偏离

---

## 1. CLI 命令规格

包名：`context-engine`（已存在于 pyproject.toml）
CLI 入口：`ogc`

| 命令 | 作用 |
|------|------|
| `ogc serve` | 启动服务：① 通过 `subprocess` 启动 `agfs-server` 进程，轮询健康检查确认就绪 ② 启动 API server。openGauss 不由 ogc 管理（用 `ogon` alias 单独启动），ogc 仅检查连通性 |
| `ogc stop` | 停止服务：发送 SIGTERM 给 AGFS 子进程 → 等 5s → SIGKILL → 验证端口已释放。API server 随主进程退出 |
| `ogc setup` | 交互式初始化：生成 .env |
| `ogc health-check` | 检查依赖状态：AGFS、openGauss、环境变量 |
| `ogc plugin install` | 安装 OpenClaw 插件（自动清理旧版本） |
| `ogc plugin uninstall` | 卸载 OpenClaw 插件 |
| `ogc plugin status` | 查看插件安装状态 |

**框架**：Click（Python CLI 库）

---

## 2. AGFS 启动规格

> 数据来源：Phase 0 报告 §1

| 项目 | 精确值 |
|------|--------|
| 二进制名 | `agfs-server` |
| 检测路径优先级 | 1. `AGFS_BINARY` 环境变量 2. `~/.local/bin/agfs-server` 3. 系统 PATH |
| 版本 | 1.4.0（参考值，不强制） |
| 启动命令 | `agfs-server -c <config_path>` |
| 监听地址 | `0.0.0.0:1833` |
| 健康检查 URL | `GET http://localhost:1833/` |
| 健康检查预期响应 | `{"service":"AGFS Server","status":"running",...}` |
| 就绪等待 | 轮询健康检查，间隔 0.5s，超时 10s |
| 停止流程 | SIGTERM → 等 5s → SIGKILL |
| 外部 AGFS 支持 | 环境变量 `AGFS_EXTERNAL_URL` 已设置时，跳过检测和进程管理 |

### AGFS 配置文件生成规格

**生成位置**：`<project_root>/agfs-config.yaml`（项目根目录）

**配置文件内容**（每个字段名精确匹配 Phase 0 调查结果）：
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
      local_dir: <AGFS_DATA_DIR>
```

**`local_dir` 取值优先级**：
1. `AGFS_DATA_DIR` 环境变量
2. 默认值：`/data/agfs`

**注意**：`path: /local` 是 AGFS 挂载路径前缀，统一使用 `/local`（Phase 0 ISSUE-01 决策）。代码中 `agfs_mount_prefix` 的 `/local/plugin` bug 用单独 commit 修复，不在本项目范围。

---

## 3. openGauss 连接规格

> 数据来源：Phase 0 报告 §2 + 用户提供的 ~/.bashrc alias

### 连接参数

| 项目 | 精确值 |
|------|--------|
| 环境变量名 | `OPENGUASS_CONNECTION_STRING`（来源：providers/config.py:77） |
| 连接串格式 | `host=127.0.0.1 port=8888 dbname=postgres user=aaa password=aaa@123123` |
| 宿主机端口 | `8888` |
| 容器内端口 | `5432` |
| 数据库名 | `postgres` |
| 用户名 | `aaa` |
| 密码 | `aaa@123123` |

### openGauss Docker 启动命令（参考，ogc 不负责启动）

```bash
docker run --name opengauss --privileged=true -d \
  -e GS_PASSWORD=Gauss@1234 \
  -e GS_USERNAME=aaa \
  -e GS_USER_PASSWORD=aaa@123123 \
  -p 8888:5432 \
  -v /data/opengauss_data:/var/lib/opengauss \
  opengauss:6.0.3
```

### 连通性检测方式

```python
import psycopg2
try:
    conn = psycopg2.connect(os.environ["OPENGUASS_CONNECTION_STRING"])
    conn.close()
    return True
except Exception:
    return False
```

**openGauss 无需额外安装 pgvector 扩展**，已内置向量能力。

**DB 初始化由业务代码自动处理**（`opengauss_index.py` 的 `ensure_table` 方法，首次写入时自动建表），ogc 不提供 `db init` 命令。代码中 `ensure_table` 方法会在首次使用时自动建表，无需独立初始化步骤。

> 数据来源：Phase 0 报告 §3

| 项目 | 精确值 |
|------|--------|
| 插件源码目录 | `<project_root>/openclaw_context_engine_plugin/` |
| 安装目标目录 | `~/.openclaw/extensions/og-memory-context-engine/` |
| 插件 ID | `og-memory-context-engine` |
| 插件版本 | 0.1.0（来自 package.json） |
| OpenClaw 配置文件 | `~/.openclaw/openclaw.json` |

### `ogc plugin install` 行为

1. 检查 `~/.openclaw/extensions/og-memory-context-engine/` 是否已存在
   - 已存在 → 先删除整个目录
2. 复制 `<project_root>/openclaw_context_engine_plugin/` 到 `~/.openclaw/extensions/og-memory-context-engine/`
3. 检查 `~/.openclaw/openclaw.json` 中是否已注册插件配置
   - 未注册 → 在 `plugins` 段中注入以下配置：

```json
{
  "plugins": {
    "enabled": true,
    "allow": ["og-memory-context-engine"],
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

**地址配置注入方式**：`memoryApiBaseUrl` 的值取自 `OGMEM_HTTP_PORT` 环境变量（默认 `8090`），拼接为 `http://127.0.0.1:{port}`。

### `ogc plugin uninstall` 行为

1. 删除 `~/.openclaw/extensions/og-memory-context-engine/` 整个目录

### `ogc plugin status` 行为

1. 检查 `~/.openclaw/extensions/og-memory-context-engine/` 是否存在
2. 检查 `~/.openclaw/openclaw.json` 中是否注册了插件配置
3. 输出状态信息

**注意**：插件是 JS 代码，`ogc` 只是搬运工。不改写为 Python，不修改 index.js 业务逻辑。

---

## 5. API Server 启动规格

> 数据来源：Phase 0 报告 §4.4

| 项目 | 精确值 |
|------|--------|
| 入口模块 | `server/app.py` |
| 框架 | Flask |
| 端口环境变量 | `OGMEM_HTTP_PORT`（来源：server/app.py:141） |
| 默认端口 | `8090` |
| 监听地址 | `0.0.0.0` |
| 开发启动命令 | `python server/app.py` |
| 生产启动命令 | `gunicorn -w 2 -b 0.0.0.0:8090 server.app:app` |

**`ogc serve` 启动 API server 的方式**：用 `subprocess.Popen` 启动 `python server/app.py`，设置 `PYTHONPATH` 确保模块可导入。

---

## 6. pyproject.toml 规格

> 数据来源：Phase 0 报告 §4.3

### 需要修改的字段

**1. 添加 `[project.scripts]` 段**（当前不存在）：
```toml
[project.scripts]
ogc = "cli:main"
```

**2. 补充 dependencies**（修复 ISSUE-02, ISSUE-03）：
```toml
dependencies = [
    "pyagfs>=1.4.0",
    "openai>=1.0.0",
    "apscheduler>=3.10.0",
    "flask>=3.0.0",
    "psycopg2-binary>=2.9.0",
    "click>=8.0.0",
    "python-dotenv>=1.0.0",
]
```

**3. 在 `[tool.hatch.build.targets.wheel]` 的 `packages` 中添加 CLI 包**：
```toml
packages = [
    "core",
    "fs",
    "extraction",
    "commit",
    "index",
    "retrieval",
    "providers",
    "service",
    "cli",           # ← 新增
]
```

**4. 添加数据文件包含（JS 插件）**：
```toml
[tool.hatch.build.targets.wheel.shared-data]
# 插件源码保留在包内，供 ogc plugin install 复制
```

实际上插件源码在 `openclaw_context_engine_plugin/` 目录，通过 `package-data` 或在 CLI 代码中用 `__file__` 相对路径定位。

---

## 7. .env.example 规格

> 数据来源：Phase 0 报告 §4.6 + §4.2

```bash
# ============================================
# Context Engine 配置
# 复制此文件为 .env 并填入实际值
# ============================================

# ---- openGauss 数据库 ----
# 启动: ogon (alias in ~/.bashrc)
OPENGUASS_CONNECTION_STRING=host=127.0.0.1 port=8888 dbname=postgres user=aaa password=aaa@123123
VECTOR_DB_TYPE=opengauss

# ---- OpenAI API ----
OPENAI_API_KEY=sk-your-api-key-here
OPENAI_BASE_URL=https://api.openai.com/v1
OPENAI_EMBEDDING_MODEL=text-embedding-ada-002
OPENAI_LLM_MODEL=gpt-4o-mini

# ---- oG-Memory Server ----
OGMEM_HTTP_PORT=8090

# ---- AGFS ----
AGFS_BASE_URL=http://localhost:1833
# AGFS_EXTERNAL_URL=http://some-host:1833   # 设此值则跳过本地 AGFS 启动
# AGFS_DATA_DIR=/data/agfs                  # AGFS 数据存储目录

# ---- 默认账户 ----
OG_ACCOUNT_ID=acct-demo
OG_USER_ID=u-alice
OG_AGENT_ID=main

# ---- Provider ----
CONTEXTENGINE_PROVIDER=openai
```

**所有变量名必须与 Phase 0 报告 §4.2 中的拼写完全一致**，不得重命名。

---

## 8. CLI 模块结构规格

新增文件：`cli.py`（项目根目录）

```
oG-Memory/
├── cli.py                   # ← 新增：CLI 入口，Click 命令组
├── server/
│   └── app.py               # 已存在，不修改
├── openclaw_context_engine_plugin/  # 已存在，不修改
└── ...
```

**不创建新的子目录**。CLI 代码放在项目根目录的单个 `cli.py` 文件中。

### CLI 函数结构

```python
# cli.py

import click

@click.group()
def main():
    """Context Engine CLI - oG-Memory management tool"""
    pass

@main.command()
def serve():
    """Start Context Engine service (AGFS + API server)"""
    pass

@main.command()
def stop():
    """Stop running Context Engine service"""
    pass

@main.command()
def setup():
    """Interactive setup: generate .env"""
    pass

@main.command(name="health-check")
def health_check():
    """Check dependency status: AGFS, openGauss, env vars"""
    pass

@main.group()
def plugin():
    """OpenClaw plugin management"""
    pass

@plugin.command(name="install")
def plugin_install():
    """Install OpenClaw plugin (auto-cleanup old version)"""
    pass

@plugin.command(name="uninstall")
def plugin_uninstall():
    """Uninstall OpenClaw plugin"""
    pass

@plugin.command(name="status")
def plugin_status():
    """Show plugin installation status"""
    pass

if __name__ == "__main__":
    main()
```

---

## 9. 实施顺序

| 步骤 | 任务 | 验证方式 | commit 类型 |
|------|------|----------|-------------|
| 1 | 修改 pyproject.toml：添加 scripts、补充 dependencies | `pip install -e .` 成功，`which ogc` 能找到 | `feat` |
| 2 | 新增 cli.py 骨架 | `ogc --help` 输出命令列表 | `feat` |
| 3 | 实现 AGFS 检测 + 子进程启动（`subprocess.Popen`）| `ogc serve` 能启动 AGFS 子进程 + 健康检查通过，`ogc stop` 能关闭并验证端口释放 | `feat` |
| 4 | 串联 API server 启动 | `ogc serve` 后 health 端点可访问 | `feat` |
| 5 | 新增 .env.example + dotenv 加载 | 环境变量从 .env 正确加载 | `feat` |
| 6 | 实现 ogc setup | 交互式生成 .env | `feat` |
| 7 | 实现 ogc health-check | 输出各组件状态 | `feat` |
| 8 | 实现 ogc plugin install/uninstall/status | 插件正确安装到 ~/.openclaw/extensions/ | `feat` |

每步完成后立即 commit 并验证，不攒大 commit。

### 错误处理要求

**所有 CLI 命令函数必须使用 try/except 包裹业务逻辑**，捕获所有异常并打印完整 traceback，方便用户 debug。不要吞掉异常。

```python
import traceback

@main.command()
def serve():
    """Start Context Engine service (AGFS + API server)"""
    try:
        # ... business logic ...
        pass
    except Exception as e:
        click.echo(f"Error: {e}", err=True)
        traceback.print_exc()
        raise SystemExit(1)
```

**每个命令都遵循此模式**：`try` 包裹整个业务逻辑 → `except Exception` 打印完整错误 → `SystemExit(1)`。

---

## 10. 不做的事情

| 禁止项 | 原因 |
|--------|------|
| 修改现有业务代码 | 仅新增部署封装层 |
| 修改现有 `os.getenv()` 调用或变量名 | 保留环境变量方式 |
| 引入 config.toml / config.yaml 作为项目配置 | 保留环境变量方式 |
| 创建新的子目录来重组现有代码 | 不修改目录结构 |
| 修改 server/app.py 或 server/memory_service.py | CLI 复用现有启动逻辑 |
| 将 JS 插件改写为 Python | 不改语言 |
| 修改 index.js 业务逻辑 | 只搬运 |
| 实现 AGFS 自动下载 | 本阶段只做检测+启动 |
| 凭记忆填写端口、路径、配置字段名 | 必须从本规格文档取值 |
| 修改 agfs_mount_prefix 的 bug | 单独 commit，不在本项目范围 |
