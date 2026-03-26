# PLAN-CHANGE — Phase 2 补充：Search 链路接入 OpenGauss

## 背景

Phase 2 完成了写入链路的修正（插件不管 IndexService、outbox event 正确生成），但 **search 链路还没有接到 OpenGauss**。

分析 `MemoryReadAPI` 源码后，发现 search 链路的结构如下：

```
bridge/memory_api.py::assemble()
  → MemoryReadAPI.search_memory(query, ctx, ...)
    → RetrievalPipeline.run(query, ctx, ...)
      → 内部某处调用 VectorIndex.search()
        → ??? 用的是 InMemoryVectorIndex 还是 OpenGaussVectorIndex？
```

`MemoryReadAPI` 的构造函数：
```python
class MemoryReadAPI:
    def __init__(
        self,
        pipeline: RetrievalPipeline,   # ← 关键：这个 pipeline 内部用哪个 VectorIndex？
        read_service: MemoryReadService,
        config: RetrievalConfig | None = None,
    )
```

**核心问题：`RetrievalPipeline` 是怎么被创建的？它的 VectorIndex 从哪来？**

插件进程是 `execFileSync` 短生命周期进程。如果 `_get_read_api()` 每次都创建一个新的 `InMemoryVectorIndex()`，那就是每次都查一个空索引，永远返回空。必须让它用 OpenGaussVectorIndex 连到真实数据库去查。

## 需要调查的问题

### Q1: bridge 层怎么实例化 MemoryReadAPI？

```bash
# 查看 bridge 层怎么创建 read api
grep -n -A 20 "_get_read_api\|MemoryReadAPI\|read_api" \
  openclaw_context_engine_plugin/bridge/memory_api.py
```

要搞清楚：
- 有没有 `_get_read_api()` 单例函数？
- `RetrievalPipeline` 是怎么创建的？
- 传入的 VectorIndex 是哪个实现？

### Q2: RetrievalPipeline 内部怎么用 VectorIndex？

```bash
# 查看 RetrievalPipeline 的构造函数
grep -n -A 20 "class RetrievalPipeline" retrieval/pipeline.py

# 查看它怎么调用 VectorIndex
grep -n "vector_index\|VectorIndex\|\.search(" retrieval/pipeline.py
```

### Q3: 有没有现成的工厂方法创建 OpenGauss 版的 pipeline？

```bash
# 查看 service/api.py 或 providers/ 中有没有创建 read api 的工厂
grep -rn "def.*read_api\|def.*retrieval\|def.*pipeline" service/ providers/ --include="*.py"

# 查看 ProviderConfig 是否能自动选择 VectorIndex 后端
grep -rn "VECTOR_DB_TYPE\|vector_index\|OpenGauss" providers/config.py providers/ --include="*.py"
```

### Q4: 环境变量 VECTOR_DB_TYPE 在 search 链路中生效吗？

Phase 2 已经设了 `os.environ["VECTOR_DB_TYPE"] = "opengauss"`，但这只在插件进程中。需要确认：
- `ProviderConfig` 或工厂方法是否读这个环境变量？
- 读了之后是否会创建 OpenGaussVectorIndex 而不是 InMemoryVectorIndex？
- 创建的时候 OpenGauss 连接参数从哪来？（也是环境变量？）

## 预期修改

根据调查结果，大概率是以下两种情况之一：

### 情况 A：有 ProviderConfig 自动选择后端

如果 `providers/config.py` 已经实现了根据 `VECTOR_DB_TYPE` 创建对应 VectorIndex 的逻辑：

```python
# bridge/memory_api.py
def _get_read_api():
    # 确保环境变量设置正确
    os.environ.setdefault("VECTOR_DB_TYPE", "opengauss")
    # 确保 OG_HOST, OG_PORT 等也在环境变量中
    
    # 使用 ProviderConfig 自动选择后端
    config = ProviderConfig()
    vector_index = config.create_vector_index()  # 自动用 OpenGauss
    pipeline = RetrievalPipeline(vector_index=vector_index, ...)
    read_service = MemoryReadService(...)
    return MemoryReadAPI(pipeline, read_service)
```

### 情况 B：需要手动创建 OpenGaussVectorIndex

如果没有自动选择机制：

```python
# bridge/memory_api.py
def _get_read_api():
    from providers.vector_index import OpenGaussVectorIndex
    
    vector_index = OpenGaussVectorIndex(
        host=os.environ.get("OG_HOST", "localhost"),
        port=int(os.environ.get("OG_PORT", "8888")),
        ...
    )
    pipeline = RetrievalPipeline(vector_index=vector_index, ...)
    read_service = MemoryReadService(...)
    return MemoryReadAPI(pipeline, read_service)
```

### 两种情况都要注意

- `RetrievalPipeline` 的构造函数可能不只要 VectorIndex，还要 Embedder（把 query 向量化才能查）
- 如果 Embedder 也硬编码了 MockEmbedder，也要换成真实的（OpenAI Embedder）
- 确保 `OPENAI_API_KEY` 在插件进程中可用（search 时需要把 query embed）

## 执行计划

```
Phase 2.5（补充任务）：
  1. [INVESTIGATOR] 调查 Q1-Q4，搞清楚 search 链路的 VectorIndex 怎么来的
  2. [CODER] 根据调查结果修改 _get_read_api()，让 search 用 OpenGaussVectorIndex
  3. [RUNNER] 单元验证：
     - 确认 MemoryReadAPI 能正常实例化
     - 确认 search_memory 能连到 OpenGauss（需要 IndexService 已经写入了数据）
  4. [LEAD] 更新 PROGRESS.md，然后进 Phase 3

完成标准：
  - search_memory 使用 OpenGaussVectorIndex 查询
  - query 使用真实 Embedder 向量化（不是 MockEmbedder）
  - 单元验证通过
```
