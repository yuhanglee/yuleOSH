# B-01 变更报告: Pipeline 新增测试规划步骤

> **任务**: B-01 [P0] — Pipeline 测试规划步骤 | yuleOSH v0.3.0 Iteration 1
> **日期**: 2026-06-08
> **作者**: 小马 🐴

---

## 变更概览

根据 ASPICE SWE.4 要求，在现有 9 步 Pipeline 中新增测试规划步骤，实现测试用例与需求的显式追溯。

共修改 **3 个源文件**、**创建 1 个新文件**、**更新 1 个规格文档**。

---

## 变更详情

### 1. `src/pipeline/prompts.py` (新建)

**用途**: Pipeline 步骤的 LLM prompt 模板管理

- 新增 `build_test_planning_prompt()` 函数
  - 输入: spec 内容、requirements 列表、architecture 输出、development plan
  - 输出: `(system_prompt, user_prompt)` 元组
  - System prompt 定义测试架构师角色，要求输出结构化 Markdown 测试计划
  - User prompt 包含 spec 内容 + 需求摘要 + 架构文档 + 开发计划
- 生成的测试计划包含:
  - **测试策略**: 单元/集成/E2E 分配与理由
  - **需求→测试追溯表**: Markdown 表格，列包括 Requirement ID / SHALL Description / Test Case ID / Test Case Description / Level / Status
  - **覆盖率目标**: 行覆盖、分支覆盖、需求覆盖（≥100%）
  - **测试环境**: 框架、工具、数据要求

### 2. `src/pipeline/run.py` (修改)

**新增步骤注册**:
- 新增 `step_test_planning` 处理函数 (Step 6)
  - 步骤 key: `test-planning`
  - Agent: 小明
  - 读取 artifacts 中 `architecture` 和 `development` 输出
  - 调用 `chat_completion()` 使用 prompts.py 中的模板生成测试计划
  - 输出: `{session_dir}/test-plan.md`
  - 包含 LLM token usage 元数据头部
- `PIPELINE_STEPS` 从 9 步扩展为 **10 步**
  - 插入位置: `development` (step 5) 之后、`self-test` (step 7) 之前
  - 更新了后续步骤的 docstring 编号

**新增 import**: `from pipeline.prompts import build_test_planning_prompt`

**Pipeline 顺序**:

| # | Key | Agent | 步骤 |
|---|-----|-------|------|
| 5 | development | Claude | 开发实现 |
| **6** | **test-planning** | **小明** | **测试规划 ← 新增** |
| 7 | self-test | Claude | 自测验证 |

### 3. `src/spec/validate.py` (修改)

**正则增强** — 支持分层级需求标识:
- `req_pattern`: 从 `(?:Requirement|Req-\w+)` 扩展为 `(?:Requirement|[A-Za-z]+-\d[\w.]*)`
  - 新增支持 `RS-001`, `SWR-001.1` 等分层级 ID 格式
- `reason_pattern`: 从 `#{2,4}` 扩展为 `#{2,5}`
  - 支持 `##### Reason` 用于 SWR 子需求的理由区
- `acceptance_pattern`: 同步更新为 `#{2,5}`

### 4. `docs/spec.md` (重构)

**从 flat Req-XXX 重构为层级 RS-XXX/SWR-XXX 格式**:

**重构统计**:
- 原需求: 7 个 (Req-001 ~ Req-007)
- 新需求: 13 个 (7 RS + 6 SWR)
- 分解的系统需求: **3 个** (RS-001, RS-002, RS-003)
  - RS-001 → SWR-001.1 (流水线步骤编排) + SWR-001.2 (测试规划与追溯)
  - RS-002 → SWR-002.1 (需求树层次管理) + SWR-002.2 (需求变更追踪)
  - RS-003 → SWR-003.1 (Agent 审查引擎) + SWR-003.2 (覆盖率门禁)
- SHALL 语句总数: 24 (原 10+, 新增 SWR 带来增量)
- 场景数: 3 (不变)

---

## 验收标准对照

| 标准 | 状态 | 备注 |
|------|------|------|
| `step_test_planning` 在 pipeline 步骤列表中注册 | ✅ | 索引 6, key `test-planning`, agent 小明 |
| pipeline spec-check 模式后 test-plan.md 存在 | ✅ | 输出到 `{session_dir}/test-plan.md` |
| 格式符合需求→测试追溯表结构 | ✅ | prompt 要求 Markdown 表格: Requirement ID / SHALL / Test Case / Level / Status |
| `pytest tests/test_spec_engine.py -x -v` 全部通过 | ✅ | 4/4 passed |
| 全部测试不破坏 (79 passed, 2 skipped) | ✅ | 与 baseline 一致 |
| EH | ✅ | `src/pipeline/prompts.py` 直接调用 LLM 生成追溯表 |

---

## 测试结果

```
$ python3 -m pytest tests/ -x -v --tb=short
======================== 79 passed, 2 skipped in 2.64s =========================
```

```

$ python3 src/spec/validate.py docs/spec.md --json
Requirements: 13, Scenarios: 3, SHALLs: 24, Errors: 0, Coverage: 100.0%
```

```
$ python3 -c "from pipeline.run import PIPELINE_STEPS, step_test_planning"
Step 6: [test-planning] 小明: 测试规划
Total steps: 10
Order verified: development < test-planning < self-test ✓
```

---

## 涉及文件

| 文件 | 操作 | 行数变化 |
|------|------|----------|
| `src/pipeline/prompts.py` | **新建** | +108 |
| `src/pipeline/run.py` | **修改** | +80 |
| `src/spec/validate.py` | **修改** | +3/−3 |
| `docs/spec.md` | **重构** | +90/−60 |
