---
name: verification-sim_tb_arch
description: "读取 SIM 测试用例三元组，分析所有测试用例的 Stim/Coverage/Checker 需求，决策如何构建基于 UVM 的 Testbench 架构——包括需要哪些组件、组件间的连接关系、哪些组件可复用已有 VIP、哪些组件需要自行开发。输出为 verification-tb-arch.md，包含完整的 TB 组件清单、层次结构图、接口/VIP 决策以及开发任务分配。触发条件：用户需要为一批 SIM 测试用例规划 UVM TB 架构时触发。"
---

# UVM Testbench 架构设计

## 概述

**目标**：基于 SIM 测试用例三元组中所有测试用例的 Stim / Coverage / Checker 需求，决策构建一个**能覆盖全部 SIM 测试点**的 UVM Testbench 架构，并明确每个组件的来源（复用 / VIP / 新开发）。

**输出**：`verification-tb-arch.md` —— UVM TB 架构设计文档

---

## 第一阶段：输入约束收集

> **Workflow 上下文**：若本 skill 由 `verification-workflow` 编排器调用，DUT 名称、Spec、RTL、`verification-targets.md`、`verification-sim-tc-defines.md` 已由前序步骤产出并存在于全局上下文中，直接使用，**不重复询问用户**。仅对下表中全局上下文未覆盖的信息项（如 VIP 库可用性、UVM 基础库版本）进行询问。
>
> 若本 skill 被**独立使用**（非 workflow 上下文），则按下表逐项收集。

在开始分析之前，必须收集以下信息。对于尚未提供的信息，主动询问。

| 信息项 | 获取方式 | 缺失时处理 |
|--------|---------|-----------|
| SIM 测试用例三元组（`verification-sim-tc-defines.md` 或等效描述） | 询问用户提供 | 必须获取，这是架构决策的主要输入 |
| DUT Spec 文档 | 询问路径 | 用于了解接口协议与模块边界 |
| RTL 顶层文件 | 询问路径 | 确认端口信号、时钟域、复位策略 |
| 验证目标文档（`verification-targets.md` 或等效描述） | 询问用户提供 | 确认集成边界与排除范围 |
| 是否有内部 VIP 库 / 商用 VIP 授权 | 询问用户 | 影响 Agent 开发策略 |
| 项目 UVM 基础库版本（如有） | 询问用户 | 影响 API 兼容性 |

---

## 第二阶段：从 TC 需求提取 TB 能力需求

遍历 SIM 测试用例三元组中每条 TC 的三元组，提取以下四类能力需求：

| 需求类型 | 来源字段 | 问题 |
|---------|---------|------|
| **激励能力** | Stim | 需要驱动哪些接口？施加什么类型的激励？ |
| **监控能力** | Coverage | 需要在哪些接口/信号上采样？需要哪些 covergroup？ |
| **检查能力** | Checker | 需要哪些断言？是否需要参考模型进行数据比对？ |
| **同步/控制** | Stim 时序 | 是否需要跨接口事件同步？是否需要 virtual sequencer？ |

---

## 第三阶段：组件决策清单

### 1. 需要哪些 UVM Agent？

对 DUT 的每个独立接口（或接口组），判断：
- 该接口是否需要主动驱动激励 → 需要 **Driver**
- 该接口是否需要被动监控 → 需要 **Monitor**
- 两者都需要 → **Active Agent**；只监控 → **Passive Agent**

### 2. 是否存在可用 VIP？

对每个需要 Agent 的接口，按以下顺序判断：

| 判断顺序 | 条件 | 结论 |
|---------|------|------|
| ① | 接口为行业标准协议（AXI、AHB、APB、PCIe、USB、I2C、SPI 等） | **询问用户是否有商用/开源 VIP 可用** |
| ② | 用户有内部已验证的 VIP 库 | **复用已有 VIP，记录版本与约束** |
| ③ | 接口为项目自定义协议 | **标记为需要新开发** |
| ④ | 接口极简（单信号握手） | **内联实现，不单独建 Agent** |

> **向用户确认**：对每个标准协议接口，明确询问：
> "该接口（`<接口名>`，协议：`<协议>`）是否有可用的 VIP？如有，请提供名称和路径；如无，我将规划新开发任务。"

### 3. 是否需要参考模型（Reference Model）？

若任意 TC 的 Checker 含有"输出数据与期望值比对"的逻辑（Scoreboard 模式），则需要：
- **Reference Model**（纯软件行为模型）
- **Scoreboard**（比对 Monitor 采集的实际输出与 Reference Model 预测值）

