# oG-Memory OpenClaw Plugin - 开发日志

## 项目信息

- **开始时间**: 2026-03-19 19:40
- **完成时间**: 2026-03-19 20:45
- **总耗时**: ~1 小时
- **开发阶段**: Phase 0-3 全部完成

---

## 关键问题与解决方案

### 问题 1: AGFS 后端选择

**初始困惑**: 尝试 memfs、localfs、kvfs 多个后端，都遇到各种错误

**根本原因**:
- memfs: 返回 502 而不是 404
- localfs: 返回 500 而不是 404
- kvfs: 未正确挂载

**最终方案**: 使用 /local 后端
- 理由: 读写功能正常，只是状态码非标准
- 解决: AGFSContextFS 的 `_read_file` 已检查 "no such file" 消息，能正确处理

---

### 问题 2: 502 Bad Gateway 错误

**症状**:
```python
AGFSClientError: Bad Gateway - backend service unavailable
URL: http://localhost:8080/api/v1/files?path=/local/...
```

**诊断过程**:
1. 直接 curl AGFS → 返回 500 ✅
2. requests 库调用 → 返回 502 ❌
3. 检查环境变量 → 发现 http_proxy 设置

**根本原因**: requests 库使用代理，导致 500 被转换为 502

**解决方案**:
```python
# 在 write_memory.py 开头
os.environ.pop('http_proxy', None)
os.environ.pop('https_proxy', None)
```

```typescript
// 在 index.ts 中
env: {
  ...
  http_proxy: undefined,
  https_proxy: undefined,
}
```

---

### 问题 3: Python 桥接调用失败

**症状**:
```
[og-memory] Python exec failed: Command failed
code: 1 | stderr: (empty)
```

**诊断过程**:
1. 直接测试 Python 脚本 → 正常运行 ✅
2. 通过 OpenClaw 调用 → 失败 ❌
3. 增加日志 → 无 stderr 输出

**根本原因**: 缺少 OPENAI_API_KEY 环境变量

**解决方案**:
1. 在 `openclaw.plugin.json` 中添加 apiKey 字段
2. 从 Agent 模型配置读取 API key
3. 在 Python 进程环境中设置 OPENAI_API_KEY

```typescript
const apiKey = config.apiKey || "";
env: {
  OPENAI_API_KEY: apiKey || process.env.OPENAI_API_KEY,
}
```

---

### 问题 4: 工具不可见

**症状**: Agent 不调用 og_memory_write，只说"记住了"

**诊断**:
```bash
openclaw plugins info og-memory
# ✅ 工具已注册

openclaw config get tools.profile
# ❌ "coding" - 限制工具可见性
```

**解决方案**:
```bash
openclaw config set tools.profile full
```

---

## 技术债务与改进

### 当前限制

1. **硬编码 API Key**
   - 现状: 从模型配置读取
   - 改进: 支持动态配置或 secrets 管理

2. **固定存储路径**
   - 现状: `/tmp/agfs-data`
   - 改进: 可配置存储路径

3. **错误信息简化**
   - 现状: 只返回 error.message
   - 改进: 保留详细 trace 用于调试

4. **无重试机制**
   - 现状: 失败立即返回
   - 改进: 对 transient 错误重试

### 性能优化

1. **进程启动开销**
   - 每次 Python 调用都启动新进程
   - 改进: 使用持久化 Python 守护进程

2. **LLM 调用延迟**
   - 每次 write_memory 都调用 LLM
   - 改进: 缓存简单写入，批量处理

---

## 测试用例

### 成功用例

| # | 场景 | 输入 | 输出 |
|---|------|------|------|
| 1 | 英文偏好 | "I like coffee" | ✅ preference_xxx |
| 2 | 中文偏好 | "我喜欢茶" | ✅ preference_xxx |
| 3 | 标签 | tags: ["food", "drink"] | ✅ 标签保存 |
| 4 | 重复写入 | 相同内容两次 | ✅ 两次写入 (merge 未实现) |

