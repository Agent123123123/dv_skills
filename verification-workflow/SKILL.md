---
name: verification-workflow
description: "重要：用于任何芯片 IP/模块验证规划任务时必须加载。定义强制性十一步工作流（第 1～6 步串行 + 第 7/8/9 步可并行 + 第 10 步 Smoke 验证 + 第 11 步回归测试），将输入的 DUT Spec + RTL 推进为完整的、经编译验证与回归覆盖的验证交付物（规划文档 + 可运行 TB 代码 + 覆盖率报告）。若不加载本 skill，将跳过关键阶段，产生不完整的临时验证计划。触发条件：任何涉及为 Verilog/SystemVerilog IP、模块或 RTL 块进行规划/定义/准备/验证/策略制定的任务——尤其是从头开始时。"
---

# 验证工作流编排器

## 概述

本 skill 是验证规划流水线的**顶层编排器**。

它定义了一套**强制性工作流**，将输入的 DUT Spec + RTL 推进为一份完整的、经编译验证与回归覆盖的验证交付物。

工作流分为四个阶段：
- **串行阶段（第 1 → 6 步）**：每步严格依赖前一步输出，必须按顺序执行。
- **并行阶段（第 7 / 8 / 9 步）**：三者仅依赖第 4、5、6 步的输出，彼此之间无数据依赖，可同时执行。
- **Smoke 阶段（第 10 步）**：编译 + 冒烟运行，验证生成代码的可用性。
- **回归阶段（第 11 步）**：全量回归仿真 + 覆盖率收敛分析，验证所有 TC 通过并达成覆盖率目标。

```
┌──────────────────────────────────────────────┐
│         输入：DUT Spec + RTL 源代码           │
└──────────────────────┬───────────────────────┘
                       │
                       ▼
        ┌──────────────────────────────┐
        │  第 1 步                     │
        │  verification-target         │
        │  定义验证目标                │
        └──────────────┬───────────────┘
                       │ 输出：verification-targets.md
                       ▼
        ┌──────────────────────────────┐
        │  第 2 步                     │
        │  verification-testpoint      │
        │  分解测试点                  │
        └──────────────┬───────────────┘
                       │ 输出：verification-testpoints.md
                       ▼
        ┌──────────────────────────────┐
        │  第 3 步                     │
        │  verification-strategy       │
        │  规划验证策略                │
        └──────────────┬───────────────┘
                       │ 输出：verification-strategy.md
                       ▼
        ┌──────────────────────────────┐
        │  第 4 步                     │
        │  verification-sim_tc_define  │
        │  定义 SIM 三元组             │
        └──────────────┬───────────────┘
                       │ 输出：verification-sim-tc-defines.md
                       ▼
        ┌──────────────────────────────┐
        │  第 5 步                     │
        │  verification-sim_tb_arch    │
        │  设计 UVM TB 架构            │
        └──────────────┬───────────────┘
                       │ 输出：verification-tb-arch.md
                       ▼
   ┌────────────────────────────────────────┐
   │  【Coding Style 统一检查】             │
   │  扫描已加载的 skill 列表，             │
   │  若有 UVM/TB coding style skill，      │
   │  立即加载。                            │
   │  后续第 6～9 步均共用此结论。          │
   └───────────────────┬────────────────────┘
                       │
                       ▼
        ┌──────────────────────────────┐
        │  第 6 步                     │
        │  verification-sim_tb_writer  │
        │  实现 TB 结构骨架            │
        └──────────────┬───────────────┘
                       │ 输出：tb/ 骨架代码 + tb_filelist.f
                       │
          ┌────────────┼────────────┐
          │            │            │
          ▼            ▼            ▼
   ┌────────────┐┌────────────┐┌────────────┐
   │  第 7 步   ││  第 8 步   ││  第 9 步   │
   │  stim_     ││  fcov_     ││  checker_  │
   │  writer    ││  writer    ││  writer    │
   │  激励序列  ││  功能覆盖  ││  检查器    │
   └─────┬──────┘└─────┬──────┘└─────┬──────┘
         │             │             │
         │  sequences  │  func_cov   │  sva+scoreboard
         └─────────────┼─────────────┘
                       │
                       ▼
        ┌──────────────────────────────┐
        │  第 10 步                    │
        │  Smoke 验证                  │
        │  加载 sim_tool + sim_debug   │
        │                              │
        │  10a. 编译检查 → 失败→fix循环│
        │  10b. 冒烟运行 → 失败→fix循环│
        └──────────────┬───────────────┘
                       │
                       ▼
        ┌──────────────────────────────┐
        │  第 11 步                    │
        │  回归测试                    │
        │  加载 sim_tool + simforge    │
        │  + cov_reader                │
        │                              │
        │  11a. 全量回归仿真           │
        │  11b. 覆盖率收集与分析       │
        │  11c. 覆盖率收敛判定         │
        └──────────────┬───────────────┘
                       │
                       ▼
┌──────────────────────────────────────────────┐
│  输出：完整验证交付物（经回归验证）           │
│  （5 份规划文档 + TB 代码 + 覆盖率报告）      │
└──────────────────────────────────────────────┘
```

