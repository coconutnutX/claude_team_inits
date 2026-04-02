# oG-Memory 目录重构 + 配置管理重构

## 背景

oG-Memory 项目当前存在两个层面的混乱：

1. **目录结构混乱** — 功能代码散落在多个顶层目录，没有统一的 `src/` 组织
2. **配置管理混乱** — 环境变量散布在多个模块中、部分值硬编码在源码里、`.env` / `.env.example` / 代码中的 key 名不一致

目标是先整理目录结构，再将所有可配置项统一收敛到一个 `config.yaml` 文件，代码中**不再直接依赖任何环境变量**。

### 当前状态

- 功能代码可以直接运行，但目录组织和配置管理需要在早期就理清，避免后续返工

### 核心原则

1. **用户只接触 `config.yaml`** — 提供 `config.example.yaml`，用户复制为 `config.yaml` 后按需修改
2. **代码中零 `os.getenv()`** — 所有配置统一从 `Config` 对象读取
3. **外部依赖的环境变量由代码注入** — agfs、opengauss 等如需环境变量，在启动子进程时从 config 中取值注入，对用户透明
4. **不改变现有功能行为** — 本次重构仅改目录结构和配置管理方式，不改业务逻辑
5. **小步提交** — 每完成一个最小可验证的改动就提交一次，commit message 清晰描述改动内容

### 涉及的仓库/代码范围

- og-memory 核心代码（Python）
- agfs 相关启动/配置逻辑
- opengauss 连接层
- openclaw context engine plugin（如涉及配置传递）

### Git 提交规范

**全程遵循小步多次提交原则**，严禁攒一大堆改动后一次性提交。参考粒度：

- ✅ `refactor: move og_memory/ into src/og_memory/`
- ✅ `refactor: update imports after directory move`
- ✅ `refactor: update pyproject.toml packages path`
- ❌ `refactor: restructure entire project` （太大，无法 review，出问题难以定位）

每次提交后确保项目仍可正常运行（至少 import 不报错）。如果一个 Task 涉及多个文件的改动，按"挪一组相关文件 → 修 import → 提交 → 验证"的节奏推进。

---

## Phase 0：目录结构重构

**目标**：将项目目录整理成规范结构，所有功能代码收敛到 `src/` 下。本阶段**只挪文件、修 import、改打包配置**，不改任何业务逻辑，不改配置管理方式。

### Task 0.1：盘点现有目录结构

记录当前项目的完整目录树：

```bash
find . -type f -not -path './.git/*' -not -path './node_modules/*' -not -path './__pycache__/*' -not -path './*.pyc' | sort
```

产出一份当前目录结构文档，标注每个目录/文件的职责。识别出：
- 哪些是功能代码（应该进 `src/`）
- 哪些是测试代码（应该进 `tests/`）
- 哪些是配置/文档/脚本（留在项目根目录）
- 哪些是历史遗留的垃圾文件（可以删除）

### Task 0.2：设计目标目录结构

基于 Task 0.1 的盘点，设计目标结构。参考模板：

```
og-memory/
├── src/
│   └── og_memory/
│       ├── __init__.py
│       ├── config.py            # Phase 1-3 再创建
│       ├── core/                # 核心业务逻辑
│       │   ├── store.py
│       │   ├── retriever.py
│       │   └── ...
│       ├── agfs/                # agfs 管理相关
│       │   └── manager.py
│       ├── db/                  # opengauss 连接层
│       │   └── ...
│       └── ...
├── tests/
│   └── ...
├── config.example.yaml          # Phase 1-3 再创建
├── pyproject.toml
├── README.md
└── ...
```

> **注意**：目标结构必须根据实际代码内容来定，上面只是参考。Task 0.1 盘点后再确定具体的子目录划分。

### Task 0.3：执行目录迁移

按照目标结构逐步迁移文件。**每挪一组相关文件就提交一次**：

1. 创建 `src/og_memory/` 目录结构
2. 逐模块迁移文件（每移一个模块 → 修 import → 验证 → 提交）
3. 迁移测试文件到 `tests/`
4. 清理空目录和垃圾文件

提交粒度示例：
```
refactor: create src/og_memory/ directory structure
refactor: move core modules into src/og_memory/core/
refactor: update imports for core module move
refactor: move agfs manager into src/og_memory/agfs/
refactor: move tests into tests/ directory
chore: remove empty legacy directories
```

### Task 0.4：更新打包配置

