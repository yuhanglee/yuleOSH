# yuleOSH v0.3.0 — Iteration 5 Report

> **Iteration**: 5  
> **Date**: 2026-06-08  
> **Focus**: D-02 需求-测试双向追溯 + D-03 验收矩阵自动化  
> **Baseline after Iteration 4**: 243 passed, 0 skipped  
> **Current**: 256 passed, 0 skipped (+13 new tests)

---

## D-02 [P1] 需求-测试双向追溯

### 改动：`src/evidence/pack.py`

**新增方法 `_parse_covers_from_file()`**：
- 解析测试文件中的 `Covers:` 标记
- 支持两种格式：
  1. 模块 docstring 中的 `Covers:` 行（如 `test_llm_client.py`）
  2. `# Covers:` 行注释
- 返回解析出的关键词列表

**新增方法 `_collect_test_coverage()`**：
- 扫描 `tests/` 目录下的所有 `test_*.py`
- 对每个文件调用 `_parse_covers_from_file()`
- 返回 `{test_file: [covered_keywords]}` 字典
- 无 `tests/` 目录或无可解析文件时优雅降级

**新增方法 `_build_requirement_to_test_map()`**：
- 遍历每个 requirement 的 SHALL 语句
- 提取关键词（过滤 stop words: the, and, for, with 等）
- 与测试文件 Covers 关键词做模糊匹配
- 构建双向映射：`req_name → [test_files]` 和 `test_file → [req_names]`

**修改 `generate_traceability_matrix()`**：
- 新增 Test coverage mapping 区块：每个 requirement 显示匹配的测试文件
- 每个 SHALL 前有 ✅/❌ 标记是否被测试覆盖
- Summary 新增 `Requirements with test coverage` 和 `Uncovered SHALLs` 指标

**新增方法 `_check_traceability_completeness()`**：
- 检查每个 SHALL 是否至少被一个测试覆盖
- 返回 uncovered SHALLs 列表（含 req_name, shall, req_id）
- 在 evidence 生成过程中输出 ⚠️ 警告信息

### 测试（6个新测试）

| 测试 | 说明 |
|:----|:----|
| `test_collect_test_coverage_basic` | 基本 Covers 解析（docstring 格式） |
| `test_collect_test_coverage_comment_style` | Covers 行注释格式 |
| `test_collect_test_coverage_empty` | 无 Covers 文件返回空字典 |
| `test_collect_test_coverage_no_tests_dir` | 无 tests/ 目录优雅降级 |
| `test_build_requirement_to_test_map` | 双向 map 构建 |
| `test_traceability_matrix_includes_tests` | 追溯矩阵含测试信息 |
| `test_traceability_matrix_shows_uncovered` | 未覆盖显示 ❌ |
| `test_check_uncovered_shalls` | 未覆盖检测 |

---

## D-03 [P1] 验收矩阵自动化

### 改动：`src/evidence/pack.py`

**新增方法 `generate_acceptance_matrix()`**：
- 遍历所有 requirement 的每个 SHALL 语句
- 自动匹配 tests/ 下相关测试（基于 Covers 关键词）
- 生成 Markdown 表格格式：
  ```
  | Req ID | Requirement | SHALL | 验证方法 | 测试文件 | 状态 |
  ```
- 每个 SHALL 一行，独立标记 ✅/❌
- Summary 显示覆盖率统计 + PASS/FAIL 判定

**集成到生成流程**：
- `generate_evidence()` 中调用 `generate_acceptance_matrix()`
- 输出到 `acceptance-matrix.md`
- 包含在 compliance pack (ZIP) 中

### 测试（5个新测试）

| 测试 | 说明 |
|:----|:----|
| `test_acceptance_matrix_generation` | 验收矩阵基本结构 |
| `test_acceptance_matrix_covers_all_shalls` | 所有 SHALL 都在矩阵中 |
| `test_acceptance_matrix_no_tests` | 无测试覆盖时显示 ❌ |
| `test_acceptance_matrix_summary` | Summary 准确性 |
| `test_full_evidence_flow_with_traceability` | E2E：全部证据链 + 追溯 + 合规包 |

---

## 测试结果

```
============================= 256 passed in 11.12s =============================
```

- ✅ All original 243 tests pass (unchanged behavior)
- ✅ 13 new extended tests pass
- ✅ No regressions

---

## 文件改动清单

| 文件 | 改动 |
|:----|:-----|
| `src/evidence/pack.py` | +5 methods, ~180 lines |
| `tests/test_evidence_engine_extended.py` | 新建，13 tests |
| `sessions/it1/Iteration-5-report.md` | 本报告 |

---

## 核心架构

```
EvidenceCollector
├── collect_requirements()       ← 已有
├── _parse_covers_from_file()    ← 新增: 解析 Covers: 标记
├── _collect_test_coverage()     ← 新增: 扫描测试文件
├── _build_requirement_to_test_map()  ← 新增: 双向映射
├── _check_traceability_completeness()  ← 新增: 未覆盖检测
├── generate_traceability_matrix()  ← 增强: 含测试映射
├── generate_acceptance_matrix()  ← 新增: 验收矩阵
├── generate_requirement_coverage()  ← 已有
├── generate_code_coverage_report()  ← 已有
├── aggregate_review_logs()     ← 已有
└── pack_compliance_zip()       ← 增强: 含 acceptance-matrix.md
```