---

## 强制执行规则

1. **串行阶段（第 1 → 6 步）严格按顺序执行。** 不允许跳步。
2. **并行阶段（第 7 / 8 / 9 步）可同时执行。** 三者仅依赖第 4 步（三元组）和第 5 步（TB 架构）的输出以及第 6 步（骨架代码）中的命名约定，彼此之间无数据依赖。
3. **串行阶段中，每步必须完成并经用户确认后，才能开始下一步。** 并行阶段的确认方式取决于执行模式（见规则 4）。
4. **执行模式选择**：进入并行阶段前，询问用户选择执行模式：
   - **快速模式**（推荐）：连续执行第 7 / 8 / 9 步，全部完成后统一确认。
   - **逐步模式**：按 7 → 8 → 9 顺序逐步执行，每步单独确认（适合首次使用或需要精细调整时）。
5. **执行每步之前，必须先加载该步的子 skill。** 将读取子 skill 的 SKILL.md 作为该步的第一个操作。
6. **输出向下传递。** 每步的输出文档是下一步的输入，需在步骤间明确说明这一传递关系。
7. **第 5 步完成后，Coding Style 统一检查。** 在进入第 6 步之前，执行一次 coding style skill 扫描，后续第 6～9 步共用此结论，不再重复扫描。
8. **第 10 步 Smoke 在并行阶段确认通过后自动执行。** 不单独询问是否执行。Smoke 失败时自动进入 debug → fix → retry 循环（最多 3 轮），3 轮后仍失败则输出诊断报告交用户处理。9. **第 11 步回归测试在 Smoke 通过后自动进入。** 包含全量 TC 回归 + 覆盖率收集与分析。若回归中有 TC 失败或覆盖率未达标，输出详细报告交用户处理。若 Smoke 状态为 SKIPPED 或 FAILED，第 11 步同样跳过。
---

## 回退规则

当某步发现问题需要修改前序输出时，遵循以下规则：

| 发现问题的步骤 | 允许回退到 | 需要级联更新的文档 |
|--------------|----------|-----------------|
| 第 2 步（测试点） | 第 1 步（目标） | verification-targets.md → verification-testpoints.md |
| 第 3 步（策略） | 第 2 步（测试点） | verification-testpoints.md → verification-strategy.md |
| 第 4 步（三元组） | 第 3 步（策略）或第 2 步 | 对应文档 + 所有后续文档 |
| 第 6~9 步（编码） | 第 4 步（三元组）或第 5 步（架构） | 修改后须重新执行受影响的编码步骤 |
| 第 10 步（Smoke） | 第 6～9 步（查看编译/运行报错定位到哪个组件） | 修复后重新执行第 10 步（无需重复 7/8/9 确认流程） |
| 第 11 步（回归） | 第 7～9 步（失败 TC 对应的 sequence/checker） | 修复后重新执行第 10 步 Smoke + 第 11 步回归 |

回退时，告知用户："需要回退到第 X 步修改 `<文档名>`，修改完成后将从第 X+1 步重新继续。"

---

## 预检：收集 DUT 输入信息

开始第 1 步之前，收集：

