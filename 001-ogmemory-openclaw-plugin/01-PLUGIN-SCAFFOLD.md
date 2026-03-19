# Phase 1: 插件脚手架 — @plugin-dev

## 你的角色

你是插件开发者。你的任务是在 oG-Memory 项目内创建一个符合 OpenClaw 规范的插件脚手架，注册一个 agent tool，让 OpenClaw 能够发现并加载这个插件。

## ⚠️ PROGRESS.md 规则

在执行本阶段时，你必须维护 `.claude-team/PROGRESS.md`：

- **开始时**：更新总览表 Phase 1 状态为 🔄，追加 `[STARTED]` 记录
- **插件安装成功时**：填写"关键路径信息"中的"插件安装方式"
- **遇到问题时**：追加 `[NOTE]` 或 `[BLOCKED]` 记录
- **需要 human 操作时**：追加 `[ASK-HUMAN]` 记录
- **完成时**：更新总览表为 ✅，追加 `[DONE]` 记录

---

## 前置条件

- Phase 0 已完成，环境正常
- 你已经在 `feature/openclaw-plugin` 分支上
- **先查 PROGRESS.md 的"关键路径信息"获取项目路径等信息**

---

## OpenClaw 插件核心概念（必读）

### 插件是什么
OpenClaw 插件是 TypeScript 模块，通过 jiti 在运行时加载。插件通过 `openclaw.plugin.json` 声明元数据，通过入口文件的 `register(api)` 函数注册运行时行为。

### Memory 类型插件的特殊性
OpenClaw 有一个 `plugins.slots.memory` 的排他槽位。默认是 `memory-core`。

**MVP 不占 memory slot**。我们注册为普通 non-capability 插件，只提供一个 agent tool。

```
// MVP: 注册为普通 tool 插件，不占 memory slot
// TODO: Phase N - 如果要替代内置 memory，需要改为 kind: "memory" 并配置 slots
```

### 插件发现
我们使用 `openclaw plugins install -l <path>` 以链接模式安装（开发推荐）。

---

## 目录结构

在 oG-Memory 项目根目录下创建：

```
oG-Memory/
├── ... (现有项目文件)
└── openclaw-plugin/
    ├── package.json
    ├── openclaw.plugin.json
    ├── index.ts
    └── tsconfig.json        # 可选，编辑器智能提示用
```

---

## 文件内容

### `openclaw-plugin/package.json`

```json
{
  "name": "og-memory",
  "version": "0.1.0",
  "description": "oG-Memory plugin for OpenClaw — graph memory backed by agfs",
  "type": "module",
  "openclaw": {
    "extensions": ["./index.ts"]
  },
  "dependencies": {}
}
```

### `openclaw-plugin/openclaw.plugin.json`

```json
{
  "id": "og-memory",
  "name": "oG-Memory",
  "description": "Graph memory write tool backed by agfs storage",
  "configSchema": {
    "type": "object",
    "additionalProperties": false,
    "properties": {
      "pythonPath": {
        "type": "string",
        "description": "Path to Python executable in conda py11 environment"
      },
      "projectRoot": {
        "type": "string",
        "description": "Path to oG-Memory project root"
      }
    }
  }
}
```

### `openclaw-plugin/index.ts`

```typescript
import { Type } from "@sinclair/typebox";

export default function register(api: any) {
  api.registerTool({
    name: "og_memory_write",
    description:
      "Write a memory entry to the oG-Memory graph storage (agfs-backed). " +
      "Use this when the user asks you to remember something, or when you " +
      "encounter important information that should be persisted.",
    parameters: Type.Object({
      content: Type.String({
        description: "The memory content to store",
      }),
      tags: Type.Optional(
        Type.Array(Type.String(), {
          description: "Optional tags for categorizing the memory",
        })
      ),
      // TODO: Phase N - Add metadata fields (source, confidence, relations)
    }),
    async execute(_id: string, params: { content: string; tags?: string[] }) {
      try {
        const result = await writeMemory(params.content, params.tags, api);
        return {
          content: [
            {
              type: "text" as const,
              text: `Memory stored successfully. ${result}`,
            },
          ],
        };
      } catch (err: any) {
        return {
          content: [
            {
              type: "text" as const,
              text: `Failed to store memory: ${err.message}`,
            },
          ],
        };
      }
    },
  });

  // TODO: Phase N - Register og_memory_search tool
  // TODO: Phase N - Register og_memory_get tool
  // TODO: Phase N - Consider registering as memory slot plugin (kind: "memory")

  api.logger?.info?.("[og-memory] Plugin registered successfully");
}

async function writeMemory(
  content: string,
  tags: string[] | undefined,
  api: any
): Promise<string> {
  // STUB: Will be replaced by @integrator in Phase 2
  const timestamp = new Date().toISOString();
  api.logger?.info?.(
    `[og-memory] Write request: content="${content.substring(0, 50)}..." tags=${JSON.stringify(tags)} time=${timestamp}`
  );
  return `[STUB] Would write to agfs at ${timestamp}`;
}
```

---

## 安装和验证

```bash
cd <oG-Memory项目根目录>/openclaw-plugin
openclaw plugins install -l .
```

如果 `-l` 不行，备选：
```bash
# 方案 A：手动符号链接
ln -s $(pwd) ~/.openclaw/extensions/og-memory

# 方案 B：配置 load path（编辑 ~/.openclaw/openclaw.json）
# "plugins": { "load": { "paths": ["<绝对路径>/openclaw-plugin"] } }
```

**无论用了哪种方式，都要记录到 PROGRESS.md 的"关键路径信息"表格里。**

验证：
```bash
openclaw plugins list
openclaw plugins info og-memory
openclaw plugins doctor
```

快速功能测试：
```bash
openclaw agent --local -m "Please remember that my favorite color is blue" --session-id test-og-memory
```

---

## 完成动作

更新 PROGRESS.md，示例：

```markdown
### [DONE] @plugin-dev — Phase 1 脚手架完成

插件已创建并通过链接模式安装。
- plugins list: ✅ 显示 og-memory enabled
- plugins doctor: ✅ 无错误
- agent test: ✅ agent 调用了 og_memory_write（stub 返回）
安装方式：openclaw plugins install -l ./openclaw-plugin
```
