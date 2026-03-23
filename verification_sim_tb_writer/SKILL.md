---
name: verification_sim_tb_writer
description: "verification_workflow 流水线的第 9 步。读取第 5 步输出的 verification-tb-arch.md，实现 UVM Testbench 的结构性骨架（接口 + clocking block、seq_item、Driver、Monitor、Agent、Package、Ref Model、Virtual Sequencer、Env 连接层、Tests、tb_top），并整合第 6~8 步已生成的 sequences / fcov / checker 代码，输出完整 tb_filelist.f。若存在 UVM/TB coding style 相关 skill，在开始编码前必须先加载。触发条件：由 verification_workflow 在第 9 步调用，或用户直接需要为已设计的 TB 架构生成结构性骨架代码时触发。"
---

# UVM Testbench 代码实现（第 9 步）

## 概述

本 skill 是 `verification_workflow` 流水线的**第 9 步**。

**目标**：依据 `verification-tb-arch.md` 中的组件清单与层次结构，实现 TB **结构性骨架**（接口、Agent 组件、Env 连接层、Tests、tb_top），并整合第 6~8 步已生成的 sequences / fcov / checker 代码，输出完整的 `tb_filelist.f`。

> **注意**：激励 Sequence（第 6 步）、功能覆盖率（第 7 步）、SVA 断言与 Scoreboard（第 8 步）已在前序步骤实现，本步骤不重复生成。

**输入**：
- `verification-tb-arch.md`（来自第 5 步）——主要输入：组件清单、层次结构、接口列表、开发任务表
- `tb/sequences/`（来自第 6 步 stim_writer）——各 TC 激励 sequence 文件，已存在，Test 中引用
- `tb/env/<dut>_func_cov.sv`（来自第 7 步 fcov_writer）——功能覆盖率文件，已存在，Env 中连接 analysis port
- `tb/env/<dut>_scoreboard.sv` 及接口 SVA（来自第 8 步 checker_writer）——检查器，已存在，Env 中连接

**输出**：`tb/` 结构性骨架代码 + `tb_filelist.f`（涵盖所有第 6~9 步生成的文件）

---

## 前置：Coding Style Skill 检查

**在开始任何编码之前，必须执行以下检查：**

扫描当前已加载的 skill 列表，判断是否存在 UVM / TB coding style 相关 skill（名称含 `uvm`、`tb`、`coding-style`、`rtl-coding-style` 等关键字）。

- **若存在**：立即加载该 skill，并在整个编码过程中严格遵循其中的命名规范、文件结构规范和代码风格要求。
- **若不存在**：采用以下默认规范（见下方"默认编码规范"节），并告知用户：
  > "未检测到 TB coding style skill，将采用内置默认 UVM 编码规范。如需定制风格，可加载对应 skill 后重新执行本步骤。"

---

## 默认编码规范

在无专项 coding style skill 时，遵循以下基线规范：

### 命名规范

| 类型 | 命名模式 | 示例 |
|------|---------|------|
| Package | `<dut>_<intf>_pkg` | `uart_apb_pkg` |
| Interface | `<intf>_if` | `apb_if` |
| Agent | `<intf>_agent` | `apb_agent` |
| Driver | `<intf>_driver` | `apb_driver` |
| Monitor | `<intf>_monitor` | `apb_monitor` |
| Sequencer | `<intf>_sequencer` | `apb_sequencer` |
| Sequence Item | `<intf>_seq_item` | `apb_seq_item` |
| Sequence | `<intf>_<场景>_seq` | `apb_write_seq` |
| Env | `<dut>_env` | `uart_env` |
| Scoreboard | `<dut>_scoreboard` | `uart_scoreboard` |
| Reference Model | `<dut>_ref_model` | `uart_ref_model` |
| Coverage | `<dut>_func_cov` | `uart_func_cov` |
| Test | `<dut>_<场景>_test` | `uart_reset_test` |
| Virtual Sequencer | `<dut>_vseqr` | `uart_vseqr` |
| TB Top | `tb_top` | `tb_top` |

### 文件结构规范