### 边界情况

| # | 场景 | 行为 | 备注 |
|---|------|------|------|
| 1 | 空内容 | 拒绝 | ValueError |
| 2 | 超长内容 | 截断 | abstract 100字 |
| 3 | 特殊字符 | 正常 | JSON 转义 |
| 4 | 无 API Key | 失败 | 500 error |

---

## 配置清单

### 必需配置

```bash
# 1. Python 环境
openclaw config set plugins.entries.og-memory.config.pythonPath "/home/aaa/miniconda3/envs/py11/bin/python"

# 2. 项目路径
openclaw config set plugins.entries.og-memory.config.projectRoot "/data/Workspace2/oG-Memory"

# 3. API Key
openclaw config set plugins.entries.og-memory.config.apiKey "sk-..."

# 4. 工具配置
openclaw config set tools.profile full
```

### 可选配置

```bash
# 5. 存储路径 (未实现)
openclaw config set plugins.entries.og-memory.config.storagePath "~/.og-memory/data"

# 6. AGFS URL (代码硬编码)
# 目前固定为 http://localhost:8080
```

---

## 性能指标

| 指标 | 数值 | 备注 |
|------|------|------|
| 端到端延迟 | ~2-3秒 | 含 LLM 调用 |
| Python 启动 | ~0.5秒 | conda 环境初始化 |
| AGFS 写入 | ~0.1秒 | 本地文件系统 |
| LLM 调用 | ~1-2秒 | gpt-4o-mini |
| 文件大小 | ~1KB | 含元数据 |

---

## 经验教训

### 成功因素

1. **增量开发**: Phase 0→1→2→3 逐步验证
2. **详细日志**: 每个问题都有 trace 追踪
3. **直接测试**: Python 脚本可独立测试
4. **文档先行**: PROGRESS.md 和 ENV-LOG.md 及时更新

### 踩坑提醒

1. **代理陷阱**: 本地服务调用一定要禁用代理
2. **环境变量**: Python 子进程不会自动继承所有变量
3. **工具可见性**: OpenClaw 的 profile 系统会限制插件工具
4. **错误传播**: stderr 不会自动出现在主日志，需主动记录

### 改进建议

1. **自动化测试**: 添加端到端测试脚本
2. **配置验证**: 启动时检查所有必需配置
3. **健康检查**: 提供 `openclaw plugins diagnose` 命令
4. **文档完善**: 使用 example 而不是 理论说明

---

## 附录

### 相关命令

```bash
# 查看插件状态
openclaw plugins info og-memory

# 查看插件配置
openclaw config get plugins.entries.og-memory

# 测试工具调用
openclaw agent --agent main --local --message "test"

# 查看 AGFS 状态
curl http://localhost:8080/api/v1/health
curl http://localhost:8080/api/v1/plugins

# 查看写入的文件
find /tmp/agfs-data -name "*.meta.json"
ls -la /tmp/agfs-data/accounts/default/users/default/memories/
```

### 文件清单

```
openclaw-plugin/
├── README.md                   # 用户文档
├── DEVELOPMENT.md              # 本文档
├── package.json
├── openclaw.plugin.json
├── index.ts                    # 147 行
├── tsconfig.json
├── bridge/
│   └── write_memory.py         # 95 行
└── node_modules/@sinclair      # 符号链接
```

### 代码统计

| 语言 | 文件数 | 代码行数 | 备注 |
|------|--------|----------|------|
| TypeScript | 1 | ~147 | 插件入口 |
| Python | 1 | ~95 | 桥接脚本 |
| JSON | 2 | ~50 | 配置文件 |
| Markdown | 2 | ~500 | 文档 |
| **总计** | **6** | **~792** | - |

---

**项目状态**: ✅ 完成并可用
**最后更新**: 2026-03-19 20:45
