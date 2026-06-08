# yuleOSH v0.3.0 Iteration 4 — 嵌入式基础

**Date:** 2026-06-08  
**Branch:** main  
**Previous commit:** 22b1035 (v0.3.0 Iteration 3)  
**Base test:** 225 passed, 0 skipped

---

## C-03 [P0] 交叉编译基础容器化

### Changes

#### 1. `Dockerfile.cross` (NEW)

Ubuntu 22.04 基础镜像，安装：
- `gcc-arm-none-eabi` (APT 官方源)
- `riscv64-unknown-elf-gcc` (APT 优先 → 下载预编译包 fallback)
- `build-essential`, `make`, `wget`
- 容器入口默认为 `make TARGET=all`
- 构建验证步骤打印所有工具链版本

**使用方式：**
```bash
docker build -t yuleosh-cross -f Dockerfile.cross .
docker run --rm -v $(pwd):/work yuleosh-cross make TARGET=arm
```

#### 2. `Makefile` (NEW)

项目根目录交叉编译 Makefile，支持：
- `make TARGET=arm` — ARM .elf 编译（ARM 为必选目标）
- `make TARGET=riscv` — RISC-V .elf 编译（可选，无工具链则提示跳过）
- `make TARGET=all` — 所有目标
- `make clean` — 清理 build/
- `make check-tools` — 验证工具链安装状态

ARM 编译使用 `-mcpu=cortex-m4 -mthumb -specs=nano.specs`  
RISC-V 编译使用 `-march=rv64imac -mabi=lp64`

#### 3. `src/cross/hello.c` (NEW)

最小交叉编译测试程序，打印架构信息（ARM / RISC-V / x86）。

#### 4. `src/ci/run.py` — cross_compile 阶段重写 (C-03)

**替换原有 info 占位**，现在实际执行交叉编译：

1. 检查 `src/cross/hello.c` 是否存在
2. **Attempt 1:** 直接运行 `make TARGET=arm`
   - 成功 → 检测 build/*.elf 文件 → 标记 passed
3. **Attempt 2:** make 不可用时尝试 Docker
   - `docker build -t yuleosh-cross -f Dockerfile.cross .`
   - `docker run ... make TARGET=arm`
4. **None:** make 和 Docker 都不可用 → 给出明确错误信息（不静默跳过）

---

## A-03 [P1] CI 层级依赖链

### Changes

#### `src/ci/run.py` — 层依赖链 (A-03)

新增结构：

```python
layer_dependencies: dict[int, list[int]] = {
    1: [],          # L1 无依赖
    2: [1],         # L2 依赖 L1
    3: [1, 2],      # L3 依赖 L1 + L2
}
```

新增函数：
- `get_latest_layer_result(layer, project_dir)` — 读取 `.osh/ci/layer{N}-{hash}.json` 最新结果
- `check_layer_dependency(target_layer, project_dir)` — 检查前置层是否通过
  - 无结果 → 返回阻断原因："Layer N has no recorded result"
  - 失败 → 返回阻断原因："Layer N status is 'failed' — layer M blocked"
  - 通过 → 返回 None
- `run_all(project_dir)` — 完整流水线：L1 → L2 → L3
  - 层前检查依赖
  - 失败时打印阻断原因并停止下游
  - 不执行不产生新文件

`main()` 扩展：新增 `python3 run.py all` 命令执行全流水线。

### 测试文件 `tests/test_ci_layers.py` (NEW, 18 tests)

覆盖验证：

| 测试 | 验证内容 |
|------|----------|
| `test_layer_dependencies_config` | 配置字典结构完整性 |
| `test_layer_dependencies_chain` | L1=[], L2=[1], L3=[1,2] |
| `test_layer_dependencies_no_cycles` | 无循环依赖 |
| `test_get_latest_layer_result_*` ×4 | 结果文件读写、缺失、最新 |
| `test_check_dependency_*` ×6 | 无依赖/未运行/失败/通过/两级 |
| `test_run_all_no_ci_dir` | 空项目运行 |
| `test_run_all_l1_failure_blocks_l2_l3` | L1 失败 → 阻断上下游 |
| `test_run_all_l2_failure_blocks_l3` | L2 失败 → 阻断 L3 |
| `test_l1_result_saved` | L1 结果文件保存 |
| `test_dependency_chain_integrity` | 完整约束检查 |
| `test_run_all_skips_blocked_layer` | run_all 阻断信息 |

---

## Test Results

```
243 passed in 11.08s
```

- 225 existing tests: all passed (unchanged)
- 18 new tests (test_ci_layers.py): all passed

---

## File Summary

| File | Status | Description |
|------|--------|-------------|
| `src/cross/hello.c` | NEW | 交叉编译测试源文件 |
| `Dockerfile.cross` | NEW | ARM + RISC-V 交叉编译容器 |
| `Makefile` | NEW | 交叉编译 Makefile（ARM/RISC-V/all） |
| `src/ci/run.py` | MODIFIED | C-03: cross_compile 实际执行; A-03: 层依赖链 |
| `tests/test_ci_layers.py` | NEW | 层依赖 18 个单元测试 |

---

## 验证

### 交叉编译验证（当前机器无 ARM 工具链）

```bash
$ make TARGET=arm
mkdir -p build
  ⏭️  ARM toolchain not found — install gcc-arm-none-eabi
make: *** [build/hello-arm.elf] Error 1

$ make clean && make check-tools
  ✅ Cleaned build artifacts
=== Toolchain check ===
arm-none-eabi-gcc: NOT FOUND
riscv64-unknown-elf-gcc: NOT FOUND (optional)
=== Done ===
```

**ARM .elf 生成需在安装了 gcc-arm-none-eabi 的环境或 Docker 中运行。**

```bash
# Docker 验证（推荐 CI 环境）
docker build -t yuleosh-cross -f Dockerfile.cross .
docker run --rm -v $(pwd):/work yuleosh-cross make TARGET=arm
```

### 层依赖链验证

```bash
# 完整流水线（从空白开始 → L1 尝试运行）
python3 src/ci/run.py all

# 单层运行
python3 src/ci/run.py 1
python3 src/ci/run.py 2
python3 src/ci/run.py 3
```
