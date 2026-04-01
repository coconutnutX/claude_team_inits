# Context Engine (原 oG-Memory) 重构计划

> **交付目标**：用户通过 `pip install context-engine` 安装后，执行 `ogc serve` 即可启动完整服务（API + AGFS），无需手动运行 Docker 或部署脚本。
>
> **⚠️ 核心约束（贯穿全程，违反任何一条都需要停下来确认）**：
> 1. **不调整现有目录结构**——不搬家、不重命名、不重新归类现有文件
> 2. **尽量减少代码修改**——业务代码不改，仅新增部署封装层
> 3. **保留环境变量作为配置方式**——不引入 config.toml，现有 `os.getenv()` 调用全部保持原样
> 4. **发现问题只记录不修改**——迁移过程中发现的业务 bug、硬编码、命名不一致等问题，记录到 issues 清单，不在本次修复
> 5. **Plan Compliance**——不得擅自修改计划中约定的端口、路径、变量名、接口参数。遇到冲突时 ask human

---

## 阶段概览

```
Phase 0: 环境调查  →  输出调查报告  →  ⚠️ 人工审核通过后才能继续
Phase 1: 精确规格  →  输出开发规格文档（每个组件的端口、配置、路径全部写死）
Phase 2: 实施      →  按规格文档开发，不得偏离
```

---

## Phase 0：环境调查（必须先完成，必须人工审核）

### 目标

在不修改任何代码的前提下，完整记录当前环境中三个组件的**实际运行方式**。产出一份调查报告，作为后续所有开发的唯一事实依据。

### ⚠️ 完成条件

> **Phase 0 的产出物必须经过人类检查确认后，才能进入 Phase 1。**
> 不要在人类确认前开始写任何代码。

### 调查内容

#### 0.1 AGFS 调查

需要搞清楚并记录的信息：

**安装与二进制**
- agfs-server 二进制的实际路径（`which agfs-server`、`ls ~/.local/bin/agfs-server`）
- 版本信息（`agfs-server --version` 或 `agfs-server -v`）
- `agfs-server --help` 的完整输出（记录所有命令行参数）

**启动方式**
- 当前项目中 AGFS 是如何启动的？（Docker？直接运行？脚本？）
- 启动命令的完整内容（包括所有参数）
- 是否需要配置文件？如果需要，配置文件的完整路径和完整内容

**网络配置**
- 实际监听地址和端口：是 `0.0.0.0:1833`？`127.0.0.1:1833`？`:8080`？
- 确认方式：启动后 `ss -tlnp | grep agfs` 或 `netstat -tlnp | grep agfs` 或查看启动日志
- 健康检查端点：URL 是什么？（`/health`？`/healthz`？）返回什么？

**配置文件**
- 配置文件格式（YAML？TOML？JSON？）
- 配置文件中每个字段的**准确名称**（是 `local_dir` 还是 `local_path`？是 `path` 还是 `mount_path`？）
- 把当前使用的配置文件**原样复制**到调查报告中，不做任何修改

**存储路径**
- 数据实际存储在哪里？（哪个目录？Docker volume？）
- 该路径是在配置文件中指定的还是默认值？

**现有代码中的调用方式**
- `grep -rn "agfs\|AGFS\|1833\|8080" --include="*.py" --include="*.js" --include="*.yaml" --include="*.yml"` 的完整输出
- 从结果中整理：代码中使用的 AGFS 地址（硬编码还是环境变量？变量名是什么？）

#### 0.2 openGauss 调查

**启动方式**
- 当前的 Docker 启动命令（完整复制，包括所有 -e -p -v 参数）
- 镜像名和版本（`docker images | grep gauss`）
- 容器名（`docker ps | grep gauss`）

**网络配置**
- 容器内端口 → 宿主机端口映射（是 5432:5432 还是 8888:5432？）
- 现有代码中连接用的端口是哪个？

