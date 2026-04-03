# Phase 2: 配置迁移改造计划

> 编写时间：2026-04-02
> 基础文档：`phase1-config-inventory.md`

---

## 1. 最终 `config.example.yaml`

```yaml
# config.example.yaml
# cp config.example.yaml config.yaml

# ============================================================
# oG-Memory HTTP 服务
# ============================================================
ogmem-server:
  host: "0.0.0.0"           # 监听地址
  port: 8090                 # HTTP API 端口
  workers: 2                 # Gunicorn worker 数（仅 standalone 模式）
  log_level: "info"          # debug / info / warning / error

# ============================================================
# AGFS 文件存储服务
# ============================================================
agfs:
  base_url: "http://127.0.0.1:1833"   # AGFS 连接地址（Python 代码使用）
  mount_prefix: "/local/plugin"        # AGFS 挂载前缀
  config_path: "docker/agfs-config.yaml"  # AGFS 自身配置文件路径（用于启动子进程）
  data_dir: "/data/agfs"              # AGFS 数据目录（仅启动子进程时使用）
  bin_path: ""                     # agfs-server 二进制路径，空则自动查找 PATH
  # external: false          # 设为 true 则跳过本地 AGFS 进程管理，直接使用 base_url

# ============================================================
# openGauss 向量数据库
# ============================================================
opengauss:
  host: "127.0.0.1"
  port: 5432
  dbname: "postgres"
  user: "aaa"
  password: ""               # [必填]
  table_name: "vector_index"
  pool_size: 5
  dimension: 1536

# ============================================================
# LLM / Embedding 服务
# ============================================================
llm:
  provider: "mock"           # mock / openai / openai-cached
  vector_db_type: "memory"   # memory / opengauss
  api_key: ""                # [openai 模式必填]
  base_url: "https://api.openai.com/v1"
  llm_model: "gpt-4o-mini"
  embedding_model: "text-embedding-ada-002"
  temperature: 0.7
  max_tokens: 4096

# ============================================================
# IndexService（异步索引）
# ============================================================
index_service:
  interval: 30               # 轮询间隔（秒）
  workers: 1                 # 工作线程数

# ============================================================
# 默认身份（无 RequestContext 时的回退值）
# ============================================================
defaults:
  account_id: "acct-demo"
  user_id: "u-alice"
  agent_id: "main"
```

### 与 Phase 1 草案的差异说明

| 变更 | 原因 |
|------|------|
| `server` → `ogmem-server` | 用户要求更具体 |
| `llm.llm_model` 保留（与 `embedding_model` 命名一致）| 用户要求 |
| 新增 `index_service` 节 | `scripts/run_index_service.py` 单独的配置项需要归属 |
| 新增 `agfs.mount_prefix` | `MemoryService.__init__` 和 `run_index_service.py` 都用到 |
| 新增 `agfs.config_path` | 需要告诉 entrypoint AGFS 的配置文件在哪 |
| 新增 `agfs.external` | 支持使用外部 AGFS（不启动子进程）的场景 |
| `opengauss` 拆分为独立字段 | 替代原来的 `connection_string` 拼接，更清晰 |

---

## 2. `config.py` 模块设计

### 2.1 文件位置

`src/og_memory/config.py`

### 2.2 数据结构

```python
# 使用嵌套 dataclass，保持简单，不引入框架

from dataclasses import dataclass, field

@dataclass
class OgmemServerConfig:
    host: str = "0.0.0.0"
    port: int = 8090
    workers: int = 2
    log_level: str = "info"

@dataclass
class AgfsConfig:
    base_url: str = "http://127.0.0.1:1833"
    mount_prefix: str = "/local/plugin"
    config_path: str = "docker/agfs-config.yaml"
    data_dir: str = "/data/agfs"
    bin_path: str = ""              # agfs-server 二进制路径，空则自动查找 PATH
    external: bool = False

@dataclass
class OpenGaussConfig:
    host: str = "127.0.0.1"
    port: int = 5432
    dbname: str = "postgres"
    user: str = "aaa"
    password: str = ""
    table_name: str = "vector_index"
    pool_size: int = 5
    dimension: int = 1536

    @property
    def connection_string(self) -> str:
        """生成 libpq 格式连接串，供 OpenGaussVectorIndex 使用。"""
        return (
            f"host={self.host} port={self.port} dbname={self.dbname}"
            f" user={self.user} password={self.password}"
        )

@dataclass
class LlmConfig:
    provider: str = "mock"
    vector_db_type: str = "memory"   # memory / opengauss
    api_key: str = ""
    base_url: str = "https://api.openai.com/v1"
    llm_model: str = "gpt-4o-mini"
    embedding_model: str = "text-embedding-ada-002"
    temperature: float = 0.7
    max_tokens: int = 4096

@dataclass
class IndexServiceConfig:
    interval: int = 30
    workers: int = 1

@dataclass
class DefaultsConfig:
    account_id: str = "acct-demo"
    user_id: str = "u-alice"
    agent_id: str = "main"

@dataclass
class Config:
    ogmem_server: OgmemServerConfig = field(default_factory=OgmemServerConfig)
    agfs: AgfsConfig = field(default_factory=AgfsConfig)
    opengauss: OpenGaussConfig = field(default_factory=OpenGaussConfig)
    llm: LlmConfig = field(default_factory=LlmConfig)
    index_service: IndexServiceConfig = field(default_factory=IndexServiceConfig)
    defaults: DefaultsConfig = field(default_factory=DefaultsConfig)
```