> ⚠️ **注意**：Reference Model 对复杂 DUT（加密引擎、DMA 控制器、协议转换桥等）可能是数百行代码的独立实现，需在开发任务清单中单独列出，重点标注复杂度。

若所有 Checker 均为纯断言（SVA 不变式，无数据比对），则 Scoreboard 可省略。

### 4. 是否需要 Virtual Sequencer？

若存在以下任一情况，需要 **Virtual Sequencer**：
- 多个 Agent 的激励需要有序编排（如先通过接口 A 配置，再通过接口 B 发送数据）
- TC 的 Stim 中明确描述了跨接口的时序依赖
- 存在超过 2 个 Active Agent

### 5. 功能覆盖组件

基于所有 TC 的 Coverage 字段，将覆盖率需求分为两类，为每条 Coverage 判定采用哪种 Collector：

#### 5a. Transaction-based Fcov Collector（事务级覆盖率收集器）

**实现方式**：继承 `uvm_subscriber #(<intf>_seq_item)`，在 `write()` 中对事务对象字段调用 `covergroup.sample()`。

**适用场景**：
- 需要对 UVM 事务对象（seq_item）的字段进行覆盖率采样
- 覆盖点来源于 Monitor 通过 analysis_port 广播的事务内容
- 典型示例：命令类型覆盖、地址范围覆盖、数据模式覆盖、burst 长度覆盖

**决策规则**：若 Coverage 描述中的采样对象可以从 Monitor 广播的 seq_item 字段中直接获取 → 使用 Transaction-based Collector。

#### 5b. Interface-based Fcov Collector（接口级覆盖率收集器）

**实现方式**：在 Interface 中直接定义 `covergroup`，或使用 SVA `cover property` 捕获时序覆盖，通过 `bind` 机制绑定到 DUT 或接口实例。

**适用场景**：
- 需要对接口信号的时序行为进行覆盖率采样（如握手时序、背压持续周期）
- 需要对 DUT 内部信号或状态机状态进行覆盖率采样
- 需要捕获多信号在同一时钟沿的组合关系
- 典型示例：FSM 状态转移覆盖、FIFO 水位覆盖、协议时序覆盖（valid-to-ready 延迟分布）

**决策规则**：若 Coverage 描述涉及接口时序、DUT 内部信号、或状态机转移 → 使用 Interface-based Collector。

**内部信号访问**：当需要采样 DUT 内部信号时，必须通过 `bind` 机制：
- 创建独立的覆盖率模块（`<dut>_intf_fcov`），包含引用内部信号的 covergroup
- 使用 `bind <dut_module> <dut>_intf_fcov <inst>(.clk(...), .internal_sig(...))` 绑定
- 在开发任务清单中标注需要访问的内部信号路径

#### Coverage 分类汇总

遍历所有 TC 的 Coverage 字段后，输出分类表：

| TC ID | Coverage 描述 | 采样对象类型 | Collector 类型 | 备注 |
|-------|-------------|-------------|---------------|------|
| TP-FUNC-001 | cmd 类型覆盖 | seq_item.cmd | Transaction-based | — |
| TP-FUNC-003 | FSM S1→S2 转移 | DUT 内部信号 | Interface-based (bind) | 需内部信号路径 |

---

## 第四阶段：层次结构设计

```
uvm_test
  └─ uvm_env (tb_env)
       ├─ <intf_a>_agent  [Active/Passive]
       │    ├─ <intf_a>_driver       ← 复用VIP / 新开发
       │    ├─ <intf_a>_monitor
       │    └─ <intf_a>_sequencer
       ├─ <intf_b>_agent  [Active/Passive]
       │    └─ ...
       ├─ ref_model        ← 若有数据比对需求
       ├─ scoreboard       ← 若有数据比对需求
       ├─ txn_fcov_collector    ← uvm_subscriber，事务级覆盖率
       ├─ intf_fcov_collector   ← Interface covergroup / SVA cover（可含 bind）
       └─ virtual_sequencer  ← 若有跨 Agent 编排需求
```

---

## 第五阶段：用户确认流程

**Step A —— VIP 可用性确认**
> "以下接口需要 Agent，请确认 VIP 情况：
> - `<接口名>`（协议 `<协议>`）：是否有可用 VIP？
> - `<接口名>`（自定义协议）：已标记为新开发，是否认可？"

**Step B —— 架构决策确认**
> "建议 TB 架构包含以下组件：[展示层次图]
> 是否需要调整任何组件的存在、类型（Active/Passive）或实现方式？"

**Step C —— 开发范围确认**
> "需要新开发的组件共 `<N>` 个：[展示开发任务清单]
> 是否有任何组件可从其他项目复用或有内部库可用？"

