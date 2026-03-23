---
name: verification_sim_fcov_writer
description: "verification_workflow 流水线的第 7 步。读取第 4 步输出的 verification-sim-tc-defines.md 中每条 TC 的 Coverage 字段，结合第 5 步输出的 verification-tb-arch.md 中的 Monitor/Subscriber 分配，为每条 TC 编写 UVM 功能覆盖率代码（covergroup / coverpoint / bins）。若存在 UVM/TB coding style 相关 skill，在开始编码前必须先加载。输出为 tb/env/<dut>_func_cov.sv 覆盖率收集器文件。触发条件：由 verification_workflow 在第 7 步调用，或用户直接需要为已定义的 TC Coverage 生成功能覆盖率代码时触发。"
---

# 功能覆盖率编写（第 7 步）

## 概述

本 skill 是 `verification_workflow` 流水线的**第 7 步**。

**目标**：依据 `verification-sim-tc-defines.md` 中每条 TC 的 **Coverage 字段**，结合 `verification-tb-arch.md` 中确定的 Monitor/Subscriber 架构，实现 `<dut>_func_cov.sv`——包含所有 covergroup 定义及采样触发逻辑。

**输入**：
- `verification-sim-tc-defines.md`（来自第 4 步）——每条 TC 的 Coverage 字段：必须采样的状态、值、bin 描述
- `verification-tb-arch.md`（来自第 5 步）——确认 coverage collector 的连接方式（连接哪些 Monitor 的 analysis port，使用哪些 seq_item 字段）

**输出**：`tb/env/<dut>_func_cov.sv`

---

## 前置：Coding Style Skill 检查

**开始编码前必须执行：** 扫描已加载 skill 列表，若存在名称含 `uvm`、`tb`、`coding-style` 的 skill，立即加载并遵循；否则使用默认规范：
> "未检测到 TB coding style skill，将采用内置默认 UVM 编码规范。"

---

## Coverage 字段解析规则

读取每条 TC 的 Coverage 字段，将自然语言描述映射为 covergroup 元素：

| Coverage 描述模式 | 映射为 |
|-----------------|------|
| "必须采样到信号 X = 值 V" | `coverpoint x { bins v = {V}; }` |
| "覆盖 X 的低/中/高边界（A, B, C）" | `coverpoint x { bins lo = {A}; bins mid = {B}; bins hi = {C}; }` |
| "采样 X 处于范围 [A:B]" | `coverpoint x { bins range = {[A:B]}; }` |
| "状态转移 S1→S2" | `coverpoint state { bins s1_to_s2 = (S1 => S2); }` |
| "X 和 Y 的交叉覆盖" | `cross x_cp, y_cp;` |
| "握手成立（valid && ready）" | `coverpoint valid_ready_handshake { bins handshake = {1}; }` |
| "FSM 经过状态 S" | `coverpoint fsm_state { bins s = {S}; }` |

若 Coverage 字段描述模糊（如"覆盖所有合法输入"），**暂停并向用户确认**具体 bin 划分，不得自行扩展。

---

## 编码规范（默认）

| 要素 | 规范 |
|------|------|
| 类名 | `<dut>_func_cov` |
| 文件名 | `<dut>_func_cov.sv` |
| 继承 | `uvm_subscriber #(<intf>_seq_item)`（若仅监听一个接口） 或 `uvm_component`（多接口需多个 analysis_imp） |
| 注册 | `` `uvm_component_utils(<dut>_func_cov) `` |
| Covergroup 命名 | `cg_tp_<tc_id>`，例如 `cg_tp_func_001` |
| 采样触发 | 在 `write()` 函数或 `run_phase` 的 `@(posedge vif.clk)` 中调用 `<cg>.sample()` |

---

## 实现模板

```systemverilog
class <dut>_func_cov extends uvm_subscriber #(<intf>_seq_item);
  `uvm_component_utils(<dut>_func_cov)

  // ── TC: TP-FUNC-001 — <TC 描述> ──────────────────────────────────────
  // Coverage: <Coverage 字段原文摘要>
  covergroup cg_tp_func_001;
    cp_<signal>: coverpoint trans.<field> {
      bins <bin_name> = {<value>};
      // ... 直接映射 Coverage 字段中描述的每个采样条件
    }
    // 若有交叉覆盖：
    // cx_<a>_<b>: cross cp_<a>, cp_<b>;
  endgroup

  // ── TC: TP-FUNC-002 — <TC 描述> ──────────────────────────────────────
  covergroup cg_tp_func_002;
    // ...
  endgroup

  // （每条 TC 一个 covergroup，命名唯一）

  <intf>_seq_item trans;  // 当前采样事务

  function new(string name, uvm_component parent);
    super.new(name, parent);
    cg_tp_func_001 = new();
    cg_tp_func_002 = new();
    // 初始化所有 covergroup
  endfunction

  // write() 由 Monitor 的 analysis_port 调用触发采样
  function void write(<intf>_seq_item t);
    trans = t;
    // 根据事务类型或状态触发对应 covergroup
    cg_tp_func_001.sample();
    cg_tp_func_002.sample();
    // 若存在条件触发（仅当某 TC 场景发生时采样），使用 if 判断
  endfunction

endclass
```