### 2.3 加载接口

```python
import yaml
from pathlib import Path

class ConfigError(Exception):
    """配置加载/校验错误。"""
    pass

def load_config(path: str | Path | None = None) -> Config:
    """从 YAML 文件加载配置。

    path 为 None 时，按以下顺序查找：
      1. 环境变量 CONFIG_PATH
      2. ./config.yaml
      3. 无配置文件 → 使用全部默认值 + 打印 warning

    Raises:
        ConfigError: YAML 语法错误、必填字段缺失
    """
    ...
```

### 2.4 校验规则

| 场景 | 规则 | 错误提示 |
|------|------|---------|
| YAML 语法错误 | `yaml.YAMLError` 捕获 | `"config.yaml 语法错误（第 {line} 行）: {detail}"` |
| 文件不存在且无 CONFIG_PATH | 打 warning，用默认值 | `"未找到 config.yaml，使用默认配置。如需自定义: cp config.example.yaml config.yaml"` |
| `llm.provider` = `"openai"` | `llm.api_key` 非空 | `"llm.provider='openai' 时 llm.api_key 为必填项"` |
| `llm.vector_db_type` = `"opengauss"` | `opengauss.password` 非空 | `"llm.vector_db_type='opengauss' 时 opengauss.password 为必填项"` |
| 未知顶级字段 | 忽略，不报错 | — |

### 2.5 与现有 `ProviderConfig` 的桥接

```python
# 在 ProviderConfig 中新增类方法
@classmethod
def from_config(cls, config: Config) -> "ProviderConfig":
    """从新的 Config 对象构造 ProviderConfig（迁移期桥接）。"""
    llm = config.llm
    og = config.opengauss
    base_url = llm.base_url
    if base_url and not base_url.endswith("/v1"):
        base_url = base_url.rstrip("/") + "/v1"
    return cls(
        provider=llm.provider,
        openai_api_key=llm.api_key or None,
        openai_base_url=base_url,
        openai_embedding_model=llm.embedding_model,
        openai_llm_model=llm.llm_model,
        vector_db_type=llm.vector_db_type,
        opengauss_connection_string=og.connection_string if llm.vector_db_type == "opengauss" else None,
        opengauss_dimension=og.dimension,
        opengauss_table_name=og.table_name,
        opengauss_pool_size=og.pool_size,
    )
```

> 注意：`ProviderConfig` 已有的工厂方法 `from_env()` 在 Phase 3 改造中逐步替换，最终删除。

---

## 3. 文件改造清单

### 3.1 按文件列出具体改造方案

