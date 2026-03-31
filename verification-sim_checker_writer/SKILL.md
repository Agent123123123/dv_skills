---
name: verification-sim_checker_writer
description: "读取 SIM 测试用例三元组中每条 TC 的 Checker 字段，编写两类检查代码：（1）生成独立的 SVA 属性文件（tb/sva/<dut>_properties.sv）捕获时序不变式；（2）若 TB 架构规划了 Scoreboard，实现 Scoreboard 的数据比对逻辑。输出为 tb/sva/ 下的 SVA 文件 + tb/env/<dut>_scoreboard.sv。触发条件：用户需要为已定义的 TC Checker 生成断言或 Scoreboard 代码时触发。"
---

# 检查器编写

## 概述

**目标**：依据 SIM 测试用例三元组中每条 TC 的 **Checker 字段**，实现两类检查代码：
1. **SVA 断言**（`assert property`）——写入独立文件 `tb/sva/<dut>_properties.sv`，通过 bind 机制绑定到 DUT，自动在每个时钟边沿检查时序不变式
2. **Scoreboard 数据比对逻辑**——若 TB 架构文档中规划了 Scoreboard，实现 `tb/env/<dut>_scoreboard.sv` 的比对逻辑

**输出**：
- `tb/sva/<dut>_properties.sv`（SVA 属性模块 + bind 声明）
- `tb/env/<dut>_scoreboard.sv`（若架构含 Scoreboard）

---

## 第一阶段：输入约束收集

> **Workflow 上下文**：若本 skill 由 `verification-workflow` 编排器调用，DUT 名称、RTL 顶层模块名、时钟/复位信号名、`verification-sim-tc-defines.md`、`verification-tb-arch.md`、Coding Style 结论已由前序步骤产出并存在于全局上下文中，直接使用，**不重复询问用户**。
>
> 若本 skill 被**独立使用**（非 workflow 上下文），则按下表逐项收集。

在开始编码之前，必须收集以下信息。对于尚未提供的信息，主动询问。

| 信息项 | 获取方式 | 缺失时处理 |
|--------|---------|-----------|
| SIM 测试用例三元组（`verification-sim-tc-defines.md` 或等效描述） | 询问用户提供 | 必须获取，每条 TC 的 Checker 字段是编码直接依据 |
| TB 架构文档（`verification-tb-arch.md` 或等效描述） | 询问用户提供 | 必须获取，确认是否有 Scoreboard 以及其输入/输出事务类型 |
| RTL DUT 顶层模块名 | 询问用户 | 必须获取，用于 bind 声明 |
| 时钟信号名和复位信号名 | 询问用户或读取 RTL | 必须获取，SVA 中的时钟和复位引用 |
| UVM/TB coding style 规范（若项目有要求） | 询问用户 | 无则使用本文档内置默认规范 |
| DUT 名称 | 询问用户 | 必须获取，用于文件/类命名 |

---

## 第二阶段：Checker 字段分类

读取每条 TC 的 Checker 字段，将其分为两类：

| 类型 | 特征 | 实现方式 |
|------|------|---------|
| **时序不变式** | "若 A 则非 B"、"X 不得在 Y 期间为高"、"VALID 拜高后 DATA 必须稳定" | SVA `assert property` 写入 `tb/sva/<dut>_properties.sv` |
| **数据比对** | "输出数据必须与输入经过算法 F 变换后的结果相等" | Scoreboard `write()` 比对逻辑 |

若一条 TC 的 Checker 同时含有两类，分别输出 SVA 和 Scoreboard 代码。

---

## 第三阶段：SVA 属性实现

### Checker 字段 → SVA 语法映射

| Checker 描述模式 | SVA 写法 |
|----------------|---------|
| `若 A 则非 B`（`A \|-> !B`） | `assert property (@(posedge clk) A \|-> !B)` |
| `A 为高时，B 必须保持稳定` | `assert property (@(posedge clk) A \|-> $stable(B))` |
| `A 之后 N 拍，B 必须为高` | `assert property (@(posedge clk) A \|-> ##N B)` |
| `A 期间，B 不得翻转` | `assert property (@(posedge clk) $rose(A) \|-> B throughout A)` |
| `A 发生时，B 和 C 不能同时为高` | `assert property (@(posedge clk) A \|-> !(B && C))` |
| `复位释放后 N 拍内，X 必须为某值` | `assert property (@(posedge clk) $fell(rst_n) \|-> ##[1:N] X == V)` |

### SVA 文件结构

生成 `tb/sva/<dut>_properties.sv`，包含一个专用模块及 bind 声明：

```systemverilog
// ══════════════════════════════════════════════
//  SVA Properties for <DUT 名称>
//  自动检查 SIM 测试用例三元组中所有 TC 的 Checker
// ══════════════════════════════════════════════

module <dut>_sva_props (
  input logic clk,
  input logic rst_n,
  // 其他需要引用的信号（从 DUT 端口列表提取）
  input logic <signal_a>,
  input logic <signal_b>
);

// ── TC: TP-FUNC-001 — <TC 描述> ──────────────────────────
// Checker: <Checker 字段原文摘要>
property p_tp_func_001_<描述>;
  @(posedge clk) disable iff (!rst_n)
  <蕴含式条件> |-> <结论>;
endproperty
assert property (p_tp_func_001_<描述>)
  else `uvm_error("SVA", "TP-FUNC-001: <违规描述>")

// ── TC: TP-FUNC-002 — <TC 描述> ──────────────────────────
property p_tp_func_002_<描述>;
  @(posedge clk) disable iff (!rst_n)
  ...
