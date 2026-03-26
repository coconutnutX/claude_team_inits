# 02-PLUGIN-FIX-OPENGAUSS — Phase 2：修复插件接入 OpenGauss

## 目标

根据 Phase 1 的分析结果，修复 `openclaw_context_engine_plugin`，让它：

1. **按正确方式使用 outbox_store**（不硬编码，通过配置/工厂加载）
2. **写入链路完整落地到 OpenGauss**（不只写 AGFS，还要写数据库）
3. **检索链路从 OpenGauss 读取**（不用 mock VectorIndex）
4. **配置驱动**：切换后端只需改配置，不需改代码

**前置：Phase 1 必须完成。** 以下所有 `???` 需要用 Phase 1 的实际发现替换。

---

## 设计原则

### 1. 插件不应该知道 OpenGauss 的存在

```
❌ 错误：bridge/memory_api.py 直接 import psycopg2 连 OpenGauss
✅ 正确：bridge/memory_api.py 调用 service/api.py，由 service 层根据配置选择后端
```

### 2. outbox_store 应从配置加载

```
❌ 错误：
    outbox_store = InMemoryOutboxStore()  # 硬编码

✅ 正确（伪代码，以 Phase 1 发现的实际方式为准）：
    outbox_store = config.get_outbox_store()  # 或工厂方法
    # 或
    outbox_store = OutboxStoreFactory.create(config)
    # 或
    outbox_store = create_outbox_store_from_env()
```

### 3. 不修改 core/ 和 service/api.py 的接口

如果 service/api.py 已有正确的初始化逻辑但插件没有正确调用，应该修复插件的调用方式，而不是改 service 层。

---

## 任务清单

### T2.1 [CODER] 确认修改方案

在动手前，先回答以下问题（基于 Phase 1 的分析）：

1. **service/api.py 的初始化流程是什么？**
   - 它是否有 `init()` / `setup()` / `configure()` 方法？
   - 它是否需要外部传入 outbox_store 实例？
   - 还是它内部根据配置自动选择后端？

2. **bridge/memory_api.py 当前如何初始化 service 层？**
   - 是直接 import 函数调用？
   - 还是实例化某个 API 对象？
   - 有没有传入配置？

3. **需要改什么才能让数据流入 OpenGauss？**
   - 是改配置文件（如 yaml/toml）？
   - 是设环境变量？
   - 是修改 bridge 层的初始化代码？
   - 还是以上都需要？

将方案写入 PROGRESS.md 的 `[DECISION]` 记录中，等 LEAD 审核。

### T2.2 [CODER] 修改 bridge/memory_api.py 的初始化逻辑

**基于 T2.1 的方案，修改文件。** 以下是几种可能的修改方向（以 Phase 1 实际发现为准）：

#### 方向 A：如果 service 层有统一的初始化入口

```python
# bridge/memory_api.py

# 之前（硬编码）：
from service.api import MemoryWriteAPI, MemoryReadAPI
write_api = MemoryWriteAPI()  # 可能内部用了 mock

# 之后（通过配置）：
from service.api import MemoryWriteAPI, MemoryReadAPI
from service.config import load_config  # 或实际的配置加载方式

config = load_config()  # 自动读取 OpenGauss 连接信息
write_api = MemoryWriteAPI(config=config)
read_api = MemoryReadAPI(config=config)
```

#### 方向 B：如果需要手动传入 outbox_store

```python
# bridge/memory_api.py

# 之前：
from service.api import commit_messages, search_memory
# 没有初始化 outbox_store

# 之后：
from service.api import commit_messages, search_memory
from service.outbox_store import OpenGaussOutboxStore  # 或实际类名
from service.index_service import IndexService  # 如果需要

# 读取 OpenGauss 连接参数（从环境变量）
import os
og_config = {
    "host": os.environ.get("OG_HOST", "localhost"),
    "port": os.environ.get("OG_PORT", "5432"),
    "user": os.environ.get("OG_USER"),
    "password": os.environ.get("OG_PASSWORD"),
    "database": os.environ.get("OG_DATABASE"),
}
outbox_store = OpenGaussOutboxStore(og_config)
# 注册到 service 层
```

#### 方向 C：如果只需要设环境变量

```python
# bridge/memory_api.py
# 不需要改 Python 代码，service 层自动从环境变量读取
# 但需要确保环境变量在插件执行时可用

# 在 index.js 的 callPython 中传递环境变量
# 或在 openclaw.json 的 plugin config 中设置
```

**⚠️ 以上是假设方案，具体以 Phase 1 发现为准。如果 Phase 1 的分析不足以确定方案，写 [ASK-HUMAN]。**

### T2.3 [CODER] 修改 OpenGauss 配置

根据 Phase 1 发现的配置方式，设置 OpenGauss 连接参数：

```bash
# 可能需要修改的配置文件（以 Phase 1 发现为准）：
# - ~/.agfs-config.yaml
# - oG-Memory 项目根目录下的某个 config 文件
# - 环境变量
# - openclaw.json 中的 plugin config
```

### T2.4 [CODER] 确认 index.js 不需要改动

检查 `index.js` 中的 `callPython` 是否需要传递额外的环境变量或参数：

```bash
cat openclaw_context_engine_plugin/index.js
```

