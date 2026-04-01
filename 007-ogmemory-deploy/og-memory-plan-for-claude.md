# oG-Memory 重构计划：收敛为 pip 可安装包

> **交付目标**：用户通过 `pip install og-memory` 安装后，执行 `og-memory serve` 即可启动完整服务（API + AGFS），无需手动运行 Docker 或部署脚本。
>
> **⚠️ 核心约束（贯穿全程，违反任何一条都需要停下来确认）**：
> 1. **不调整现有目录结构**——不搬家、不重命名、不重新归类现有文件
> 2. **尽量减少代码修改**——业务代码不改，仅新增部署封装层
> 3. **保留环境变量作为配置方式**——不引入 config.toml，现有 `os.getenv()` 调用全部保持原样
> 4. **发现问题只记录不修改**——迁移过程中发现的业务 bug、硬编码、命名不一致等问题，记录到 issues 清单，不在本次修复

---

## 一、现状与目标对比

```
现状：
  1. docker run openGauss                    ← 手动
  2. 安装/启动 AGFS                           ← 手动，二进制可能缺失
  3. docker compose 或手动构建启动 oG-Memory   ← 手动
  4. 配置 OpenClaw 插件                       ← 手动，且重装需先删旧目录
  = 4 步手动操作，多个独立进程

目标：
  pip install og-memory                      ← 一次
  og-memory serve                            ← 一个命令启动 API + AGFS
  og-memory plugin install                   ← 一个命令安装 OpenClaw 插件
  openGauss 仍为外部依赖，用户自行启动（Docker 或已有实例）
```

---

## 二、要做的事情（仅新增，不改现有代码）

### 2.1 在项目根目录新增 pyproject.toml

作用：将现有项目打包为 pip 可安装的 Python 包，注册 `og-memory` CLI 命令。

要点：
- `[project.scripts]` 中注册 `og-memory = "og_memory.cli:cli"`（或按现有入口模块调整路径）
- `dependencies` 中列出现有代码已经使用的全部依赖 + 新增的 `click`、`python-dotenv`
- `packages` 指向现有源码目录，**不移动文件**

> **⚠️ 注意**：需要先梳理现有代码的 import 结构和模块路径。pyproject.toml 的 `packages` 配置必须与现有目录结构匹配，不能为了适配打包而移动文件。如果现有代码不在标准的 `src/` 布局下，使用 `[tool.hatch.build.targets.wheel] packages = ["."]` 或 setuptools 的 `package_dir` 来适配。

> **⚠️ 注意**：梳理现有的全部 Python 依赖（requirements.txt 或 setup.py 中的），完整搬入 `dependencies`，不能遗漏。

### 2.2 新增 CLI 入口模块

新建一个文件（如 `og_memory/cli.py` 或项目根目录下合适位置），使用 `click` 实现以下命令：

**`og-memory serve`**：
1. 使用 `python-dotenv` 加载 `.env` 文件（当前目录 → `~/.og-memory/.env`）
2. 检查必要的环境变量是否存在（openGauss 连接串、API Key），缺失则报错并提示
3. 检查 AGFS 是否可用（见 2.3）
4. 启动 AGFS 子进程，等待就绪（轮询健康检查）
5. 检查 openGauss 连通性，失败则报错退出
6. 启动现有的 API server（调用现有的启动逻辑，不重写）

> **⚠️ 注意**：第 6 步必须复用现有的 server 启动逻辑（比如现有的 `app.py`、`main.py` 或 uvicorn 启动方式），**不要重新实现 FastAPI app 或重新注册路由**。CLI 只是一个启动入口，调用现有代码。

> **⚠️ 注意**：`python-dotenv` 的 `load_dotenv()` 不会覆盖已存在的环境变量，所以不会影响用户直接 `export` 设置的变量。这保证了与现有部署方式的完全兼容。

**`og-memory setup`**：
1. 交互式引导用户填写：openGauss 连接信息、LLM API Key 等
2. 输出到 `.env` 文件（**不是** config.toml）
3. 可选：初始化数据库表

**`og-memory check`**：
1. 检查 AGFS 是否已安装（`which agfs-server`）
2. 测试 openGauss 连接
3. 检查关键环境变量是否设置

**`og-memory db init`**：执行建表 SQL（幂等）

**`og-memory stop`**：停止服务

**`og-memory plugin install` / `og-memory plugin uninstall`**：管理 OpenClaw 插件（见 2.5）

> **⚠️ 注意**：`og-memory serve` 需要处理 SIGINT/SIGTERM 信号，退出时先停 uvicorn 再停 AGFS 子进程。用 `atexit` 注册清理回调作为兜底。

