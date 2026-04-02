# Phase 0 目录结构重构计划

> 编写时间：2026-04-02
> 状态：待用户确认

---

## 1. 现状分析

### 1.1 当前顶层结构

```
oG-Memory/
├── core/                    # 领域对象 + 接口 (无外部依赖)
├── fs/agfs_adapter/         # ContextFS 的 AGFS 实现 (依赖 core)
├── extraction/              # 候选抽取 (依赖 core)
├── providers/               # 外部能力适配 (依赖 core)
│   ├── config.py, llm/, embedder/, vector_index/, relation_store/
├── commit/                  # 写入编排 (依赖 core, fs)
├── index/                   # 索引同步 (依赖 core, fs)
├── retrieval/               # 检索链路 (依赖 core, fs)
├── service/                 # 服务层 API (依赖所有包)
├── server/                  # HTTP Flask 服务 (依赖所有包)
├── og_memory/               # 仅有 __pycache__，内容为空壳
├── openclaw_context_engine_plugin/  # JS 插件（独立包，有 package.json）
├── openclaw-plugin/                  # TS 插件（独立包，有 package.json）
├── tests/                   # 测试代码
├── scripts/                 # 工具脚本
├── docker/                  # Docker 配置
├── docs/                    # 文档
├── architecture/            # 架构文档
├── agfs/                    # AGFS Go 源码（独立仓库克隆）
├── examples/                # 示例代码
├── pyproject.toml           # 打包配置
└── ...其他文件
```

### 1.2 当前 import 模式

所有功能包作为**顶层包**直接引用：
```python
from core.models import RequestContext
from fs.agfs_adapter import AGFSContextFS
from providers.config import ProviderConfig
from server.memory_service import MemoryService
```

`server/app.py` 还使用了 `sys.path.insert(0, ...)` 的 hack。

### 1.3 当前打包配置

`pyproject.toml` 中 packages 列表为 8 个顶层包：`core`, `fs`, `extraction`, `commit`, `index`, `retrieval`, `providers`, `service`。
注意 `server` **不在** packages 列表中。

---

## 2. 目标结构

### 2.1 目标目录树

```
oG-Memory/
├── src/
│   └── og_memory/
│       ├── __init__.py
│       ├── core/                    # 领域对象 + 接口 (无外部依赖)
│       │   ├── __init__.py
│       │   ├── models.py
│       │   ├── interfaces.py
│       │   ├── enums.py
│       │   ├── errors.py
│       │   ├── validation.py
│       │   └── logging_config.py
│       ├── fs/                      # 文件系统适配 (依赖 core)
│       │   ├── __init__.py
│       │   └── agfs_adapter/
│       │       ├── __init__.py
│       │       └── agfs_context_fs.py
│       ├── extraction/              # 候选抽取 (依赖 core)
│       │   ├── __init__.py
│       │   └── tools.py
│       ├── commit/                  # 写入编排 (依赖 core, fs)
│       │   ├── __init__.py
│       │   ├── context_writer.py
│       │   ├── candidate_pipeline.py
│       │   ├── merge_policies.py
│       │   ├── policy_router.py
│       │   ├── archive_builder.py
│       │   └── outbox_store.py
│       ├── index/                   # 索引同步 (依赖 core, fs)
│       │   ├── __init__.py
│       │   ├── outbox_worker.py
│       │   ├── index_record_builder.py
│       │   ├── directory_event_handler.py
│       │   ├── directory_summarizer.py
│       │   ├── repair_job.py
│       │   └── scheduler.py
│       ├── retrieval/               # 检索链路 (依赖 core, fs)
│       │   ├── __init__.py
│       │   ├── pipeline.py
│       │   ├── query_planner.py
│       │   ├── seed_retriever.py
│       │   ├── hierarchical_searcher.py
│       │   ├── assembly_service.py
│       │   ├── memory_read_service.py
│       │   ├── hotness.py
│       │   ├── trace.py
│       │   └── l0_retriever.py
│       ├── providers/               # 外部能力适配 (依赖 core)
│       │   ├── __init__.py
│       │   ├── config.py
│       │   ├── llm/
│       │   ├── embedder/
│       │   ├── vector_index/
│       │   └── relation_store/
│       ├── service/                 # 服务层 (依赖所有包)
│       │   ├── __init__.py
│       │   ├── api.py
│       │   ├── index_service.py
│       │   └── memory_fs.py
│       └── server/                  # HTTP API (依赖所有包)
│           ├── __init__.py
│           ├── app.py
│           └── memory_service.py
│
├── openclaw_context_engine_plugin/  # JS 插件 — 不动（独立包）
├── openclaw-plugin/                  # TS 插件 — 不动（独立包）
├── tests/                           # 测试 — 位置不动，import 更新
│   ├── conftest.py
│   ├── contract/
│   ├── integration/
│   ├── unit/
│   └── fixtures/
├── scripts/                         # 工具脚本 — 不动，import 更新
├── examples/                        # 示例 — 不动，import 更新
├── docker/                          # Docker 配置 — 不动
├── docs/                            # 文档 — 不动
├── architecture/                    # 架构文档 — 不动
├── agfs/                            # AGFS Go 源码 — 不动
├── pyproject.toml                   # 更新 packages 路径
└── README.md
```

### 2.2 不动的目录/文件

| 目录/文件 | 原因 |
|-----------|------|
| `openclaw_context_engine_plugin/` | 独立 JS 包，有自己的 package.json |
| `openclaw-plugin/` | 独立 TS 包，有自己的 package.json |
| `tests/` | pytest 的 testpaths 配置，保持顶层即可 |
| `scripts/` | 工具脚本，非打包对象 |
| `examples/` | 示例代码，非打包对象 |
| `docker/` | 部署配置，非打包对象 |
| `docs/`, `architecture/` | 文档，非打包对象 |
| `agfs/` | 独立 Go 仓库克隆 |