- DUT 名称 / 描述（询问用户）
- Spec 文档路径（若未提供则询问）
- RTL 顶层文件路径（若未提供则询问）
- RTL filelist 路径（编译 DUT 所需的 `.f` 文件；若未提供则询问，第 10 步 Smoke 编译必需）
- 已有约束（时间节点、排除的子模块、已知的 Golden IP）

若 Spec 或 RTL 不可用，告知用户：
> "需要 DUT Spec 和至少一个 RTL 顶层文件才能继续。请提供这些文件。"

若 RTL filelist 不可用，警告用户：
> "未提供 RTL filelist（.f 文件）。第 1～9 步可正常执行，但第 10 步 Smoke 编译将无法自动执行。建议尽早提供。"

### 全局上下文传递

预检阶段收集的信息构成**全局上下文**，在后续所有步骤中自动继承，子 skill **不得重复询问**。同时，每步完成后产生的输出文档也自动纳入全局上下文。

全局上下文包含以下变量（随流水线推进逐步积累）：

| 变量 | 来源 | 可用范围 |
|------|------|---------|
| DUT 名称 | 预检 | 第 1～10 步 |
| Spec 文档路径 / 内容 | 预检 | 第 1～10 步 |
| RTL 顶层文件路径 / 内容 | 预检 | 第 1～10 步 |
| RTL filelist 路径 | 预检 | 第 10 步 |
| DUT 参数配置 | 第 1 步确认 | 第 2～10 步 |
| 已知排除的子模块 / Golden IP | 预检 | 第 1～10 步 |
| Coding Style 结论 | 第 5 步后检查 | 第 6～9 步 |
| `verification-targets.md` | 第 1 步输出 | 第 2～10 步 |
| `verification-testpoints.md` | 第 2 步输出 | 第 3～10 步 |
| `verification-strategy.md` | 第 3 步输出 | 第 4～10 步 |
| `verification-sim-tc-defines.md` | 第 4 步输出 | 第 5～10 步 |
| `verification-tb-arch.md` | 第 5 步输出 | 第 6～10 步 |

> **执行约束**：加载子 skill 后，若其"输入约束收集"段要求收集的信息已在全局上下文中，直接使用已有值，不向用户重复询问。仅对全局上下文中**不存在**的、该步骤特有的信息项向用户询问。

---

## 步骤执行流程

### 第 1 步：定义验证目标

**加载 skill**：读取 `../verification-target/SKILL.md`
**执行**：完整遵循该 skill 中定义的流程。
**退出条件**：`verification-targets.md` 已保存，用户已确认其内容。

---

### 第 2 步：分解测试点

**加载 skill**：读取 `../verification-testpoint/SKILL.md`
**输入**：`verification-targets.md`（DUT 参数配置 + 验证目标列表）
**执行**：完整遵循该 skill 中定义的流程。
**退出条件**：`verification-testpoints.md` 已保存，用户已确认其内容。

---

### 第 3 步：规划验证策略

**加载 skill**：读取 `../verification-strategy/SKILL.md`
**输入**：`verification-targets.md` + `verification-testpoints.md`
**执行**：完整遵循该 skill 中定义的流程。
**退出条件**：`verification-strategy.md` 已保存，用户已确认其内容（含覆盖率收敛目标）。

> **FORMAL 测试点处理**：若第 3 步分配了 FORMAL 手段的测试点，这些测试点将由独立的 FPV（Formal Property Verification）流程处理，不进入第 4～9 步的 SIM 实现路径。提示用户："共有 `<n>` 条测试点分配给 FORMAL，建议为其单独创建 FPV 工程（`assert property` / `assume property` / `cover property`）。"

---

### 第 4 步：定义 SIM 测试用例三元组

**加载 skill**：读取 `../verification-sim_tc_define/SKILL.md`
**输入**：`verification-strategy.md`（筛选手段为 SIM 的测试点）+ DUT Spec + RTL
**执行**：完整遵循该 skill 中定义的流程。
**退出条件**：`verification-sim-tc-defines.md` 已保存，用户已确认其内容。

---

### 第 5 步：设计 UVM Testbench 架构

**加载 skill**：读取 `../verification-sim_tb_arch/SKILL.md`
**输入**：`verification-sim-tc-defines.md` + DUT Spec + RTL 顶层文件 + `verification-targets.md`
**执行**：完整遵循该 skill 中定义的流程。
**退出条件**：`verification-tb-arch.md` 已保存，用户已确认其内容。