| # | 文件 | 当前方式 | 改造方案 | 优先级 |
|---|------|---------|---------|--------|
| 1 | `src/og_memory/config.py` | 不存在（新建） | 创建 Config 体系：6 个 dataclass + `load_config()` + `ConfigError` | P0 |
| 2 | `src/og_memory/providers/config.py` | `ProviderConfig.from_env()` 读 11 个 `os.environ.get()` | 新增 `from_config(cls, Config)` 类方法；保留 `from_env()` 直到全部调用方迁移完毕 | P0 |
| 3 | `src/og_memory/server/memory_service.py` | `__init__` 读 `AGFS_BASE_URL`/`OG_ACCOUNT_ID`/`OG_USER_ID`；`get_llm()` 和 `get_read_api()` 调 `ProviderConfig.from_env()` | `__init__(self, config: Config)` 替代所有 `os.environ.get()`；`get_llm()` 和 `get_read_api()` 改用 `ProviderConfig.from_config(self._config)` | P0 |
| 4 | `src/og_memory/server/app.py` | `__main__` 读 `OGMEM_HTTP_PORT` | 从 `Config.ogmem_server.port` 获取 | P0 |
| 5 | `scripts/run_index_service.py` | 读 7 个环境变量 + `ProviderConfig.from_env()` | 接收 `Config` 对象（通过 `load_config()` 加载） | P1 |
| 6 | `docker/entrypoint-standalone.sh` | 读 `OGMEM_HTTP_PORT`、`OGMEM_WORKERS` | 用 Python 辅助脚本从 `config.yaml` 提取值 | P1 |
| 7 | `docker/entrypoint.sh` | 读 `OPENAI_API_KEY`、`OPENAI_LLM_MODEL` 注入 openclaw.json | 用 Python 辅助脚本从 `config.yaml` 提取值 | P1 |
| 8 | `pyproject.toml` | 无 `pyyaml` 依赖 | `dependencies` 新增 `pyyaml>=6.0` | P0 |
| 9 | `.gitignore` | 无 `config.yaml` 条目 | 新增 `config.yaml`（保留 `config.example.yaml`） | P0 |
| 10 | `config.example.yaml` | 不存在（新建） | 见第 1 节 | P0 |
| 11 | `.env.example` | 存在，含真实 API Key | Phase 3 最后删除 | P2 |

### 3.2 不需要改造的文件

| 文件 | 原因 |
|------|------|
| `openclaw_context_engine_plugin/index.js` | JS 插件从 `openclaw.json` 读配置（`memoryApiBaseUrl`），不读 `config.yaml`，不涉及本次改造 |
| `openclaw_context_engine_plugin/bridge/memory_api.py` | subprocess 模式桥接，接收参数而非读环境变量 |
| `docker/agfs-config.yaml` | AGFS 自身配置，由 entrypoint 脚本管理 |
| `src/og_memory/providers/llm/openai_llm.py` | 通过构造函数接收 `api_key`/`base_url`/`model`，不直接读环境变量 |
| `src/og_memory/providers/embedder/openai_embedder.py` | 同上 |
| `src/og_memory/providers/vector_index/opengauss_index.py` | 通过构造函数接收 `connection_string`，不直接读环境变量 |

---

## 4. 外部依赖配置注入方案

### 4.1 AGFS

**现状**：`entrypoint-standalone.sh` 和 `entrypoint.sh` 中直接启动 AGFS：
```sh
/opt/agfs/agfs-server -c /opt/agfs/config.yaml &
```

**改造后**：
- AGFS 使用自己的 `agfs-config.yaml`（容器内 `/opt/agfs/config.yaml`），保持不变
- 主 `config.yaml` 中的 `agfs.data_dir` 如与 AGFS 默认不同，entrypoint 启动前需更新 AGFS 配置
- `agfs.external: true` 时，entrypoint 跳过 AGFS 启动

**entrypoint 中获取 AGFS 配置**：通过 Python 辅助脚本（见 4.3）

### 4.2 openGauss

**现状**：`ProviderConfig.from_env()` 读取 `OPENGAUSS_CONNECTION_STRING`（libpq 格式拼接字符串）。
> 注：旧代码中拼写为 `OPENGUASS`（少一个 A），本次迁移一并修正为 `OPENGAUSS`。

**改造后**：
- `config.yaml` 中 `opengauss` 节提供 `host`/`port`/`dbname`/`user`/`password` 独立字段
- `OpenGaussConfig.connection_string` 属性自动拼接为 libpq 格式
- `OpenGaussVectorIndex` 仍通过 `connection_string` 参数接收，不感知配置来源

**不使用 PG* 环境变量**（Phase 1 已确认 psycopg2 支持纯参数传入）。

### 4.3 Shell 脚本中获取配置值

**方案**：创建 `scripts/config_helper.py`，提供 CLI 接口从 `config.yaml` 提取指定值。

```python
#!/usr/bin/env python3
"""从 config.yaml 提取配置值，供 shell 脚本使用。

Usage:
    python3 scripts/config_helper.py ogmem-server.port
    python3 scripts/config_helper.py ogmem-server.workers
    python3 scripts/config_helper.py agfs.data_dir
"""
```

entrypoint 中的用法：
```sh
OGMEM_PORT=$(python3 /opt/ogmem/scripts/config_helper.py ogmem-server.port)
OGMEM_WORKERS=$(python3 /opt/ogmem/scripts/config_helper.py ogmem-server.workers)
```

**依赖**：仅需 `pyyaml`（已在容器中安装）。

---

## 5. 改造顺序

按依赖关系排序，每个步骤对应 1-3 个 commit：

### Step 1: 创建配置基础设施 [P0]

