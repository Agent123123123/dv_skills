---
name: verification-sim_tool
description: "SIM 编译与仿真运行工具指引。当任何任务需要执行 SIM 编译（vcs/xrun/questa 编译命令）、运行仿真（单条 TC 或批量回归）、配置 filelist 或 Makefile 时，必须加载本 skill。触发条件：用户提及 compile、elaboration、simulation、run testcase、回归提交、filelist 配置——即使未明确说「SIM 工具」。Debug 相关场景请加载 verification-sim_debug skill。"
---

# SIM 编译 / 仿真运行工具指引

## 概述

本 skill 是**独立的仿真工具参考手册**，不属于验证工作流的任何一步。

提供编译命令、运行命令、回归管理、目录约定和 Makefile 模板，随时可查阅。

---

## 第一阶段：输入约束收集

| 信息项 | 获取方式 | 缺失时处理 |
|--------|---------|-----------|
| 仿真器类型（VCS / Xrun / Questa） | 询问用户，或自动探测（见下方流程） | 缺失则默认 **VCS** |

---

## 默认工具选择

当用户未指定时，使用以下默认值：

| 分类 | 默认值 |
|------|--------|
| 仿真器 | **VCS** |
| 回归管理 | **simforge** |
| 覆盖率分析 | **cov_reader** |
| 持久终端 | **terminal_manager** |

---

## 仿真器环境探测与加载

### 探测策略（按优先级依次尝试）

```
优先级 1: which <tool>          → 已在 PATH → 直接使用
优先级 2: module av <keyword>   → Environment Modules 发现 → module load 加载
优先级 3: $VCS_HOME / $XRUN_HOME → 手动环境变量 → 参考 eda-toolchain-debug
全部失败 → 报告 BLOCKED
```

### 完整探测流程

Agent 在需要仿真器时，**必须按以下步骤顺序执行**：

**Step 1：直接检测 PATH**

```bash
which vcs 2>/dev/null && echo "VCS_FOUND" || echo "VCS_NOT_FOUND"
which xrun 2>/dev/null && echo "XRUN_FOUND" || echo "XRUN_NOT_FOUND"
which vsim 2>/dev/null && echo "VSIM_FOUND" || echo "VSIM_NOT_FOUND"
```

若目标仿真器已在 PATH → 跳到对应仿真器章节，直接使用。

**Step 2：检测 module 命令可用性**

```bash
type module &>/dev/null && echo "MODULE_AVAILABLE" || echo "MODULE_NOT_AVAILABLE"
```

若 `module` 不可用 → 跳到 Step 3。

**Step 3：module av 探测可用仿真器**

使用 `module av` 列出所有可用的 EDA 工具版本：

```bash
# 探测 VCS
module av vcs 2>&1 | grep -iE 'vcs|synopsys'

# 探测 Xrun (Cadence Xcelium)
module av xcelium 2>&1 | grep -iE 'xcelium|xrun|cadence'
module av incisive 2>&1 | grep -iE 'incisive|cadence'   # 旧版 Cadence

# 探测 Questa (Siemens/Mentor)
module av questa 2>&1 | grep -iE 'questa|mentor|siemens'
module av modelsim 2>&1 | grep -iE 'modelsim|mentor'     # 旧版 Mentor
```

**Step 4：选择并加载最新版本**

从 `module av` 输出中提取版本列表，**按版本号排序取最新**：

```bash
# VCS 示例：提取 module 名，按版本排序，取最新
LATEST=$(module av vcs 2>&1 | grep -oP '\S*vcs\S*' | sort -V | tail -1)
if [[ -n "$LATEST" ]]; then
    module load "$LATEST"
    which vcs && vcs -id
else
    echo "BLOCKED: No VCS module found"
fi
```

```bash
# Xrun 示例
LATEST=$(module av xcelium 2>&1 | grep -oP '\S*xcelium\S*' | sort -V | tail -1)
if [[ -n "$LATEST" ]]; then
    module load "$LATEST"
    which xrun && xrun -version
else
    echo "BLOCKED: No Xcelium module found"
fi
```

```bash
# Questa 示例
LATEST=$(module av questa 2>&1 | grep -oP '\S*questa\S*' | sort -V | tail -1)
if [[ -n "$LATEST" ]]; then
    module load "$LATEST"
    which vsim && vsim -version
else
    echo "BLOCKED: No Questa module found"
fi
```

**Step 5：验证加载结果**

