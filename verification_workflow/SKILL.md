---
name: verification_workflow
description: "重要：用于任何芯片 IP/模块验证规划任务时必须加载。定义强制性的九步串行工作流（verification-target → verification-testpoint → verification-strategy → verification_sim_tc_define → verification_sim_tb_arch → verification_sim_stim_writer → verification_sim_fcov_writer → verification_sim_checker_writer → verification_sim_tb_writer），并按顺序编排各子 skill 的加载。若不加载本 skill，将跳过关键阶段，产生不完整的临时验证计划。触发条件：任何涉及为 Verilog/SystemVerilog IP、模块或 RTL 块进行规划/定义/准备/验证/策略制定的任务——尤其是从头开始时。"
---

# 验证工作流（新版）—— 编排器

## 概述

本 skill 是验证规划流水线的**顶层编排器**。

它定义了一套**强制性九步串行工作流**，将输入的 DUT Spec + RTL 推进为一份完整的、含可运行代码的验证交付物。

```
┌─────────────────────────────────────────┐
│     输入：DUT Spec + RTL 源代码          │
└───────────────────┬─────────────────────┘
                    │
                    ▼
        ┌───────────────────────┐
        │  第 1 步              │
        │  verification-target  │
        │  定义验证目标         │
        └───────────┬───────────┘
                    │ 输出：verification-targets.md
                    ▼
        ┌───────────────────────┐
        │  第 2 步              │
        │  verification-        │
        │  testpoint            │
        │  分解测试点           │
        └───────────┬───────────┘
                    │ 输出：verification-testpoints.md
                    ▼
        ┌───────────────────────┐
        │  第 3 步              │
        │  verification-        │
        │  strategy             │
        │  规划验证策略         │
        └───────────┬───────────┘
                    │ 输出：verification-strategy.md
                    ▼
        ┌───────────────────────┐
        │  第 4 步              │
        │  verification_        │
        │  sim_tc_define        │
        │  定义 SIM 三元组      │
        └───────────┬───────────┘
                    │ 输出：verification-sim-tc-defines.md
                    ▼
        ┌───────────────────────┐
        │  第 5 步              │
        │  verification_        │
        │  sim_tb_arch          │
        │  设计 UVM TB 架构     │
        └───────────┬───────────┘
                    │ 输出：verification-tb-arch.md
                    ▼
        ┌───────────────────────┐
        │  第 6 步              │
        │  verification_        │
        │  sim_tb_writer        │
        │  实现 UVM TB 代码     │
        └───────────┬───────────┘
                    │ 输出：tb/ 代码目录
                    ▼
┌─────────────────────────────────────────┐
│  输出：完整验证交付物                    │
│  （5 份规划文档 + 可运行 TB 代码）       │
└─────────────────────────────────────────┘
```

---

## 强制执行规则

1. **始终按顺序执行（第 1 → 2 → 3 步）。** 不允许跳步或并行执行。
2. **每步必须完成并经用户确认后，才能开始下一步。** 不允许一次性预先生成全部三份输出。
3. **执行每步之前，必须先加载该步的子 skill。** 将读取子 skill 的 SKILL.md 作为该步的第一个操作。
4. **输出向下传递。** 每步的输出文档是下一步的输入，需在步骤间明确说明这一传递关系。

---

## 步骤执行流程

### 预检：收集 DUT 输入信息

开始第 1 步之前，收集：

- DUT 名称 / 描述（询问用户）
- Spec 文档路径（若未提供则询问）
- RTL 顶层文件路径（若未提供则询问）
- 已有约束（时间节点、排除的子模块、已知的 Golden IP）

若 Spec 或 RTL 不可用，告知用户：
> "需要 DUT Spec 和至少一个 RTL 顶层文件才能继续。请提供这些文件。"

---

### 第 1 步：定义验证目标

**加载 skill**：读取 `../verification-target/SKILL.md`
**执行**：完整遵循该 skill 中定义的流程。
**退出条件**：`verification-targets.md` 已保存，用户已确认其内容。

---

### 第 2 步：分解测试点

**加载 skill**：读取 `../verification-testpoint/SKILL.md`
**输入**：将第 1 步的 `verification-targets.md` 作为上下文传入。
**执行**：完整遵循该 skill 中定义的流程。
**退出条件**：`verification-testpoints.md` 已保存，用户已确认其内容。

---

### 第 3 步：规划验证策略

**加载 skill**：读取 `../verification-strategy/SKILL.md`
**输入**：将 `verification-targets.md` 和 `verification-testpoints.md` 同时作为上下文传入。
**执行**：完整遵循该 skill 中定义的流程。
**退出条件**：`verification-strategy.md` 已保存，用户已确认其内容。

---

### 第 4 步：定义 SIM 测试用例三元组

**加载 skill**：读取 `../verification_sim_tc_define/SKILL.md`
**输入**：将 `verification-strategy.md` 作为上下文传入（筛选手段为 SIM 的测试点）；同时保留对 DUT Spec 和 RTL 的引用。
**执行**：完整遵循该 skill 中定义的流程，为每条 SIM 测试点输出 Stim / Coverage / Checker 三元组。
**退出条件**：`verification-sim-tc-defines.md` 已保存，用户已确认其内容。