---

### 【Coding Style 统一检查】（第 5 步后，第 6 步前执行）

扫描已加载的 skill 列表，判断是否存在 UVM / TB coding style 相关 skill（名称含 `uvm`、`tb`、`coding-style`、`rtl-coding-style` 等关键字）。

- **若存在**：立即加载，并告知用户："已加载 coding style skill，后续所有代码生成步骤将遵循其规范。"
- **若不存在**：告知用户："未检测到 TB coding style skill，第 6～9 步将采用内置默认 UVM 编码规范。"

此后第 6～9 步的子 skill **不再重复扫描 coding style**，使用本步骤的结论。

---

### 第 6 步：实现 TB 结构性骨架

**加载 skill**：读取 `../verification-sim_tb_writer/SKILL.md`
**输入**：`verification-tb-arch.md` + `verification-sim-tc-defines.md`
**执行**：完整遵循该 skill 中定义的流程，实现接口、Agent 组件、Env 连接层、Tests、tb_top。`tb_filelist.f` 中已根据 TC 定义预留第 7～9 步将生成文件的路径占位。
**退出条件**：`tb/` 目录下所有结构性骨架文件已生成，`tb_filelist.f` 已创建（含占位路径），用户已确认交付物。

---

### 并行阶段：第 7 / 8 / 9 步（激励 + 覆盖率 + 检查器）

> **数据依赖分析**：第 7 / 8 / 9 步的输入均来自第 4 步（三元组的 Stim / Coverage / Checker 字段）和第 5 步（TB 架构中的 Agent / Monitor / Scoreboard 分配），以及第 6 步建立的命名约定和 `tb_filelist.f` 占位路径。三步之间**无交叉依赖**——激励序列不需要覆盖率代码，覆盖率不需要检查器代码，检查器不需要激励序列。

**进入并行阶段前，询问用户**：
> "第 7（激励序列）、8（功能覆盖）、9（检查器）步之间无数据依赖，可以连续执行。请选择执行模式：
> - **快速模式**（推荐）：连续执行三步，全部完成后统一确认
> - **逐步模式**：按 7 → 8 → 9 顺序逐步执行，每步单独确认"

---

### 第 7 步：编写激励序列

**加载 skill**：读取 `../verification-sim_stim_writer/SKILL.md`
**输入**：`verification-sim-tc-defines.md`（Stim 字段）+ `verification-tb-arch.md`（Agent/Sequencer 分配）
**执行**：完整遵循该 skill 中定义的流程。生成的文件路径须与第 6 步 `tb_filelist.f` 中的占位路径一致。
**退出条件**：`tb/sequences/` 目录下所有 TC 的 sequence 文件已生成。
**确认**：快速模式下延迟至三步全部完成后统一确认；逐步模式下立即确认。

---

### 第 8 步：编写功能覆盖率

**加载 skill**：读取 `../verification-sim_fcov_writer/SKILL.md`
**输入**：`verification-sim-tc-defines.md`（Coverage 字段）+ `verification-tb-arch.md`（Monitor/Subscriber 分配）
**执行**：完整遵循该 skill 中定义的流程。生成的文件路径须与第 6 步 `tb_filelist.f` 中的占位路径一致。
**退出条件**：`tb/env/<dut>_func_cov.sv` 已生成，所有 TC 的 covergroup 定义完备。
**确认**：同上。

---

### 第 9 步：编写检查器

**加载 skill**：读取 `../verification-sim_checker_writer/SKILL.md`
**输入**：`verification-sim-tc-defines.md`（Checker 字段）+ `verification-tb-arch.md`（SVA 位置、Scoreboard 规划）
**执行**：完整遵循该 skill 中定义的流程。生成的文件路径须与第 6 步 `tb_filelist.f` 中的占位路径一致。
**退出条件**：`tb/sva/<dut>_properties.sv` 和（若有）`tb/env/<dut>_scoreboard.sv` 已生成。
**确认**：同上。

---

### 并行阶段统一确认（快速模式）

三步全部完成后，向用户展示汇总：