**认证信息**
- 用户名、密码、数据库名（从启动命令的 -e 参数中提取）
- 现有代码中的连接串格式（完整复制 `OPENGUASS_CONNECTION_STRING` 的值）

**初始化**
- 有无 init.sql？内容是什么？
- 需要安装什么插件/扩展？（向量索引相关的）
- 建了哪些表？`\dt` 或 `SELECT * FROM pg_tables WHERE schemaname='public'` 的输出

**现有代码中的调用方式**
- `grep -rn "OPENGUASS\|opengauss\|5432\|8888\|psycopg\|connection_string" --include="*.py"` 的完整输出
- 确认环境变量的准确拼写

#### 0.3 OpenClaw 插件调查

**插件目录结构**
- `ls -la ~/.openclaw/extensions/og-memory-context-engine/` 的完整输出
- `cat ~/.openclaw/extensions/og-memory-context-engine/package.json` 的完整内容
- `cat ~/.openclaw/extensions/og-memory-context-engine/index.js` 的完整内容（或至少关键部分）

**插件配置**
- og-memory（context engine）的地址是从哪里读取的？
  - 环境变量？变量名是什么？
  - package.json 中的字段？
  - OpenClaw 的配置文件？
  - 硬编码？

**安装行为**
- `openclaw plugins install` 实际做了什么？
  - 执行前后 `ls -la ~/.openclaw/extensions/` 的对比
  - 是纯复制还是有 npm install 等步骤？
  - 有没有注册到某个配置文件中？

**启动日志**
- 插件启动时的完整日志输出（包含那行 `[plugins] [og-memory] mode=http url=...`）
- 安全警告的完整内容

#### 0.4 现有代码全局信息

**环境变量完整清单**
```bash
grep -rn "os.getenv\|os.environ\|process.env" --include="*.py" --include="*.js"
```
整理成表格：变量名 | 所在文件:行号 | 默认值 | 用途

**Python 依赖**
- `cat requirements.txt` 或 `cat setup.py` 或 `cat pyproject.toml`（如果已存在）
- `pip freeze` 当前环境的完整输出

**入口模块**
- 现有 API server 是如何启动的？入口文件是哪个？
- 启动命令是什么？（`python app.py`？`uvicorn main:app`？）
- Dockerfile 中的 CMD 是什么？

### 调查报告输出格式

```markdown
# Context Engine 环境调查报告

> 调查时间：YYYY-MM-DD
> 调查环境：（操作系统、Python 版本等）

## 1. AGFS
### 1.1 二进制
- 路径：
- 版本：
- --help 输出：（原样粘贴）

### 1.2 启动
- 启动命令：（原样粘贴）
- 配置文件路径：
- 配置文件内容：（原样粘贴）

### 1.3 网络
- 监听地址：
- 监听端口：
- 健康检查：GET http://______ → 返回 ______

### 1.4 存储
- 数据目录：

### 1.5 代码中的引用
| 文件:行号 | 引用内容 | 硬编码/环境变量 |
|-----------|----------|----------------|

## 2. openGauss
（同样结构）

## 3. OpenClaw 插件
（同样结构）

## 4. 全局信息
### 4.1 环境变量清单
| 变量名 | 文件:行号 | 默认值 | 用途 |
|--------|-----------|--------|------|

### 4.2 Python 依赖
（原样粘贴）

### 4.3 入口模块
- 文件：
- 启动命令：

## 5. 发现的问题
（按 issues 模板记录，不修复）
```

---

## Phase 1：精确开发规格（基于 Phase 0 产出）

### 目标

根据 Phase 0 的调查报告，编写精确到每一个字段的开发规格文档。后续实施阶段的所有代码必须严格遵循此文档，不得偏离。

### 产出物

一份开发规格文档，包含以下内容：

**CLI 命令规格**