```bash
which <tool> || { echo "BLOCKED: <tool> still not found after module load"; }
```

### 注意事项

- `module av` 的输出格式因站点配置而异，grep 匹配模式需灵活适配
- 若 `module av` 返回多个仿真器可用（如 VCS 和 Xrun 均有），默认选 VCS，或询问用户
- `module load` 可能需要先 `module purge` 或 `module unload` 冲突模块
- 某些站点使用 `ml` 作为 `module load` 的别名（Lmod 环境）
- 加载后应通过 `which` + 版本命令双重确认

---

## 仿真器：VCS（默认）

### 环境检测

```bash
# 优先级 1: 直接检测
if which vcs &>/dev/null; then
    echo "VCS found: $(which vcs)"
    vcs -id
# 优先级 2: module load
elif type module &>/dev/null; then
    LATEST_VCS=$(module av vcs 2>&1 | grep -oP '\S*vcs\S*' | sort -V | tail -1)
    if [[ -n "$LATEST_VCS" ]]; then
        echo "Loading: $LATEST_VCS"
        module load "$LATEST_VCS"
        which vcs && vcs -id
    else
        echo "BLOCKED: No VCS module available"
    fi
else
    echo "BLOCKED: VCS not found and module system unavailable"
fi
```

若 VCS 未找到：**向用户报告 BLOCKED，不继续执行后续步骤。** 参考 `eda-toolchain-debug` skill 排查问题。

### 编译单条 TC

```bash
vcs -full64 -sverilog -timescale=1ns/1ps \
    +incdir+$UVM_HOME/src \
    $UVM_HOME/src/uvm_pkg.sv \
    -f filelist/rtl.f \
    -f tb/tb_filelist.f \
    -top tb_top \
    -Mdir=work/csrc_<tc_name> \
    -o work/simv_<tc_name> \
    -l work/compile_<tc_name>.log
echo "Exit: $?"
```

**编译成功条件**：exit code 为 0，且 log 中无 `Error` 行：

```bash
grep -i "^Error\|^Fatal\|\*E\|\*F" work/compile_<tc_name>.log | head -20
```

### 运行单条 TC

```bash
./work/simv_<tc_name> \
    +UVM_TESTNAME=<dut>_<scenario>_test \
    +UVM_VERBOSITY=UVM_MEDIUM \
    -l work/sim_<tc_name>.log
grep "UVM_FATAL\|UVM_ERROR\|PASSED\|FAILED" work/sim_<tc_name>.log | tail -20
```

### 冒烟测试流程

```
1. which vcs              → 不存在 → module av vcs → module load → 再次 which vcs
   若仍不存在             → 报告 BLOCKED
2. 编译 smoke TC          → 必须 Exit: 0
3. 运行 smoke TC          → log 末尾无 UVM_FATAL / UVM_ERROR
4. grep "PASSED"          → 确认测试通过
```

---

## 仿真器：Xrun（Cadence）

### 环境检测

```bash
# 优先级 1: 直接检测
if which xrun &>/dev/null; then
    echo "Xrun found: $(which xrun)"
    xrun -version
# 优先级 2: module load
elif type module &>/dev/null; then
    LATEST_XRUN=$(module av xcelium 2>&1 | grep -oP '\S*xcelium\S*' | sort -V | tail -1)
    if [[ -n "$LATEST_XRUN" ]]; then
        echo "Loading: $LATEST_XRUN"
        module load "$LATEST_XRUN"
        which xrun && xrun -version
    else
        echo "BLOCKED: No Xcelium module available"
    fi
else
    echo "BLOCKED: Xrun not found and module system unavailable"
fi
```

### 编译与仿真

```bash
xrun -64bit -sv -uvm \
    -timescale 1ns/1ps \
    -f filelist/rtl.f \
    -f tb/tb_filelist.f \
    -top tb_top \
    -xmlibdirpath work/xcel_<tc_name> \
    -log work/compile_<tc_name>.log \
    +UVM_TESTNAME=<dut>_<scenario>_test
```

Xrun 默认 compile + elaborate + simulate 一步完成。若需分步：
- 编译：`xrun ... -compile`
- 仿真：`xrun ... -run +UVM_TESTNAME=...`

---

## 仿真器：Questa（Mentor/Siemens）

### 环境检测

