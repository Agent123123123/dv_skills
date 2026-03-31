---
name: verification-sim_tb_writer
description: "读取 TB 架构文档，实现 UVM Testbench 的结构性骨架（接口 + clocking block、seq_item、Driver、Monitor、Agent、Package、Ref Model、Virtual Sequencer、Env 连接层、Tests、tb_top），并整合已生成的 sequences / fcov / sva / scoreboard 代码，输出完整 tb_filelist.f。若存在 UVM/TB coding style 相关 skill，在开始编码前必须先加载。触发条件：用户需要为已设计的 TB 架构生成结构性骨架代码时触发。"
---

# UVM Testbench 代码实现

## 概述

**目标**：依据 TB 架构文档中的组件清单与层次结构，实现 TB **结构性骨架**（接口、Agent 组件、Env 连接层、Tests、tb_top），并输出含占位路径的 `tb_filelist.f`。后续激励序列、功能覆盖率、SVA/Scoreboard 由各自的 skill 填入对应路径。

> **骨架优先原则**：激励 Sequence（`tb/sequences/`）、功能覆盖率（`tb/env/<dut>_func_cov.sv`）、SVA 属性（`tb/sva/<dut>_properties.sv`）、Scoreboard（`tb/env/<dut>_scoreboard.sv`）在本 skill 执行时**尚未生成**。`tb_filelist.f` 中为这些文件预留路径占位符，待后续步骤生成后自动填入。

**输出**：`tb/` 结构性骨架代码 + `tb_filelist.f`（含后续步骤的文件路径占位）

---

## 第一阶段：输入约束收集

> **Workflow 上下文**：若本 skill 由 `verification-workflow` 编排器调用，DUT 名称、Spec、RTL、`verification-tb-arch.md`、`verification-sim-tc-defines.md`、Coding Style 结论已由前序步骤产出并存在于全局上下文中，直接使用，**不重复询问用户**。
>
> 若本 skill 被**独立使用**（非 workflow 上下文），则按下表逐项收集。

在开始编码之前，必须收集以下信息。对于尚未提供的信息，主动询问。

| 信息项 | 获取方式 | 缺失时处理 |
|--------|---------|-----------|
| TB 架构文档（`verification-tb-arch.md` 或等效描述） | 询问用户提供 | 必须获取，这是 TB 骨架编码的主要输入 |
| SIM 测试用例三元组（`verification-sim-tc-defines.md` 或等效描述） | 询问用户提供 | 必须获取，确认接口信号与组件边界 |
| `tb/sequences/` 预期文件列表 | 从 `verification-sim-tc-defines.md` 中推导 TC 数量和名称 | 根据 TC 名称自动生成占位路径，后续由 stim_writer 填入 |
| `tb/env/<dut>_func_cov.sv` | 无需提供，尚未生成 | 在 filelist 中写入预期路径占位，后续由 fcov_writer 填入 |
| `tb/sva/<dut>_properties.sv` | 无需提供，尚未生成 | 在 filelist 中写入预期路径占位，后续由 checker_writer 填入 |
| `tb/env/<dut>_scoreboard.sv` | 无需提供（若架构含 Scoreboard） | 在 filelist 中写入预期路径占位，后续由 checker_writer 填入 |
| UVM/TB coding style 规范（若项目有要求） | 询问用户是否有 coding style skill 或文档 | 无则使用本文档内置默认规范 |
| DUT 名称 | 询问用户 | 必须获取 |

---

## 前置：Coding Style Skill 检查

**在开始任何编码之前，必须执行以下检查：**

扫描当前已加载的 skill 列表，判断是否存在 UVM / TB coding style 相关 skill（名称含 `uvm`、`tb`、`coding-style`、`rtl-coding-style` 等关键字）。

- **若存在**：立即加载该 skill，并在整个编码过程中严格遵循其命名规范、文件结构规范和代码风格要求。
- **若不存在**：采用以下默认规范，并告知用户：
  > "未检测到 TB coding style skill，将采用内置默认 UVM 编码规范。如需定制风格，可加载对应 skill 后重新执行。"