### 2.3 import 变化

| 之前 | 之后 |
|------|------|
| `from core.models import RequestContext` | `from og_memory.core.models import RequestContext` |
| `from fs.agfs_adapter import AGFSContextFS` | `from og_memory.fs.agfs_adapter import AGFSContextFS` |
| `from providers.config import ProviderConfig` | `from og_memory.providers.config import ProviderConfig` |
| `from server.memory_service import MemoryService` | `from og_memory.server.memory_service import MemoryService` |

包内相对引用（如 `core/__init__.py` 中的 `from core.logging_config import ...`）也统一改为 `from og_memory.core.logging_config import ...`。

---

## 3. 执行步骤（每步一个 commit）

### Step 0: 清理垃圾文件

- 删除 `og_memory/` 空壳目录（仅含 `__pycache__`）
- 删除根目录下的 `coverage.json`（应被 gitignore）
- 确认 `config/` 目录为空，如是则删除

**Commit**: `chore: clean up empty/obsolete directories`

### Step 1: 创建目标目录骨架

- 创建 `src/og_memory/` 及所有子目录
- 创建 `src/og_memory/__init__.py`

**Commit**: `refactor: create src/og_memory/ directory skeleton`

### Step 2: 迁移 core 包（无依赖）

- `git mv core/ src/og_memory/core/`
- 更新 `core/__init__.py` 及内部所有 `from core.` → `from og_memory.core.`
- 验证：`python -c "from og_memory.core.models import RequestContext"`

**Commit**: `refactor: move core into src/og_memory/core/`

### Step 3: 迁移 fs 包（依赖 core）

- `git mv fs/ src/og_memory/fs/`
- 更新内部 imports: `from core.` → `from og_memory.core.`, `from fs.` → `from og_memory.fs.`
- 验证：`python -c "from og_memory.fs.agfs_adapter import AGFSContextFS"`

**Commit**: `refactor: move fs into src/og_memory/fs/`

### Step 4: 迁移 extraction + providers（依赖 core）

- `git mv extraction/ src/og_memory/extraction/`
- `git mv providers/ src/og_memory/providers/`
- 更新所有内部 imports
- 验证

**Commit**: `refactor: move extraction and providers into src/og_memory/`

### Step 5: 迁移 commit + index + retrieval（依赖 core, fs）

- `git mv commit/ src/og_memory/commit/`
- `git mv index/ src/og_memory/index/`
- `git mv retrieval/ src/og_memory/retrieval/`
- 更新所有内部 imports
- 验证

**Commit**: `refactor: move commit, index, retrieval into src/og_memory/`

### Step 6: 迁移 service + server（依赖所有包）

- `git mv service/ src/og_memory/service/`
- `git mv server/ src/og_memory/server/`
- 更新所有内部 imports
- 移除 `server/app.py` 中的 `sys.path.insert(0, ...)` hack
- 验证

**Commit**: `refactor: move service and server into src/og_memory/`

### Step 7: 更新 tests/ 的 imports

- 更新 `tests/conftest.py` 及所有 `tests/` 下的 import 语句
- `from core.` → `from og_memory.core.`, etc.
- 运行 `pytest tests/` 验证

**Commit**: `refactor: update test imports for new package structure`

### Step 8: 更新 scripts/ 和 examples/ 的 imports

- 更新 `scripts/` 下所有 `.py` 文件的 imports
- 更新 `examples/` 下所有 `.py` 文件的 imports

**Commit**: `refactor: update scripts and examples imports`

### Step 9: 更新 pyproject.toml

```toml
[tool.hatch.build.targets.wheel]
packages = ["src/og_memory"]

[tool.hatch.build.targets.sdist]
include = [
    "/README.md",
    "/LICENSE",
    "/src/og_memory",
]

[tool.coverage.run]
source = ["og_memory"]

[tool.ruff.lint.isort]
known-first-party = ["og_memory"]
```

**Commit**: `refactor: update pyproject.toml for src layout`

### Step 10: 更新 .gitignore

- 添加 `coverage.json`（如果还没有）

**Commit**: `chore: update .gitignore`

### Step 11: 全量验证

- `python -c "import og_memory"`
- `python -c "from og_memory.core.models import RequestContext"`
- `python -c "from og_memory.server.app import app"`
- `grep -rn "^from core\.\|^import core\|^from fs\.\|^from providers\.\|^from service\.\|^from server\.\|^from commit\.\|^from index\.\|^from retrieval\.\|^from extraction\." --include="*.py" src/` → 应为空（无旧 import 残留）
- `pip install -e . && python -c "import og_memory"`
- `pytest tests/` — 所有测试通过

**Commit**: `milestone(phase-0): directory restructuring complete`

---

## 4. 风险与注意事项

1. **import 替换必须精确** — 使用 `from core.` → `from og_memory.core.` 的前缀替换，避免误改字符串或注释中的 `core`
2. **`__init__.py` 中的 re-export** — 部分包的 `__init__.py` 做了 re-export（如 `core/__init__.py` 导出了 `setup_logging` 等），迁移后需要更新这些引用
3. **循环 import** — 当前 `server/memory_service.py` 直接 import 大量模块，迁移后路径变长但不应引入新循环
4. **tests 中的 conftest.py** — 是测试基础设施，import 更新要特别注意
5. **不做的事** — 不改业务逻辑，不改配置管理方式，不引入新功能

## 5. 预计产出

- 10-12 个小步 git commits
- 所有 Python 功能代码统一收敛在 `src/og_memory/` 下
- `pip install -e .` 正常工作
- 所有现有测试通过
