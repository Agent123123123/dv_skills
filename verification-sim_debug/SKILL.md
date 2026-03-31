---
name: verification-sim_debug
description: "SIM 仿真 Debug 与覆盖率分析指引。当仿真失败、出现 UVM_FATAL/UVM_ERROR、编译报错无法定位、波形异常、EDA 工具链异常，或需要分析覆盖率（查看覆盖空洞、覆盖率不达标、需要 waiver、merge 覆盖率数据库）时，必须加载本 skill。触发条件：用户提及仿真失败、跑挂、UVM fatal/error、信号值异常、波形 debug、log 分析、filelist 报错、license 问题、段错误、coverage hole、覆盖率低、cov_reader、waiver——即使未明确说「debug」。"
---

# SIM 仿真 Debug 与覆盖率分析指引

## 概述

本 skill 是**独立的仿真 Debug 与覆盖率分析参考手册**，不属于验证工作流的任何一步。

当仿真编译或运行出错、或需要分析覆盖率时，随时加载本 skill 获取系统化的排查方法。

---

## Debug 快速定位流程

拿到失败用例后，按以下顺序排查：

```
Step 1：确认 compile 是否成功
  → grep -i "^Error|\*E|\*F" work/compile_<tc>.log
  → 有报错 → 进入「编译错误排查」
  → 无报错 → Step 2

Step 2：查看仿真 log 末尾
  → tail -80 work/sim_<tc>.log
  → 有 UVM_FATAL → 进入「UVM 致命错误」
  → 有 UVM_ERROR → 进入「UVM 一般错误」
  → 仿真超时 → 进入「超时 / 卡死」
  → 无明显错误 → 检查 PASSED/FAILED 关键字

Step 3：需要更细粒度信息
  → 提升 verbosity：+UVM_VERBOSITY=UVM_HIGH
  → 或开启波形 → 进入「波形 Debug」
```

---

## 编译错误排查

### 快速筛查

```bash
# 所有编译 Error/Fatal
grep -n "^Error\|^Fatal\|\*E\|\*F" work/compile_<tc>.log | head -30

# 查看 filelist 是否缺文件
grep "cannot open file\|No such file" work/compile_<tc>.log

# 展开 filelist 确认文件数量
vcs -full64 -sverilog -f tb/tb_filelist.f -dry_run 2>&1 | grep "^Compiling" | wc -l
```

若 filelist 存在层级嵌套或宏变量，加载 **`fexpand`** skill 展开 filelist。

### 常见编译错误

| 错误信息 | 原因 | 解决方法 |
|---------|------|---------|
| `cannot open file '...'` | filelist 路径错误 | 确认文件路径相对/绝对引用正确 |
| `Undefined variable 'xxx'` | `ifdef 宏未定义 | 添加 `+define+xxx` 编译选项 |
| `undeclared symbol 'uvm_pkg'` | UVM 库未正确 include | 确认 `$UVM_HOME/src/uvm_pkg.sv` 在 filelist 中且路径正确 |
| `package 'xxx_pkg' not found` | package 编译顺序错误 | 将 package 文件挪到 filelist 头部 |
| `Unexpected token` | SV 语法不被当前版本支持 | 参考 `eda-toolchain-debug` skill，检查工具版本 |

---

## UVM 致命错误（UVM_FATAL）

### 快速筛查

```bash
grep -n "UVM_FATAL" work/sim_<tc>.log
# 查看上下文（前后各 10 行）
grep -n "UVM_FATAL" work/sim_<tc>.log | while read line; do
  lineno=$(echo $line | cut -d: -f1)
  sed -n "$((lineno-5)),$((lineno+10))p" work/sim_<tc>.log
done
```

### UVM_FATAL 常见类型

| 报错标识 | 含义 | 排查步骤 |
|---------|------|---------|
| `[TPRSE]` | Test class 未找到或未注册 | 检查 `` `uvm_component_utils(<classname>) `` 宏；确认 `+UVM_TESTNAME=` 与类名完全一致（大小写） |
| `[NO_TLM]` | TLM port/export 未连接 | 检查 `env.connect_phase()` 中所有 `analysis_port.connect()` 调用 |
| `[CNTXT]` | `uvm_config_db::get()` 失败 | 确认 `tb_top` 的 `uvm_config_db::set()` 路径与 component 层级中 `get()` 的路径完全一致 |
| `[RNDSC]` | `randomize()` 约束无解 | 检查 constraint 块，是否存在矛盾约束；使用 `constraint_mode(0)` 临时禁用某约束排查 |
| `[PHSBER]` | Phase 执行异常 | 确认每个 `run_phase` 任务有正确的 `phase.raise_objection()` / `phase.drop_objection()` |

---

## UVM 一般错误（UVM_ERROR）

```bash
# 查看所有 UVM_ERROR 及其消息
grep -n "UVM_ERROR" work/sim_<tc>.log

# 仅查看 scoreboard / checker 相关报错
grep -n "UVM_ERROR.*score\|UVM_ERROR.*check\|UVM_ERROR.*mismatch" work/sim_<tc>.log -i
```