修改 `pyproject.toml`（或 `setup.py`），确保：

1. `packages` 路径指向新的 `src/og_memory/`
2. 如有 CLI entry_points，路径同步更新
3. `pip install -e .` 能正常工作

### Task 0.5：验证

1. `python -c "import og_memory"` 成功
2. 如有现有测试，全部通过
3. 如有现有启动方式（如 `python app.py`），仍然能正常启动
4. `grep -rn "from og_memory\|import og_memory" --include="*.py" .` — 确认所有 import 路径正确

### Phase 0 产出物

1. 整理后的目录结构
2. 更新后的 `pyproject.toml`
3. 一系列小步 git commits

---

## Phase 1：调查与盘点

**目标**：产出一份完整的"配置项清单"，覆盖所有环境变量、硬编码值、外部依赖的配置需求。

> **前提**：Phase 0 已完成，代码已收敛到 `src/` 下，以下扫描基于新目录结构。

### Task 1.1：扫描所有环境变量引用

扫描整个项目代码库，找出所有环境变量的使用点：

```bash
grep -rn "os\.getenv\|os\.environ\|getenv\|dotenv" --include="*.py" .
grep -rn "process\.env" --include="*.js" --include="*.ts" .
```

对每个找到的环境变量，记录：

| 字段 | 说明 |
|------|------|
| 变量名 | 如 `OPENAI_API_KEY` |
| 所在文件:行号 | 如 `og_memory/store.py:15` |
| 默认值 | 如有 |
| 用途 | 简要说明（如"OpenGauss 连接密码"） |
| 是否必填 | 是/否 |
| 已知问题 | 如拼写错误、重复定义等 |

### Task 1.2：扫描硬编码的可配置项

寻找应该可配置但被硬编码的值。重点关注：

**网络相关**：
```bash
grep -rn "127\.0\.0\.1\|localhost\|0\.0\.0\.0" --include="*.py" .
grep -rn ":[0-9]\{4,5\}" --include="*.py" .  # 端口号
```

**路径相关**：
```bash
grep -rn "\.og-memory\|/tmp/\|/var/\|~/\." --include="*.py" .
grep -rn "\.json\|\.yaml\|\.conf\|\.db\|\.log" --include="*.py" .  # 配置/数据文件路径
```

**其他常见硬编码**：
- API base URLs（如 OpenAI 端点）
- 模型名称（如 `gpt-4o`）
- 超时时间、重试次数
- 日志级别
- 批处理大小、并发数

对每个找到的硬编码项，记录同 Task 1.1 的表格格式，额外加一列"当前硬编码值"。

### Task 1.3：调查 agfs 的配置需求

agfs 作为 Go 编写的外部依赖，有自己的配置体系。需要搞清楚：

1. **agfs-server 支持的所有命令行参数**
   ```bash
   agfs-server --help
   ```
2. **agfs 是否支持配置文件**（如 agfs-config.yaml）
3. **agfs 自身依赖的环境变量**
4. **agfs 的存储路径在哪里、是否可配置**
5. **agfs 的端口号如何指定**
6. **当前项目中如何启动和管理 agfs 进程**

产出一份 agfs 配置项清单。

### Task 1.4：调查 opengauss 客户端的配置传递方式

确认 opengauss 的 Python 客户端库（psycopg2 或其他）的配置传递方式：

1. 是否支持纯参数传入（不依赖环境变量）
2. 是否有 `PGHOST`、`PGPORT`、`PGDATABASE` 等隐式环境变量依赖
3. 连接字符串的完整格式

### Task 1.5：调查 .env.example 与代码的一致性

对比现有 `.env.example`（如果存在）与 Task 1.1 扫描结果：
- 哪些变量在 `.env.example` 中有、但代码中未使用
- 哪些变量在代码中使用、但 `.env.example` 中缺失
- 变量名拼写是否一致（已知问题：`OPENGUASS` vs `OPENGAUSS`）

### Phase 1 产出物

一份 `phase1-config-inventory.md` 文档，包含：

1. **环境变量完整清单**（表格形式）
2. **硬编码可配置项清单**（表格形式）
3. **agfs 配置项清单**
4. **opengauss 客户端配置方式说明**
5. **一致性问题列表**
6. **建议的 config.yaml 结构草案**（基于以上发现）

---

## Phase 2：设计与计划

**目标**：产出详细的改造计划文档，明确每个文件要怎么改、配置如何流转。