### 2.3 AGFS 处理方案

#### 当前阶段：假定已安装，检测并启动

**默认假定开发者已经通过官方方式安装了 AGFS**。og-memory 只负责：

1. **检测**：启动时检查 `agfs-server` 是否在 PATH 中可用（优先检查 `~/.local/bin/agfs-server`，这是 AGFS 官方 install.sh 的默认安装位置）
2. **启动**：作为子进程启动 `agfs-server`，传入配置文件路径
3. **就绪等待**：轮询健康检查端点，超时 10s 报错
4. **停止**：og-memory 退出时 SIGTERM → 等 5s → SIGKILL
5. **未安装时的提示**：

```
错误：未找到 agfs-server。请先安装 AGFS：
  curl -fsSL https://raw.githubusercontent.com/c4pt0r/agfs/master/install.sh | sh
```

**外部 AGFS 支持**：如果环境变量 `AGFS_EXTERNAL_URL` 已设置，跳过检测和进程管理，直接使用该地址。适用于 AGFS 已在别处运行或通过 Docker 部署的场景。

#### 未来阶段（TODO，本次不实现）

```python
# TODO: 自动下载 AGFS 二进制
# AGFS 仓库 https://github.com/c4pt0r/agfs 已具备自动下载条件：
# - Go 编写，GitHub Actions 每日构建 nightly release
# - 已有官方 install.sh 支持多平台
# - 已有多平台预编译二进制：linux-amd64/arm64, darwin-amd64/arm64
# - nightly release 发布在 GitHub Releases nightly tag 下
# - 下载 URL 格式：https://github.com/c4pt0r/agfs/releases/download/nightly/agfs-{os}-{arch}-{date}.tar.gz
# - 安装后二进制位于 ~/.local/bin/agfs-server
#
# 实现思路：
#   方案 A：直接 subprocess 调用官方 install.sh
#   方案 B：用 Python 复刻下载逻辑（平台检测 → 下载 tar.gz → 解压）
```

> **⚠️ 注意**：AGFS 的 agfs-server 是 Go 编写的二进制。现有项目中 AGFS 可能监听端口 1833（从现有部署文档看），但 AGFS 官方默认端口是 8080。需要确认现有代码中使用的端口，以及通过什么方式配置 AGFS 的监听端口（命令行参数？配置文件 `agfs-config.yaml`？）。

> **⚠️ 注意**：现有代码中访问 AGFS 的地址可能是硬编码的 `http://127.0.0.1:1833`。如果是，本次不改，但需要记录到 issues。

### 2.4 新增 .env.example

在项目根目录提供完整的环境变量模板。

> **⚠️ 注意**：变量 key 名称必须与现有代码中的 `os.getenv()` 调用完全一致。需要先全局搜索现有代码中所有 `os.getenv` / `os.environ` 调用，整理出完整清单。不要自己发明新的变量名。

> **⚠️ 注意**：目前已知的变量包括（从现有部署文档中提取）：
> - `OPENAI_API_KEY`
> - `OPENGUASS_CONNECTION_STRING`（注意拼写，现有代码中可能就是 OPENGUASS 不是 OPENGAUSS，需 grep 确认，如确认是这样则保留原样并记录到 issues）
> - `OPENAI_BASE_URL`
> - `OPENAI_EMBEDDING_MODEL`
> - `OPENAI_LLM_MODEL`
> - `OGMEM_PORT`
> 以上可能不完整，必须从源码中确认完整列表。

### 2.5 openclaw_context_engine_plugin 管理

#### 2.5.1 现状

从实际运行信息确认了以下事实：

- **插件是 JavaScript/Node.js 编写的**（index.js），不是 Python
- **插件名称**：`og-memory-context-engine`
- **安装位置**：`~/.openclaw/extensions/og-memory-context-engine/`
- **安装命令**：`openclaw plugins install openclaw_context_engine_plugin`
- **配置**：插件启动时自动读取 og-memory 地址（`mode=http url=http://127.0.0.1:8090`）
- **痛点**：重装时必须先手动删除旧目录 `~/.openclaw/extensions/og-memory-context-engine/`，否则报 `plugin already exists`
- **安全警告**：OpenClaw 会检测到 shell 命令执行和环境变量+网络发送模式，报出 WARNING（功能不受影响但体验不好）
- **hook pack 问题**：`package.json` 缺少 `openclaw.hooks` 字段，报 `not a valid hook pack` 错误

