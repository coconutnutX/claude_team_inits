# 00-REBASE-FIX — Phase 0：跑现有测试，暴露 rebase 问题

## 目标

主线 rebase 后可能引入了一些测试失败。在开始 search 集成之前，先跑通 `oG-Memory/tests/` 下的所有现有测试，发现问题、记录问题、等人类确认修复。

**这个 Phase 不要求你修 bug。** 只需要：跑测试 → 记录失败 → 写 [ASK-HUMAN] → 等人类。

---

## 任务清单

### T0.1 [INVESTIGATOR] 扫描测试目录结构

先了解有哪些测试：

```bash
cd /data/Workspace2/oG-Memory

# 查看 tests 目录结构
find tests/ -name "*.py" -type f | sort

# 看有没有 pytest 配置
cat pytest.ini 2>/dev/null || cat pyproject.toml 2>/dev/null | grep -A 10 "\[tool.pytest"

# 看有没有测试依赖
grep -i "test\|pytest" requirements*.txt 2>/dev/null
```

将发现的测试文件列表记录到 PROGRESS.md。

### T0.2 [INVESTIGATOR] 跑所有测试

```bash
cd /data/Workspace2/oG-Memory
conda activate py11

# 尝试用 pytest 跑（推荐）
python -m pytest tests/ -v --tb=short 2>&1 | tee /tmp/test-results.txt

# 如果没有 pytest，尝试逐个跑
# for f in tests/test_*.py; do echo "=== $f ===" && python "$f" 2>&1; done
```

**不要跳过任何测试。** 全部跑一遍，包括可能很慢的集成测试。

### T0.3 [INVESTIGATOR] 整理失败报告

对于每个失败的测试，记录以下信息到 PROGRESS.md：

```markdown
### [时间] [INVESTIGATOR] [FAILED] 测试失败报告

**失败测试汇总：X 个通过 / Y 个失败 / Z 个错误**

#### 失败 1：tests/test_xxx.py::test_yyy
- **错误类型**：ImportError / AssertionError / AttributeError / ...
- **错误信息**：完整的 traceback 最后几行
- **可能原因**：rebase 导致的接口变更 / 依赖缺失 / 配置问题 / ...

#### 失败 2：tests/test_xxx.py::test_zzz
- ...

[ASK-HUMAN] 以上 Y 个测试失败，请确认：
1. 哪些是 rebase 导致的问题，需要你修？
2. 哪些是预期内的（比如需要 LLM key、需要 AGFS 服务等），可以先跳过？
3. 修好后告诉我，我再重新跑验证。
```

### T0.4 [LEAD] 等待人类确认

在 PROGRESS.md 中写完 [ASK-HUMAN] 后，**停下来等人类回复**。

人类可能会：
- 自己修代码，然后让你重跑测试验证
- 告诉你哪些失败可以忽略（如需要外部服务的集成测试）
- 让你尝试修某个特定问题

### T0.5 [INVESTIGATOR] 验证修复（人类修完后）

人类确认修复后，重新跑测试：

```bash
cd /data/Workspace2/oG-Memory
python -m pytest tests/ -v --tb=short 2>&1 | tee /tmp/test-results-fixed.txt
```

对比修复前后的结果，记录到 PROGRESS.md。

---

## 完成标准

满足以下条件后在 PROGRESS.md 标记 Phase 0 为 ✅：

- [ ] 所有测试文件已跑过一遍
- [ ] 失败的测试已记录到 PROGRESS.md（含 traceback 和可能原因）
- [ ] 已写 [ASK-HUMAN] 等待人类确认
- [ ] 人类确认后，该修的已修，该跳过的已标记
- [ ] 最终测试结果已记录

---

## 注意事项

- **不要自己改 oG-Memory 的核心代码来修测试。** 这些可能是 rebase 引入的问题，需要人类判断正确的修复方式。
- 如果某个测试需要环境依赖（AGFS 服务、LLM key 等），在报告中说明，不算为"rebase 问题"。
- 如果 pytest 本身没装，先 `pip install pytest --break-system-packages` 或用 conda 装。