---

## 默认编码规范

| 类型 | 命名模式 | 示例 |
|------|---------|------|
| Package | `<dut>_<intf>_pkg` | `uart_apb_pkg` |
| Interface | `<intf>_if` | `apb_if` |
| Agent | `<intf>_agent` | `apb_agent` |
| Driver | `<intf>_driver` | `apb_driver` |
| Monitor | `<intf>_monitor` | `apb_monitor` |
| Sequencer | `<intf>_sequencer` | `apb_sequencer` |
| Sequence Item | `<intf>_seq_item` | `apb_seq_item` |
| Env | `<dut>_env` | `uart_env` |
| Reference Model | `<dut>_ref_model` | `uart_ref_model` |
| Test | `<dut>_<场景>_test` | `uart_reset_test` |
| Virtual Sequencer | `<dut>_vseqr` | `uart_vseqr` |
| TB Top | `tb_top` | `tb_top` |

---

## 第二阶段：目录结构

```
tb/
├── interfaces/
│   └── <intf>_if.sv
├── <intf>_agent/
│   ├── <intf>_seq_item.sv
│   ├── <intf>_sequencer.sv
│   ├── <intf>_driver.sv
│   ├── <intf>_monitor.sv
│   ├── <intf>_agent.sv
│   └── <intf>_pkg.sv
├── env/
│   ├── <dut>_ref_model.sv     ← 若架构中含参考模型
│   ├── <dut>_scoreboard.sv    ← 由 checker_writer 生成，本 skill 不重建
│   ├── <dut>_func_cov.sv      ← 由 fcov_writer 生成，本 skill 不重建
│   ├── <dut>_vseqr.sv         ← 若架构含 Virtual Sequencer
│   └── <dut>_env.sv
├── sequences/                 ← 由 stim_writer 生成，本 skill 不重建
│   └── <intf>_<场景>_seq.sv
├── sva/                       ← 由 checker_writer 生成，本 skill 不重建
│   └── <dut>_properties.sv
├── tests/
│   ├── <dut>_base_test.sv
│   ├── <dut>_smoke_test.sv    ← 必须生成，Smoke 验证目标
│   └── <dut>_<场景>_test.sv
└── tb_top.sv
```

---

## 第三阶段：编码执行流程

### 编码顺序（依赖关系由低到高）

```
1. Interface (.sv)          ← 信号声明 + clocking block
2. Seq Item                 ← rand 字段 + constraint（对应 Stim 随机维度）
3. Sequencer
4. Driver                   ← 按 Stim 时序协议驱动 clocking block
5. Monitor                  ← 采样 monitor_cb，打包为 seq_item 广播
6. Agent                    ← 例化 Driver/Monitor/Sequencer
7. Package（per agent）     ← `include 上述所有类
8. Reference Model（若有） ← 纯行为级，预测 DUT 输出
9. Virtual Sequencer（若有）
10. Env                     ← 例化所有组件，连接 fcov + scoreboard 的 analysis_port
11. Base Test + 各 TC Test  ← Test 调用已生成的 sequences
12. tb_top                  ← 例化 DUT + interfaces，注入 vif，调用 run_test()
13. tb_filelist.f           ← 汇总所有文件，含已有的 sequences/fcov/sva/scoreboard
```

### 关键实现要点

**Interface**：
- 声明 DUT 所有相关端口的 `logic` 信号
- 包含 `clocking block`（driver_cb / monitor_cb）

**Seq Item**：
- `rand` 字段对应 TC Stim 中的随机化维度
- `constraint` 对应 Stim 中描述的合法范围/约束
- 实现 `do_copy`、`do_compare`、`convert2string`

**Driver**：
- `run_phase` 中循环调用 `seq_item_port.get_next_item()`
- 按 Stim 描述的时序协议驱动 `clocking block`
- 调用 `seq_item_port.item_done()` 释放

**Monitor**：
- 在 `run_phase` 中持续采样 `monitor_cb`
- 打包为 seq_item 后通过 `ap`（analysis port）广播

**Env**（连接层）：
- 例化所有 Agent、Ref Model（若有）、Virtual Sequencer（若有）
- 在 `connect_phase` 中将所有 Monitor 的 `analysis_port` 连接到：
  - `<dut>_func_cov` 的 analysis_imp（由 fcov_writer 已生成）
  - `<dut>_scoreboard` 的 analysis_imp（由 checker_writer 已生成，若有）
- 同时连接 Ref Model 输出到 Scoreboard（若有）

**Test（每条 TC）**：
- 继承自 base_test
- 在 `run_phase` 中通过 virtual sequencer 调用已生成的对应 sequence
- 设置仿真超时

**Smoke Test（必须生成）**：
- 文件名：`<dut>_smoke_test.sv`，类名：`<dut>_smoke_test`
- 继承自 base_test
- `run_phase`：仅执行 reset 序列 + env 启停，不需要复杂激励
- 用途：作为第 10 步 Smoke 验证的运行目标，验证 TB 框架可编译/可启动

**tb_top**：
- 例化 DUT 与所有 interface
- 通过 `uvm_config_db #(virtual <intf>_if)::set()` 将 vif 注入 UVM 配置数据库
- 调用 `run_test()`

