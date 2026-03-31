---
name: verification-sim_stim_writer
description: "读取 SIM 测试用例三元组中每条 TC 的 Stim 字段，结合 TB 架构文档中的 Agent/Sequencer 分配，为每条 TC 编写 UVM Sequence 激励代码。输出为 tb/sequences/ 目录下每条 TC 对应的 .sv 激励序列文件。触发条件：用户需要为已定义的 TC Stim 生成 UVM Sequence 代码时触发。"
---

# SIM 激励序列编写

## 概述

**目标**：依据 SIM 测试用例三元组中每条 TC 的 **Stim 字段**，结合 TB 架构文档确定的 Agent 分配，逐条实现 UVM Sequence 激励源文件，交付 `tb/sequences/` 目录。

**输出**：`tb/sequences/` 目录下的 `.sv` 文件，每条 TC 对应一个 Sequence 类

---

## 第一阶段：输入约束收集

> **Workflow 上下文**：若本 skill 由 `verification-workflow` 编排器调用，DUT 名称、`verification-sim-tc-defines.md`、`verification-tb-arch.md`、已有 seq_item 定义、Coding Style 结论已由前序步骤产出并存在于全局上下文中，直接使用，**不重复询问用户**。
>
> 若本 skill 被**独立使用**（非 workflow 上下文），则按下表逐项收集。

在开始编码之前，必须收集以下信息。对于尚未提供的信息，主动询问。

| 信息项 | 获取方式 | 缺失时处理 |
|--------|---------|-----------|
| SIM 测试用例三元组（`verification-sim-tc-defines.md` 或等效描述） | 询问用户提供 | 必须获取，这是 Stim 编码的直接依据 |
| TB 架构文档（`verification-tb-arch.md` 或等效描述） | 询问用户提供 | 必须获取，确认每条 TC 关联的 Agent/Sequencer 及 seq_item 结构 |
| 已有的 seq_item 类定义（若 TB 骨架代码已生成） | 询问路径 | 若有，确认字段名称以避免不一致 |
| UVM/TB coding style 规范（若项目有要求） | 询问用户是否有 coding style skill 或文档 | 无则使用本文档内置默认规范 |
| 项目 DUT 名称（用于文件/类命名前缀） | 询问用户 | 必须获取 |

---

## 第二阶段：编码前确认

**Seq Item 字段约定**：在编写 Sequence 之前，确认各接口 `seq_item` 中可随机化字段与 Stim 描述的激励维度对应关系。若 Stim 引用的字段在 TB 架构文档中未定义，**立即暂停并向用户确认字段名称**，不得自行定义新字段名。

---

## 编码规范（默认）

若项目未提供专项 coding style 要求，使用以下默认规范：

| 要素 | 规范 |
|------|------|
| 类名 | `<dut>_<tc_id>_seq`，例如 `uart_tp_func_001_seq` |
| 文件名 | 与类名一致，后缀 `.sv` |
| 继承 | 继承自 `uvm_sequence #(<intf>_seq_item)` |
| 注册 | `` `uvm_object_utils(<classname>) `` |
| 构造函数 | `function new(string name = "<classname>"); super.new(name); endfunction` |
| body() | `task body(); ... endtask` |
| 发送事务 | `start_item(req); assert(req.randomize() with {...}); finish_item(req);` |
| 注释头 | 每个文件顶部注释标注关联 TC ID 和 Stim 摘要 |

---

## 第三阶段：执行步骤

### Step 1：建立 TC → Sequence 映射表

遍历 SIM 测试用例三元组，列出所有 TC 的 ID 和 Stim 摘要，从 TB 架构文档中确认每条 TC 对应的目标 Agent 和 seq_item 类名。

| TC ID | Stim 摘要 | 目标 Agent | seq_item |
|-------|---------|-----------|---------|
| TP-FUNC-001 | ... | apb_agent | apb_seq_item |

### Step 2：逐条实现 Sequence

对每条 TC，按以下模板实现：

```systemverilog
// TC: <TC_ID> — <TC 描述>
// Stim: <Stim 字段原文摘要>
class <dut>_<tc_id>_seq extends uvm_sequence #(<intf>_seq_item);
  `uvm_object_utils(<dut>_<tc_id>_seq)

  // Stim 中的随机化参数暴露为 public rand 字段
  rand int unsigned num_txn;
  constraint c_num_txn { num_txn inside {[1:16]}; }

  function new(string name = "<dut>_<tc_id>_seq");
    super.new(name);
  endfunction

  task body();
    // --- 前置条件 ---
    // (若需要等待复位或特定状态，在此处理)

    // --- 激励主体（直接对应 Stim 字段描述） ---
    repeat(num_txn) begin
      req = <intf>_seq_item::type_id::create("req");
      start_item(req);
      assert(req.randomize() with {
        // 约束直接映射 Stim 中的合法范围描述
      }) else `uvm_fatal("SEQ", "Randomization failed")
      finish_item(req);
    end
  endtask