### Task 2.1：设计 config.yaml 结构

基于 Phase 1 的清单，设计最终的 `config.example.yaml`。设计原则：

- **分组清晰**：按 `opengauss` / `agfs` / `llm` / `server` 等分组
- **合理默认值**：非敏感配置项提供合理默认值
- **注释充分**：每个字段有中英文注释，说明用途和格式
- **扁平优先**：避免过深嵌套，两层为宜

### Task 2.2：设计配置加载层

设计 `config.py` 模块的接口：

- `Config.load(path)` — 从 yaml 文件加载
- 各子配置的访问方式（如 `config.opengauss.host`）
- 必填字段的校验逻辑
- 配置缺失时的错误提示（用户友好）
- **不设计环境变量回退机制** — 只从 config.yaml 读取

### Task 2.3：规划每个文件的改造方案

对 Phase 1 中识别出的每个使用环境变量/硬编码的文件，规划改造方式：

| 文件 | 当前方式 | 改造方案 | 优先级 |
|------|---------|---------|--------|
| `og_memory/store.py` | `os.getenv("OPENGUASS_CONNECTION_STRING")` | 改为接收 `config.opengauss` 参数 | P0 |
| `og_memory/retriever.py` | 硬编码 `127.0.0.1:1833` | 改为 `config.agfs.port` | P0 |
| ... | ... | ... | ... |

### Task 2.4：规划外部依赖的配置注入方案

明确 agfs 和 opengauss 的配置传递策略：

**agfs**：
- 优先使用命令行参数（如 `--port`）
- 如必须用环境变量，通过 `subprocess.Popen(env=...)` 注入
- 列出具体的映射关系：`config.agfs.port` → `--port` 或 `AGFS_PORT`

**opengauss**：
- 确认使用连接参数直接传入（不依赖 PG* 环境变量）
- 连接参数从 `config.opengauss` 获取

### Phase 2 产出物

一份 `phase2-migration-plan.md` 文档，包含：

1. **最终 `config.example.yaml`**（完整内容）
2. **`config.py` 模块设计**（接口定义、校验规则）
3. **文件改造清单**（每个文件的具体改造方案）
4. **外部依赖配置注入方案**
5. **改造顺序**（依赖关系和优先级）

---

## Phase 3：编码实现

**目标**：按照 Phase 2 的计划执行代码改造。

### Task 3.1：创建配置基础设施

1. 创建 `config.example.yaml`
2. 实现 `config.py` 模块（加载、校验、访问）
3. 添加 `pyyaml` 到项目依赖
4. 在 `.gitignore` 中添加 `config.yaml`

### Task 3.2：改造核心模块

按 Phase 2 中的改造清单，逐文件修改：

1. 消除所有 `os.getenv()` / `os.environ` 调用
2. 改为从 `Config` 对象获取配置
3. 消除硬编码值，替换为配置引用
4. 确保 `Config` 对象在程序入口点加载，并通过参数传递到各模块

### Task 3.3：改造外部依赖启动逻辑

1. 改造 agfs 启动代码，从 config 中取参数
2. 改造 opengauss 连接代码，从 config 中取连接参数
3. 如有子进程需要环境变量，在启动时注入

### Task 3.4：改造程序入口

> **注意**：当前项目在主线重新开发中，尚未实现 CLI（没有 `og-memory serve` 等命令）。本 Task 针对现有的程序入口点（如 `app.py`、`server.py`、或 `__main__.py`），而非尚不存在的 CLI。

1. 在程序入口点加载 `config.yaml`，创建 `Config` 对象
2. 将 `Config` 对象传递给各模块（而非让各模块自行读取环境变量）
3. 支持通过参数或环境变量指定配置文件路径（如 `CONFIG_PATH=./config.yaml`，这是唯一允许的环境变量）
4. 如果后续实现 CLI，在 CLI 层调用相同的配置加载逻辑即可

### Task 3.5：清理旧配置机制

1. 删除 `.env.example`（如存在）
2. 删除 `python-dotenv` 相关代码
3. 删除旧的环境变量文档/注释中对环境变量的引用
4. 更新 README 中的配置说明

### 编码规范

- **小步多次提交** — 每完成一个最小可验证的改动就 commit，不要攒改动。一个 Task 可能对应 3-5 个 commits
- 每次 commit 后确保代码能通过语法检查（至少 `python -c "import og_memory"` 不报错）
- 修改后保持原有导入结构，不做不必要的重构
- 对于不确定的改动（如某个环境变量的用途不明），标记 `# TODO: verify` 注释