```
并行阶段交付摘要
────────────────────────────────────────
第 7 步（激励序列）：tb/sequences/ — <N> 个 sequence 文件
第 8 步（功能覆盖）：tb/env/<dut>_func_cov.sv — <N> 个 covergroup
第 9 步（检查器）  ：tb/sva/<dut>_properties.sv — <N> 条 SVA
                     tb/env/<dut>_scoreboard.sv — <有/无>
────────────────────────────────────────
```

询问："以上三步的输出是否符合预期？如需调整，请指出具体步骤和修改内容。"

---

### 第 10 步：Smoke 验证（编译 + 冒烟运行）

> 并行阶段确认通过后**自动进入**本步骤，无需单独询问。

**前置检查**：
- 确认全局上下文中有 `RTL filelist 路径`。若缺失，此时向用户询问：
  > "第 10 步需要 RTL filelist（.f 文件）来编译 DUT + TB。请提供路径。"
- 确认仿真器可用（`which vcs` 或 `which xrun`）。若不可用：
  > "编译器不可用，Smoke 步骤跳过。生成的代码已就绪，请在获取仿真器后手动执行：`vcs -f <rtl.f> -f tb/tb_filelist.f -top tb_top`"
  >
  > 跳过后直接进入流水线完成，Smoke 状态标记为 **SKIPPED**。

**加载 skill**：读取 `../verification-sim_tool/SKILL.md`（编译/运行命令参考）

#### 10a. 编译检查

1. 按 `verification-sim_tool` 的编译流程执行：
   ```bash
   vcs -full64 -sverilog -timescale=1ns/1ps \
       +incdir+$UVM_HOME/src \
       $UVM_HOME/src/uvm_pkg.sv \
       -f <rtl_filelist> \
       -f tb/tb_filelist.f \
       -top tb_top \
       -Mdir=work/csrc_smoke \
       -o work/simv_smoke \
       -l work/compile_smoke.log
   ```
2. 检查退出码 + grep 错误：
   ```bash
   grep -i "^Error\|^Fatal\|\*E\|\*F" work/compile_smoke.log | head -20
   ```
3. **编译成功**（exit code = 0，无 Error/Fatal）→ 进入 10b
4. **编译失败** → 进入 fix 循环（见下）

#### 10b. 冒烟运行

1. 运行 smoke test（第 6 步 tb_writer 生成的 `<dut>_smoke_test`）：
   ```bash
   ./work/simv_smoke \
       +UVM_TESTNAME=<dut>_smoke_test \
       +UVM_VERBOSITY=UVM_MEDIUM \
       -l work/sim_smoke.log
   ```
2. 检查运行结果：
   ```bash
   grep "UVM_FATAL\|UVM_ERROR\|PASSED\|FAILED" work/sim_smoke.log | tail -20
   ```
3. **运行成功**（无 UVM_FATAL/UVM_ERROR，有 PASSED）→ Smoke 通过
4. **运行失败** → 进入 fix 循环

#### Fix 循环（编译失败或运行失败时）

1. 加载 `../verification-sim_debug/SKILL.md`
2. 分析编译/运行 log，定位出错的组件属于哪一步生成的代码：
   - 接口/Agent/Env/tb_top → 第 6 步（tb_writer）
   - Sequence 文件 → 第 7 步（stim_writer）
   - Fcov 文件 → 第 8 步（fcov_writer）
   - SVA/Scoreboard → 第 9 步（checker_writer）
3. 修复代码并告知用户："Smoke 编译/运行失败，已定位到 `<文件名>`（第 X 步生成），修复内容：`<修改摘要>`。"
4. 重新执行 10a 编译 → 10b 运行
5. **最多重试 3 轮**。3 轮后仍失败：
   - 输出诊断报告（错误 log + 已尝试的修复 + 残留问题）
   - 告知用户："Smoke 验证在 3 轮自动修复后仍未通过，请查看诊断报告手动处理。"
   - Smoke 状态标记为 **FAILED**

**退出条件**：Smoke 编译 + 运行均通过，或超过 3 轮自动修复后交用户处理。

---

### 第 11 步：回归测试（全量仿真 + 覆盖率收敛）

> Smoke 验证通过后**自动进入**本步骤，无需单独询问。
> 若第 10 步状态为 **SKIPPED** 或 **FAILED**，本步骤同样跳过。