| 命令 | 作用 |
|------|------|
| `ogc serve` | 启动服务（AGFS 子进程 + API server） |
| `ogc stop` | 停止服务 |
| `ogc setup` | 交互式初始化：生成 .env + 初始化 DB |
| `ogc health-check` | 检查依赖状态：AGFS、openGauss、环境变量 |
| `ogc db init` | 初始化数据库表（幂等） |
| `ogc plugin install` | 安装 OpenClaw 插件（自动清理旧版本） |
| `ogc plugin uninstall` | 卸载 OpenClaw 插件 |
| `ogc plugin status` | 查看插件安装状态 |

**AGFS 启动规格**（基于 Phase 0 调查结果填入精确值）
- 二进制路径：______（从调查报告 1.1 取）
- 启动命令：`______ ______ ______`（从调查报告 1.2 取，精确到每个参数）
- 配置文件：生成到 ______，内容为 ______（从调查报告 1.2 取，精确到每个字段名）
- 监听地址：______（从调查报告 1.3 取）
- 健康检查：______（从调查报告 1.3 取）

**openGauss 连接规格**（基于 Phase 0 调查结果填入精确值）
- 连接串格式：______（从调查报告 2 取）
- 环境变量名：______（从调查报告 4.1 取，精确拼写）
- 连通性检测方式：______

**插件安装规格**（基于 Phase 0 调查结果填入精确值）
- 插件源码目录：______
- 安装目标目录：______（从调查报告 3 取）
- 安装方式：复制 / `openclaw plugins install` / 软链接（从调查报告 3 确认）
- 地址配置注入方式：______（从调查报告 3 取）

**pyproject.toml 规格**
- 包名：`context-engine`
- CLI 入口：`ogc = "______:cli"`（从调查报告 4.3 确认入口模块后填入）
- dependencies：______（从调查报告 4.2 取）
- 包含的数据文件（JS 插件）：______

**.env.example 规格**
- 完整的变量清单及默认值（从调查报告 4.1 逐条取，拼写必须完全一致）

---

## Phase 2：实施（严格按 Phase 1 规格执行）

### 约束

- **Phase 1 规格文档是唯一依据**。不得凭记忆或猜测填写端口、路径、配置字段名
- **每一步 commit 后验证**。不要写完全部代码再测试
- **遇到规格文档与现实不符时 stop and ask**，不得自行修改

### 实施顺序

| 步骤 | 任务 | 验证方式 |
|------|------|----------|
| 1 | 新增 pyproject.toml | `pip install -e .` 成功，`which ogc` 能找到 |
| 2 | 新增 CLI 骨架（ogc serve / ogc health-check） | `ogc --help` 输出命令列表 |
| 3 | 实现 AGFS 检测 + 子进程启动 | `ogc serve` 能启动 AGFS 子进程，Ctrl+C 能停 |
| 4 | 串联 API server 启动 | `ogc serve` 后 health 端点可访问 |
| 5 | 新增 .env.example + dotenv 加载 | 环境变量从 .env 正确加载 |
| 6 | 实现 ogc setup | 交互式生成 .env |
| 7 | 实现 ogc health-check | 输出各组件状态 |
| 8 | 实现 ogc plugin install/uninstall/status | 插件正确安装到 ~/.openclaw/extensions/ |
| 9 | 编写 README | Quick Start 流程走通 |

### AGFS 处理方案

#### 当前阶段：假定已安装，检测并启动

**默认假定开发者已通过官方方式安装了 AGFS**。ogc 只负责：

1. **检测**：检查 `agfs-server` 是否在 PATH 中（优先 `~/.local/bin/agfs-server`）
2. **启动**：作为子进程启动，参数和配置文件严格按 Phase 1 规格
3. **就绪等待**：轮询健康检查端点，超时 10s 报错
4. **停止**：SIGTERM → 等 5s → SIGKILL
5. **未安装时提示**：

```
错误：未找到 agfs-server。请先安装 AGFS：
  curl -fsSL https://raw.githubusercontent.com/c4pt0r/agfs/master/install.sh | sh
```

**外部 AGFS 支持**：环境变量 `AGFS_EXTERNAL_URL` 已设置时，跳过检测和进程管理。