```bash
# 优先级 1: 直接检测
if which vsim &>/dev/null; then
    echo "Questa found: $(which vsim)"
    vsim -version
# 优先级 2: module load
elif type module &>/dev/null; then
    LATEST_QUESTA=$(module av questa 2>&1 | grep -oP '\S*questa\S*' | sort -V | tail -1)
    if [[ -n "$LATEST_QUESTA" ]]; then
        echo "Loading: $LATEST_QUESTA"
        module load "$LATEST_QUESTA"
        which vsim && vsim -version
    else
        echo "BLOCKED: No Questa module available"
    fi
else
    echo "BLOCKED: Questa not found and module system unavailable"
fi
```

### 编译与仿真

```bash
# 编译
vlog -sv -mfcu +incdir+$UVM_HOME/src $UVM_HOME/src/uvm_pkg.sv \
     -f filelist/rtl.f -f tb/tb_filelist.f \
     -work work/questa_lib

# 仿真
vsim -c work/questa_lib.tb_top \
     +UVM_TESTNAME=<dut>_<scenario>_test \
     -do "run -all; quit" \
     -l work/sim_<tc_name>.log
```

---

## 回归管理：simforge（默认）

> 若需完整 CLI 参考，加载 `simforge` skill。

### testplan.yaml 模板

```yaml
global:
  simulator: vcs
  top_module: tb_top
  compile_opts: >-
    -full64 -sverilog -timescale=1ns/1ps
    +incdir+$UVM_HOME/src
    $UVM_HOME/src/uvm_pkg.sv
    -f filelist/rtl.f
    -f tb/tb_filelist.f
  sim_opts: "+UVM_VERBOSITY=UVM_MEDIUM"
  timeout: 3600

cases:
  - name: <dut>_smoke_test
    tags: [smoke, p0]
    sim_opts: "+UVM_TESTNAME=<dut>_smoke_test"

  - name: <dut>_reset_test
    tags: [regress, p1]
    sim_opts: "+UVM_TESTNAME=<dut>_reset_test"
```

### 提交与监控

```bash
# 提交 smoke 回归
simforge submit --tags smoke -j 4

# 查看状态
simforge status

# 查看失败用例
simforge results --filter failed
```

### 全量回归

```bash
simforge submit --tags regress -j 16
simforge status --json
simforge results --filter failed --json
```

---

## Makefile 模板

```makefile
SIM       ?= vcs
WORK_DIR  ?= work
UVM_HOME  ?= $(VCS_HOME)/etc/uvm-1.2

.PHONY: compile_% run_% regression clean

compile_%:
	mkdir -p $(WORK_DIR)
	$(SIM) -full64 -sverilog -timescale=1ns/1ps \
	       +incdir+$(UVM_HOME)/src \
	       $(UVM_HOME)/src/uvm_pkg.sv \
	       -f filelist/rtl.f \
	       -f tb/tb_filelist.f \
	       -top tb_top \
	       -Mdir=$(WORK_DIR)/csrc_$* \
	       -o $(WORK_DIR)/simv_$* \
	       -l $(WORK_DIR)/compile_$*.log

run_%: compile_%
	$(WORK_DIR)/simv_$* \
	    +UVM_TESTNAME=$(DUT)_$*_test \
	    +UVM_VERBOSITY=UVM_MEDIUM \
	    -l $(WORK_DIR)/sim_$*.log

regression:
	simforge submit --tags regress -j 16

clean:
	rm -rf $(WORK_DIR)
```

**关键约定**：
- `SIM ?= vcs`：不硬编码仿真器名称
- 所有产物均在 `$(WORK_DIR)/` 下，Makefile 自我包含（不依赖 shell alias）

---

## 持久终端会话

编译或仿真时间较长时，使用 **`terminal_manager`** skill 创建持久 tmux 会话：

```bash
terminal_manager create sim_session
terminal_manager exec sim_session "make regression"
terminal_manager logs sim_session
```

---

## 相关 skill 速查

| Skill | 触发场景 |
|-------|---------|
| `verification-sim_debug` | 仿真失败 debug、UVM 报错分析、波形分析、覆盖率 debug |
| `simforge` | 提交回归、查看回归状态、跨批次 merge 覆盖率 |
| `wave_reader` | 分析 VCD/FST/FSDB 波形文件 |
| `cov_reader` | 查看覆盖率报告、定位覆盖空洞、管理 waiver |
| `eda-toolchain-debug` | VCS/Xrun/Questa 工具链环境问题 |
| `terminal_manager` | 长时间编译/仿真的持久终端管理 |
| `fexpand` | 展开/调试层级 filelist（.f 文件） |