**加载 skill**：
- 读取 `../verification-sim_tool/SKILL.md`（编译/运行/回归命令参考）
- 若使用 simforge 回归管理：读取 `../simforge/SKILL.md`
- 覆盖率分析：提示可使用 `cov_reader` skill

#### 11a. 准备回归配置

1. 基于第 10 步已验证通过的编译命令，为每个 TC 生成运行条目。
2. **推荐方式**：生成 `testplan.yaml`（simforge 格式），将第 7 步所有 sequence 对应的 test case 注册为回归条目：

```yaml
global:
  simulator: vcs
  top_module: tb_top
  compile_opts: >-
    -full64 -sverilog -timescale=1ns/1ps
    -ntb_opts uvm-1.2
    -f <rtl_filelist>
    -f tb/tb_filelist.f
  sim_opts: "+UVM_VERBOSITY=UVM_MEDIUM"
  timeout: 3600

cases:
  - name: <dut>_smoke_test
    tags: [smoke, p0]
    sim_opts: "+UVM_TESTNAME=<dut>_smoke_test"
  # ... 为每个已生成的 test 添加条目
```

3. **备选方式**（无 simforge 时）：使用 Makefile 或 shell 脚本批量运行。

#### 11b. 提交全量回归

```bash
# 使用 simforge
simforge submit --tags regress -j <并发数>
simforge status
simforge results --filter failed
```

或手动批量运行：

```bash
for tc in <test_list>; do
    ./work/simv_smoke +UVM_TESTNAME=${tc} +UVM_VERBOSITY=UVM_MEDIUM \
        -l work/sim_${tc}.log &
done
wait
```

#### 11c. 回归结果分析

1. **Pass/Fail 汇总**：
   ```
   Total: <N> TCs | PASS: <n> | FAIL: <n> | TIMEOUT: <n>
   ```

2. **失败分析**：对每个 FAIL 的 TC：
   - 提取 UVM_FATAL / UVM_ERROR 信息
   - 定位失败源头（sequence / checker / scoreboard / SVA）
   - 按优先级（P0 > P1 > P2）排序展示

3. **P0 通过率要求**：所有 P0 标记的 TC 必须 100% 通过。若有 P0 失败，明确标记为 **CRITICAL**。

#### 11d. 覆盖率收集与分析

1. **收集覆盖率**：若编译时开启了覆盖率选项（`-cm line+cond+fsm+tgl+branch+assert`），合并所有 TC 的覆盖率数据：
   ```bash
   # VCS coverage merge
   urg -dir work/simv_smoke.vdb -report work/coverage_report
   ```

2. **覆盖率指标检查**（对照第 3 步 `verification-strategy.md` 中的收敛目标）：

   | 覆盖率类型 | 典型目标 |
   |-----------|---------|
   | 功能覆盖（Functional） | ≥ 95% |
   | 代码行覆盖（Line） | ≥ 90% |
   | 条件覆盖（Condition） | ≥ 85% |
   | 分支覆盖（Branch） | ≥ 85% |
   | Toggle 覆盖 | ≥ 80% |
   | FSM 覆盖 | 100% |
   | 断言覆盖（Assertion） | 100% |

3. **覆盖率空洞分析**：若有未覆盖项，分类报告：
   - **可补充**：建议增加新 TC 或扩展现有 sequence 的 constraint
   - **设计不可达**：建议添加 waiver / exclusion

#### 11e. 回归报告

向用户展示回归汇总：

```
╔══════════════════════════════════════════════════════╗
║              回归测试报告                            ║
╠══════════════════════════════════════════════════════╣
║ 总 TC 数：   <N>                                     ║
║ PASS：       <n>                                     ║
║ FAIL：       <n> (<列出失败 TC 名称>)                ║
║ P0 通过率：  <n>/<N> (<百分比>)                      ║
╠══════════════════════════════════════════════════════╣
║              覆盖率汇总                              ║
╠══════════════════════════════════════════════════════╣
║ 功能覆盖：   <xx>%    (目标 ≥ 95%)  <PASS/GAP>      ║
║ 行覆盖：     <xx>%    (目标 ≥ 90%)  <PASS/GAP>      ║
║ 条件覆盖：   <xx>%    (目标 ≥ 85%)  <PASS/GAP>      ║
║ 分支覆盖：   <xx>%    (目标 ≥ 85%)  <PASS/GAP>      ║
║ Toggle：     <xx>%    (目标 ≥ 80%)  <PASS/GAP>      ║
║ FSM：        <xx>%    (目标 100%)   <PASS/GAP>      ║
║ 断言：       <xx>%    (目标 100%)   <PASS/GAP>      ║
╚══════════════════════════════════════════════════════╝
```