**tb_filelist.f**：
- 按编译顺序列出所有 `.sv` 文件（interfaces → packages → env → tests → tb_top → sva）
- **包含已有文件**：`tb/sequences/*.sv`、`tb/env/<dut>_func_cov.sv`、`tb/sva/<dut>_properties.sv`、`tb/env/<dut>_scoreboard.sv`（若有）

---

## 编码过程中的用户交互

遇到以下情况时，**暂停编码并向用户确认**：

| 情况 | 询问内容 |
|------|---------|
| Stim 描述的时序未指定具体拍数 | "TP-XXX 的 Stim 中 `<信号>` 的保持时间未明确，请确认是 1 拍还是需要随机化" |
| Checker 中存在数据比对但 Spec 未给出算法 | "`<TC>` 的 Checker 需要数据比对，Reference Model 的变换算法未在 Spec 中明确，请提供" |
| 接口信号名与 RTL 端口名不一致 | "发现 tb-arch 中 `<信号名>` 与 RTL 顶层端口 `<实际端口名>` 命名不符，请确认" |
| VIP 集成需要特定注册方式 | "架构中 `<接口>` 使用 VIP，请提供 VIP 的 agent class 名称和 package 路径" |

---

## 质量检查清单

- [ ] 所有 `uvm_component` 子类已注册 `uvm_component_utils`
- [ ] 所有 `uvm_object` 子类已注册 `uvm_object_utils`
- [ ] 每个 Driver 的 `run_phase` 存在 `item_done()` 调用路径（无遗漏）
- [ ] Env 的 `connect_phase` 已将所有 Monitor analysis_port 正确连接到 fcov 和 scoreboard
- [ ] `tb_top` 中所有 interface 均已通过 `uvm_config_db::set` 注入
- [ ] `tb_filelist.f` 中文件编译顺序正确（无前向引用问题）
- [ ] `tb_filelist.f` 包含 stim/fcov/sva/scoreboard 所有已有文件

---

## 输出交付物格式

```
tb/
├── interfaces/     <n> 个 .sv
├── <intf>_agent/   <n> 个（seq_item / driver / monitor / agent / pkg）
├── env/            <n> 个（ref_model / env，不含已有的 fcov/scoreboard）
├── tests/          <n> 个（base + smoke + 各 TC test）
├── tb_top.sv       1 个
└── tb_filelist.f   1 个（含所有已有文件的引用）
```

完成后向用户汇报：

```
TB 代码交付摘要 — <DUT 名称>
────────────────────────────────────────
新开发文件：<N> 个
已有文件（引用）：sequences/<n>个 + func_cov + sva + scoreboard
────────────────────────────────────────
可执行验证：vcs/xrun/questa -f tb/tb_filelist.f +UVM_TESTNAME=<dut>_smoke_test
```

询问用户："TB 骨架是否符合预期？"
