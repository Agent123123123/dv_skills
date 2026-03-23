---
name: verification_sim_tb_arch
description: "verification_workflow 流水线的第 5 步。读取第 4 步输出的 verification-sim-tc-defines.md，分析所有测试用例三元组的 Stim/Coverage/Checker 需求，决策如何构建基于 UVM 的 Testbench 架构——包括需要哪些组件、组件间的连接关系、哪些组件可复用已有 VIP、哪些组件需要自行开发。输出为 verification-tb-arch.md，包含完整的 TB 组件清单、层次结构图、接口/VIP 决策以及开发任务分配。触发条件：由 verification_workflow 在第 5 步调用，或用户直接需要为一批 SIM 测试用例规划 UVM TB 架构时触发。"
---

# UVM Testbench 架构设计（第 5 步）

## 概述

本 skill 是 `verification_workflow` 流水线的**第 5 步**。

**目标**：基于 `verification-sim-tc-defines.md` 中所有测试用例的 Stim / Coverage / Checker 需求，决策构建一个**能覆盖全部 SIM 测试点**的 UVM Testbench 架构，并明确每个组件的来源（复用 / VIP / 新开发）。

**输入**：
- `verification-sim-tc-defines.md`（来自第 4 步）——提取每条 TC 对组件能力的需求
- DUT Spec 文档——用于了解接口协议与模块边界
- RTL 顶层文件——确认端口信号、时钟域、复位策略
- `verification-targets.md`（来自第 1 步）——确认集成边界与排除范围

**输出**：`verification-tb-arch.md` —— UVM TB 架构设计文档

---

## 分析框架：从 TC 需求到 TB 组件

### 第一阶段：提取 TB 能力需求

遍历 `verification-sim-tc-defines.md` 中每条 TC 的三元组，提取以下四类能力需求：

| 需求类型 | 来源字段 | 问题 |
|---------|---------|------|
| **激励能力** | Stim | 需要驱动哪些接口？施加什么类型的激励（顺序、随机、边界）？ |
| **监控能力** | Coverage | 需要在哪些接口/信号上采样？需要哪些 covergroup？ |
| **检查能力** | Checker | 需要哪些断言？是否需要参考模型（Scoreboard）进行数据比对？ |
| **同步/控制** | Stim 时序 | 是否需要跨接口事件同步？是否需要 virtual sequencer 编排？ |

---

### 第二阶段：组件决策清单

对每类需求，逐一决策以下问题：

#### 1. 需要哪些 UVM Agent？

对 DUT 的每个独立接口（或接口组），判断：
- 该接口是否需要主动驱动激励 → 需要 **Driver**
- 该接口是否需要被动监控 → 需要 **Monitor**
- 是否两者都需要 → **Active Agent**；只监控 → **Passive Agent**

#### 2. 是否存在可用 VIP？

对每个需要 Agent 的接口，按以下顺序判断：

| 判断顺序 | 条件 | 结论 |
|---------|------|------|
| ① | 接口为行业标准协议（AXI、AHB、APB、PCIe、USB、I2C、SPI 等） | **询问用户是否有商用/开源 VIP 可用** |
| ② | 用户有内部已验证的 VIP 库 | **复用已有 VIP，记录版本与约束** |
| ③ | 接口为项目自定义协议 | **标记为需要新开发** |
| ④ | 接口极简（单信号握手） | **内联实现，不单独建 Agent** |

> **向用户确认**：对每个标准协议接口，明确询问：
> "该接口（`<接口名>`，协议：`<协议>`）是否有可用的 VIP？如有，请提供名称和路径；如无，我将规划新开发任务。"

#### 3. 是否需要参考模型（Reference Model）？

若任意 TC 的 Checker 含有"输出数据与期望值比对"的逻辑（即 Scoreboard 模式），则需要：
- **Reference Model**（纯软件行为模型，用 SV/C/Python 实现）
- **Scoreboard**（比对 Monitor 采集到的实际输出与 Reference Model 预测值）

若所有 Checker 均为纯断言（SVA 不变式，无数据比对），则 Scoreboard 可省略。

#### 4. 是否需要 Virtual Sequencer？

若存在以下任一情况，需要 **Virtual Sequencer**：
- 多个 Agent 的激励需要**有序编排**（如：先通过接口 A 配置，再通过接口 B 发送数据）
- TC 的 Stim 中明确描述了跨接口的时序依赖
- 存在超过 2 个 Active Agent

#### 5. 功能覆盖组件

基于所有 TC 的 Coverage 字段，规划：
- **Functional Coverage Collector**：收集 covergroup 采样的专用组件（可内嵌于 Monitor 或独立 Subscriber）
- **Coverage Group 列表**：按测试点分组，列出需要实现的 covergroup 名称与关键 bin

---

### 第三阶段：层次结构设计

确定上述所有组件后，输出标准 UVM TB 层次结构：

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
       ├─ func_cov_collector
       └─ virtual_sequencer  ← 若有跨 Agent 编排需求

