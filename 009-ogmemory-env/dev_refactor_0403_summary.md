# dev_refactor_0403 分支工作总结

> 日期：2026-04-03
> 分支：`dev_refactor_0403`
> 基于：`dev` (e95d7c1)

---

## 一、目标

将 oG-Memory 从环境变量配置迁移到 YAML 配置文件，同时将项目重构为 `src/` 布局。

---

## 二、Commit 列表（6 个）

| # | Hash | 说明 |
|---|------|------|
| 1 | `dba96a4` | `docker/ogmemory.example.yaml` → `config.example.yaml` |
| 2 | `0f9d0f5` | 清理过时文件，创建 `src/og_memory/` 骨架 |
| 3 | `a30b6cc` | 将所有模块移入 `src/og_memory/` 包 |
| 4 | `2c00803` | 更新 imports、pyproject、tests 适配 src layout |
| 5 | `98c700c` | 新增 `config.py`（YAML 加载、校验、Config dataclass） |
| 6 | `a5b64df` | 迁移 server、scripts、Docker 到 config.yaml |
| 7 | `fe48bcf` | 删除 `.env.example`、`from_env()`、`DEFAULT_CONFIG` |
| 8 | `8307d71` | 迁移 tests 和验证脚本到 config.yaml |
| 9 | `dc39734` | 合并 unified_config → config.py，ProviderConfig → ProviderFactory |
| 10 | `ae8d681` | 清理根目录文档，更新 README 和 OGMEMORY_ENV.md |

---

## 三、关键改动

### 3.1 配置系统：三合一 → 二合一

| 旧文件 | 职责 | 处理 |
|--------|------|------|
| `src/og_memory/config.py` | 环境变量 + `from_env()` | 重写为 YAML 加载 + 嵌套 dataclass |
| `src/og_memory/providers/unified_config.py` | `OgMemConfig` 扁平 dataclass + env fallback | **已删除**，有用功能合并入 config.py |
| `src/og_memory/providers/config.py` | `ProviderConfig` 工厂 | 重命名为 `provider_factory.py`，删除重复 dataclass |

**最终结构：**

```
config.py              — YAML 加载 + Config dataclass（无外部依赖）
provider_factory.py    — ProviderFactory 静态方法，直接读 Config 创建实例
```

### 3.2 Config dataclass 结构

```yaml
llm:           LlmConfig        # provider, api_key, base_url, model, temperature, max_tokens
embedding:     EmbeddingConfig   # model, base_url, api_key, multimodal
vector_db:     VectorDbConfig    # type, connection_string, dimension, table_name, pool_size
agfs:          AgfsConfig        # base_url, mount_prefix, config_path, data_dir, bin_path
index_service: IndexServiceConfig # interval, workers
ogmem-service: OgmemServiceConfig # host, port, workers, log_level
identity:      IdentityConfig   # account_id, user_id, agent_id
cache:         CacheConfig       # enabled, max_size
```

### 3.3 ProviderFactory

```python
class ProviderFactory:
    @staticmethod
    def create_llm(config: Config) -> LLM           # mock / openai / openai-cached
    @staticmethod
    def create_embedder(config: Config) -> Embedder   # 自动 fallback embedding→llm 配置
    @staticmethod
    def create_vector_index(config: Config) -> VectorIndex  # memory / opengauss
```

### 3.4 src layout

所有 Python 包移入 `src/og_memory/`，通过 `pyproject.toml` 的 `[tool.setuptools.packages.find]` 发现。

### 3.5 文档清理

| 操作 | 文件 |
|------|------|
| 删除 | `ENV.md`（被 OGMEMORY_ENV.md 取代） |
| 删除 | `check-env.sh`（旧 import 路径） |
| 删除 | `providers/unified_config.py` |
| 删除 | `providers/config.py`（→ `provider_factory.py`） |
| 重写 | `OGMEMORY_ENV.md` — 匹配实际 YAML 结构 |
| 重写 | `README.md` — 精简，使用 `service/api` API 示例 |
| 更新 | `docs/deployment_operations_guide.md` — Dockerfile 路径修正 |
| 移动 | `Dockerfile.openclaw` → `docker/Dockerfile.openclaw` |

---

## 四、测试结果

| 指标 | 值 |
|------|---|
| 总测试 | ~485 |
| 通过 | 471 |
| 失败 | 14 |
| 跳过 | 40 |

14 个失败均为 **预存在的外部问题**（openGauss SQL 断言、AGFS URI、远程 LLM 503），非本次改动引入。

---

## 五、合并后待办

- [ ] CLAUDE.md 中的目录结构需更新为 `src/og_memory/` 布局（已有外部修改未提交）
- [ ] `doubao-seed-2-0-code-preview-260215` 模型已不在可用列表，config.yaml 需更新
- [ ] `_normalize_url()` 在 `config.py` 和 `provider_factory.py` 中各有一份，可去重