如果 OpenGauss 连接信息通过环境变量传递，确认 `callPython` 的 `execFileSync` 能继承这些环境变量。

### T2.5 [RUNNER] 单元验证

在修改完成后，先做最小验证：

```bash
cd /data/Workspace2/oG-Memory
conda activate py11

# 1. 验证 import 不报错
python -c "
import sys
sys.path.insert(0, '.')
from openclaw_context_engine_plugin.bridge.memory_api import after_turn, assemble
print('Import OK')
"

# 2. 验证 OpenGauss 连接
python -c "
# 根据实际的连接方式验证
# 如果是 psycopg2:
import psycopg2
conn = psycopg2.connect(host='???', port='???', user='???', password='???', dbname='???')
cur = conn.cursor()
cur.execute('SELECT 1')
print('OpenGauss connection OK:', cur.fetchone())
conn.close()
"

# 3. 验证 outbox_store 初始化
python -c "
import sys
sys.path.insert(0, '.')
# 根据 Phase 1 发现的实际方式验证 outbox_store 能正确初始化
# ...
print('OutboxStore init OK')
"
```

### T2.6 [CODER] 编写 pytest 测试

位置：`tests/integration/test_opengauss_write.py`

```python
"""
验证写入链路能完整落地到 AGFS + OpenGauss。
运行：cd /data/Workspace2/oG-Memory && pytest tests/integration/test_opengauss_write.py -v
"""
import pytest


def test_write_reaches_opengauss():
    """写入一条记忆后，OpenGauss 中应该能查到。"""
    # 1. 构造测试消息
    messages = [
        {"role": "user", "content": "我叫测试用户，是一名前端工程师"},
        {"role": "assistant", "content": "你好！"},
    ]

    # 2. 构造 RequestContext
    # ctx = ...

    # 3. 调用写入（通过 service/api.py）
    # result = commit_messages(messages, ctx)

    # 4. 验证 AGFS 中有数据
    # assert ... AGFS 文件存在

    # 5. 验证 OpenGauss 中有数据
    # 连接 OpenGauss，查询刚写入的数据
    # assert ... 数据库中有对应记录

    pass  # ← CODER 替换为实际代码


def test_search_from_opengauss():
    """检索应该能从 OpenGauss 的向量索引返回结果。"""
    # 1. 确保有数据（复用上面的写入或依赖已有数据）
    # 2. 调用 search_memory
    # 3. 验证返回了非空结果
    # 4. 验证结果与查询相关

    pass  # ← CODER 替换为实际代码
```

**要求：**
- 使用 pytest 标准格式，用 `assert` 验证
- 不需要 `if __name__ == "__main__"`
- 覆盖写入和检索两个方向
- 如果 OpenGauss 未启动，测试应 skip 而不是报错（用 `pytest.mark.skipif`）

### T2.7 [RUNNER] 运行 pytest 测试

```bash
cd /data/Workspace2/oG-Memory
conda activate py11
pytest tests/integration/test_opengauss_write.py -v
```

### T2.8 [LEAD] 记录结果到 PROGRESS.md

记录：
- 修改了哪些文件，改了什么
- outbox_store 的实际加载方式
- OpenGauss 连接是否成功
- pytest 测试是否通过
- 如果有问题，写 [ASK-HUMAN]

---

## 完成标准

全部满足后在 PROGRESS.md 标记 Phase 2 为 ✅：

- [ ] bridge/memory_api.py 不再硬编码 mock VectorIndex / outbox_store
- [ ] 写入链路完整：after_turn → service → AGFS + OpenGauss
- [ ] 检索链路完整：assemble → search_memory → OpenGauss 向量索引
- [ ] outbox_store 按正确方式加载（配置驱动，非硬编码）
- [ ] pytest 测试通过
- [ ] 所有修改已记录到 PROGRESS.md

---

## 常见问题

**Q: Phase 1 分析结果不够明确，不知道怎么改？**
- 在 PROGRESS.md 写 [ASK-HUMAN]，列出具体不确定的地方
- 不要猜测，不要硬改

**Q: OpenGauss 连接失败？**
- 确认 OpenGauss 服务已启动：`systemctl status opengauss` 或 `ps aux | grep gauss`
- 确认连接参数正确（从 ~/.bashrc 读取）
- 确认防火墙 / pg_hba.conf 允许连接
- 在 PROGRESS.md 写 [ASK-HUMAN]

**Q: outbox_store 有多种实现，不确定用哪个？**
- 查看 oG-Memory 的配置文档或 CLAUDE.md
- 搜索 `if.*gauss\|if.*postgres\|backend.*=` 等条件分支
- 在 PROGRESS.md 记录发现的所有实现，写 [ASK-HUMAN]

**Q: 修改后 import 报错？**
- 确认 Python path 包含 oG-Memory 项目根目录
- 确认 conda 环境正确（py11）
- 检查新依赖是否已安装（如 psycopg2 / openGauss connector）

**Q: 写入成功但 OpenGauss 中没有数据？**
- 这正是本次任务要解决的核心问题
- 检查 outbox_store 是否真的被调用了（加 log）
- 检查 outbox_store 内部是否有异步处理（可能需要等待/flush）
- 检查是否有事务未提交
