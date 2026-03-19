# 共享知识库 — 所有角色必读

本文档包含所有角色在执行任务时需要了解的 OpenClaw 插件开发关键信息。这些信息来自 OpenClaw 官方文档，普通 AI agent 可能并不知道这些细节。

---

## 0. PROGRESS.md — 团队共享追踪板

`.claude-team/PROGRESS.md` 是全队共用的进度文件，**human 通过它掌握项目状态**。

**每个角色在工作时必须更新它**。规则写在各阶段文档开头。核心要点：

- 状态变更时更新顶部总览表
- 获得新的路径/版本信息时填入"关键路径信息"表格（后续阶段依赖这些）
- 需要讨论的事、需要 human 操作的事，都用 `[ASK-HUMAN]` 标签追加记录
- 记录追加到 `<!-- APPEND HERE -->` 标记之前

---

## 1. OpenClaw 插件系统速查

### 插件本质
- TypeScript 模块，由 jiti（即时编译器）在运行时加载
- **不需要构建步骤**，直接写 `.ts` 文件即可
- 运行在 Gateway 进程内（in-process），与核心代码共享信任边界

### 插件必需文件
1. **`openclaw.plugin.json`** — manifest，声明 id、configSchema 等元数据
2. **入口文件**（如 `index.ts`）— 导出 `register(api)` 函数或对象
3. **`package.json`** — 声明 `openclaw.extensions` 指向入口文件

### 最小 manifest
```json
{
  "id": "my-plugin",
  "configSchema": {
    "type": "object",
    "additionalProperties": false,
    "properties": {}
  }
}
```

### 最小入口文件
```typescript
export default function register(api: any) {
  api.registerTool({
    name: "my_tool",
    description: "Does a thing",
    parameters: { type: "object", properties: { input: { type: "string" } }, required: ["input"] },
    async execute(_id: string, params: any) {
      return { content: [{ type: "text", text: "done" }] };
    },
  });
}
```

### 最小 package.json
```json
{
  "name": "my-plugin",
  "version": "0.1.0",
  "type": "module",
  "openclaw": {
    "extensions": ["./index.ts"]
  }
}
```

---

## 2. 插件安装方式（开发阶段）

我们使用**链接模式**，这样修改源码后不需要重新安装：

```bash
openclaw plugins install -l /path/to/plugin-dir
```

如果这个命令不可用或出错，备选方案：

**方案 A**：手动链接到 extensions 目录
```bash
ln -s /path/to/plugin-dir ~/.openclaw/extensions/og-memory
```

**方案 B**：在配置中指定 load path
```json
{
  "plugins": {
    "load": {
      "paths": ["/path/to/plugin-dir"]
    }
  }
}
```

**重要**：修改插件代码后需要重启 Gateway（如果在 Gateway 模式下）。Local 模式（`openclaw agent --local`）每次运行都重新加载，不需要重启。

---

## 3. Memory 插件的特殊机制

OpenClaw 有一个 memory slot 系统：

- `plugins.slots.memory` 控制哪个插件占据 memory 位
- 默认是 `memory-core`（内置的 Markdown 文件 + 向量搜索）
- 还有 `memory-lancedb`（长期记忆，需要额外安装）

### MVP 策略：不占 memory slot

我们的插件 **不声明 `kind: "memory"`**，这意味着：
- 不替代内置 memory 系统
- 只作为一个普通 tool 插件
- 用户可以同时使用内置 memory 和我们的 og-memory tool
- 风险最低，不干扰现有功能

后续如果要成为正式的 memory 提供者：
```json
// openclaw.plugin.json 中添加
{ "kind": "memory" }

// 用户配置中切换
{ "plugins": { "slots": { "memory": "og-memory" } } }
```

### 内置 memory 的工作方式（供参考）

内置 memory 使用 Markdown 文件：
- `MEMORY.md` — 长期记忆
- `memory/YYYY-MM-DD.md` — 每日日志
- 提供 `memory_search`（语义搜索）和 `memory_get`（读取文件）两个 tool