### 常见 UVM_ERROR 场景

| 场景 | 常见原因 | 排查方法 |
|------|---------|---------|
| Scoreboard 数据不匹配 | DUT 输出与 Ref Model 预测不符 | 对比 expected_queue 与 actual_queue 中的事务内容；使用 `convert2string()` 打印详情 |
| `$cast failed` | 类型转换失败，类层级不匹配 | 检查 seq_item 的继承关系；确认 `create()` 传入正确类型 |
| Monitor 未采样到事务 | 接口信号时序问题 | 开启波形，检查 monitor_cb 是否在正确时钟沿采样 |
| 预期事务未收到 | `finish_item()` 未完成或 Driver 卡住 | 检查 Driver 是否调用 `item_done()`；检查协议握手逻辑 |

---

## 超时 / 卡死

```bash
# 确认是否真的超时
grep -n "TIMOUT\|timeout\|Time limit" work/sim_<tc>.log -i

# 查看仿真在哪个时刻停止
tail -30 work/sim_<tc>.log
```

**排查步骤**：
1. 增大超时限制：在仿真命令中加 `+UVM_TIMEOUT=10000000,YES`
2. 若仿真并未超时而是无限运行：检查 `run_phase` 中 `objection` 是否正确 drop
3. 若仿真在某个时刻卡住：开启波形，查看卡住时刻的信号状态（是否等待握手信号）
4. 若 Driver 卡在 `finish_item()`：检查 Sequencer 是否正常运行，用 `+UVM_VERBOSITY=UVM_HIGH` 观察 seq 发包流程

---

## 波形 Debug

### 生成波形

```bash
# VCS：生成 VCD 波形
vcs ... +vcs+dumpvars+<dut>.vcd

# VCS：生成 FSDB（需 Verdi license）
vcs ... -debug_access+all -kdb
./work/simv_<tc> ... -verdi -ssf <dut>.fsdb

# VCS：生成 VPD
vcs ... +vcs+vpdfile+work/<dut>.vpd
```

### 使用 wave_reader 分析（无需 GUI）

若需命令行分析波形，加载 **`wave_reader`** skill：

```bash
# 示例：查看指定信号在特定时刻的值
wave_reader open work/<dut>.vcd
wave_reader signal <dut>.clk <dut>.valid <dut>.data
wave_reader at 1000ns
```

### 关键信号排查路径

```
1. 检查 clk/reset 是否正常（reset 释放时序）
2. 检查 valid/ready 握手是否完成
3. 检查 DUT 输入数据是否与激励一致
4. 检查 DUT 输出数据是否与预期一致
5. 若有流水线延迟，按拍数逐步追踪
```

---

## 覆盖率 Debug

### 编译加覆盖率 flag（VCS）

```bash
vcs -full64 -sverilog -timescale=1ns/1ps \
    -f filelist/rtl.f -f tb/tb_filelist.f \
    -top tb_top \
    -cm line+tgl+branch+cond+fsm \
    -cm_dir work/cov_merge.vdb \
    -cm_hier cm_hier.cfg \
    -o work/simv_cov \
    -l work/compile_cov.log
```

**cm_hier.cfg**（排除 TB 组件）：
```
-tree tb_top exclude
```

### 分析覆盖空洞

```bash
# 使用 cov_reader 分析，加载 cov_reader skill
cov_reader open work/cov_merge.vdb
cov_reader summary
cov_reader navigate <tb_top>.u_dut
```

### 覆盖率收敛循环

```
while 覆盖率 < 目标:
1. cov_reader drill-down 定位最高价值空洞
2. 编写 1-2 条针对性 TC
3. 重跑回归 → 检查覆盖率增量
4. 若 3 次尝试后无法覆盖 → 写 waiver（.cwv.yaml）
```

---

## EDA 工具链环境问题

遇到以下情况，立即加载 **`eda-toolchain-debug`** skill：

| 症状 | 说明 |
|------|------|
| `Failed to check out license` | License server 问题 |
| `ld: cannot find -l...` | 链接库缺失 |
| `Unsupported SystemVerilog construct` | 工具版本不支持该语法 |
| PLI/DPI/VPI 库加载失败 | 共享库路径或 ABI 问题 |
| Segmentation fault | 内存或库冲突 |

---

## 相关 skill 速查

| Skill | 触发场景 |
|-------|---------|
| `verification-sim_tool` | 编译命令、运行命令、Makefile 模板、testplan.yaml 模板 |
| `wave_reader` | 命令行分析 VCD/FST/FSDB 波形文件 |
| `cov_reader` | 覆盖率报告分析、空洞定位、waiver 管理 |
| `eda-toolchain-debug` | 工具链 license / 链接 / 版本兼容问题 |
| `fexpand` | 展开/调试层级 filelist（.f 文件） |
| `terminal_manager` | 长时间仿真的持久终端管理 |
