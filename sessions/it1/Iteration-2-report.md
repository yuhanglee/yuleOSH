# v0.3.0 Iteration 2 — V 左半侧规范化 · 完成报告

> 日期: 2026-06-08 | 提交: a03f0e3

---

## 完成任务

| ID | 任务 | 优先级 | 工时 | 状态 |
|:---|:-----|:------:|:----:|:----:|
| B-02 | 需求层级 ID 规范 | P0 | 1天 | ✅ |
| B-03 | 需求状态跟踪 | P0 | 0.5天 | ✅ |
| D-01 | spec-diff 影响分析 | P0 | 1.5天 | ✅ |

---

## 变更汇总

### `src/spec/validate.py` — 核心改造

**B-02 — 层级 ID:**
- `SpecRequirement` 新增字段：`req_id`, `level`, `parent`, `status`
- `to_dict()` 输出包含上述字段
- 新增 `HEADER_ID_PATTERN` → 解析 `### RS-001: xxx` 格式自动提取 ID
- 新增 `_parse_id()` → 将 `RS-001`/`SWR-001.1` 分解为 (prefix, major, minor)
- 新增 `_id_to_level()` → RS→SYS, SWR→SW, FEATURE→FEATURE
- 新增 `_id_to_parent()` → SWR-001.1 自动推导父 ID = RS-001
- 不支持层级 ID 的旧格式需求保持空值兼容

**B-03 — 状态跟踪:**
- 支持 `Status: PROPOSED | APPROVED | IMPLEMENTED | VERIFIED` 标记
- 默认状态 = PROPOSED
- 新增 `validate_status_transition()` — 严格线性迁移
  - 迁移链：`(None) -> PROPOSED -> APPROVED -> IMPLEMENTED -> VERIFIED`
  - 不允许跳级、回退、旁路
- `validate_spec()` 新增 `invalid_status` 错误检测

**D-01 — 影响分析:**
- `diff_specs()` 输出新增 `impact_analysis` 章节:
  - `affected_requirements` — 直接变更的需求列表
  - `affected_children` — 受父需求变更影响的子需求
  - `affected_scenarios` — 受影响的场景
  - `affected_architecture_components` — 受影响的架构组件
  - `recommended_actions` — 带明确建议的操作列表
- 新增 `status_changed` / `status_changed_count` 字段
- diff 输出支持 SHOULD/MAY 变更检测

### `src/spec/diff.py` — CLI 升级
- 可视化展示 `📌 IMPACT ANALYSIS` 章节
- 显示状态变更（🔀 Status）
- 展示推荐操作及其数量

---

## 测试结果

```
tests/ — 152 passed, 2 skipped, 0 failed ✅
```

| 测试文件 | 状态 |
|:---------|:----:|
| test_spec_engine.py (4) | ✅ |
| test_spec_engine_extended.py (9) | ✅ |
| test_spec_v03_it2.py (35) **新增** | ✅ |
| test_ci_engine.py | ✅ |
| test_pipeline_errors.py | ✅ |
| test_llm_client.py | ✅ |
| test_e2e.py | ✅ |
| test_review_engine.py | ✅ |
| test_evidence_engine.py | ✅ |
| test_store.py | ✅ |
| test_perf.py | ✅ |

### 新增测试覆盖

| 测试类 | 测试数 | 覆盖内容 |
|:-------|:------:|:---------|
| TestIDParsing | 11 | ID 解析、层级推导、大小写 |
| TestParseHierarchicalSpec | 4 | RS/SWR 头部解析、to_dict |
| TestStatusTracking | 8 | 默认状态、Status: 标记解析、有效/无效状态 |
| TestStatusTransitions | 8 | 全迁移链校验、跳级/回退/终态 |
| TestImpactAnalysis | 4 | impact 结构、新增/删除/状态变更 |

---

## 验收确认

### B-02 ✅
- [x] spec 解析后支持层级 ID （RS-XXX / SWR-XXX.Y）
- [x] 自动推导 level（SYS / SW / FEATURE）
- [x] 自动推导父需求关系
- [x] 旧格式兼容（无 ID 的需求保持空值）

### B-03 ✅
- [x] Status: 标记支持 PROPOSED → APPROVED → IMPLEMENTED → VERIFIED
- [x] `validate_status_transition()` 校验状态迁移
- [x] `validate_spec()` 标记无效状态

### D-01 ✅
- [x] `diff_specs` 输出包含 `impact_analysis`
- [x] 影响分析含需求/子需求/场景/架构组件/推荐操作
- [x] 变更 spec 后 diff 输出 actionable 建议

---

## 下一 Iteration 预告

Iteration 3 — **可测试性 I** (Day 5-7):

| 任务 | 工时 | 负责人 |
|:-----|:----:|:------:|
| C-01 pipeline 依赖注入重构 | 1天 | 小克 |
| C-01 pipeline 单元测试 | 1天 | 小克 |
| C-02 E2E 测试修复 | 1天 | 小克 |
