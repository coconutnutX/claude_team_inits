# 01-CODEBASE-ANALYSIS — Phase 1：代码仓状态分析

## 目标

在修改任何代码之前，先彻底搞清楚当前代码仓的状态。主线合了很多新代码（包括 OpenGauss 支持），需要理解：

1. **写入链路的完整调用图**：从 `after_turn` 到数据最终落盘的每一步
2. **`index_service` 是什么**：在 `service/` 下新增的模块，用途和调用方式
3. **`outbox_store` 是什么**：它在写入链路中的位置，正确的加载/配置方式
4. **当前插件的问题**：哪些地方硬编码了、哪些该用配置加载
5. **OpenGauss 连接参数**：从 `~/.bashrc` 读取

**Phase 1 不修改任何代码，只分析和记录。**

---

## 任务清单

### T1.1 [INVESTIGATOR] 读取 OpenGauss 连接信息

```bash
# 读取 ~/.bashrc 末尾的 OpenGauss 环境变量
tail -30 ~/.bashrc
# 或者搜索 OpenGauss 相关的环境变量
grep -i "gauss\|GAUSS\|og_\|OG_\|5432\|opengauss" ~/.bashrc
```

记录到 PROGRESS.md "关键发现 → OpenGauss 连接参数"。

### T1.2 [INVESTIGATOR] 阅读 oG-Memory 核心文档

```bash
cd /data/Workspace2/oG-Memory
# 阅读项目主文档
cat CLAUDE.md
```

重点关注：
- 是否提到了 `outbox_store` / `index_service` / OpenGauss
- 配置加载机制（DI？config file？环境变量？）
- 写入 pipeline 的最新描述

### T1.3 [INVESTIGATOR] 分析 service/ 目录结构

```bash
cd /data/Workspace2/oG-Memory

# 查看 service 目录下所有文件
find service/ -type f -name "*.py" | sort

# 查看 index_service 的内容
cat service/index_service.py  # 如果存在

# 查看 api.py 的完整结构
grep -n "def \|class \|import " service/api.py

# 搜索 outbox_store 在哪里被定义和使用
grep -rn "outbox_store\|OutboxStore\|outbox" service/ core/ --include="*.py"

# 搜索 index_service 在哪里被定义和使用
grep -rn "index_service\|IndexService" service/ core/ --include="*.py"

# 搜索 VectorIndex 的定义和使用
grep -rn "VectorIndex\|vector_index\|MockVector" service/ core/ --include="*.py"

# 搜索 OpenGauss 相关代码
grep -rn "opengauss\|gaussdb\|openGauss\|og_" service/ core/ --include="*.py" -i
```

**关键问题要回答：**
- `index_service` 是一个新的服务层还是对 VectorIndex 的封装？
- 它和 `api.py` 是平级关系还是被 `api.py` 调用？
- 它和 `outbox_store` 是什么关系？

### T1.4 [INVESTIGATOR] 追踪写入链路

从 `after_turn` 开始，跟踪整条写入链路：

```bash
cd /data/Workspace2/oG-Memory

# 1. 从插件的 after_turn 开始
grep -n -A 20 "def after_turn" openclaw_context_engine_plugin/bridge/memory_api.py

# 2. 看 after_turn 调用了 service/api.py 的什么方法
grep -n "commit\|write\|save\|store" openclaw_context_engine_plugin/bridge/memory_api.py

# 3. 跟踪 service/api.py 中对应方法的实现
grep -n -A 30 "def commit_messages\|def commit_session\|def write" service/api.py

# 4. 看 service/api.py 调用了什么（extract → plan → write → ???）
grep -n "extract\|plan\|write\|outbox\|index_service\|vector" service/api.py

# 5. 查看 ContextWriter 的实现（写入 AGFS 的部分）
grep -rn "class ContextWriter\|def write" core/ --include="*.py" | head -20

# 6. 查看 OutboxEvent 的定义和流向
grep -rn "OutboxEvent\|outbox_event\|OutboxStore" core/ service/ --include="*.py"

# 7. 查看 Embedder 的定义（向量化）
grep -rn "class Embedder\|def embed\|embedding" core/ service/ --include="*.py" | head -20
```

**画出完整调用图：**

```
after_turn(params)
  → bridge/memory_api.py::after_turn()
    → service/api.py::???()
      → CandidateExtractor.extract() — LLM 抽取知识
      → MergePolicy.plan() — 决定 create/merge/append/skip
      → ContextWriter.write() — 写入 AGFS
      → OutboxEvent — ???
        → outbox_store — ???
          → Embedder — 向量化
          → VectorIndex.upsert() — 写向量索引
          → OpenGauss ??? — 写数据库
```