**退出条件**：
- **PASSED**：所有 P0 TC 通过 + 覆盖率达标
- **PARTIAL**：存在 P1/P2 失败或覆盖率部分未达标，输出改善建议
- **FAILED**：存在 P0 失败，需用户介入修复

---

## 流水线完成

全部十步完成后，向用户生成**验证总结**：

```
╔══════════════════════════════════════════════════════╗
║              验证交付物已完成                        ║
╠══════════════════════════════════════════════════════╣
║ DUT: <名称>                                          ║
║ 验证目标：      已定义 <N> 条                        ║
║ 测试点：        已分解 <N> 条                        ║
║ 策略分配：      已映射 <N> 条                        ║
║    ├─ FORMAL：  <n> 条（独立 FPV 工程）              ║
║    └─ SIM：     <n> 条                               ║
║ SIM 三元组：    已定义 <N> 条                        ║
║ TB 组件：       已规划 <N> 个（其中新开发 <n> 个）   ║
║ 激励序列：      已生成 <N> 个 sequence 文件          ║
║ 功能覆盖：      已生成 <N> 个 covergroup             ║
║ 检查器：        已生成 <n> 条 SVA + <n> 个 scoreboard║
║ TB 骨架：       已生成 <N> 个结构性 .sv 文件         ║
║ Smoke 验证：    <PASSED / FAILED / SKIPPED>          ║
║ P0（必须通过）：<n> 条                               ║
╚══════════════════════════════════════════════════════╝

交付物：
  📄 verification-targets.md
  📄 verification-testpoints.md
  📄 verification-strategy.md
  📄 verification-sim-tc-defines.md
  📄 verification-tb-arch.md
  📁 tb/sequences/（激励序列）
  📁 tb/env/（fcov + scoreboard + env）
  📁 tb/sva/（SVA 属性 + bind）
  📁 tb/（interfaces + agents + tests + tb_top + filelist）
  📄 回归报告（TC 通过率 + 覆盖率汇总）

下一步建议：
  1. 覆盖率空洞补充：根据回归报告中的 GAP 项，增加新 TC 或扩展 sequence constraint
  2. 正式签收：确认所有 P0 通过且覆盖率达标后，验证可签收
  3. FPV 工程：若有 FORMAL 测试点，单独创建 FPV 项目
```

---

## 子 Skill 参考

| 步骤 | Skill 名称 | 文件 |
|------|-----------|------|
| 1 | `verification-target` | `../verification-target/SKILL.md` |
| 2 | `verification-testpoint` | `../verification-testpoint/SKILL.md` |
| 3 | `verification-strategy` | `../verification-strategy/SKILL.md` |
| 4 | `verification-sim_tc_define` | `../verification-sim_tc_define/SKILL.md` |
| 5 | `verification-sim_tb_arch` | `../verification-sim_tb_arch/SKILL.md` |
| 6 | `verification-sim_tb_writer` | `../verification-sim_tb_writer/SKILL.md` |
| 7 | `verification-sim_stim_writer` | `../verification-sim_stim_writer/SKILL.md` |
| 8 | `verification-sim_fcov_writer` | `../verification-sim_fcov_writer/SKILL.md` |
| 9 | `verification-sim_checker_writer` | `../verification-sim_checker_writer/SKILL.md` |
| 10 | `verification-sim_tool` + `verification-sim_debug` | `../verification-sim_tool/SKILL.md` + `../verification-sim_debug/SKILL.md` |
| 11 | `verification-sim_tool` + `simforge` + `cov_reader` | `../verification-sim_tool/SKILL.md` + `../simforge/SKILL.md` + `../cov-reader/SKILL.md` |
