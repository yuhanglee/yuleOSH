---
kind: refactor
lint: RED / GREEN / REFACTOR
---

# v0.5.0 Sprint: Flash Abstraction Layer (FAL) + HIL Testing Framework

> 基于三位一体 | OpenSpec + Superpowers + Harness Engineering
> 规划: 小明 🧑‍💼(PM) → 小马 🐴(质量架构) → 小克 👨‍💻(架构/实现)

---

## 目标

在 v0.4.0 SIL 基础上，新增 Flash Abstraction Layer（统一烧录接口）和 HIL 硬件测试框架，覆盖 OpenOCD/JLink/pyOCD 三大后端，实现硬件测试全流程自动化。

### 验收标准

- [ ] 三个 Flash 后端均可通过统一接口调用
- [ ] 串口监视器支持物理端口 + 管道双模式
- [ ] HIL 测试运行器覆盖 flash→serial→assert 全生命周期
- [ ] 91+ 测试通过，覆盖率 ≥80%（cross/ 模块加权）
- [ ] CI L2.5 hardware-test stage 集成

---

## 迭代规划

### Iteration 1: Flash Abstraction Layer (FAL)
- `src/cross/flash.py` — FlashTool ABC + OpenOCDRunner + JLinkRunner + PyOCDRunner
- `src/cross/flash.py` — FlashRunner facade + auto-detect/fallback
- `FlashResult` / `FlashError` / `flash_firmware()` / `detect_hardware()`

### Iteration 2: 串口监视器
- `src/cross/serial_monitor.py` — SerialMonitor (pyserial) + PipeSerialMonitor
- expect / expect_all / read_until / assert_text_present / assert_text_absent
- 后台线程采集，线程安全 lock

### Iteration 3: HIL 测试框架
- `src/cross/hil_runner.py` — HilTestRunner flash→serial→assert 全生命周期
- HilTestResult 含 phase_timings
- 测试脚本引擎: expect / expect_re / assert / assert_not / wait / read_until
- 快捷方法: flash_and_expect / flash_and_boot / skip_flash_test / hil_test()

### Iteration 4: 测试 + 规范 + CI 集成（本迭代）
- `tests/test_flash.py` (49 tests)
- `tests/test_serial_monitor.py` (31 tests)
- `tests/test_hil_runner.py` (21 tests)
- `docs/spec.md` 更新 RS-009 / SWR-009.x
- `src/cross/__init__.py` 统一导出
- `docs/spec-delta.md` 变更追踪

---

## 技术设计

详见 `docs/hw-testing-architecture.md`

### 架构层级

```
FlashTool(ABC)                 SerialMonitor(ABC)
 ├─ OpenOCDRunner               ├─ SerialMonitor (pyserial)
 ├─ JLinkRunner                 └─ PipeSerialMonitor (pipe)
 └─ PyOCDRunner
         │
FlashRunner (facade + auto-detect + fallback)
         │
    load_target_config_safe() → YAML target config
         │
    HilTestRunner
         │
    flash → serial → assert 全生命周期
```

### YAML 目标配置

```yaml
# .yuleosh/targets/stm32f4.yaml
flash_openocd: "interface/stlink-v2.cfg target/stm32f4x.cfg"
flash_jlink: {device: "STM32F407VG", interface: "swd", speed: 4000}
flash_pyocd: {target: "stm32f407vg", frequency: 4000000}
```

---

## 产出物

| # | 产出物 | 说明 |
|:--|:-------|:-----|
| 1 | `src/cross/flash.py` | Flash Abstraction Layer + 3 个后端 + 自动检测 + fallback |
| 2 | `src/cross/serial_monitor.py` | 串口监视器（物理端口 + 管道） |
| 3 | `src/cross/hil_runner.py` | HIL 测试框架 + 脚本引擎 |
| 4 | `src/cross/__init__.py` | 统一模块导出 |
| 5 | `tests/test_flash.py` | 49 测试，覆盖所有 runner |
| 6 | `tests/test_serial_monitor.py` | 31 测试，覆盖双模式 |
| 7 | `tests/test_hil_runner.py` | 21 测试，覆盖全生命周期 |
| 8 | `docs/spec.md` RS-009 | FAL + HIL 需求规范 |
| 9 | `docs/hw-testing-architecture.md` | 技术方案文档 |
| 10 | `docs/spec-delta.md` | v0.5.0 变更追踪 |

---

## CI 流水线

### 新增 L2.5 阶段

```
当前: L1(unit+coverage) → L2(compilation+static+sil+integration) → L3(system)
新增: L2.5(hardware-test):
  1. 检测可用硬件 target
  2. 选择对应 flash 工具
  3. 烧录固件至目标板
  4. 执行 HIL 测试脚本
  5. 收集日志 → 报告 → 证据包
```

---

## 版本

- pyproject.toml: `0.3.0` → `0.5.0`（跳 0.4.x 因 SIL 层已独立迭代）