- 每个类单独一个文件，文件名与类名一致（小写下划线），后缀 `.sv`
- Package 文件汇总该接口所有类的 `` `include ``，文件名为 `<pkg_name>.sv`
- Interface 文件单独存放，文件名 `<intf>_if.sv`

### UVM 类规范

- 所有类使用 `uvm_component_utils` / `uvm_object_utils` 宏注册
- `new()` 构造函数固定签名：component 用 `(string name, uvm_component parent)`，object 用 `(string name = "<class>")`
- `build_phase` 中使用 `uvm_config_db` 获取 virtual interface
- `run_phase` 中的永久循环使用 `forever` + `@(posedge vif.clk)` 采样

---

## 目录结构

生成的 TB 代码按以下目录结构组织（根据实际组件增减）：

```
tb/
├── interfaces/
│   └── <intf>_if.sv              ← 每个接口一个文件
├── <intf>_agent/
│   ├── <intf>_seq_item.sv
│   ├── <intf>_sequencer.sv
│   ├── <intf>_driver.sv
│   ├── <intf>_monitor.sv
│   ├── <intf>_agent.sv
│   └── <intf>_pkg.sv             ← `include 上述所有文件
├── env/
│   ├── <dut>_ref_model.sv        ← 若架构中含参考模型
│   ├── <dut>_scoreboard.sv       ← 若架构中含 Scoreboard
│   ├── <dut>_func_cov.sv
│   ├── <dut>_vseqr.sv            ← 若架构中含 Virtual Sequencer
│   └── <dut>_env.sv
├── sequences/
│   └── <intf>_<场景>_seq.sv      ← 每条 TC 对应一个或多个 sequence
├── tests/
│   ├── <dut>_base_test.sv
│   └── <dut>_<场景>_test.sv
└── tb_top.sv
```

---

## 编码执行流程

### 阶段一：解析 tb-arch.md，建立编码任务列表

读取 `verification-tb-arch.md` 中的"开发任务清单"，按以下顺序排列编码任务（依赖关系由低到高）：

```
1. Interface (.sv)          ← 信号声明 + clocking block（SVA 由第 8 步已追加）
2. Seq Item
3. Sequencer
4. Driver
5. Monitor
6. Agent
7. Package (per agent)
8. Reference Model（若有）  ← 为第 8 步 Scoreboard 提供预测值
9. Virtual Sequencer（若有）
10. Env                     ← 例化所有组件，连接 analysis port 到第 7/8 步组件
11. Base Test + 各 TC Test  ← Test 调用第 6 步已生成的 sequences
12. tb_top
13. tb_filelist.f           ← 汇总第 6~9 步所有 .sv 文件的编译顺序
```

> `tb/sequences/`（第 6 步）、`func_cov.sv`（第 7 步）、`scoreboard.sv` 和 SVA（第 8 步）均已存在，**本步骤不重新生成**，仅在 Env / filelist 中引用。

### 阶段二：逐组件编码

对每个组件，执行以下步骤：

1. **查阅输入文档**：
   - 从 `verification-tb-arch.md` 获取该组件的接口、职责描述、关联 TC 列表
   - 从 `verification-sim-tc-defines.md` 获取关联 TC 的具体 Stim/Coverage/Checker 内容

2. **生成代码**：按照 coding style 规范（或默认规范）实现该组件

3. **关键实现要点**：

   **Interface**：
   - 声明 DUT 所有相关端口的 `logic` 信号
   - 包含 `clocking block`（driver_cb / monitor_cb）
   - 包含简单的协议检查 `assert property`（直接从 Checker 字段提取）

   **Seq Item**：
   - `rand` 字段对应 TC Stim 中的随机化维度
   - `constraint` 对应 Stim 中描述的合法范围/约束
   - 实现 `do_copy`、`do_compare`、`convert2string`

   **Driver**：
   - `run_phase` 中循环调用 `seq_item_port.get_next_item()`
   - 严格按 Stim 描述的时序协议驱动 `clocking block`
   - 调用 `seq_item_port.item_done()` 释放

   **Monitor**：
   - 在 `run_phase` 中持续采样 `monitor_cb`
   - 打包为 seq_item 后通过 `ap`（analysis port）广播
   - Coverage 采样点在此处触发

   **Env**（连接层）：
   - 例化所有 Agent、Ref Model（若有）、Virtual Sequencer（若有）
   - 在 `connect_phase` 中将所有 Monitor 的 `analysis_port` 连接到：
     - 第 7 步生成的 `<dut>_func_cov` 的 analysis_imp
     - 第 8 步生成的 `<dut>_scoreboard` 的 analysis_imp（若有）
   - 同时连接 Ref Model 输出到 Scoreboard 的 expected analysis_imp（若有）

   **Test（每条 TC）**：
   - 继承自 base_test
   - 在 `run_phase` 中通过 virtual sequencer 调用对应 sequence
   - 设置仿真超时

   **tb_top**：
   - 例化 DUT 与所有 interface
   - 通过 `uvm_config_db #(virtual <intf>_if)::set()` 将 vif 注入 UVM 配置数据库
   - 调用 `run_test()`