#### 未来阶段（TODO，本次不实现）

```python
# TODO: 自动下载 AGFS 二进制
# AGFS 仓库 https://github.com/c4pt0r/agfs 已具备条件：
# - Go 编写，GitHub Actions 每日构建 nightly release
# - 官方 install.sh：curl -fsSL https://raw.githubusercontent.com/c4pt0r/agfs/master/install.sh | sh
# - 多平台二进制：linux-amd64/arm64, darwin-amd64/arm64
# - 下载 URL：https://github.com/c4pt0r/agfs/releases/download/nightly/agfs-{os}-{arch}-{date}.tar.gz
# - 安装到 ~/.local/bin/agfs-server
```

### OpenClaw 插件管理

插件是 JavaScript/Node.js 编写的（index.js），安装到 `~/.openclaw/extensions/og-memory-context-engine/`。

`ogc plugin install`：
1. 检查目标目录是否已存在 → 已存在则先删除
2. 安装插件（方式依据 Phase 0 调查结果确定）

`ogc plugin uninstall`：
1. 删除 `~/.openclaw/extensions/og-memory-context-engine/`

`ogc plugin status`：
1. 检查插件目录是否存在，输出信息

> **⚠️ 注意**：插件是 JS 代码，pip 包只是搬运工。不要改写为 Python，不要修改 index.js 业务逻辑。

---

## 不要做的事情

| 禁止项 | 原因 |
|--------|------|
| ❌ 在 Phase 0 完成前写任何代码 | 没有事实依据，会写错 |
| ❌ 在 Phase 0 未经人工确认前进入 Phase 1 | 调查结果可能有误 |
| ❌ 移动现有文件到新目录 | 不调整目录结构 |
| ❌ 修改现有业务逻辑 | 减少代码修改 |
| ❌ 修改现有 `os.getenv()` 调用或变量名 | 保留环境变量方式 |
| ❌ 引入 config.toml / config.yaml 作为项目配置 | 保留环境变量方式 |
| ❌ 重新实现 FastAPI app 或路由注册 | CLI 复用现有启动逻辑 |
| ❌ 将 deploy-all.sh 作为推荐方式 | 目标是 pip + CLI |
| ❌ 实现 AGFS 自动下载 | 本阶段只做检测+启动 |
| ❌ 把 JS 插件改写为 Python | 不改语言 |
| ❌ 凭记忆填写端口、路径、配置字段名 | 必须从 Phase 0 报告中取 |
| ❌ 自行解决规格与现实的冲突 | stop and ask |

---

## Issues 记录模板

实施过程中发现问题时**不修改，只记录**：

```
### [ISSUE] 标题
- **位置**：文件路径:行号
- **描述**：具体问题
- **影响**：可能导致什么
- **建议**：后续如何修复
```

---

## 验收标准

```bash
# 前提：openGauss 已运行，AGFS 已安装

# 安装
pip install -e .
which ogc                      # ✓

# 配置
ogc setup                      # 交互式生成 .env ✓

# 启动
ogc serve                      # AGFS 子进程自动启动 ✓
                               # API health 端点可访问 ✓
                               # assemble / after_turn 正常 ✓

# 停止
Ctrl+C                         # AGFS 子进程自动停止 ✓

# 健康检查
ogc health-check               # 输出各组件状态 ✓

# AGFS 未安装时
ogc serve                      # → 报错 + 提示安装命令 ✓

# 外部 AGFS
export AGFS_EXTERNAL_URL=http://some-host:8080
ogc serve                      # 不启动子进程 ✓

# 环境变量兼容
export OPENGUASS_CONNECTION_STRING="..."
export OPENAI_API_KEY="sk-xxx"
ogc serve                      # 不依赖 .env ✓

# 插件
ogc plugin install              # 安装到 ~/.openclaw/extensions/ ✓
ogc plugin install              # 再次：自动清理旧版本 ✓
ogc plugin status               # 已安装 ✓
ogc plugin uninstall            # 清理 ✓
```