---

## 质量检查清单

保存前确认：

- [ ] 每条 TC 的 Stim 均有对应的 Agent/Sequence 能力覆盖
- [ ] 每条 TC 的 Coverage 均已分类（Transaction-based / Interface-based）且有对应 covergroup 或 cover property 规划
- [ ] 需要 `bind` 访问的 DUT 内部信号路径已在开发任务清单中明确标注
- [ ] 每条 TC 的 Checker 均有对应的 assert 或 scoreboard 比对实现路径
- [ ] 所有新开发组件已列入开发任务清单，无遗漏
- [ ] VIP 可用性已经用户明确确认（非假设）
- [ ] 参考模型的输入/输出边界与 DUT 端口对应关系已明确
- [ ] 若 Reference Model 复杂度为"高"，已在开发任务清单中单独标注

---

## 输出交付物格式

文件名：`verification-tb-arch.md`

```markdown
# UVM Testbench 架构 — <DUT 名称>

## 摘要

| 组件类型 | 数量 | 来源 |
|---------|------|------|
| UVM Agent（Active） | <n> | 新开发 / VIP |
| UVM Agent（Passive） | <n> | 新开发 / VIP |
| Reference Model | <1 或 N/A> | 新开发 |
| Scoreboard | <1 或 N/A> | 新开发 |
| Transaction-based Fcov Collector | <n 或 N/A> | 新开发 |
| Interface-based Fcov Collector | <n 或 N/A> | 新开发 |
| Virtual Sequencer | <1 或 N/A> | 新开发 |
| **需要新开发的组件合计** | **<N>** | - |

---

## DUT 接口清单

| 接口名 | 协议类型 | 方向（DUT视角） | Agent 类型 | VIP 来源 |
|--------|---------|---------------|-----------|---------|
| `<intf_name>` | AXI4-Lite | Slave | Active | Synopsys VIP v2.3 |
| `<intf_name>` | 自定义握手 | Master | Active | 新开发 |

---

## TB 层次结构

（ASCII 图，参见本文档第四阶段模板）

---

## 组件详细说明

### `<intf>_agent`（Active）
- **VIP / 来源**：<VIP名称 或 "新开发">
- **Driver 职责**：<描述>
- **Monitor 职责**：<描述>
- **关联 TC**：TP-FUNC-001、TP-INTF-003

### `ref_model`（若存在）
- **实现语言**：SystemVerilog / C model via DPI / Python via vpi
- **输入**：<镜像哪些接口的输入事务>
- **输出**：<预测哪些接口的输出事务>
- **复杂度**：低/中/高（高复杂度需单独规划实现）
- **关联 TC**：<列出依赖 Scoreboard 比对的 TC>

### `txn_fcov_collector`（Transaction-based，若存在）
- **基类**：`uvm_subscriber #(<intf>_seq_item)`
- **采样触发**：`write()` 回调
- **Covergroup 列表**：
  - `cg_<name>`：<覆盖哪条 TC 的 Coverage 需求，关键 bin>
- **关联 TC**：<列出使用 Transaction-based 覆盖的 TC>

### `intf_fcov_collector`（Interface-based，若存在）
- **实现方式**：Interface 内 covergroup / SVA `cover property`
- **绑定方式**：<说明是直接在接口中定义，还是通过 `bind` 绑定到 DUT 内部>
- **Covergroup / Cover Property 列表**：
  - `cg_<name>` / `cp_<name>`：<覆盖哪条 TC 的 Coverage 需求，关键 bin / 时序条件>
- **需访问的内部信号**（若使用 bind）：
  - `<dut_module>.<signal_path>` — <用途说明>
- **关联 TC**：<列出使用 Interface-based 覆盖的 TC>

---

## 开发任务清单

| # | 组件名 | 类型 | 复杂度 | 依赖 | 备注 |
|---|--------|------|--------|------|------|
| 1 | `<name>` | Driver | 中 | 自定义协议规范 | - |
| 2 | `ref_model` | Reference Model | 高 | DUT Spec 全文 | 需独立规划实现 |

---

## 接口连接关系

<描述 tb_top 中 DUT 与各 Agent Interface 的连接方式，包括 clocking block 设计建议>

---

## 假设与约束

<架构决策过程中依赖的假设，以及已知限制>
```

---

## 完成确认

1. 告知用户：**"UVM Testbench 架构已定义完毕。"**
2. 说明：`verification-tb-arch.md` 可直接指导 TB 目录结构搭建、组件文件创建与分工。
3. 询问用户是否继续进行各组件代码实现。