---

### 第 5 步：设计 UVM Testbench 架构

**加载 skill**：读取 `../verification_sim_tb_arch/SKILL.md`
**输入**：将 `verification-sim-tc-defines.md` 作为主要输入；同时保留对 DUT Spec、RTL 顶层文件和 `verification-targets.md` 的引用。
**执行**：完整遵循该 skill 中定义的流程，决策 UVM TB 组件清单、层次结构、VIP 复用方案和开发任务清单。
**退出条件**：`verification-tb-arch.md` 已保存，用户已确认其内容。

---

### 第 6 步：编写激励序列

**加载 skill**：读取 `../verification_sim_stim_writer/SKILL.md`
**输入**：将 `verification-sim-tc-defines.md`（Stim 字段）和 `verification-tb-arch.md`（Agent/Sequencer 分配）作为输入。
**执行**：完整遵循该 skill 中定义的流程，为每条 SIM TC 生成 UVM Sequence `.sv` 文件。
**退出条件**：`tb/sequences/` 目录下所有 TC 的 sequence 文件已生成，用户已确认。

---

### 第 7 步：编写功能覆盖率

**加载 skill**：读取 `../verification_sim_fcov_writer/SKILL.md`
**输入**：将 `verification-sim-tc-defines.md`（Coverage 字段）和 `verification-tb-arch.md`（Monitor/Subscriber 分配）作为输入。
**执行**：完整遵循该 skill 中定义的流程，生成 `tb/env/<dut>_func_cov.sv` 覆盖率文件。
**退出条件**：`<dut>_func_cov.sv` 已生成，所有 TC 的 covergroup 定义已确认。

---

### 第 8 步：编写检查器

**加载 skill**：读取 `../verification_sim_checker_writer/SKILL.md`
**输入**：将 `verification-sim-tc-defines.md`（Checker 字段）和 `verification-tb-arch.md`（SVA 位置、Scoreboard 规划）作为输入。
**执行**：完整遵循该 skill 中定义的流程，在接口文件中追加 SVA 断言，并生成 `tb/env/<dut>_scoreboard.sv`（若有数据比对需求）。
**退出条件**：所有 SVA 断言和 Scoreboard 代码已生成并经用户确认。

---

### 第 9 步：实现 TB 结构性骨架

**加载 skill**：读取 `../verification_sim_tb_writer/SKILL.md`
**输入**：将 `verification-tb-arch.md` 作为主要输入；同时引用第 6~8 步已生成的 sequences / fcov / checker 文件。
**执行**：完整遵循该 skill 中定义的流程，实现接口、Agent 组件、Env 连接层、Tests、tb_top，并生成涵盖全部文件的 `tb_filelist.f`。
**退出条件**：`tb/` 目录下所有结构性组件代码已生成，`tb_filelist.f` 已创建，用户已确认交付物。

---

## 流水线完成

全部九步确认完成后，向用户生成**验证总结**：

```
╔══════════════════════════════════════════════════════╗
║              验证交付物已完成                        ║
╠══════════════════════════════════════════════════════╣
║ DUT: <名称>                                          ║
║ 验证目标：      已定义 <N> 条                        ║
║ 测试点：        已分解 <N> 条                        ║
║ 策略分配：      已映射 <N> 条                        ║
║    ├─ FORMAL：  <n> 条                               ║
║    └─ SIM：     <n> 条                               ║
║ SIM 三元组：    已定义 <N> 条 SIM 测试点             ║
║ TB 组件：       已规划 <N> 个（其中新开发 <n> 个）   ║
║ 激励序列：      已生成 <N> 个 sequence 文件          ║
║ 功能覆盖：      已生成 <N> 个 covergroup             ║
║ 检查器：        已生成 <n> 条 SVA + <n> 个 scoreboard║
║ TB 骨架：       已生成 <N> 个结构性 .sv 文件         ║
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
  📁 tb/（interfaces + agents + tests + tb_top + filelist）

下一步：配置仿真器编译命令（`vcs/xrun -f tb/tb_filelist.f`），执行冒烟测试，根据覆盖率报告迭代完善约束。
```

---

## 子 Skill 参考

| 步骤 | Skill 名称 | 文件 |
|------|-----------|------|
| 1 | `verification-target` | `../verification-target/SKILL.md` |
| 2 | `verification-testpoint` | `../verification-testpoint/SKILL.md` |
| 3 | `verification-strategy` | `../verification-strategy/SKILL.md` |
| 4 | `verification_sim_tc_define` | `../verification_sim_tc_define/SKILL.md` |
| 5 | `verification_sim_tb_arch` | `../verification_sim_tb_arch/SKILL.md` |
| 6 | `verification_sim_stim_writer` | `../verification_sim_stim_writer/SKILL.md` |
| 7 | `verification_sim_fcov_writer` | `../verification_sim_fcov_writer/SKILL.md` |
| 8 | `verification_sim_checker_writer` | `../verification_sim_checker_writer/SKILL.md` |
| 9 | `verification_sim_tb_writer` | `../verification_sim_tb_writer/SKILL.md` |