**关键实现规则**：

- **每条 TC 独立 covergroup**：即使多条 TC 采样相同信号，也不合并 covergroup，保持与 TC ID 一对一的可追溯性
- **条件采样**：若 Coverage 字段描述"当 X 为 Y 时才采样"，必须在 `write()` 中加 `if` 条件，不得无条件调用所有 `sample()`
- **多接口覆盖**：若 TC 的 Coverage 跨越多个接口（如同时监听 input 和 output），在 `build_phase` 中为每个接口各声明一个 `uvm_analysis_imp`，并分别实现 `write_<intf>()` 函数
- **状态转移覆盖**：直接使用 SystemVerilog coverpoint 的序列 bin 语法 `bins name = (S1 => S2)` 实现，不使用临时变量拼凑

---

## 多接口 Analysis Imp（若需要）

当需要监听多个 Monitor 时，使用宏扩展：

```systemverilog
`uvm_analysis_imp_decl(_intf_a)
`uvm_analysis_imp_decl(_intf_b)

class <dut>_func_cov extends uvm_component;
  uvm_analysis_imp_intf_a #(<intf_a>_seq_item, <dut>_func_cov) ap_a;
  uvm_analysis_imp_intf_b #(<intf_b>_seq_item, <dut>_func_cov) ap_b;

  function void write_intf_a(<intf_a>_seq_item t); ... endfunction
  function void write_intf_b(<intf_b>_seq_item t); ... endfunction
endclass
```

---

## 执行步骤

1. 汇总所有 TC 的 Coverage 字段，按接口分组（确认每条 Coverage 来自哪个 Monitor）
2. 为每条 TC 创建对应 covergroup，bins 直接映射 Coverage 字段描述
3. 实现 `write()` 函数，加入必要的条件采样逻辑
4. 生成 `<dut>_func_cov.sv`

完成后向用户汇报：

```
功能覆盖交付摘要 — <DUT 名称>
────────────────────────────────────────
tb/env/<dut>_func_cov.sv：
  Covergroup 总数：<N> 个（每条 TC 一个）
  Coverpoint 总数：<n> 个
  Cross 覆盖：<n> 个
────────────────────────────────────────
待确认事项（若有）：
  - TP-XXX：Coverage 描述"覆盖所有合法输入"未明确 bin 划分，请确认
```

询问：**"以上覆盖率定义是否符合预期？如有 bin 需要调整，请指出。"**
用户确认后进入第 8 步。

---

## 暂停交互规则

| 情况 | 询问 |
|------|------|
| Coverage 字段描述模糊（"覆盖所有输入"） | "TP-XXX 的 Coverage 未指明具体 bin 划分，请提供枚举值或范围" |
| 同一信号在多条 TC 的 bins 有重叠 | "TP-XXX 和 TP-YYY 的 Coverage 覆盖同一信号的相同 bin，是否合并为一个 covergroup？" |
| Coverage 字段引用了 tb-arch 中未出现的信号 | "Coverage 引用信号 `<signal>`，但该信号未在 seq_item 或 Monitor 输出中定义，请确认" |

---

## 质量检查清单

- [ ] 每条 TC 对应一个独立命名的 covergroup（命名含 TC ID）
- [ ] 每个 bin 均直接来源于 Coverage 字段，无自行添加的额外 bin
- [ ] 条件采样逻辑正确——不会在无关事务上触发错误 covergroup
- [ ] 多接口场景下，所有 analysis_imp 均已声明且命名不冲突
- [ ] 所有 covergroup 在构造函数中完成初始化（`new()`）
- [ ] 文件顶部注释覆盖 TC ID 与 Coverage 字段摘要