### 阶段三：Package 文件与文件列表

所有 `.sv` 文件生成后：
- 为每个 agent 生成 `<intf>_pkg.sv`（汇总 `` `include ``）
- 生成顶层 `tb_filelist.f`，列出所有文件的编译顺序（interfaces → packages → env → tests → tb_top）

---

## 编码过程中的用户交互

遇到以下情况时，**暂停编码并向用户确认**，不得自行假设：

| 情况 | 询问内容 |
|------|---------|
| Stim 描述的时序未指定具体拍数 | "TP-XXX 的 Stim 中 `<信号>` 的保持时间未明确，请确认是 1 拍还是需要随机化（范围？）" |
| Checker 中存在"数据比对"但 Spec 未给出算法 | "`<TC>` 的 Checker 需要数据比对，Reference Model 的变换算法未在 Spec 中明确，请提供或确认建模方式" |
| 接口信号名与 RTL 端口名不一致 | "发现 tb-arch 中 `<信号名>` 与 RTL 顶层端口 `<实际端口名>` 命名不符，请确认使用哪个" |
| VIP 集成需要特定 UVM 注册方式 | "架构中 `<接口>` 使用 VIP，请提供 VIP 的 agent class 名称和 package 路径以便正确例化" |

---

## 输出格式

本步骤完成后，在对话中汇报交付摘要：

```
TB 代码交付摘要 — <DUT 名称>
────────────────────────────────────────
新开发文件：
  interfaces/     <n> 个  (.sv)
  <intf>_agent/   <n> 个  (seq_item / driver / monitor / agent / pkg)
  env/            <n> 个  (ref_model / scoreboard / cov / env)
  sequences/      <n> 个  (每条 TC 一个)
  tests/          <n> 个  (base + 各 TC test)
  tb_top.sv       1  个
  tb_filelist.f   1  个
────────────────────────────────────────
复用 / VIP：
  <intf>_agent    来自 <VIP 名称>（未生成代码，已在 pkg 中 include）
────────────────────────────────────────
总计新建文件：<N> 个
```

---

## 质量检查清单

提交交付物前确认：

- [ ] 所有 `uvm_component` 子类已注册 `uvm_component_utils`
- [ ] 所有 `uvm_object` 子类已注册 `uvm_object_utils`
- [ ] 每个 Driver 的 `run_phase` 存在 `item_done()` 调用路径（无遗漏）
- [ ] 每个 Monitor 的 `analysis_port` 已在 connect_phase 中连接到 Scoreboard / Coverage
- [ ] `tb/sequences/`（第 6 步）、`func_cov.sv`（第 7 步）、`scoreboard.sv/SVA`（第 8 步）均已存在，无缺失文件
- [ ] Env 的 `connect_phase` 已将所有 Monitor analysis_port 正确连接到 fcov 和 scoreboard
- [ ] `tb_top` 中所有 interface 均已通过 `uvm_config_db::set` 注入
- [ ] `tb_filelist.f` 中文件编译顺序正确（无前向引用问题）

---

## 输出交接

本步骤完成后：

1. 告知用户：**"第 9 步已完成。UVM Testbench 结构性骨架代码已全部实现。结合第 6~8 步的 sequences / fcov / checker，`tb/` 目录现已构成完整可运行的 Testbench。"**
2. 说明后续可直接执行的操作：
   - 配置仿真器编译命令：`vcs/xrun/questa -f tb/tb_filelist.f`
   - 执行基础冒烟测试：`run_test("smoke_test")`
   - 根据覆盖率报告迭代补充约束
3. 告知用户：**"验证规划与实现流水线（第 1~9 步）已全部完成。"**