**填充每个 `???` 的实际代码路径。**

### T1.5 [INVESTIGATOR] 分析 outbox_store 的配置加载方式

```bash
cd /data/Workspace2/oG-Memory

# 查看 outbox_store 的类定义
grep -rn "class.*OutboxStore\|class.*Outbox" . --include="*.py" | head -10

# 查看它的实例化方式（是 new 出来的还是从配置加载的）
grep -rn "OutboxStore(" . --include="*.py" | head -10

# 查看配置文件格式
find . -name "*.yaml" -o -name "*.yml" -o -name "*.toml" -o -name "config*.py" | head -10
cat config*.py 2>/dev/null || cat config/*.py 2>/dev/null

# 查看是否有依赖注入或工厂模式
grep -rn "def create_\|def build_\|def make_\|Factory\|Registry\|Container" service/ core/ --include="*.py" | head -20

# 查看主程序入口如何初始化各组件
grep -rn "def main\|def setup\|def init\|def bootstrap" service/ --include="*.py" | head -10
```

**关键问题要回答：**
- `outbox_store` 应该如何正确实例化？（通过配置文件？环境变量？工厂方法？）
- 是否有多种 outbox_store 实现（mock vs OpenGauss）？
- 切换后端（mock → OpenGauss）需要改哪些配置？

### T1.6 [INVESTIGATOR] 分析当前插件的硬编码问题

```bash
cd /data/Workspace2/oG-Memory/openclaw_context_engine_plugin

# 查看 bridge/memory_api.py 的完整内容
cat bridge/memory_api.py

# 搜索硬编码的实例化
grep -n "VectorIndex(\|MockVector\|InMemory\|outbox\|index_service" bridge/memory_api.py

# 查看 import 语句
head -30 bridge/memory_api.py
```

**关键问题要回答：**
- 插件当前是如何初始化 VectorIndex 的？（硬编码了 mock？）
- 插件是否使用了 outbox_store？如果没有，写入链路如何完成？
- 哪些初始化逻辑应该从配置加载而不是硬编码？

### T1.7 [INVESTIGATOR] 检索链路分析

```bash
cd /data/Workspace2/oG-Memory

# 查看 search_memory 的实现
grep -n -A 30 "def search_memory" service/api.py

# 查看它用的是哪个 VectorIndex
grep -n "vector_index\|VectorIndex\|index_service" service/api.py

# 查看 index_service 中的检索逻辑（如果存在）
grep -n -A 20 "def search\|def query" service/index_service.py 2>/dev/null
```

### T1.8 [LEAD] 汇总分析结果

将以上所有发现整理到 PROGRESS.md 的"关键发现"区域：

```markdown
## 关键发现

### OpenGauss 连接参数
- host: ???
- port: ???
- user: ???
- password: ???
- database: ???

### 写入链路完整调用图
after_turn → ??? → ??? → ??? → AGFS + OpenGauss

### index_service 用途
- 定义位置：???
- 功能：???
- 被谁调用：???
- 和 VectorIndex 的关系：???

### outbox_store 用途和加载方式
- 定义位置：???
- 功能：???
- 正确加载方式：???（配置文件？工厂方法？）
- 和 index_service 的关系：???

### 当前插件问题清单
1. ??? 硬编码了 ???
2. ??? 没有使用 outbox_store
3. ...
```

---

## 完成标准

全部满足后在 PROGRESS.md 标记 Phase 1 为 ✅：

- [ ] OpenGauss 连接参数已记录
- [ ] 写入链路完整调用图已画出（从 after_turn 到 OpenGauss 的每一步）
- [ ] index_service 的用途和调用方式已搞清楚
- [ ] outbox_store 的用途和正确加载方式已搞清楚
- [ ] index_service 和 outbox_store 的关系已搞清楚
- [ ] 当前插件的硬编码问题已列出
- [ ] 所有发现已记录到 PROGRESS.md "关键发现"区域

---

## 注意事项

- **不要修改任何代码**，Phase 1 只做分析
- 如果发现的架构比预期更复杂（如多层抽象、多种配置方式），在 PROGRESS.md 写 [ASK-HUMAN] 让人类确认理解是否正确
- 如果 `CLAUDE.md` 中有关于 outbox_store / index_service 的说明，优先以文档为准
- 如果文档和代码不一致，以代码为准，但在 PROGRESS.md 中标注差异
