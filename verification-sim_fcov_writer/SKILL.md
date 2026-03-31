---
name: verification-sim_fcov_writer
description: "读取 SIM 测试用例三元组中每条 TC 的 Coverage 字段，结合 TB 架构文档中的 Monitor/Subscriber 分配，为每条 TC 编写 UVM 功能覆盖率代码（covergroup / coverpoint / bins）。输出为 tb/env/<dut>_func_cov.sv 覆盖率收集器文件。触发条件：用户需要为已定义的 TC Coverage 生成功能覆盖率代码时触发。"
---

# 功能覆盖率编写

## 概述

**目标**：依据 SIM 测试用例三元组中每条 TC 的 **Coverage 字段**，结合 TB 架构文档确定的 Monitor/Subscriber 架构，实现 `<dut>_func_cov.sv`——包含所有 covergroup 定义及采样触发逻辑。

**输出**：`tb/env/<dut>_func_cov.sv`

---

## 第一阶段：输入约束收集

> **Workflow 上下文**：若本 skill 由 `verification-workflow` 编排器调用，DUT 名称、`verification-sim-tc-defines.md`、`verification-tb-arch.md`、已有 seq_item 定义、Coding Style 结论已由前序步骤产出并存在于全局上下文中，直接使用，**不重复询问用户**。
>
> 若本 skill 被**独立使用**（非 workflow 上下文），则按下表逐项收集。

在开始编码之前，必须收集以下信息。对于尚未提供的信息，主动询问。

| 信息项 | 获取方式 | 缺失时处理 |
|--------|---------|-----------|
| SIM 测试用例三元组（`verification-sim-tc-defines.md` 或等效描述） | 询问用户提供 | 必须获取，每条 TC 的 Coverage 字段是编码直接依据 |
| TB 架构文档（`verification-tb-arch.md` 或等效描述） | 询问用户提供 | 必须获取，确认 coverage collector 连接哪些 Monitor 的 analysis port |
| 已有 seq_item 类定义（若 TB 代码已生成） | 询问路径 | 若有，确认采样字段名称 |
| UVM/TB coding style 规范（若项目有要求） | 询问用户 | 无则使用本文档内置默认规范 |
| DUT 名称 | 询问用户 | 必须获取，用于文件/类命名 |

---

## 第二阶段：Coverage 字段解析规则

读取每条 TC 的 Coverage 字段，将自然语言描述映射为 covergroup 元素：

| Coverage 描述模式 | 映射为 |
|-----------------|------|
| "必须采样到信号 X = 值 V" | `coverpoint x { bins v = {V}; }` |
| "覆盖 X 的低/中/高边界（A, B, C）" | `coverpoint x { bins lo = {A}; bins mid = {B}; bins hi = {C}; }` |
| "采样 X 处于范围 [A:B]" | `coverpoint x { bins range = {[A:B]}; }` |
| "状态转移 S1→S2" | `coverpoint state { bins s1_to_s2 = (S1 => S2); }` |
| "X 和 Y 的交叉覆盖" | `cross x_cp, y_cp;` |
| "握手成立（valid && ready）" | `coverpoint valid_ready_handshake { bins handshake = {1}; }` |

若 Coverage 字段描述模糊（如"覆盖所有合法输入"），**暂停并向用户确认**具体 bin 划分，不得自行扩展。

---

## 编码规范（默认）

若项目未提供专项 coding style 要求，使用以下默认规范：

| 要素 | 规范 |
|------|------|
| 类名 | `<dut>_func_cov` |
| 文件名 | `<dut>_func_cov.sv` |
| 继承 | `uvm_subscriber #(<intf>_seq_item)`（仅监听一个接口）或 `uvm_component`（多接口） |
| 注册 | `` `uvm_component_utils(<dut>_func_cov) `` |
| Covergroup 命名 | `cg_tp_<tc_id>`，例如 `cg_tp_func_001` |
| 采样触发 | 在 `write()` 函数中调用 `<cg>.sample()` |

---

## 第三阶段：执行步骤

### Step 1：Coverage 字段汇总

汇总所有 TC 的 Coverage 字段，按接口分组，确认每条 Coverage 来自哪个 Monitor 的 analysis port。

### Step 2：逐条实现 Covergroup

对每条 TC，按以下模板实现（每条 TC 独立一个 covergroup）：

```systemverilog
class <dut>_func_cov extends uvm_subscriber #(<intf>_seq_item);
  `uvm_component_utils(<dut>_func_cov)

  // ── TC: TP-FUNC-001 — <TC 描述> ─────────────────────
  // Coverage: <Coverage 字段原文摘要>
  covergroup cg_tp_func_001;
    cp_<signal>: coverpoint trans.<field> {
      bins <bin_name> = {<value>};
    }
    // 若有交叉覆盖：
    // cx_<a>_<b>: cross cp_<a>, cp_<b>;
  endgroup

  // ── TC: TP-FUNC-002 — <TC 描述> ─────────────────────
  covergroup cg_tp_func_002;
    ...
  endgroup

  <intf>_seq_item trans;

  function new(string name, uvm_component parent);
    super.new(name, parent);
    cg_tp_func_001 = new();
    cg_tp_func_002 = new();
  endfunction

  function void write(<intf>_seq_item t);
    trans = t;
    // 根据事务类型触发对应 covergroup
    cg_tp_func_001.sample();
    cg_tp_func_002.sample();
  endfunction

endclass
```

**关键实现规则**：

- **每条 TC 独立 covergroup**：即使多条 TC 采样相同信号，也不合并 covergroup，保持与 TC ID 一对一的可追溯性
- **条件采样**：若 Coverage 字段描述"当 X 为 Y 时才采样"，必须在 `write()` 中加 `if` 条件，不得无条件调用所有 `sample()`
- **状态转移覆盖**：使用 coverpoint 的序列 bin 语法 `bins name = (S1 => S2)` 实现，不使用临时变量拼凑

### Step 3：多接口 Analysis Imp（若需要）

当需要监听多个 Monitor 时：

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

## 暂停交互规则

| 情况 | 询问 |
|------|------|
| Coverage 字段描述模糊（"覆盖所有输入"） | "TP-XXX 的 Coverage 未指明具体 bin 划分，请提供枚举值或范围" |
| 同一信号在多条 TC 的 bins 有重叠 | "TP-XXX 和 TP-YYY 的 Coverage 覆盖同一信号的相同 bin，是否合并为一个 covergroup？" |
| Coverage 字段引用了未在 seq_item 中定义的信号 | "Coverage 引用信号 `<signal>`，但该信号未在 seq_item 或 Monitor 输出中定义，请确认" |

---

## 质量检查清单

- [ ] 每条 TC 对应一个独立命名的 covergroup（命名含 TC ID）
- [ ] 每个 bin 均直接来源于 Coverage 字段，无自行添加的额外 bin
- [ ] 条件采样逻辑正确——不会在无关事务上触发错误 covergroup
- [ ] 多接口场景下，所有 analysis_imp 均已声明且命名不冲突
- [ ] 所有 covergroup 在构造函数中完成初始化（`new()`）

---

## 输出交付物格式

文件名：`tb/env/<dut>_func_cov.sv`

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

询问："以上覆盖率定义是否符合预期？如有 bin 需要调整，请指出。"