#### 2.5.2 方案：在 og-memory CLI 中封装插件管理

在 `og-memory` CLI 中新增 `plugin` 子命令组，封装 OpenClaw 插件的安装/卸载/更新流程：

**`og-memory plugin install`**：
1. 检查 `~/.openclaw/extensions/og-memory-context-engine/` 是否已存在
2. 如果已存在 → 自动删除旧目录（解决"必须先手动删"的痛点）
3. 定位插件源码目录（pip 包内随附的 `openclaw_context_engine_plugin/` 目录）
4. 调用 `openclaw plugins install <路径>`，或直接将插件文件复制/软链接到 `~/.openclaw/extensions/og-memory-context-engine/`

> **⚠️ 注意**：需要确认 `openclaw plugins install` 内部做了什么。如果只是把目录复制到 `~/.openclaw/extensions/` 下，那 og-memory 可以直接做复制/软链接，不依赖 openclaw 命令。如果 openclaw 做了额外的注册逻辑，则必须走 openclaw 命令。先查看 `~/.openclaw/extensions/og-memory-context-engine/` 的内容来判断。

**`og-memory plugin uninstall`**：
1. 删除 `~/.openclaw/extensions/og-memory-context-engine/` 目录
2. 提示卸载完成

**`og-memory plugin status`**：
1. 检查插件目录是否存在
2. 输出插件版本、og-memory 地址等信息

#### 2.5.3 插件源码随 pip 包分发

将 `openclaw_context_engine_plugin/` 目录作为 **数据文件** 包含在 pip 包中（通过 pyproject.toml 的 `[tool.hatch.build.targets.wheel.shared-data]` 或 `package-data`），但**不作为 Python 模块导入**。它只是一个 JS 插件的文件包，由 `og-memory plugin install` 命令负责安装到 OpenClaw 的扩展目录。

> **⚠️ 注意**：插件是 JS 代码，pip 包只是搬运工。不要试图将其改写为 Python。

> **⚠️ 注意**：`package.json` 缺少 `openclaw.hooks` 的问题需要确认——这是插件本身的 bug 还是 OpenClaw 版本不兼容？如果是插件 bug，在本次记录到 issues。

#### 2.5.4 需要确认的问题

> **[待确认]** `openclaw plugins install` 的内部行为：是纯复制还是有注册逻辑？  
> **[待确认]** 插件的 `og-memory` 地址是从哪里读取的？环境变量？package.json 中的配置字段？OpenClaw 的全局配置？  
> **[待确认]** `package.json` 中 `openclaw.hooks` 缺失是 bug 还是版本差异？  
> **[待确认]** 安全警告（shell 命令执行、credential harvesting）是否有实际安全风险，还是误报？index.js:14 和 index.js:69 的代码需要审查。

---

## 三、不要做的事情

| 禁止项 | 原因 |
|--------|------|
| ❌ 移动现有文件到新目录 | 违反"不调整目录结构"约束 |
| ❌ 重命名现有模块或文件 | 同上 |
| ❌ 修改现有业务逻辑 | 违反"减少代码修改"约束 |
| ❌ 修改现有 `os.getenv()` 调用或变量名 | 违反"保留环境变量"约束 |
| ❌ 引入 config.toml / config.yaml | 违反"保留环境变量"约束 |
| ❌ 重新实现 FastAPI app 或路由注册 | CLI 应复用现有启动逻辑 |
| ❌ 将 oG-Memory API 放入 Docker 容器 | 目标是原生 Python 进程 |
| ❌ 将 deploy-all.sh 作为推荐方式 | 目标是 pip install + CLI |
| ❌ 修改现有的硬编码地址或端口 | 记录到 issues，不在本次修改 |
| ❌ 实现 AGFS 自动下载功能 | 本次只做检测+启动，自动下载留 TODO |
| ❌ 把 JS 插件改写为 Python | 插件是 JS 生态，不改语言 |
| ❌ 修改插件的业务逻辑（index.js） | 只封装安装/卸载流程 |

---

## 四、启动流程（`og-memory serve` 时序）

