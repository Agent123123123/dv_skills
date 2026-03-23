---
name: verification_sim_stim_writer
description: "verification_workflow 流水线的第 6 步。读取第 4 步输出的 verification-sim-tc-defines.md 中每条 TC 的 Stim 字段，结合第 5 步输出的 verification-tb-arch.md 中的 Agent/Sequencer 分配，为每条 TC 编写 UVM Sequence 激励代码。若存在 UVM/TB coding style 相关 skill，在开始编码前必须先加载。输出为 tb/sequences/ 目录下每条 TC 对应的 .sv 激励序列文件。触发条件：由 verification_workflow 在第 6 步调用，或用户直接需要为已定义的 TC Stim 生成 UVM Sequence 代码时触发。"
---

# SIM 激励序列编写（第 6 步）

## 概述

本 skill 是 `verification_workflow` 流水线的**第 6 步**。

**目标**：依据 `verification-sim-tc-defines.md` 中每条 TC 的 **Stim 字段**，结合 `verification-tb-arch.md` 中确定的 Agent 分配，逐条实现 UVM Sequence 激励源文件，交付 `tb/sequences/` 目录。

**输入**：
- `verification-sim-tc-defines.md`（来自第 4 步）——每条 TC 的 Stim 字段：前置条件、激励动作、时序、随机化约束
- `verification-tb-arch.md`（来自第 5 步）——确认每条 TC 关联的 Agent/Sequencer 及 seq_item 结构

**输出**：`tb/sequences/` 目录下的 `.sv` 文件，每条 TC 对应一个 Sequence 类

---

## 前置：Coding Style Skill 检查

**开始编码前必须执行：** 扫描已加载 skill 列表，若存在名称含 `uvm`、`tb`、`coding-style` 的 skill，立即加载并遵循；否则使用默认规范，并告知用户：
> "未检测到 TB coding style skill，将采用内置默认 UVM 编码规范。"

---

## Seq Item 字段约定（与 tb_writer 对齐）

在编写 Sequence 之前，确认各接口 `seq_item` 中可随机化字段与 Stim 描述的激励维度对应关系。若 Stim 引用的字段在 `verification-tb-arch.md` 中未定义，**立即暂停并向用户确认字段名称**，不得自行定义新字段名。

---

## 编码规范（默认）

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

## 执行步骤

### Step 1：建立 TC → Sequence 映射表

遍历 `verification-sim-tc-defines.md`，列出所有 TC 的 ID 和 Stim 摘要，并从 `verification-tb-arch.md` 中确认每条 TC 对应的：
- 目标 Agent（哪个接口的 sequencer）
- seq_item 类名

输出映射表（仅内部使用，不需要写入文件）：

| TC ID | Stim 摘要 | 目标 Agent | seq_item |
|-------|---------|-----------|---------|
| TP-FUNC-001 | ... | apb_agent | apb_seq_item |

---

### Step 2：逐条实现 Sequence

对每条 TC，按以下模板实现：

```systemverilog
// TC: <TC_ID> — <TC 描述>
// Stim: <Stim 字段原文摘要>
class <dut>_<tc_id>_seq extends uvm_sequence #(<intf>_seq_item);
  `uvm_object_utils(<dut>_<tc_id>_seq)

  // Stim 中的随机化参数（如有）暴露为 public rand 字段
  rand int unsigned num_txn;  // 示例
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
- **随机化范围**：Stim 中给出明确范围（如"addr 范围 0~255"）的，必须写入 `constraint` 块；未给出范围的，使用 `rand` 字段不加约束，并在注释中标注"Stim 未约束"
- **多步骤激励**：若 Stim 描述多个阶段（配置→激活→观测），拆分为 `if-else` 或顺序 `start_item/finish_item` 块，每阶段加注释

---

### Step 3：Virtual Sequence（若架构含 Virtual Sequencer）

若 `verification-tb-arch.md` 中规划了 Virtual Sequencer，为每条涉及跨 Agent 编排的 TC 额外生成 `<dut>_<tc_id>_vseq.sv`，在其 `body()` 中按 Stim 描述的跨接口时序编排各子 sequence：

```systemverilog
class <dut>_<tc_id>_vseq extends uvm_sequence;
  `uvm_object_utils(<dut>_<tc_id>_vseq)
  `uvm_declare_p_sequencer(<dut>_vseqr)

  task body();
    <intf_a>_<tc_id>_seq seq_a = <intf_a>_<tc_id>_seq::type_id::create("seq_a");
    <intf_b>_<tc_id>_seq seq_b = <intf_b>_<tc_id>_seq::type_id::create("seq_b");
    // 按 Stim 时序描述串行或 fork-join 编排
    seq_a.start(p_sequencer.intf_a_seqr);
    seq_b.start(p_sequencer.intf_b_seqr);
  endtask
endclass
```

---

### Step 4：确认与交付

完成所有 Sequence 文件后，向用户汇报：

```
激励序列交付摘要 — <DUT 名称>
────────────────────────────────────────
tb/sequences/ 目录：
  <dut>_<tc_id>_seq.sv    × <N> 条 TC
  <dut>_<tc_id>_vseq.sv   × <n> 条（跨 Agent TC）
────────────────────────────────────────
待确认事项（若有）：
  - TP-XXX：Stim 时序 <X> 拍未明确，已默认设为 1 拍，请确认
```

询问用户：**"以上激励序列是否符合预期？如有时序或约束需要调整，请指出。"**
用户确认后进入第 7 步。

---

## 暂停交互规则

遇到以下情况**必须暂停，向用户确认后再继续**：

| 情况 | 询问 |
|------|------|
| Stim 中时序未给出具体拍数 | "TP-XXX 中 `<信号>` 保持时间未明确，请确认拍数或随机范围" |
| Stim 引用了 tb-arch 中不存在的 seq_item 字段 | "Stim 引用字段 `<field>`，但 seq_item 中未定义，请确认字段名" |
| Stim 描述跨 Agent 时序但架构中无 Virtual Sequencer | "TC `<id>` 需要跨 Agent 编排，但 tb-arch 未规划 Virtual Sequencer，是否补充？" |

---

## 质量检查清单

- [ ] 每条 TC 的 Sequence 文件顶部有 TC ID 和 Stim 摘要注释
- [ ] 所有前置条件（复位、状态等待）均已实现，未被跳过
- [ ] 所有显式时序约束（N 拍延迟、先后顺序）已用代码精确表达
- [ ] 所有 Stim 中明确给出的随机化范围已转化为 `constraint` 块
- [ ] 每个 `start_item` 都有对应的 `finish_item`，无遗漏
- [ ] 跨 Agent TC 已生成对应 vseq，且跨 Agent 时序正确