endclass
```

**关键实现规则**：

- **前置条件**：若 Stim 描述"复位释放后"、"FSM 处于 IDLE"等状态，使用 `wait(vif.<signal>)` 或延迟拍数实现，不得跳过
- **时序约束**：Stim 中描述的"先 A 后 B，间隔 N 拍"，必须用 `repeat(N) @(posedge vif.clk)` 精确实现
- **随机化范围**：Stim 中给出明确范围的，必须写入 `constraint` 块；未给出范围的，使用 `rand` 字段不加约束，并在注释中标注"Stim 未约束"
- **多步骤激励**：若 Stim 描述多个阶段，拆分为顺序 `start_item/finish_item` 块，每阶段加注释

### Step 3：Virtual Sequence（若架构含 Virtual Sequencer）

若 TB 架构文档中规划了 Virtual Sequencer，为每条涉及跨 Agent 编排的 TC 额外生成 `<dut>_<tc_id>_vseq.sv`，在其 `body()` 中按 Stim 描述的跨接口时序编排各子 sequence：

```systemverilog
class <dut>_<tc_id>_vseq extends uvm_sequence;
  `uvm_object_utils(<dut>_<tc_id>_vseq)
  `uvm_declare_p_sequencer(<dut>_vseqr)

  task body();
    <intf_a>_<tc_id>_seq seq_a = <intf_a>_<tc_id>_seq::type_id::create("seq_a");
    <intf_b>_<tc_id>_seq seq_b = <intf_b>_<tc_id>_seq::type_id::create("seq_b");
    seq_a.start(p_sequencer.intf_a_seqr);
    seq_b.start(p_sequencer.intf_b_seqr);
  endtask
endclass
```

---

## 暂停交互规则

遇到以下情况**必须暂停，向用户确认后再继续**：

| 情况 | 询问 |
|------|------|
| Stim 中时序未给出具体拍数 | "TP-XXX 中 `<信号>` 保持时间未明确，请确认拍数或随机范围" |
| Stim 引用了 seq_item 中不存在的字段 | "Stim 引用字段 `<field>`，但 seq_item 中未定义，请确认字段名" |
| Stim 描述跨 Agent 时序但架构中无 Virtual Sequencer | "TC `<id>` 需要跨 Agent 编排，但 TB 架构未规划 Virtual Sequencer，是否补充？" |

---

## 质量检查清单

- [ ] 每条 TC 的 Sequence 文件顶部有 TC ID 和 Stim 摘要注释
- [ ] 所有前置条件（复位、状态等待）均已实现，未被跳过
- [ ] 所有显式时序约束（N 拍延迟、先后顺序）已用代码精确表达
- [ ] 所有 Stim 中明确给出的随机化范围已转化为 `constraint` 块
- [ ] 每个 `start_item` 都有对应的 `finish_item`，无遗漏
- [ ] 跨 Agent TC 已生成对应 vseq，且跨 Agent 时序正确

---

## 输出交付物格式

目录：`tb/sequences/`

```
tb/sequences/
  <dut>_<tc_id>_seq.sv    × <N> 条 TC（每条一个文件）
  <dut>_<tc_id>_vseq.sv   × <n> 条（跨 Agent TC，若有）
```

完成后向用户汇报：

```
激励序列交付摘要 — <DUT 名称>
────────────────────────────────────────
tb/sequences/ 目录：
  Sequence 文件：<N> 个
  Virtual Sequence 文件：<n> 个（跨 Agent TC）
────────────────────────────────────────
待确认事项（若有）：
  - TP-XXX：Stim 时序 <X> 拍未明确，已默认设为 1 拍，请确认
```

询问："以上激励序列是否符合预期？如有时序或约束需要调整，请指出。"