我们的 tool 命名为 `og_memory_write`，避免与内置 tool 冲突。

---

## 4. Tool 注册规范

```typescript
api.registerTool({
  name: "snake_case_name",     // 必须 snake_case，不能与内置 tool 冲突
  description: "...",           // LLM 用这个来决定何时调用
  parameters: Type.Object({     // 使用 @sinclair/typebox（OpenClaw 自带）
    field: Type.String({ description: "..." }),
  }),
  async execute(_id, params) {
    // _id 是 tool call id
    // params 是已验证的参数对象
    return {
      content: [{ type: "text", text: "result" }],
    };
  },
});
```

也可以用 `{ optional: true }` 作为第二参数，使 tool 需要显式 allow 才能使用。我们的 MVP 不需要这个。

### Tool 参数用 TypeBox 或 JSON Schema

以下两种写法等价：

```typescript
// TypeBox（推荐，有类型推导）
import { Type } from "@sinclair/typebox";
parameters: Type.Object({
  content: Type.String(),
  tags: Type.Optional(Type.Array(Type.String())),
})

// 原始 JSON Schema
parameters: {
  type: "object",
  properties: {
    content: { type: "string" },
    tags: { type: "array", items: { type: "string" } },
  },
  required: ["content"],
}
```

---

## 5. 配置系统

插件配置位于 `plugins.entries.<id>.config`：

```json
{
  "plugins": {
    "entries": {
      "og-memory": {
        "enabled": true,
        "config": {
          "pythonPath": "/path/to/python",
          "projectRoot": "/path/to/oG-Memory"
        }
      }
    }
  }
}
```

在插件代码中读取：
```typescript
const config = api.config?.plugins?.entries?.["og-memory"]?.config ?? {};
const pythonPath = config.pythonPath || "python";
```

配置也可以通过 CLI 设置：
```bash
openclaw config set plugins.entries.og-memory.config.pythonPath "/path/to/python"
```

---

## 6. 常用 CLI 命令

```bash
# 插件管理
openclaw plugins list              # 列出所有插件
openclaw plugins info <id>         # 插件详情
openclaw plugins doctor            # 诊断
openclaw plugins install -l <path> # 开发模式安装（链接）
openclaw plugins enable <id>       # 启用
openclaw plugins disable <id>      # 禁用

# 测试
openclaw agent --local -m "消息" --session-id <id>  # local 模式运行 agent

# 配置
openclaw config set <key> <value>  # 设置配置
cat ~/.openclaw/openclaw.json      # 查看配置文件

# 诊断
openclaw doctor                    # 全局诊断
openclaw status                    # 状态
```

---

## 7. 项目特有信息

### oG-Memory 项目
- 仓库：`https://gitcode.com/coconutnutx/oG-Memory`
- 当前分支：`phase1_write`（已完成写入功能）
- 我们的开发分支：`feature/openclaw-plugin`
- 核心存储：agfs（Go 实现的图存储文件系统）

### agfs 关键点
- 需要 Go 编译
- 运行后提供文件系统级别的图存储
- 写入的数据应该能在对应路径找到
- **验证写入成功的方式就是检查 agfs 路径下是否有新数据**

### 用户环境
- Windows WSL (Ubuntu)
- conda 环境 `py11`（Python 3.11）
- 之前尝试过开发，可能有残留配置

---

## 8. 不要做的事情

1. **不要写复杂的 shell 脚本**（尤其是带大量环境变量的 setup.sh）
2. **不要启动额外的 server/daemon**（MVP 用 child_process 调用即可）
3. **不要修改内置 memory 系统**（我们是加法，不是替换）
4. **不要硬编码路径**（全部通过配置传入）
5. **不要安装不必要的 npm 包**（OpenClaw 的 jiti 已经处理 TypeScript）
6. **不要覆盖内置 tool 名称**（我们的 tool 以 `og_` 为前缀）