endproperty
assert property (p_tp_func_002_<描述>)
  else `uvm_error("SVA", "TP-FUNC-002: <违规描述>")

endmodule

// Bind 声明：将 SVA 模块绑定到 DUT 实例
bind <dut_module_name> <dut>_sva_props <dut>_props_inst (
  .clk    (<dut_clk_port>),
  .rst_n  (<dut_rst_port>),
  .<signal_a> (<dut_signal_a_port>),
  .<signal_b> (<dut_signal_b_port>)
);
```

**关键实现规则**：

- **`disable iff (!rst_n)`**：所有断言默认加复位屏蔽，复位期间不检查；若 Checker 明确针对复位行为本身，去除此条
- **错误消息**：`else` 后的错误消息必须包含 TC ID 和一句人类可读的违规描述
- **属性命名**：`p_<tc_id>_<关键词>`，确保在整个项目中唯一
- **独立文件**：SVA 属性写入独立 `.sv` 文件，通过 bind 绑定到 DUT，不修改任何已有文件

---

## 第四阶段：Scoreboard 实现（若架构含 Scoreboard）

何时需要 Scoreboard：任意 TC 的 Checker 包含"输出数据与期望值相等"、"响应内容必须匹配"等描述。

```systemverilog
class <dut>_scoreboard extends uvm_scoreboard;
  `uvm_component_utils(<dut>_scoreboard)

  uvm_analysis_imp #(<intf>_seq_item, <dut>_scoreboard) ap_expected;
  uvm_analysis_imp #(<intf>_seq_item, <dut>_scoreboard) ap_actual;

  <intf>_seq_item expected_q[$];
  int unsigned pass_cnt, fail_cnt;

  function new(string name, uvm_component parent);
    super.new(name, parent);
  endfunction

  function void build_phase(uvm_phase phase);
    super.build_phase(phase);
    ap_expected = new("ap_expected", this);
    ap_actual   = new("ap_actual",   this);
  endfunction

  function void write_ap_expected(<intf>_seq_item t);
    expected_q.push_back(t);
  endfunction

  function void write_ap_actual(<intf>_seq_item actual);
    <intf>_seq_item expected;
    if (expected_q.size() == 0) begin
      `uvm_error("SB", "Actual output received but no expected item in queue")
      fail_cnt++;
      return;
    end
    expected = expected_q.pop_front();
    // ── 比对逻辑：直接对应 Checker 字段 ─────────────────
    if (actual.<field> !== expected.<field>) begin
      `uvm_error("SB", $sformatf("MISMATCH: actual=%0h expected=%0h (TC: <tc_id>)",
        actual.<field>, expected.<field>))
      fail_cnt++;
    end else begin
      `uvm_info("SB", "PASS", UVM_HIGH)
      pass_cnt++;
    end
  endfunction

  function void report_phase(uvm_phase phase);
    `uvm_info("SB", $sformatf("Scoreboard: PASS=%0d  FAIL=%0d", pass_cnt, fail_cnt), UVM_NONE)
    if (fail_cnt > 0)
      `uvm_error("SB", "Scoreboard detected failures!")
  endfunction

  function void final_phase(uvm_phase phase);
    if (expected_q.size() > 0)
      `uvm_error("SB", $sformatf("%0d expected items unconsumed at end of simulation", expected_q.size()))
  endfunction
endclass
```

**关键实现规则**：

- **比对字段精确性**：只比对 Checker 字段中明确指出需要比对的字段，不全量比对所有字段
- **孤立项报警**：`final_phase` 中检查 `expected_q` 非空情况并报警

---

## 暂停交互规则

| 情况 | 询问 |
|------|------|
| Checker 描述数据比对但未指明算法 | "`<TC>` Checker 要求数据比对，但变换算法不明确，请确认是直接相等还是需要参考模型运算" |
| Checker 描述的信号不在 DUT 顶层端口 | "Checker 引用信号 `<signal>`，但该信号不在 DUT 顶层端口，请确认信号路径（内部信号需通过 bind 机制访问）" |
| 需要跨时钟域的断言 | "TP-XXX 的断言跨越两个时钟域，请确认同步策略后再实现" |

---

## 质量检查清单

- [ ] 每条 SVA 断言均包含 `disable iff (!rst_n)`（除非明确针对复位行为）
- [ ] 每条断言的 `else` 错误消息包含 TC ID 和人类可读的违规描述
- [ ] 属性名称在整个项目中唯一（含 TC ID 前缀）
- [ ] SVA 模块通过 bind 绑定到 DUT，不修改任何已有文件
- [ ] Scoreboard 比对字段与 Checker 字段一一对应，无多余比对
- [ ] Scoreboard 中实现了孤立预期项的检测（`final_phase` 报警）

---

## 输出交付物格式

```
tb/sva/
  <dut>_properties.sv     ← SVA 属性模块 + bind 声明
tb/env/
  <dut>_scoreboard.sv     ← 若架构含 Scoreboard
```

完成后向用户汇报：

```
检查器交付摘要 — <DUT 名称>
────────────────────────────────────────
SVA 断言：
  tb/sva/<dut>_properties.sv
    共 <N> 条 assert property
    覆盖 TC：<TP-001, TP-003, ...>

Scoreboard：
  tb/env/<dut>_scoreboard.sv（比对逻辑覆盖 <n> 条 TC）
  比对字段：<field_a>、<field_b>
────────────────────────────────────────
待确认事项（若有）：
  - TP-XXX：Checker 中"数据比对"算法未明确，已按直接相等处理，请确认
```

询问："以上检查器实现是否符合预期？如有断言条件或比对字段需要调整，请指出。"
