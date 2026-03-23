---
name: verification_sim_tc_define
description: "verification_workflow 流水线的第 4 步。读取第 3 步输出的 verification-strategy.md，筛选所有手段为 SIM 的测试点，并为每个测试点精确定义三元组：Stim（如何触发）、Coverage（系统必须出现的状态）、Checker（系统绝不能出现的状态）。输出为 verification-sim-tc-defines.md，可直接指导 testbench 激励、覆盖组和断言的实现。触发条件：由 verification_workflow 在第 4 步调用，或用户直接需要为 SIM 测试点定义具体用例三元组时触发。"
---

# SIM 测试用例定义（第 4 步）

## 概述

本 skill 是 `verification_workflow` 流水线的**第 4 步**。

**目标**：针对 `verification-strategy.md` 中所有手段含 `SIM`（含 `FORMAL + SIM`）的测试点，精确定义**三元组**：

| 字段 | 定义 |
|------|------|
| **Stim（激励）** | 如何触发该测试点——描述初始条件、激励序列、关键事件及时序约束 |
| **Coverage（覆盖）** | 该测试点下系统**一定要出现**的状态——描述仿真过程中必须采样到的条件、值、时序 |
| **Checker（检查）** | 该测试点下系统**绝不能出现**的状态——描述仿真过程中必须始终为假的条件，即断言的否定面 |

**输入**：
- `verification-strategy.md`（来自第 3 步）——提取手段含 SIM 的测试点
- DUT Spec 文档——用于补充信号含义、协议规则、边界约束
- RTL 源代码——用于确认信号名称、状态机状态、数据宽度

**输出**：`verification-sim-tc-defines.md` —— 每条 SIM 测试点的三元组定义表

---

## 三元组定义规范

### Stim（激励）

描述**如何主动触发**该测试点，应包含：

- **前置条件**：DUT 需处于何种初始状态（如：复位完成、FIFO 为空、FSM 处于 IDLE）
- **激励动作**：施加什么输入序列（信号名称、值、持续时间、顺序）
- **关键时序**：关键事件发生的先后顺序（如：先拜高 valid，再 N 拍后施加 data）
- **是否需要随机化**：若 Stim 中存在可随机化的参数（如数据值、延迟周期数），明确列出随机化范围或约束

写法示例：
> "在复位释放后，将 `req` 拉高并保持 1 拍，同时驱动随机合法 `addr`（范围 0~255）；随后在 rand[1~4] 拍延迟后再拉低 `req`。"

---

### Coverage（覆盖）

描述在该测试点执行期间，仿真**必须采样到**的状态组合。对应 covergroup 的 bins 或 cover property。

应描述：
- 哪个信号或信号组合处于何种值/状态
- 该条件持续多长时间或在哪个时钟沿被采样
- 若涉及交叉覆盖（cross coverage），明确两个维度

写法示例：
> "必须采样到 `grant` 与 `req` 同时为高的周期（握手成功）；必须覆盖 `addr` 的低边界（0）、高边界（255）和中间值（128）三个 bin。"

---

### Checker（检查）

描述在该测试点执行期间，仿真**绝对不允许出现**的状态。对应 SVA `assert property` 或 `always@(posedge clk) assert()`。

应描述：
- 违规条件的信号组合及时序
- 采用"若 A 则非 B"的蕴含式格式，便于直接映射到 SVA

写法示例：
> "若 `grant` 为高，则 `err` 必须为低（|-> !err）；`req` 未拜高时，`grant` 不得为高（!req |-> !grant）。"

---

## 执行步骤

1. **筛选 SIM 测试点**  
   从 `verification-strategy.md` 的策略表中，提取所有 **手段 = `SIM` 或 `FORMAL + SIM`** 的行，形成工作列表。

2. **逐条定义三元组**  
   对工作列表中每条测试点，依次基于 Spec 和 RTL 分析，填写 Stim / Coverage / Checker。

3. **信号名精确化**  
   三元组中所有信号名均使用 RTL 顶层端口或内部关键信号的**实际名称**（不允许使用"某信号"、"输出端口"等模糊表述）。

4. **随机化约束提炼**  
   若 Stim 中含有可随机化的维度，在 Coverage 中同步补充对应的 bins 覆盖需求，确保随机化有意义地命中目标空间。

5. **确认环节**  
   完成全部三元组草稿后，向用户显示汇总表，重点标出 P0 测试点的三元组，询问：
   > "以上三元组定义是否准确？是否需要调整任何测试点的 Stim / Coverage / Checker？"
   
   根据用户反馈修改后，保存为 `verification-sim-tc-defines.md`。

---

## 输出格式

生成 `verification-sim-tc-defines.md`，结构如下：

```markdown
# SIM 测试用例三元组定义 — <DUT 名称>

## 摘要
- 本文件覆盖 <N> 条 SIM 测试点
- P0：<n> 条 | P1：<n> 条 | P2：<n> 条 | P3：<n> 条

---

## 测试点三元组

### TP-FUNC-001 — <测试点简要描述>

**优先级**：P1  
**所属目标**：<来自 verification-targets.md 的目标 ID>

| 字段 | 定义 |
|------|------|
| **Stim** | 前置条件：复位释放后，FSM 处于 IDLE。激励：拉高 `start` 1 拍，同时驱动合法 `cmd`（0x01 / 0x02 / 0x03 每种至少各驱动一次）。时序：`start` 拉低后，等待 `done` 拉高。 |
| **Coverage** | 采样 `done` 拉高时 `cmd` 的值，需覆盖 0x01、0x02、0x03 三个 bin；采样 FSM 经过 BUSY→DONE 的状态转移。 |
| **Checker** | `start` 未拜高时，`done` 不得为高（`!start \|-> !done`）；`done` 拜高期间，`err` 必须为低（`done \|-> !err`）。 |

---

### TP-FUNC-002 — <测试点简要描述>

（格式同上，每条测试点一节）

---

## 备注与假设
<分析过程中依赖的假设，以及与 Spec 存在歧义之处>
```

---

## 质量检查清单

在保存输出前，逐条确认：

- [ ] 每个三元组的 Stim 中**不存在模糊信号名**（必须是 RTL 实际信号）
- [ ] Coverage 的每个 bin 与 Stim 的激励范围**相互覆盖**（激励真的能触发该 bin）
- [ ] Checker 条件与 Spec 的功能要求**直接对应**（能从 Spec 找到依据）
- [ ] P0 测试点的 Checker 至少包含**一条安全性断言**（系统级不变式）
- [ ] 不存在三元组中 Coverage 和 Checker 同时描述**同一信号同一值**的矛盾（即：不允许某状态出现、同时又要求覆盖该状态）

---

## 输出交接

本步骤完成后：

1. 告知用户：**"第 4 步已完成。所有 SIM 测试点的 Stim / Coverage / Checker 三元组已定义完毕。"**
2. 说明后续可用输出：
   - Stim → 直接指导 driver / sequence 实现
   - Coverage → 直接映射为 covergroup bins 和 cover property
   - Checker → 直接映射为 SVA assert property 或 checker module
3. 提示用户：**"验证规划流水线已全部完成。可以开始 testbench 架构设计和用例实现。"**