```
[1] load_dotenv()  ← 加载 .env，现有 os.getenv() 自动生效，零改动
         │
[2] 检查必要环境变量
    ├── OPENGUASS_CONNECTION_STRING 是否设置 → 缺失则报错
    └── OPENAI_API_KEY 是否设置 → 缺失则警告
         │
[3] AGFS 管理
    ├── AGFS_EXTERNAL_URL 已设置？→ 跳过，直接使用该地址
    └── 未设置 →
         ├── which agfs-server（检查 PATH 和 ~/.local/bin/）
         ├── 未找到 → 报错，提示 curl install.sh 安装命令
         ├── 找到 → Popen 启动 agfs-server 子进程
         └── 轮询健康检查，超时 10s 报错
         │
[4] 检查 openGauss 连通性 → 失败则报错
         │
[5] 启动 API server
    └── 调用现有的启动逻辑（不重写）
         │
[服务就绪]

Ctrl+C 或 SIGTERM →
    ├── 停止 API server
    └── 停止 AGFS 子进程（SIGTERM → 5s → SIGKILL）
```

---

## 五、CLI 完整命令列表

| 命令 | 作用 |
|------|------|
| `og-memory serve` | 启动服务（AGFS 子进程 + API server） |
| `og-memory stop` | 停止服务 |
| `og-memory setup` | 交互式初始化：生成 .env + 初始化 DB |
| `og-memory check` | 检查依赖状态：AGFS、openGauss、环境变量 |
| `og-memory db init` | 初始化数据库表（幂等） |
| `og-memory plugin install` | 安装 OpenClaw 插件（自动处理旧版本清理） |
| `og-memory plugin uninstall` | 卸载 OpenClaw 插件 |
| `og-memory plugin status` | 查看插件安装状态 |

---

## 六、最终用户体验

```bash
# 前提：开发者已安装 AGFS 和 openGauss
# AGFS：curl -fsSL https://raw.githubusercontent.com/c4pt0r/agfs/master/install.sh | sh
# openGauss：docker run ... 或已有实例

# 首次
pip install og-memory
og-memory setup                # 交互式 → 生成 .env + 初始化 DB
og-memory serve                # 启动（自动拉起 AGFS 子进程）

# OpenClaw 集成
og-memory plugin install       # 一键安装插件到 ~/.openclaw/extensions/
                               # 已存在？自动清理旧版本再装

# 日常
og-memory serve

# 更新插件
og-memory plugin install       # 自动覆盖，无需手动删

# CI / Docker 环境
export OPENGUASS_CONNECTION_STRING="host=... port=... ..."
export OPENAI_API_KEY="sk-xxx"
og-memory serve
```

---

## 七、实施顺序

| 步骤 | 任务 | 前置条件 |
|------|------|----------|
| 1 | **摸底**：全局搜索现有代码，输出环境变量清单、入口模块路径、依赖列表、AGFS 交互方式（端口、配置文件路径） | 无 |
| 2 | **新增 pyproject.toml**：适配现有目录结构，注册 CLI 入口，列全依赖，将 openclaw_context_engine_plugin 作为数据文件包含 | 步骤 1 |
| 3 | **新增 CLI 模块**（cli.py）：实现 `og-memory serve`，复用现有 server 启动逻辑 | 步骤 2 |
| 4 | **新增 AGFS 管理模块**：检测已安装的 agfs-server + 子进程启停（自动下载留 TODO） | 步骤 1 |
| 5 | **新增 .env.example**：基于步骤 1 的环境变量清单 | 步骤 1 |
| 6 | **验证**：`pip install -e .` 后 `og-memory serve` 能否正常启动全部服务 | 步骤 2-5 |
| 7 | 实现 `og-memory setup` / `check` / `stop` | 步骤 6 |
| 8 | 实现 `og-memory plugin install/uninstall/status` | 步骤 6 + 确认 openclaw 插件机制 |
| 9 | 编写 README Quick Start | 步骤 6 |

---

## 八、待确认清单（实施前必须明确）

| # | 问题 | 如何确认 | 影响 |
|---|------|----------|------|
| 1 | 现有代码的目录结构和入口模块（main.py? app.py?） | `find . -name "*.py" \| head` + Dockerfile CMD | pyproject.toml packages、CLI 启动逻辑 |
| 2 | 现有代码的全部环境变量 key 名称 | `grep -rn "os.getenv\|os.environ" --include="*.py"` | .env.example 准确性 |
| 3 | 现有代码的全部 Python 依赖 | requirements.txt / setup.py / Pipfile | pyproject.toml dependencies |
| 4 | AGFS 在本项目中使用的端口（1833 还是 8080？） | grep 源码中的 AGFS 地址 | 启动 agfs-server 时的端口参数 |
| 5 | AGFS 配置方式（`--port`？`agfs-config.yaml`？） | docker/ 目录下的配置 + `agfs-server --help` | 进程管理器启动命令 |
| 6 | `OPENGUASS_CONNECTION_STRING` 拼写是否确实如此 | `grep -rn "OPENGUASS" --include="*.py"` | .env.example key 名 |
| 7 | `openclaw plugins install` 内部行为 | 对比安装前后 ~/.openclaw/ 目录变化 | 决定 og-memory plugin install 是调用 openclaw 命令还是直接复制 |
| 8 | 插件 og-memory 地址从哪里读取 | 查看 index.js 中 `127.0.0.1:8090` 如何配置 | plugin install 时是否需要写入地址配置 |
| 9 | `package.json` 缺少 `openclaw.hooks` 是 bug 还是版本差异 | 对比 OpenClaw 插件文档 | 是否需要修复 package.json |
| 10 | index.js 安全警告（:14 shell 命令执行, :69 credential harvesting）是否为误报 | 查看具体代码 | 是否需要记录 issue 或修复 |

