# Phase 2: 集成桥接 — @integrator

## 你的角色

你是集成工程师。你的任务是将 Phase 1 中的 stub `writeMemory` 函数替换为真实的 oG-Memory 写入调用，让 `og_memory_write` tool 真正通过 agfs 存储记忆。

## ⚠️ PROGRESS.md 规则

在执行本阶段时，你必须维护 `.claude-team/PROGRESS.md`：

- **开始时**：更新总览表 Phase 2 状态为 🔄，追加 `[STARTED]` 记录
- **搞清楚 oG-Memory API 后**：填写"关键路径信息"中的"oG-Memory 写入 API 函数签名"和"需要的环境变量"
- **遇到问题时**：追加 `[NOTE]` 或 `[BLOCKED]` 记录
- **做出设计决策时**：追加 `[DECISION]` 记录说明为什么这样做
- **完成时**：更新总览表为 ✅，追加 `[DONE]` 记录

---

## 前置条件

- Phase 1 已完成，插件脚手架可被 OpenClaw 加载
- **先查 PROGRESS.md 的"关键路径信息"获取所有路径信息**

---

## 核心思路

**TypeScript → `child_process.execFile` → Python 脚本 → oG-Memory API → agfs**

这是最简方案，没有额外依赖，不需要起服务。

```
// 为什么不用 HTTP server？
// MVP 阶段用 local 模式，进程内调用更简单可靠
// TODO: Phase N - 如果需要 server 部署，可以改为 HTTP 调用
```

---

## 步骤

### Step 1: 理解 oG-Memory 写入接口

先阅读 oG-Memory 项目代码，找到写入记忆的核心 API：

```bash
cd <项目根目录>
find . -name "*.py" | head -30
grep -r "def write" --include="*.py" .
grep -r "agfs" --include="*.py" .
grep -r "memory" --include="*.py" . | grep -i "write\|store\|save"
```

搞清楚：
1. 写入函数的签名（参数、返回值）
2. 需要的初始化步骤
3. agfs 的启动/连接方式
4. 写入后数据存储在什么路径、什么格式

**搞清楚后立刻更新 PROGRESS.md 的"关键路径信息"表格。**

### Step 2: 创建 Python 桥接脚本

在 `openclaw-plugin/bridge/write_memory.py` 创建桥接：

```python
#!/usr/bin/env python3
"""
Bridge: called by OpenClaw plugin via child_process.
JSON on stdin → oG-Memory write → JSON on stdout.
"""
import sys
import json
import os

project_root = os.environ.get("OG_MEMORY_ROOT", "")
if project_root:
    sys.path.insert(0, project_root)

def main():
    try:
        params = json.loads(sys.stdin.read())
        content = params.get("content", "")
        tags = params.get("tags", [])

        if not content:
            raise ValueError("content is required")

        # ========================================
        # TODO: 替换为实际的 oG-Memory 写入调用
        # 根据 Step 1 的发现填入具体代码
        # ========================================

        output = {
            "ok": True,
            "message": "Memory written successfully",
        }
        print(json.dumps(output))

    except Exception as e:
        print(json.dumps({"ok": False, "error": str(e)}))
        sys.exit(1)

if __name__ == "__main__":
    main()
```

### Step 3: 修改 index.ts 中的 writeMemory

将 stub 替换为 Python 桥接调用：

```typescript
import { execFile } from "child_process";
import { join } from "path";

async function writeMemory(
  content: string,
  tags: string[] | undefined,
  api: any
): Promise<string> {
  const config = api.config?.plugins?.entries?.["og-memory"]?.config ?? {};
  const pythonPath = config.pythonPath || "python";
  const projectRoot = config.projectRoot || "";

  // 从 projectRoot 推导脚本路径（最稳妥的方式）
  const scriptPath = join(projectRoot, "openclaw-plugin", "bridge", "write_memory.py");

  const input = JSON.stringify({ content, tags: tags || [] });

  return new Promise((resolve, reject) => {
    const child = execFile(
      pythonPath,
      [scriptPath],
      {
        env: {
          ...process.env,
          OG_MEMORY_ROOT: projectRoot,
          // TODO: Phase N - 添加其他需要的环境变量
        },
        timeout: 30000,
        maxBuffer: 1024 * 1024,
      },
      (error, stdout, stderr) => {
        if (stderr) {
          api.logger?.warn?.(`[og-memory] Python stderr: ${stderr}`);
        }
        if (error) {
          reject(new Error(`Python bridge failed: ${error.message}`));
          return;
        }
        try {
          const result = JSON.parse(stdout.trim());
          if (result.ok) {
            resolve(result.message || "Written successfully");
          } else {
            reject(new Error(result.error || "Unknown write error"));
          }
        } catch (parseErr) {
          reject(new Error(`Failed to parse bridge output: ${stdout}`));
        }
      }
    );
    child.stdin?.write(input);
    child.stdin?.end();
  });
}
```

### Step 4: 配置 OpenClaw

```bash
openclaw config set plugins.entries.og-memory.config.pythonPath "<PROGRESS.md中记录的python路径>"
openclaw config set plugins.entries.og-memory.config.projectRoot "<PROGRESS.md中记录的项目路径>"
```

### Step 5: 手动验证

```bash
# 直接测试 Python 桥接
echo '{"content": "test memory", "tags": ["test"]}' | \
  OG_MEMORY_ROOT=<项目路径> \
  <python路径> <项目路径>/openclaw-plugin/bridge/write_memory.py

# 检查 agfs 存储路径
ls -lt <agfs存储路径>/ | head -10
```

---

## 完成动作

更新 PROGRESS.md，示例：

```markdown
### [DONE] @integrator — Phase 2 桥接集成完成

oG-Memory 写入 API: `memory_graph.write(content, tags)` 来自 src/writer.py
桥接方式: child_process.execFile → bridge/write_memory.py → oG-Memory
手动 Python 桥接测试: ✅ 数据写入 ~/.agfs/data/memories/
agfs 路径确认: ~/.agfs/data/memories/<timestamp>.json
```