提交粒度示例：
```
feat: add config.py with yaml loading and validation
refactor(store): replace os.getenv with Config parameter
refactor(retriever): replace hardcoded agfs address with config
refactor(agfs): inject config into subprocess env
chore: remove .env.example and dotenv dependency
```

---

## Phase 4：测试与调试

**目标**：验证改造后的系统功能正常，配置管理行为符合预期。

> **注意**：当前项目没有 CLI，测试以直接运行 Python 脚本或模块的方式进行。

### Task 4.1：配置加载测试

验证以下场景：

1. ✅ 正常：`config.yaml` 存在且完整 → 正常启动
2. ✅ 缺失：`config.yaml` 不存在 → 清晰报错，提示 `cp config.example.yaml config.yaml`
3. ✅ 不完整：必填字段缺失 → 报错指出具体缺失的字段
4. ✅ 格式错误：yaml 语法错误 → 报错指出行号
5. ✅ 自定义路径：`CONFIG_PATH=/path/to/config.yaml` → 正确加载

### Task 4.2：功能回归测试

确保所有原有功能正常工作：

1. **opengauss 连接**：能正常连接、读写数据
2. **agfs 启动与停止**：子进程正常管理
3. **API 端点**：health check、assemble、after_turn 等现有端点正常响应
4. **LLM 调用**：embedding 和 completion 正常
5. **IndexService**：异步索引流程正常（如已实现）
6. **进程退出**：agfs 子进程正确清理

### Task 4.3：边界情况测试

1. config.yaml 中有多余的未知字段 → 应忽略而不报错
2. 端口号被占用 → 清晰报错
3. opengauss 密码含特殊字符 → 连接正常
4. agfs external_url 模式 → 不启动子进程，直接使用外部地址
5. 日志级别可通过 config 正确切换

### Task 4.4：文档验证

按照更新后的 README 从零开始走一遍完整流程：

```bash
git clone ...
cd og-memory
pip install -e .
cp config.example.yaml config.yaml
# 编辑 config.yaml，填入必要的密码和 API key
python -m og_memory  # 或现有的启动方式
```

确保新用户能顺畅完成部署。

### Phase 4 产出物

一份 `phase4-test-report.md`，包含：

1. 每个测试用例的执行结果
2. 发现的 bug 及修复记录
3. 最终验证通过的确认

---

## 附录 A：config.yaml 结构参考

```yaml
# config.example.yaml
# 复制为 config.yaml 并按需修改
# cp config.example.yaml config.yaml

# ============================================================
# openGauss 数据库连接
# ============================================================
opengauss:
  host: 127.0.0.1
  port: 5432
  database: ogmem
  user: ogmem
  password: ""          # [必填] 数据库密码

# ============================================================
# AGFS 文件存储服务
# ============================================================
agfs:
  port: 1833            # agfs-server 监听端口
  data_dir: ""          # 数据存储目录，空则使用默认
  bin_path: ""          # agfs-server 二进制路径，空则自动查找
  # external_url: ""    # 取消注释则使用外部 agfs，跳过本地进程管理

# ============================================================
# LLM / Embedding 服务
# ============================================================
llm:
  api_key: ""           # [必填] OpenAI 或兼容 API 的 key
  base_url: ""          # 可选，用于第三方兼容 API
  model: gpt-4o         # 默认模型
  embedding_model: text-embedding-3-small

# ============================================================
# oG-Memory 服务本身
# ============================================================
server:
  host: 0.0.0.0
  port: 8090
  log_level: info       # debug / info / warning / error
```

## 附录 B：配置流转架构

```
┌─────────────────────┐
│   config.yaml       │  ← 用户只接触这个文件
└────────┬────────────┘
         │ yaml.safe_load()
         ▼
┌─────────────────────┐
│   Config 对象        │  ← 代码中的唯一配置来源
│   config.opengauss   │
│   config.agfs        │
│   config.llm         │
│   config.server      │
└────┬───┬───┬────────┘
     │   │   │
     │   │   └──→ og-memory 自身模块：直接读属性
     │   │
     │   └──────→ opengauss 客户端：连接参数传入
     │              psycopg2.connect(host=..., port=...)
     │
     └──────────→ agfs 子进程：命令行参数 / env 注入
                    Popen(["agfs-server", "--port", ...], env={...})
```
