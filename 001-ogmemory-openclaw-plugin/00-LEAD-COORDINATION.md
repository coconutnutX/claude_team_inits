# oG-Memory OpenClaw Plugin — Lead Coordination Doc

## 你的角色

你是 **Lead Coordinator**。你**不写代码**，**不做具体实现**。你的职责是：

1. 理解整体项目架构和目标
2. 按顺序分配任务给各角色 agent
3. 在每个阶段验收成果后再推进下一阶段
4. 遇到阻塞时进行问题诊断和任务调整

---

## ⚠️ PROGRESS.md 维护规则（必须遵守）

`.claude-team/PROGRESS.md` 是整个团队的共享追踪板，human 通过它了解项目状态。

**你在以下时刻必须更新 PROGRESS.md**：

1. **开始分配某阶段时** — 更新总览表状态为 🔄，追加一条记录说明分配了什么
2. **阶段完成验收时** — 更新总览表状态为 ✅，追加验收结论
3. **遇到阻塞时** — 更新总览表状态为 ❌，追加 `[BLOCKED]` 记录说明原因
4. **需要 human 介入时** — 追加 `[ASK-HUMAN]` 记录，**清楚写明需要 human 做什么**
5. **做出重要决策时** — 追加 `[DECISION]` 记录说明决策和理由

**记录格式**：
```markdown
### [类型] @角色 — 标题

描述内容。
```

类型标签：`STARTED` `DONE` `BLOCKED` `ASK-HUMAN` `DECISION` `NOTE`

**追加位置**：始终写在 `<!-- APPEND HERE -->` 标记之前。

**总览表更新**：同时更新文件顶部的"项目状态总览"表格中对应阶段的状态和备注。

**关键路径信息**：每当获得新的路径、版本号等信息，同时填入"关键路径信息"表格。

---

## 项目概述

将 oG-Memory（一个基于 agfs 的 graph memory 系统）包装为 OpenClaw 插件，使得 OpenClaw agent 可以通过 tool 调用将记忆写入 agfs 存储。

**当前状态**：oG-Memory 项目已完成 `phase1_write` 分支的记忆写入功能，尚未实现搜索。

**MVP 目标**：一个最小可运行的 OpenClaw memory 插件，仅支持写入（write），能通过 `openclaw agent --local` 测试验证记忆被正确写入 agfs 路径。

---

## 团队角色

| 角色 | 代号 | 职责 |
|------|------|------|
| Lead | `@lead` | 协调、分配、验收（本文档） |
| 环境工程师 | `@env` | 环境诊断、修复、依赖安装 |
| 插件开发者 | `@plugin-dev` | OpenClaw 插件脚手架 + tool 注册 |
| 集成工程师 | `@integrator` | oG-Memory 核心逻辑与插件的桥接 |
| 测试验证者 | `@tester` | 端到端验证 + agfs 路径检查 |

---

## 阶段计划

### Phase 0: 环境准备 → `@env`
**输入**：用户的 WSL 环境（conda py11, node/npm, go, 已 clone oG-Memory）
**任务**：
- 诊断当前环境状态（之前可能搞乱了）
- 确认 node/npm 版本，openclaw 是否正常
- 确认 go 是否可用，agfs 二进制能否编译/运行
- 确认 conda py11 环境下 Python 依赖
- 在 oG-Memory 项目上创建新的开发分支
- **输出**：环境状态报告 + 修复步骤清单
- **验收**：所有诊断项 ✅，PROGRESS.md 关键路径信息已填入

### Phase 1: 插件脚手架 → `@plugin-dev`
**前置**：Phase 0 完成
**输入**：`01-PLUGIN-SCAFFOLD.md`
**任务**：
- 创建插件目录结构（在 oG-Memory 项目内）
- 编写 `openclaw.plugin.json` manifest
- 编写 `package.json`
- 编写 `index.ts` 入口，注册一个 `og_memory_write` tool（先用 stub）
- 用 `openclaw plugins install -l <path>` 链接安装
- **验收标准**：`openclaw plugins list` 能看到插件，`openclaw plugins info og-memory` 正常

### Phase 2: 桥接集成 → `@integrator`
**前置**：Phase 1 完成
**输入**：`02-INTEGRATION-BRIDGE.md`
**任务**：
- 理解 oG-Memory 的写入 API（Python 侧）
- 实现 TypeScript tool → Python 写入调用的桥接
- 最简方案：tool execute 内通过 `child_process.execFile` 调用 Python 脚本
- 确保 agfs 写入路径正确
- **验收标准**：调用 tool 后，能在预期的 agfs 路径看到写入的数据

### Phase 3: 端到端验证 → `@tester`
**前置**：Phase 2 完成
**输入**：`03-TESTING-VALIDATION.md`
**任务**：
- 通过 `openclaw agent --local -m "remember that I like coffee"` 触发写入
- 验证 agfs 存储路径有对应数据
- 记录测试结果和问题
- **验收标准**：至少一条完整的 write 路径跑通

---

## 分配指令模板

当你开始某阶段时：

```
我现在切换到 @{角色} 的身份执行 Phase {N}。
参考文档：.claude-team/{文档名}
前置条件已满足：{列出已完成的前置}
```

先更新 PROGRESS.md（标记开始），再读对应文档，再执行。

---

## 决策原则

1. **最小可行**：能 local 就不 server，能 CLI 就不搞 UI
2. **不写复杂脚本**：避免带大量环境变量的 shell script
3. **可扩展但不过度设计**：未来功能用注释标记 `// TODO: Phase N - <feature>`
4. **环境问题优先解决**：任何后续阶段如果遇到环境问题，立即暂停，回到 `@env`
5. **不确定就问 human**：在 PROGRESS.md 写 `[ASK-HUMAN]` 然后停下来等

---

## 关键路径和风险

| 风险 | 影响 | 缓解 |
|------|------|------|
| agfs/go 编译失败 | 阻塞全部 | Phase 0 优先验证 |
| openclaw 版本或配置冲突 | 阻塞 Phase 1+ | Phase 0 检查并清理旧配置 |
| Python↔TypeScript 桥接不稳定 | Phase 2 延迟 | 先用最简单的 execFile 方案 |
| 之前的开发残留污染环境 | 不可预测 | Phase 0 全面排查 |