uvm_test
  └─ virtual_sequence（顶层序列，编排各子 sequence）
       ├─ <intf_a>_sequence
       └─ <intf_b>_sequence
```

---

### 第四阶段：组件开发任务分解

对所有标记为"新开发"的组件，输出开发任务清单：

| 组件名 | 类型 | 所属 Agent | 功能描述 | 复杂度估计 | 依赖 |
|--------|------|-----------|---------|-----------|------|
| `<name>_driver` | Driver | `<agent>` | 驱动 `<接口>` 的合法激励序列 | 低/中/高 | `<接口>` 协议规范 |
| `<name>_monitor` | Monitor | `<agent>` | 采样 `<接口>` 上的事务并打包为 uvm_sequence_item | 低/中/高 | - |
| `ref_model` | Reference Model | - | 纯行为级功能建模，输入镜像 DUT 输入，预测 DUT 输出 | 中/高 | DUT Spec |
| `scoreboard` | Scoreboard | - | 比对 ref_model 输出与 monitor 采集的实际输出 | 低 | ref_model |
| `func_cov_collector` | Coverage | - | 实现 `<N>` 个 covergroup，覆盖所有 TC Coverage 需求 | 中 | TC 定义 |

---

## 确认流程

完成架构草案后，向用户按以下顺序确认：

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

用户确认后，保存为 `verification-tb-arch.md`。

---

## 输出格式

生成 `verification-tb-arch.md`，结构如下：

```markdown
# UVM Testbench 架构 — <DUT 名称>

## 摘要

| 组件类型 | 数量 | 来源 |
|---------|------|------|
| UVM Agent（Active） | <n> | 新开发 / VIP |
| UVM Agent（Passive） | <n> | 新开发 / VIP |
| Reference Model | <1 或 N/A> | 新开发 |
| Scoreboard | <1 或 N/A> | 新开发 |
| Functional Coverage Collector | 1 | 新开发 |
| Virtual Sequencer | <1 或 N/A> | 新开发 |
| **需要新开发的组件合计** | **<N>** | - |

---

## DUT 接口清单

| 接口名 | 协议类型 | 方向（DUT视角） | Agent 类型 | VIP 来源 |
|--------|---------|---------------|-----------|---------|
| `<intf_name>` | AXI4-Lite | Slave | Active | Synopsys VIP v2.3 |
| `<intf_name>` | 自定义握手 | Master | Active | 新开发 |
| `<intf_name>` | 仅观测 | - | Passive Monitor | 新开发 |

---

## TB 层次结构

（ASCII 图或列表，参见上方模板）

---

## 组件详细说明

### `<intf>_agent`（Active）
- **VIP / 来源**：<VIP名称 或 "新开发">
- **Driver 职责**：<描述>
- **Monitor 职责**：<描述>
- **关联 TC**：TP-FUNC-001、TP-INTF-003（列出驱动此 Agent 的 TC）

### `ref_model`（若存在）
- **实现语言**：SystemVerilog / C model via DPI / Python via vpi
- **输入**：<镜像哪些接口的输入事务>
- **输出**：<预测哪些接口的输出事务>
- **关联 TC**：<列出依赖 Scoreboard 比对的 TC>

### `func_cov_collector`
- **Covergroup 列表**：
  - `cg_<name>`：<覆盖哪条 TC 的 Coverage 需求，关键 bin>
  - ...

---

## 开发任务清单

| # | 组件名 | 类型 | 复杂度 | 依赖 | 备注 |
|---|--------|------|--------|------|------|
| 1 | `<name>` | Driver | 中 | 自定义协议规范 | - |
| 2 | `<name>` | Monitor | 低 | - | 可与 Driver 同步开发 |
| 3 | `ref_model` | Reference Model | 高 | DUT Spec 第 3~5 章 | 建议优先开发 |
...

---

## 接口连接关系

<描述 TB 顶层 (tb_top) 中 DUT 与各 Agent Interface 的连接方式，包括 clocking block 设计建议>

---

## 假设与约束

<架构决策过程中依赖的假设，以及已知限制>
```

---

## 质量检查清单

保存前确认：

- [ ] 每条 TC 的 Stim 均有对应的 Agent/Sequence 能力覆盖
- [ ] 每条 TC 的 Coverage 均有对应的 covergroup bin 规划
- [ ] 每条 TC 的 Checker 均有对应的 assert 或 scoreboard 比对实现路径
- [ ] 所有新开发组件已列入开发任务清单，无遗漏
- [ ] VIP 可用性已经用户明确确认（非假设）
- [ ] 参考模型的输入/输出边界与 DUT 端口对应关系已明确

---

## 输出交接

本步骤完成后：

1. 告知用户：**"第 5 步已完成。UVM Testbench 架构已定义完毕。"**
2. 说明各文档的用途：
   - `verification-tb-arch.md` → 指导 TB 目录结构搭建、组件文件创建与分工
   - 开发任务清单 → 可直接作为工程任务拆分输入
3. 提示用户：**"验证规划流水线（第 1~5 步）已全部完成。可以开始 Testbench 实现。"**
