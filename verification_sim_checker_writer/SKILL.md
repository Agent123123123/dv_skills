---
name: verification_sim_checker_writer
description: "verification_workflow 流水线的第 8 步。读取第 4 步输出的 verification-sim-tc-defines.md 中每条 TC 的 Checker 字段，编写两类检查代码：（1）在接口文件中实现 SVA assert property 捕获时序不变式；（2）若 tb-arch 规划了 Scoreboard，实现 Scoreboard 的数据比对逻辑。若存在 UVM/TB coding style 相关 skill，在开始编码前必须先加载。输出为接口文件中的 SVA 属性块 + tb/env/<dut>_scoreboard.sv。触发条件：由 verification_workflow 在第 8 步调用，或用户直接需要为已定义的 TC Checker 生成断言或 Scoreboard 代码时触发。"
---

# 检查器编写（第 8 步）

## 概述

本 skill 是 `verification_workflow` 流水线的**第 8 步**。

**目标**：依据 `verification-sim-tc-defines.md` 中每条 TC 的 **Checker 字段**，实现两类检查代码：
1. **SVA 断言**（`assert property`）——写入对应接口文件（`tb/interfaces/<intf>_if.sv`），自动在每个时钟边沿检查时序不变式
2. **Scoreboard 数据比对逻辑**——若 `verification-tb-arch.md` 中规划了 Scoreboard，实现 `tb/env/<dut>_scoreboard.sv` 的比对逻辑

**输入**：
- `verification-sim-tc-defines.md`（来自第 4 步）——每条 TC 的 Checker 字段
- `verification-tb-arch.md`（来自第 5 步）——确认 SVA 写入哪个接口、是否有 Scoreboard 以及其输入/输出事务类型

**输出**：
- `tb/interfaces/<intf>_if.sv` 中新增的 SVA 属性块（追加到现有接口文件）
- `tb/env/<dut>_scoreboard.sv`（若架构含 Scoreboard）

---

## 前置：Coding Style Skill 检查

**开始编码前必须执行：** 扫描已加载 skill 列表，若存在名称含 `uvm`、`tb`、`coding-style` 的 skill，立即加载并遵循；否则：
> "未检测到 TB coding style skill，将采用内置默认 UVM 编码规范。"

---

## Checker 字段分类

读取每条 TC 的 Checker 字段，将其分为两类：

| 类型 | 特征 | 实现方式 |
|------|------|---------|
| **时序不变式** | "若 A 则非 B"、"X 不得在 Y 期间为高"、"VALID 拜高后 DATA 必须稳定" | SVA `assert property` 写入接口文件 |
| **数据比对** | "输出数据必须与输入经过算法 F 变换后的结果相等" | Scoreboard `write()` 比对逻辑 |

若一条 TC 的 Checker 同时含有两类，分别输出 SVA 和 Scoreboard 代码。

---

## 一、SVA 断言实现

### Checker 字段 → SVA 语法映射

| Checker 描述模式 | SVA 写法 |
|----------------|---------|
| `若 A 则非 B`（`A \|-> !B`） | `assert property (@(posedge clk) A \|-> !B)` |
| `A 为高时，B 必须保持稳定` | `assert property (@(posedge clk) A \|-> $stable(B))` |
| `A 之后 N 拍，B 必须为高` | `assert property (@(posedge clk) A \|-> ##N B)` |
| `A 期间，B 不得翻转` | `assert property (@(posedge clk) $rose(A) \|-> B throughout A)` |
| `A 发生时，同一拍内 B 和 C 不能同时为高` | `assert property (@(posedge clk) A \|-> !(B && C))` |
| `复位释放后 N 拍内，X 必须为某值` | `assert property (@(posedge clk) $fell(rst_n) \|-> ##[1:N] X == V)` |

### SVA 代码结构

将 SVA 属性块追加到对应接口文件（`tb/interfaces/<intf>_if.sv`）末尾，位于 `endinterface` 之前：

```systemverilog
// ══════════════════════════════════════════════
//  SVA Checker Properties（来自 TC 三元组）
// ══════════════════════════════════════════════

// TC: TP-FUNC-001 — <TC 描述>
// Checker: <Checker 字段原文摘要>
property p_tp_func_001_<描述>;
  @(posedge clk) disable iff (!rst_n)
  <蕴含式条件> |-> <结论>;
endproperty
assert property (p_tp_func_001_<描述>)
  else `uvm_error("SVA", "TP-FUNC-001: <违规描述>")

// TC: TP-FUNC-002 — <TC 描述>
property p_tp_func_002_<描述>;
  @(posedge clk) disable iff (!rst_n)
  ...
endproperty
assert property (p_tp_func_002_<描述>)
  else `uvm_error("SVA", "TP-FUNC-002: <违规描述>")
```

**关键实现规则**：