---

## 九、AGFS 参考信息（供后续实现自动下载时使用）

> 以下信息来自 https://github.com/c4pt0r/agfs ，本次不使用，仅作记录。

- **语言**：Go (server) + Python (shell) + C++ + Rust
- **官方安装方式**：`curl -fsSL https://raw.githubusercontent.com/c4pt0r/agfs/master/install.sh | sh`
- **安装位置**：`~/.local/bin/agfs-server`
- **CI**：GitHub Actions daily-build.yml，每日构建 nightly release
- **Release tag**：`nightly`（滚动更新）
- **下载 URL 格式**：`https://github.com/c4pt0r/agfs/releases/download/nightly/agfs-{os}-{arch}-{date}.tar.gz`
- **支持平台**：linux-amd64, linux-arm64, darwin-amd64, darwin-arm64
- **Docker**：`docker pull c4pt0r/agfs:latest`，默认端口 8080
- **配置**：支持 YAML 配置文件（如 localfs 插件的 `local_dir` 配置）

---

## 十、Issues 记录模板

在实施过程中，遇到以下情况时**不修改，只记录**：

```
### [ISSUE] 标题
- **位置**：文件路径:行号
- **描述**：具体问题
- **影响**：可能导致什么
- **建议**：后续如何修复
```

示例：
```
### [ISSUE] AGFS 地址硬编码
- **位置**：og_memory/retriever.py:42
- **描述**：AGFS URL 硬编码为 http://127.0.0.1:1833，未使用环境变量
- **影响**：无法通过配置改变 AGFS 地址
- **建议**：改为 os.getenv("AGFS_URL", "http://127.0.0.1:1833")

### [ISSUE] 环境变量拼写错误
- **位置**：og_memory/store.py:15
- **描述**：变量名为 OPENGUASS（少一个 S），应为 OPENGAUSS
- **影响**：用户困惑，但功能不受影响
- **建议**：修正拼写，同时兼容旧名称

### [ISSUE] OpenClaw 插件安全警告
- **位置**：openclaw_context_engine_plugin/index.js:14, :69
- **描述**：OpenClaw 检测到 shell 命令执行和环境变量+网络发送模式
- **影响**：安装时输出 WARNING，影响用户信心
- **建议**：审查相关代码，消除不必要的 child_process 调用
```

---

## 十一、验收标准

完成后，以下流程必须能走通：

```bash
# 前提：openGauss 已运行，AGFS 已安装

# 1. 安装
pip install -e .
which og-memory                # ✓

# 2. 配置
og-memory setup                # 交互式生成 .env ✓

# 3. 启动
og-memory serve                # AGFS 子进程自动启动 ✓
                               # http://localhost:8090/api/v1/health 可访问 ✓
                               # assemble / after_turn 端点正常 ✓

# 4. 停止
Ctrl+C                         # AGFS 子进程自动停止 ✓

# 5. AGFS 未安装时
og-memory serve
# → 错误：未找到 agfs-server。
#    请先安装：curl -fsSL https://raw.githubusercontent.com/c4pt0r/agfs/master/install.sh | sh  ✓

# 6. 外部 AGFS
export AGFS_EXTERNAL_URL=http://some-host:8080
og-memory serve                # 不启动子进程，使用外部地址 ✓

# 7. 环境变量兼容（不依赖 .env）
export OPENGUASS_CONNECTION_STRING="..."
export OPENAI_API_KEY="sk-xxx"
og-memory serve                # ✓

# 8. 插件管理
og-memory plugin install       # 安装到 ~/.openclaw/extensions/ ✓
og-memory plugin install       # 再次执行：自动清理旧版本再装 ✓（不报 already exists）
og-memory plugin status        # 显示已安装 ✓
og-memory plugin uninstall     # 清理干净 ✓
```
