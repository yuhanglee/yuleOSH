# yuleOSH v0.3.0 Iteration 1 — CI 硬校验

## 变更报告: A-01 + A-02

---

## A-01 [P0]: CI 阻断逻辑修复

### 修改文件
- `src/ci/run.py` — 重写全部 stage handler 的错误处理逻辑

### 新增功能

#### 1. `--strict` 模式 (CI_STRICT=1)
- 新增 `is_strict()` 函数，读取 `CI_STRICT` 环境变量
- 严格模式下缺失工具记录为 `"failed"` 而非 `"skipped"`
- 非严格模式（默认）：缺失工具依然返回 `False`（阻断），但记录为 `"skipped"` 并打印 skip 原因

#### 2. `MISRA_FAIL_FAST` 环境变量支持
- 新增 `is_misra_fail_fast()` 函数
- cppcheck / clang-tidy 返回非零退出码时，启用此变量会立即阻断管道
- 未启用时也会阻断（A-01 要求），但 stage 记录为 `"failed"` 而非 `"skipped"`

#### 3. 所有 stage handler 的阻断改造

| Handler | 原行为 | 新行为 |
|---------|--------|--------|
| `run_plan_lint` | issues → warning, return True | issues → `add_stage("failed")`, return **False** |
| `run_clang_tidy` | FileNotFoundError → return True | `FileNotFoundError` / `CalledProcessError` → return **False** |
| `run_coverage_check` | FileNotFoundError/Timeout/JSONDecodeError → return True | 全部 → return **False** |
| `run_layer2` cppcheck | FileNotFoundError → return True | `FileNotFoundError` / `TimeoutExpired` → return **False** |
| `run_layer3` pytest | FileNotFoundError/Timeout → all_passed unaffected | → **False** + all_passed=False |

### 关键变更模式
- **工具缺失**: 记录 stage + 打印 skip 原因 → 返回 `False` (阻断)
- **工具失败**: 记录 stage + 打印失败信息 → 返回 `False` (阻断)
- **工具正常**: 无变化 → 返回 `True`
- **计划内跳过** (HOOK_TYPE=commit, COVERAGE_RUN=1, 无 C 文件等): 返回 `True` (不阻断)

### 测试覆盖
新增 15 个测试到 `tests/test_ci_engine.py`，覆盖:
- ✅ 严格模式/非严格模式 helper 函数
- ✅ plan-lint 阻断测试 (issue → False, no issue → True, no tasks → True)
- ✅ clang-tidy 三种场景: 无 C 文件 (True) / 工具缺失 (False) / 工具正常 (True)
- ✅ coverage 阻断: hook skip (True) / 嵌套 skip (True) / 工具缺失 (False)
- ✅ unit-tests: 无测试文件 (True)
- ✅ Layer 1 整体失败/通过
- ✅ Layer 2 MISRA_FAIL_FAST
- ✅ Layer 3 不崩溃
- ✅ Strict 模式记录为 "failed"

---

## A-02 [P0]: Pipeline 调用失败硬错误

### 修改文件
- `src/pipeline/run.py` — 替换所有静默降级路径，增加异常表达力

### 新增功能

#### 1. `PipelineStepError` 自定义异常类
继承自 `RuntimeError`，在所有 step handler 中替代通用 `RuntimeError` 和空 `except: pass`
- LLM 调用失败 → 抛出 `PipelineStepError`，包含 spec 路径和错误详情
- 所有 step handler 的顶层捕获从通用 `Exception` 改为 `PipelineStepError` → 保持 propagate
- 在 `run_pipeline()` orchestrator 中，异常捕获后调用 `session.fail_step()` + `break`

#### 2. 静默降级路径修复

| 位置 | 原行为 | 新行为 |
|------|--------|--------|
| `PipelineSession._save()` store write | `except Exception: pass` | `except Exception: log.warning(...)` |
| `_parse_spec()` cache read | `except Exception: pass` | `except Exception: log.warning(...)` + 重新解析 |
| `_parse_spec()` cache write | `except Exception: pass` | `except Exception: log.warning(...)` |
| `_parse_requirements()` 整体失败 | `except Exception: pass` | `log.warning(...)` + 返回空列表 |
| `_parse_scenarios()` 整体失败 | `except Exception: pass` | `log.warning(...)` + 返回空列表 |
| `step_claude_test` pytest/gotest | `except: pass` 级联 | 替换为 `log.warning(...)` |

#### 3. JSON 解析错误增强
- 所有 `json.JSONDecodeError` 现在包含原始响应的**前 500 字符**预览
- 明确指示哪个步骤/调用产生了错误

#### 4. `step_hermes_review` 容错 JSON 解析 (`_try_parse_hermes_json`)

支持以下格式偏差:
- ✅ 裸 JSON (`{"status": "passed", ...}`)
- ✅ Markdown ` ```json ... ``` ` fence 包裹
- ✅ 普通 ` ``` ... ``` ` fence 包裹（无 json 标记）
- ✅ 前导/后续文本 + JSON 对象
- ✅ 多个代码块 → 自动选取第一个有效 JSON
- ✅ 花括号精确匹配提取（无 fence 时查找匹配的 { }）
- ✅ 完全无效输出 → 返回 `status: "retry"` + 原始输出完整保留在 `_raw_llm_output`

### 测试覆盖
新增文件 `tests/test_pipeline_errors.py`，包含 15 个测试:
- ✅ `PipelineStepError` 类型检查
- ✅ `_parse_requirements` / `_parse_scenarios` 警告日志
- ✅ Hermes JSON 解析: bare/markdown_fence/plain_fence/leading_text/multiple_fences/brace_extraction/completely_invalid/preserves_raw_on_fallback
- ✅ `step_hermes_review` 非 JSON 输入 → retry 状态
- ✅ LLM 超时 → pipeline 阻断 (failed)
- ✅ JSON 错误包含 raw output (500 chars)
- ✅ `step_super_analysis` LLM 失败 → `PipelineStepError`
- ✅ `_save` store 失败 → `log.warning`

### 测试结果
```
======================== 117 passed, 2 skipped in 5.70s ========================
```
(原始 79 passed + 新增 38 tests)