- **`disable iff (!rst_n)`**：所有断言默认加复位屏蔽，复位期间不检查；若 Checker 明确针对复位行为本身，去除此条
- **错误消息**：`else` 后的错误消息必须包含 TC ID 和一句人类可读的违规描述，方便仿真日志定位
- **属性命名**：`p_<tc_id>_<关键词>`，确保在整个项目中唯一
- **时钟边沿**：使用接口中已定义的 `clocking block` 时钟，与 Driver/Monitor 保持一致

---

## 二、Scoreboard 实现（若架构含 Scoreboard）

### 何时需要 Scoreboard

当任意 TC 的 Checker 包含以下描述时，需要 Scoreboard：
- "输出数据与期望值相等"
- "响应内容必须匹配请求的 X 字段"
- "DUT 输出与参考模型预测一致"

### 实现模板

```systemverilog
class <dut>_scoreboard extends uvm_scoreboard;
  `uvm_component_utils(<dut>_scoreboard)

  // 接收 Reference Model 预测输出
  uvm_analysis_imp #(<intf>_seq_item, <dut>_scoreboard) ap_expected;
  // 接收 DUT 实际输出（来自 output monitor）
  uvm_analysis_imp #(<intf>_seq_item, <dut>_scoreboard) ap_actual;

  // 使用队列缓存预测值（FIFO 语义）
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

  // Reference Model 预测到达时入队
  function void write_ap_expected(<intf>_seq_item t);
    expected_q.push_back(t);
  endfunction

  // DUT 实际输出到达时出队比对
  // 比对逻辑直接对应 TC Checker 字段的数据比对描述
  function void write_ap_actual(<intf>_seq_item actual);
    <intf>_seq_item expected;
    if (expected_q.size() == 0) begin
      `uvm_error("SB", "Actual output received but no expected item in queue")
      fail_cnt++;
      return;
    end
    expected = expected_q.pop_front();
    // ── 比对逻辑：直接对应 Checker 字段 ──────────────────
    if (actual.<field> !== expected.<field>) begin
      `uvm_error("SB", $sformatf(
        "MISMATCH: actual=%0h expected=%0h (TC: <tc_id>)",
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
endclass
```

**关键实现规则**：

- **队列语义**：预测值和实际值均按 FIFO 顺序比对，若顺序无法保证，使用 `associative array` 按 transaction ID 索引
- **比对字段精确性**：只比对 Checker 字段中明确指出需要比对的字段，不全量比对 seq_item 所有字段
- **孤立项报警**：仿真结束时若 `expected_q` 非空，在 `final_phase` 中报告未被消耗的预测项数量

---

## 执行步骤

1. 遍历所有 TC 的 Checker 字段，分类为"SVA 时序不变式"或"数据比对"
2. 将所有 SVA 属性插入对应接口文件
3. 若有数据比对需求，生成 `<dut>_scoreboard.sv`（比对逻辑覆盖所有相关 TC）
4. 完成后汇报：

```
检查器交付摘要 — <DUT 名称>
────────────────────────────────────────
SVA 断言：
  <intf>_if.sv 新增 <N> 条 assert property
  覆盖 TC：<TP-001, TP-003, ...>

Scoreboard：
  <dut>_scoreboard.sv（比对逻辑覆盖 <n> 条 TC）
  比对字段：<field_a>、<field_b>
────────────────────────────────────────
待确认事项（若有）：
  - TP-XXX：Checker 中"数据比对"算法未明确，已按直接相等处理，请确认
```

询问：**"以上检查器实现是否符合预期？如有断言条件或比对字段需要调整，请指出。"**
用户确认后进入第 9 步。

---

## 暂停交互规则

| 情况 | 询问 |
|------|------|
| Checker 描述数据比对但未指明算法 | "`<TC>` Checker 要求数据比对，但变换算法不明确，请确认是直接相等还是需要参考模型运算" |
| Checker 描述的信号不在接口定义中 | "Checker 引用信号 `<signal>`，但该信号未出现在 `<intf>_if.sv` 中，请确认信号路径" |
| 需要跨时钟域的断言 | "TP-XXX 的断言跨越两个时钟域（`<clk_a>` 和 `<clk_b>`），请确认同步策略后再实现" |

---

## 质量检查清单

- [ ] 每条 SVA 断言均包含 `disable iff (!rst_n)`（除非明确针对复位行为）
- [ ] 每条断言的 `else` 错误消息包含 TC ID 和人类可读的违规描述
- [ ] 属性名称在整个项目中唯一（含 TC ID 前缀）
- [ ] Scoreboard 比对字段与 Checker 字段一一对应，无多余比对
- [ ] Scoreboard 中实现了孤立预期项的检测（expected_q 非空报警）
- [ ] 分类正确——时序不变式用 SVA，数据比对用 Scoreboard，两者不混用
