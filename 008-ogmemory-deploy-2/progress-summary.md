# oG-Memory CLI 开发进展总结

> 更新时间：2026-04-01
> 分支：dev_0401_deploy_2

---

## 已完成的工作

### Phase 0：环境调查 ✅
- 完整调查了 AGFS、openGauss、OpenClaw 插件的环境配置
- 记录了 7 个 issues，其中 5 个已决策处理
- 输出文件：`phase0-investigation-report.md`

### Phase 1：开发规格 ✅
- 精确定义了 CLI 命令、AGFS 启动、环境变量等规格
- 输出文件：`phase1-development-spec.md`

### Phase 2：实现 ✅
共 6 个 commit：

| Commit | 说明 |
|--------|------|
| `d33f7a6` | pyproject.toml 添加 CLI 入口点和缺失依赖 |
| `b699f04` | 完整 CLI 实现 (serve/stop/setup/health-check/plugin) |
| `2893f20` | 干净的 .env.example 模板 |
| `6c3b1e5` | .gitignore 更新 |
| `351ac78` | 修复 PROJECT_ROOT 使用 cwd，dotenv 加载逻辑修正 |
| `a5ab269` | .env.example 连接串加引号（兼容 bash source） |

### Phase 3：验证 + 修复 ✅
| Commit | 说明 |
|--------|------|
| `849d06f` | _load_dotenv 去引号 + 始终覆盖 env var |
| `e9d3893` | 将 IndexService 集成到 ogc serve |

端到端验证通过：
- 写入记忆 (`after_turn`) → 存储到 AGFS ✅
- IndexService 同步 → 向量索引创建 (5条记录) ✅
- 查询记忆 (`assemble`, 使用 `prompt` 字段) → 返回结果 ✅

---

## 当前未提交的改动

`cli.py` 中 health-check 新增了 IndexService 进程检测（通过 `pgrep -f run_index_service.py`）。

---

## 遗留问题

### 1. openGauss 连接 health-check 仍显示带引号
**现象**：用户在新终端运行 `ogc health-check`，openGauss Connection 显示 `"host=...`（带引号），Status: CANNOT CONNECT
**原因**：`pip install` 不是 editable 模式，修改源码不生效。需要 `pip install -e .`
**状态**：已修复代码（`_load_dotenv` 去引号 + 覆盖），用户需重新 `pip install -e .` 后验证

### 2. health-check 中 IndexService 检测
**状态**：代码已写好但未提交。需要在用户验证 openGauss 连接修复后一起提交。

---

## 文件清单

### 新增文件
| 文件 | 说明 |
|------|------|
| `cli.py` | CLI 入口，Click 命令组 (~620行) |
| `.env.example` | 环境变量模板（替代旧版） |
| `.claude-team/phase0-investigation-report.md` | Phase 0 调查报告 |
| `.claude-team/phase1-development-spec.md` | Phase 1 开发规格 |
| `.claude-team/progress-summary.md` | 本文件 |

### 修改文件
| 文件 | 改动 |
|------|------|
| `pyproject.toml` | 添加 scripts 入口、补充依赖、force-include cli.py |
| `.gitignore` | 添加 .omc |

### 未修改
- `server/app.py` — 不动
- `scripts/run_index_service.py` — 不动
- `openclaw_context_engine_plugin/` — 不动
- `core/`, `fs/`, `extraction/`, `commit/`, `index/`, `retrieval/`, `providers/`, `service/` — 不动

---

## 关键经验

1. **`.env` 中含空格的值必须用引号包裹**，否则 `source .env` 只取第一个 token
2. **`_load_dotenv` 必须去掉引号**，否则 psycopg2 收到带引号的连接串
3. **必须用 `pip install -e .`** 安装，否则修改源码不会反映到 `ogc` 命令
4. **`ogc serve` 现在启动三个服务**：AGFS + IndexService + API server，不再需要手动启动 IndexService
5. **assemble 端点使用 `prompt` 字段**（不是 `query`），这是 API 设计决定的
6. **IndexService 轮询间隔 30 秒**，写入后需要等待才能查到
