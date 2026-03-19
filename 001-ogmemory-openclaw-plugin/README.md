# oG-Memory × OpenClaw Plugin — Claude Teams 项目启动包

## 这是什么

这是一组协作文档，用于指导一个 Claude Teams 团队将 oG-Memory（基于 agfs 的图记忆系统）开发为 OpenClaw 插件。

---

## 文件索引

| 文件 | 给谁看 | 内容 |
|------|--------|------|
| `PROGRESS.md` | **所有人** | 项目进度追踪板，所有角色共同维护，human 从这里看整体状况 |
| `SHARED-KNOWLEDGE.md` | **所有人** | OpenClaw 插件开发必读知识 |
| `00-LEAD-COORDINATION.md` | `@lead` | 项目总协调文档 |
| `00-ENV-SETUP.md` | `@env` | Phase 0：环境诊断与准备 |
| `01-PLUGIN-SCAFFOLD.md` | `@plugin-dev` | Phase 1：插件脚手架搭建 |
| `02-INTEGRATION-BRIDGE.md` | `@integrator` | Phase 2：写入 API 桥接 |
| `03-TESTING-VALIDATION.md` | `@tester` | Phase 3：端到端测试 |

---

## 快速启动

### 如果你是 Lead（@lead）

1. 读 `SHARED-KNOWLEDGE.md`
2. 读 `00-LEAD-COORDINATION.md`
3. 从 Phase 0 开始执行，每个阶段先更新 `PROGRESS.md` 再干活
4. 每个 Phase 完成后验收，再推进下一个

### 如果你是被分配任务的角色

1. 先读 `SHARED-KNOWLEDGE.md`
2. 查看 `PROGRESS.md` 中的"关键路径信息"获取你需要的路径和版本
3. 读你对应的 Phase 文档
4. 执行时持续更新 `PROGRESS.md`

---

## 项目目标（MVP）

**一句话**：让 OpenClaw agent 能通过 `og_memory_write` tool 将记忆写入 agfs 存储。

**验证标准**：执行 `openclaw agent --local -m "记住某件事"` 后，能在 agfs 存储路径找到对应数据。

---

## 技术架构（MVP）

```
用户消息 → OpenClaw Agent → og_memory_write tool → TypeScript execFile
    → Python bridge/write_memory.py → oG-Memory API → agfs 存储
```

---

## 阶段依赖

```
Phase 0 (环境) → Phase 1 (脚手架) → Phase 2 (集成) → Phase 3 (验证)
     @env          @plugin-dev        @integrator       @tester
```

严格顺序。前一阶段完成并验收后才能开始下一阶段。

# 规则

## 规则 1：记录我说的话

每当我在对话中给出信息、做出决定、或回答你的问题时，
你要在 PROGRESS.md 追加一条 [HUMAN-INPUT] 记录，概括我说了什么。
例如：

### [HUMAN-INPUT] — AGFS 安装方式

用户指示：不用 Docker，从源码编译。网络问题时用 GOPROXY=https://goproxy.cn,direct

## 规则 2：环境变更日志

创建 .claude-team/ENV-LOG.md，内容如下：

# 环境变更日志

> 每次对环境做出修改（安装软件、创建目录/文件、修改配置、设置环境变量等），
> 都必须在这里追加一条记录。格式：时间 + 做了什么 + 路径/命令。