```
feat: add config.example.yaml with full configuration schema
feat: add config.py with yaml loading, validation, and Config dataclasses
chore: add pyyaml dependency to pyproject.toml
chore: add config.yaml to .gitignore
```

**产出物**：`config.example.yaml`、`src/og_memory/config.py`
**验证**：`python -c "from og_memory.config import load_config; c = load_config(); print(c)"`

### Step 2: 桥接 ProviderConfig [P0]

```
feat(providers): add ProviderConfig.from_config() bridge method
```

**产出物**：`providers/config.py` 新增方法
**验证**：`python -c "from og_memory.config import load_config; from og_memory.providers.config import ProviderConfig; c = load_config(); p = ProviderConfig.from_config(c); print(p)"`

### Step 3: 改造 MemoryService [P0]

```
refactor(server): replace os.environ in MemoryService with Config parameter
```

**改动**：
- `MemoryService.__init__` 接收 `config: Config` 参数
- 删除 `__init__` 中的 3 处 `os.environ.get()`
- `get_llm()` 改用 `ProviderConfig.from_config(self._config)`
- `get_read_api()` 改用 `ProviderConfig.from_config(self._config)`

**验证**：`python -c "from og_memory.config import load_config; from og_memory.server.memory_service import MemoryService; c = load_config(); s = MemoryService(config=c); print(s)"`

### Step 4: 改造 app.py 入口 [P0]

```
refactor(server): load config.yaml in app.py entrypoint
```

**改动**：
- `app.py` 中 `__main__` 块加载 Config 并传递给 MemoryService
- `_get_service()` 使用 Config 初始化 MemoryService
- 删除 `os.environ.get("OGMEM_HTTP_PORT")`

**验证**：`python -m og_memory.server.app` 能启动（需 AGFS 运行）

### Step 5: 改造 run_index_service.py [P1]

```
refactor(scripts): replace env vars in run_index_service.py with Config
```

**改动**：
- `main()` 函数开头调用 `load_config()`
- 删除 7 处 `os.environ.get()`
- `ProviderConfig.from_env()` → `ProviderConfig.from_config(config)`

**验证**：`python scripts/run_index_service.py` 能启动（需完整环境）

### Step 6: 改造 Docker entrypoints [P1]

```
feat(scripts): add config_helper.py for shell config extraction
refactor(docker): update entrypoint-standalone.sh to use config.yaml
refactor(docker): update entrypoint.sh to use config.yaml
```

**改动**：
- 新建 `scripts/config_helper.py`
- `entrypoint-standalone.sh` 用 `config_helper.py` 提取 `ogmem-server.port`/`workers`
- `entrypoint.sh` 用 `config_helper.py` 提取 `llm.api_key`/`llm.llm_model`
- Dockerfile COPY 新增 `config.example.yaml` 和 `scripts/config_helper.py`

**验证**：Docker 构建并启动

### Step 7: Dockerfile 更新 [P1]

```
refactor(docker): update Dockerfiles to include config.yaml
```

**改动**：
- `Dockerfile.standalone` COPY `config.example.yaml` 到 `/opt/ogmem/`
- `Dockerfile` 同样处理
- `pip install` 新增 `pyyaml`
- entrypoint 中挂载或内置 config.yaml

### Step 8: 清理旧配置 [P2]

```
chore: remove .env.example
refactor(providers): remove ProviderConfig.from_env() and DEFAULT_CONFIG
```

**改动**：
- 删除 `.env.example`
- 删除 `providers/config.py` 中的 `from_env()` 方法
- 删除 `DEFAULT_CONFIG = ProviderConfig.from_env()` 全局变量
- 删除 `get_embedder()`/`get_llm()`/`get_vector_index()` 中的 env 回退逻辑

**验证**：`grep -rn "os.getenv\|os.environ" src/og_memory/ --include="*.py"` 应返回空

---

## 6. 风险与注意事项

| 风险 | 缓解措施 |
|------|---------|
| `MemoryService` 被多处实例化（app.py、bridge/memory_api.py） | 所有入口统一改为传入 Config |
| Docker 容器内需要 `config.yaml` 存在 | Dockerfile 中 COPY `config.example.yaml`；entrypoint 检测并提示 |
| shell 脚本调用 Python 解析 YAML 有性能开销 | `config_helper.py` 每次启动只调用一次，开销可忽略 |
| `ProviderConfig.from_env()` 被测试代码广泛使用 | 先新增 `from_config()`，Phase 3 逐步替换测试中的调用 |
| `OPENGUASS` 拼写错误（少一个 A） | 本次迁移一并修正为 `OPENGAUSS`；`from_env()` 删除后旧拼写自然消除 |
