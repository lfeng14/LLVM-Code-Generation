# LLVM CodeGen 第18章：指令调度（Instruction Scheduling）完整学习笔记

***

## 一、章节概述

指令调度（Instruction Scheduling）是一种**底层代码优化**，通过重新排列指令的执行顺序来提升程序性能。本章内容是理解和修改现有 LLVM 开源后端调度启发式算法的基础，也是实现自定义调度策略的关键。[1]

**两大优化目标**：[1]

- **降低寄存器压力**：减少需要同时分配的寄存器数量，避免 spill 到内存
- **提高指令级并行度（ILP）**：增加可以并行执行的指令数量，充分利用硬件资源

**两个学习重点**：[1]

1. 如何调整调度启发式策略（Scheduling Heuristics）
2. 如何为目标架构实现调度模型（Scheduling Model）

***

## 二、背景知识：In-order vs Out-of-order 处理器

理解调度的重要性，首先要理解处理器的执行方式。[1]

| 特性 | **In-order（顺序执行）** | **Out-of-order（乱序执行）** |
|------|------------------------|----------------------------|
| 执行顺序 | 严格按程序代码顺序执行 | 硬件动态检测依赖，自动重排 |
| 遇到停顿 | 后续**所有**指令被迫等待 | 跳过停顿指令，优先执行无依赖的指令 |
| 编译器调度重要性 | **极高**，直接决定性能 | 中等，硬件可部分弥补 |
| 硬件复杂度 | 低、功耗小 | 高（需要 ROB、寄存器重命名逻辑） |
| 典型应用 | 嵌入式设备（ARM Cortex-A53）| 高性能 CPU（Intel Core、Apple M 系列）|

**关键结论**：指令调度对 in-order 处理器的性能提升尤为显著。乱序处理器可以动态适配执行，而顺序处理器完全依赖编译器的静态调度决策。[1]

### 延迟隐藏示例（in-order 处理器）

**未调度代码**：

```assembly
LD  R1, 0(R2)    ; 3 周期延迟
ADD R3, R1, R4   ; 依赖 R1，必须等待
MUL R5, R6, R7   ; 完全独立，但被迫停顿
```

执行时间线（共 **8 周期**）：

```
周期: 1    2    3    4    5    6    7    8
LD   IF   ID   EX   MEM  WB
ADD       IF   ID   停   停   EX   WB
MUL            IF   停   停   ID   EX   WB
```

**调度后代码**：

```assembly
LD  R1, 0(R2)    ; 提前发射，MUL 掩盖延迟
MUL R5, R6, R7   ; 独立指令，先执行
ADD R3, R1, R4   ; 此时 R1 已就绪
```

执行时间线（共 **5 周期**）：

```
周期: 1    2    3    4    5
LD   IF   ID   EX   MEM  WB
MUL       IF   ID   EX   WB
ADD            IF   ID   EX   WB
```

性能提升：减少 3 个停顿周期，总执行时间缩短 37.5%。[1]

***

## 三、LLVM 调度框架的三大核心组件

整个调度框架由三个主要部分组成，`MachineScheduler` pass 负责将它们串联起来。[1]

```
MachineScheduler pass
        │
        ▼
┌───────────────────────┐
│  ScheduleDAGInstrs    │  ← 构建 DDG + 执行调度策略
│  ┌─────────────────┐  │
│  │ DDG（依赖图）   │  │
│  └─────────────────┘  │
│  ┌─────────────────┐  │
│  │ MachineSchedStrategy │
│  └─────────────────┘  │
└───────────────────────┘
        │
        ▼
┌───────────────────────┐
│  TargetSchedModel     │  ← 描述硬件资源和指令特性
└───────────────────────┘
```

### 3.1 数据依赖图（DDG）— `ScheduleDAGInstrs` 类

**概念**：DDG 表示调度区域内指令之间所有的调度约束，描述了指令出现在最终基本块中的弱偏序关系。[1]

**重要细节：边的方向**（反直觉！）：[1]
- 图中的边表示 use-def chain（使用-定义链）
- **若存在边 A → B，则 B 必须排在 A 之前**
- 原因：B 使用了 A 的定义，所以 B（消费者）排在前面，A（生产者）在后面

**`ScheduleDAGInstrs` 的两大职责**：[1]
1. 构建 DDG（包括 producer/consumer 数据依赖 + 内存依赖约束）
2. 使用绑定的调度策略来调度 DDG

默认实现类为 `ScheduleDAGMILive`，它在调度 `MachineInstr` 的同时追踪寄存器活跃性（liveness）和寄存器压力。[1]

**自定义入口**：[1]

```cpp
// 重写 TargetPassConfig 的此方法来替换默认实现
ScheduleDAGInstrs *
MyPassConfig::createMachineScheduler(MachineSchedContext *C) const {
    ScheduleDAGMILive *DAG = new ScheduleDAGMILive(
        C, std::make_unique<MySchedStrategy>(C));
    // 在此添加 Mutations
    return DAG;
}
```

#### Mutations（DDG 变异）

Mutations 允许在 DDG 完全构建后对其进行修改，向调度器添加额外约束。[1]

**使用场景**：已知特定指令顺序能让目标架构性能更好，但 DDG 默认不包含该约束。

```
原始 DDG：                  Mutated DDG：
instrA                      instrA
  ├── instrB    →→→         ├── instrB ─┐
  └── instrC                └── instrC  ← 添加约束，强制 B 在 C 之前
       └── instrD                └── instrD
```

**实现方式**：[1]

```cpp
// 继承 ScheduleDAGMutation 并实现 apply 方法
class MyMutation : public ScheduleDAGMutation {
public:
    void apply(ScheduleDAGInstrs *DAG) override {
        // 修改 DAG，例如添加新边
        // ⚠️ 注意：不能创建循环，否则调度器无解
    }
};

// 注册到调度器
DAG->addMutation(std::make_unique<MyMutation>());
```

**内置 Mutations（直接可用）**：[1]
- `createLoadClusterDAGMutation`：将相同类型的 load 聚簇
- `createStoreClusterDAGMutation`：将相同类型的 store 聚簇

***

### 3.2 调度策略 — `MachineSchedStrategy` 类

**概念**：调度策略是调度算法的核心，决定每一步从 ready queue 中挑选哪条指令。[1]

**Ready Queue（就绪队列）**：[1]
> 所有依赖已被满足、可以立即调度的指令集合。
> - Top-down 调度器：所有**后继**节点已调度完的指令进入 ready queue
> - Bottom-up 调度器：所有**前驱**节点已调度完的指令进入 ready queue

**推荐基类**：[1]
- `GenericScheduler`：用于寄存器分配前（Pre-RA）调度
- `PostGenericScheduler`：用于寄存器分配后（Post-RA）调度

相比直接重写 `pickNode`，推荐重写 `tryCandidate` 方法——它只关注"哪个候选更优"这一比较逻辑，更易于实现。[1]

#### 自定义策略完整示例

以下示例来自 H2BLB 目标后端，优先调度 `mayLoad` 和 `WIDENING_SMUL` 指令：[1]

```cpp
class H2BLBPreRASchedStrategy : public GenericScheduler {
public:
    H2BLBPreRASchedStrategy(const MachineSchedContext *C)
        : GenericScheduler(C) {}

protected:
    bool tryCandidate(SchedCandidate &Cand, SchedCandidate &TryCand,
                      SchedBoundary *Zone) const override {
        
        // Step 1：先调用默认启发式
        bool BetterCand = GenericScheduler::tryCandidate(Cand, TryCand, Zone);
        
        // Step 2：如果默认选择是真正更优的，直接采纳
        if (BetterCand && TryCand.Reason != NodeOrder
                       && TryCand.Reason != NoCand)
            return true;
        
        // Step 3：自定义逻辑（只在同一 Region 内生效）
        if (Zone != nullptr) {
            // 优先调度 load 指令（隐藏内存延迟）
            if (TryCand.SU->getInstr()->mayLoad()) {
                TryCand.Reason = Stall;
                return true;
            }
            // 优先调度 WIDENING_SMUL（2 周期延迟）
            if (TryCand.SU->getInstr()->getOpcode() == H2BLB::WIDENING_SMUL) {
                TryCand.Reason = Stall;
                return true;
            }
        }
        
        return TryCand.Reason != NoCand;
    }
};
```

**细节解析**：
- `TryCand.Reason == NodeOrder`：仅因原始顺序选出，不是真正的优化选择
- `TryCand.Reason == NoCand`：没有其他候选，被迫选择
- 以上两种情况说明默认选择并不"聪明"，此时可注入自定义逻辑[1]
- `TryCand.Reason = Stall`：表明此指令若不提前调度，将导致流水线停顿

#### Scheduling Regions（调度区域）

调度区域是连续的指令序列，**最大不超过一个基本块**，LLVM 不支持跨 BB 的超级块调度。[1]

可通过重写 `TargetInstrInfo` 的 `isSchedulingBoundary` 方法来控制 Region 的边界划分。[1]

#### 调度方向（Scheduling Direction）

**三种方向**：[1]

| 方向 | 从哪里开始 | Ready Queue 内容 | 适合场景 |
|------|-----------|-----------------|---------|
| **Top-down** | Region 顶部 → 底部 | 无后继的节点 | 减少生产者到消费者的延迟 |
| **Bottom-up** | Region 底部 → 顶部 | 无前驱的节点 | 降低寄存器压力 |
| **Bidirectional**（默认）| 动态选择 | 两端 ready queue 均参与 | 平衡两个目标 |

**设置方式**：[1]

```cpp
// 在 TargetSubtargetInfo 子类中重写
void MySubtarget::overrideSchedPolicy(MachineSchedPolicy &Policy,
                                      unsigned NumRegionInstrs) const {
    Policy.OnlyTopDown = true;   // 强制 top-down
    Policy.OnlyBottomUp = false;
}
```

***

### 3.3 调度模型 — `TargetSchedModel` 类

**概念**：调度模型描述目标子架构的硬件资源信息，告诉调度器每条指令的延迟、使用的处理单元等特性。[1]

> 注意：调度模型与**子架构（subtarget）**绑定，而非整个 target。例如 x86 的 Haswell 和 Sapphire Rapids 有不同的调度模型。[1]

有两种建模方式：[1]
- **Scheduling Events（调度事件）**：推荐方式，适用于绝大多数目标
- **Instruction Itineraries（指令流水线）**：仅推荐用于 in-order VLIW 处理器

#### 调度模型的四大 TableGen 类

**整体架构**：[1]

```
指令 (opc)
├── ReadArg1   ─────→  ALU (latency: -1)  ← 转发路径
├── ReadArg2   ─────→  ALU (latency: 0)
└── WriteDef0  ─────→  ALU (latency: 1)
                └────→ LSUnit (latency: 3)  ← 如果是 load
```

**（1）`SchedMachineModel`**：调度模型的顶层类，聚合所有资源和绑定信息[1]

**（2）`SchedReadWrite` 及子类**：调度事件，附加到指令操作数上[1]
- `SchedWrite`：附加到**定义**（definitions）操作数
- `SchedRead`：附加到**使用**（uses）操作数
- 约束：每个操作数最多一个调度事件；每条指令至少一个 SchedWrite 事件

**（3）`ProcResourceUnits` 及子类**：处理单元，描述硬件资源[1]

```tablegen
def ALURes     : ProcResource<1>;        // 1 个 ALU 单元
def MemRes     : ProcResource<1>;        // 1 个 Memory 单元
def MemAndALU  : ProcResGroup<[MemRes, ALURes]>;  // 组合资源
// 内存操作同时需要 MemRes（读写）和 ALURes（地址计算）
```

`ProcResourceUnits` 的 `BufferSize` 字段用于区分 in-order 与 out-of-order 行为。[1]

**（4）`WriteRes` / `ReadAdvance` / `InstRW`**：绑定调度事件到处理资源[1]

***

## 四、调度绑定的两种方法

### 方法一：`WriteRes` + `ReadAdvance`（分散式）

调度事件分散在指令定义文件中，绑定在调度模型文件中：[1]

```tablegen
// 在指令定义文件中装饰指令
let SchedRW = [WriteWUMUL, ReadWUMULArg0, ReadWUMULArg1] in
def WIDENING_UMUL : H2BLBWidenIMul<"wumul", /*isSign=*/0>;

// 也可用继承方式
def WIDENING_SMUL : H2BLBWidenIMul<"wsmul", /*isSign=*/1>,
    Sched<[WriteWSMUL, ReadWSMULArg0, ReadWSMULArg1]>;

// 在调度模型中绑定资源
let SchedModel = H2BLBDefaultModel in {
    let Latency = 2 in
    def : WriteRes<WriteWSMUL, [ALURes]>;     // 2 周期，使用 ALU

    def : ReadAdvance<ReadWSMULArg0, 0>;      // 无转发
    def : ReadAdvance<ReadWSMULArg1, 1>;      // 有 1 周期转发路径
}
```

**`ReadAdvance` 细节**：
- 正数：表示提前 N 周期可读（转发路径，减少等待）
- 0：无效果
- 负数：表示延迟 N 周期（执行域切换惩罚）

### 方法二：`InstRW`（集中式，推荐）

调度事件和绑定全部集中在调度模型文件中：[1]

```tablegen
let SchedModel = H2BLBDefaultModel in {
    // 定义 load 指令的调度事件+绑定
    let Latency = 3 in
    def DefaultWriteLoad : SchedWriteRes<[MemRes]>;
    
    // 用正则表达式一次性装饰所有 load 指令（除 load immediate）
    def : InstRW<[DefaultWriteLoad], (instregex "^LD[^i]*$")>;
}
```

**两种方法对比**：

| | `WriteRes` + `ReadAdvance` | `InstRW`（推荐） |
|-|--------------------------|-----------------|
| 调度事件位置 | 指令定义文件 | 调度模型文件 |
| 绑定位置 | 调度模型文件 | 调度模型文件 |
| 模块化程度 | 低（混合在指令中）| 高（调度模型独立）|
| 多处理器复用 | 困难（事件共享问题）| 容易 |
| 兼容性 | 两种方法可混用 | 推荐默认选择 |

**为什么推荐 `InstRW`**：假设你定义了 `WriteFloat` 事件共享给 `fdiv` 和 `fmax`，当新处理器中两者延迟不同时，无法再共享同一个 `SchedWrite` 事件，模型需要大幅重构。而 `InstRW` 方式的绑定是局部的，只影响当前处理器的调度模型。[1]

***

## 五、调度模型的组装

### 5.1 `SchedMachineModel` 顶层类

```tablegen
def H2BLBDefaultModel : SchedMachineModel {
    let IssueWidth = 1;                    // 每周期发射指令数
    let MicroOpBufferSize = 0;             // 0 = in-order
    let LoadLatency = 4;                   // 未装饰 load 的默认延迟
    let CompleteModel = false;             // 完成后设为 true 以启用检查
}
```

**`CompleteModel` 的作用**：设为 `true` 后，TableGen 会在有指令未被调度模型覆盖时报错，防止团队协作时遗漏新指令。[1]

### 5.2 连接处理器模型

```tablegen
// 定义处理器（在 XXX.td 文件中）
def : ProcessorModel<"generic", H2BLBDefaultModel, []>;
//                    ↑ CPU名称    ↑ 调度模型         ↑ 特性集合
```

```cpp
// 在子架构构造函数中传入 CPU 名称
H2BLBSubtarget::H2BLBSubtarget(const Triple &TT, StringRef CPU,
                                StringRef FS, const TargetMachine &TM)
    : H2BLBGenSubtargetInfo(TT, CPU, /*TuneCPU=*/CPU, FS, ...) {}
```

> ⚠️ 若 TuneCPU 参数为空，LLVM 将使用默认调度模型，对所有目标均不精确，调度效果极差。[1]

### 5.3 未装饰指令的默认行为

- 未附加调度事件的指令默认假设为 **1 周期**[1]
- 带 `mayLoad` 属性但未装饰的指令，使用 `MCSchedModel::DefaultLoadLatency`（默认 4 周期）[1]
- **关键陷阱**：未装饰指令无法受益于 `ReadAdvance` 的转发路径，因为调度器找不到对应的 write 事件[1]

***

## 六、启用调度器

调度器默认**不启用**，需要在子架构中显式开启：[1]

```cpp
class MySubtarget : public TargetSubtargetInfo {
public:
    // 启用寄存器分配前调度（Pre-RA）
    bool enableMachineScheduler() const override {
        return true;
    }
    
    // 启用寄存器分配后调度（Post-RA）
    bool enablePostRAMachineScheduler() const override {
        return true;
    }
};
```

**与 SDISel 的交互**：启用 `MachineScheduler` 后，SDISel 在线性化 SelectionDAG 时会使用更简单的启发式，将复杂调度工作交给 `MachineScheduler` pass。若希望 SDISel 仍然使用自己的复杂调度，需重写 `enableMachineSchedDefaultSched` 并返回 `false`。[1]

***

## 七、Pre-RA vs Post-RA 调度

| 时机 | Pre-RA（寄存器分配前）| Post-RA（寄存器分配后）|
|------|---------------------|----------------------|
| 操作对象 | 虚拟寄存器（无限）| 物理寄存器（已固定）|
| 自由度 | 高 | 低（依赖关系更严格）|
| 风险 | 可能增加寄存器压力 | 无 spill 风险 |
| 策略基类 | `GenericScheduler` | `PostGenericScheduler` |
| 关注点 | ILP + 寄存器压力均衡 | 纯 ILP 和延迟隐藏 |

***

## 八、实现调度模型的推荐步骤

官方推荐按以下顺序使用 `InstRW` 方法逐步构建调度模型：[1]

1. **创建 `SchedMachineModel` 顶层实例**，设置 `IssueWidth`、`LoadLatency` 等基本参数
2. **确定 load 指令的默认延迟成本**（咨询硬件团队或用 `llvm-exegesis` 测量）
3. **描述处理单元**（从最常用的开始，如 ALU），设置 `BufferSize` 反映 in-order/out-of-order
4. **选择一组指令**（例如所有加法指令），创建其 `SchedWriteRes` 和 `SchedReadAdvance`
5. **用 `InstRW` 装饰该组指令**
6. **编写 `.mir` 测试用例**，验证调度前后的变化
7. **重复步骤 2-6**，逐步覆盖所有指令
8. **设置 `CompleteModel = true`**，让 TableGen 检查是否有遗漏指令

***

## 九、测试和调试工具

### 可视化 DDG

```bash
llc --view-misched-dags input.ll
# 输出 dot 格式图，用 graphviz 渲染可视化
```

### 分析微架构性能

```bash
llvm-mca input.s --mcpu=cortex-a53
# 分析吞吐量、IPC、资源使用情况
```

### 调试调度决策

```bash
llc -debug-only=machine-scheduler input.ll
# 查看每步调度的候选选择和原因
```

### MIR 测试文件格式

```llvm
# 只运行调度 pass，验证指令顺序变化
# RUN: llc -mtriple=... -run-pass=machine-scheduler %s -o - | FileCheck %s
---
name: test_func
body: |
  bb.0:
    %0:gpr = LD %1:gpr, 0
    %2:gpr = MUL %3:gpr, %4:gpr
    %5:gpr = ADD %0:gpr, %2:gpr
```

***

## 十、关键知识点汇总

### 调度效率考量维度

| 维度 | 考量内容 | 对应工具/方法 |
|------|---------|-------------|
| **指令延迟** | Load 3-4 周期，Mul 2 周期，提前调度 | `WriteRes` Latency 字段 |
| **资源冲突** | 避免多指令同时竞争同一处理单元 | `ProcResource` 建模 |
| **寄存器压力** | 减少同时活跃变量数，避免 spill | 调度方向选择（Bottom-up 有优势）|
| **转发路径** | 建模 pipeline forwarding 减少等待 | `ReadAdvance` 正数值 |
| **处理器类型** | In-order 重 ILP；Out-of-order 重寄存器压力 | `BufferSize` 字段 |

### 常见陷阱与注意事项

- **DDG 边方向反直觉**：`A → B` 表示 B 在 A 之前调度（B uses A's def）[1]
- **Mutation 不能创建环**：会导致调度器无法找到合法解[1]
- **TuneCPU 不能为空**：否则使用通用模型，调度效果极差[1]
- **未装饰指令**：无法受益于 `ReadAdvance` 转发，模型不完整时调度器可能过度或不足地隐藏延迟[1]
- **调度区域上限是 BasicBlock**：LLVM 不支持超级块（Super Block）调度[1]

### Quiz 解答（章节自测）

**Q1：DDG 代表什么？**  
DDG 表示调度区域内指令之间所有调度约束，即指令在最终基本块中出现顺序的弱偏序关系。[1]

**Q2：Mutations 的作用是什么？**  
允许在 DDG 完全构建后对其进行修改，例如通过添加额外的依赖边来限制调度器的自由度。[1]

**Q3：如何可视化查看 DDG？**  
使用 `--view-misched-dags` 命令行选项，输出 dot 格式图形。[1]

**Q4：用 `InstRW` 向调度模型添加指令的两个步骤？**  
1. 用 `SchedWriteRes` 和 `SchedReadAdvance` 描述调度事件，设置 `SchedModel` 字段；  
2. 用 `InstRW` 将这些调度信息绑定到目标指令。[1]

**Q5：实例化子架构时不设置处理器模型会怎样？**  
LLVM 将使用默认调度模型，调度器对子架构实际能力一无所知，很可能产生低质量调度代码。[1]

**Q6：从调度角度缩短 cycle 开销的技巧？**
        
#### 一、延迟隐藏类（最核心）

##### 技巧 1：提前调度高延迟指令（Load Hoisting）
把 `load`（3-4 周期）、`mul`（2 周期）等高延迟指令**尽量往前挪**，让其延迟被后续无关指令的执行掩盖掉 。 [ppl-ai-file-upload.s3.amazonaws](https://ppl-ai-file-upload.s3.amazonaws.com/web/direct-files/attachments/34980942/212fb322-057f-40c9-97ba-fa268cde4890/Chapter18_Instruction_Scheduling.txt)

```
❌ 未优化：          ✅ 优化后：
LD   R1, ...        LD   R1, ...    ← 提前 2 步
ADD  R3, R1, R4     MUL  R5, R6     ← 掩盖 LD 延迟
MUL  R5, R6, R7     ADD  R3, R1     ← R1 已就绪
```

在自定义策略里实现：

```cpp
// tryCandidate 中优先选 load
if (TryCand.SU->getInstr()->mayLoad()) {
    TryCand.Reason = Stall;  // 表示不调度会 stall
    return true;
}
```
```cpp
enum CandReason : uint8_t {
    NoCand,           // 0: 无候选（最低优先级）
    Only1,            // 1: 只有 1 个候选
    PhysReg,          // 2: 物理寄存器相关
    RegExcess,        // 3: 寄存器溢出
    RegCritical,      // 4: 临界寄存器压力
    Stall,            // 5: 避免流水线停顿（关键！）
    Cluster,          // 6: 指令聚簇（如 load/store clustering）
    Weak,             // 7: 弱优先级
    RegMax,           // 8: 寄存器最大化
    ResourceReduce,   // 9: 减少资源使用
    ResourceDemand,   // 10: 资源需求
    BotHeightReduce,  // 11: 减少底部高度
    BotPathReduce,    // 12: 减少底部路径
    TopDepthReduce,   // 13: 减少顶部深度
    TopPathReduce,    // 14: 减少顶部路径
    NodeOrder,        // 15: 原始节点顺序（几乎最低）
    FirstValid        // 16: 边界标记
};
```
##### 技巧 2：建模转发路径（Forwarding Path）
用 `ReadAdvance` 的正数值告诉调度器"这个操作数可以提前 N 周期读"，减少实际等待周期 ： [ppl-ai-file-upload.s3.amazonaws](https://ppl-ai-file-upload.s3.amazonaws.com/web/direct-files/attachments/34980942/212fb322-057f-40c9-97ba-fa268cde4890/Chapter18_Instruction_Scheduling.txt)

```tablegen
def : ReadAdvance<ReadWSMULArg1, 1>;  // 吸收 1 周期，减少等待
```

效果：原本要等 2 周期，通过流水线转发实际只等 1 周期。



#### 二、资源利用类

##### 技巧 3：优先调度关键路径上的指令（Critical Path First）
在 ready queue 里，优先选择**依赖链最长**的指令。这是 `GenericScheduler` 默认的重要启发式，可以通过 `tryCandidate` 中的 `SchedCandidate.Reason` 来判断和加强 。 [ppl-ai-file-upload.s3.amazonaws](https://ppl-ai-file-upload.s3.amazonaws.com/web/direct-files/attachments/34980942/212fb322-057f-40c9-97ba-fa268cde4890/Chapter18_Instruction_Scheduling.txt)

##### 技巧 4：Load/Store 聚簇（Clustering Mutation）
相邻的内存访问合并发射，减少地址计算开销和内存控制器切换开销 ： [ppl-ai-file-upload.s3.amazonaws](https://ppl-ai-file-upload.s3.amazonaws.com/web/direct-files/attachments/34980942/212fb322-057f-40c9-97ba-fa268cde4890/Chapter18_Instruction_Scheduling.txt)

```cpp
// 直接使用 LLVM 内置 mutation
DAG->addMutation(createLoadClusterDAGMutation(...));
DAG->addMutation(createStoreClusterDAGMutation(...));
```

##### 技巧 5：特定硬件指令优先调度
对于有多周期延迟的专用指令（如 SIMD multiply、widening multiply），在策略里优先排它们 ： [ppl-ai-file-upload.s3.amazonaws](https://ppl-ai-file-upload.s3.amazonaws.com/web/direct-files/attachments/34980942/212fb322-057f-40c9-97ba-fa268cde4890/Chapter18_Instruction_Scheduling.txt)

```cpp
if (Opc == MyTarget::WIDENING_SMUL) {
    TryCand.Reason = Stall;
    return true;
}
```

#### 三、调度方向类

##### 技巧 6：选择合适的调度方向
不同方向对 cycle 开销影响不同 ： [ppl-ai-file-upload.s3.amazonaws](https://ppl-ai-file-upload.s3.amazonaws.com/web/direct-files/attachments/34980942/212fb322-057f-40c9-97ba-fa268cde4890/Chapter18_Instruction_Scheduling.txt)

| 方向 | 特点 | 适用场景 |
|------|------|---------|
| **Top-down** | 优先调度生产者，充分暴露下游依赖 | in-order + 计算密集型 |
| **Bottom-up** | 优先调度消费者，缩短 live range | 寄存器紧张场景 |
| **Bidirectional**（默认）| 动态权衡 | 通用场景 |

```cpp
Policy.OnlyTopDown = true;  // in-order 处理器推荐
```

#### 四、模型精度类（让调度器"看清楚"才能优化准）

##### 技巧 7：精确标注每条指令的延迟
未装饰的指令默认 **1 周期**，而实际可能是 3-4 周期，调度器无法正确决策 ： [ppl-ai-file-upload.s3.amazonaws](https://ppl-ai-file-upload.s3.amazonaws.com/web/direct-files/attachments/34980942/212fb322-057f-40c9-97ba-fa268cde4890/Chapter18_Instruction_Scheduling.txt)

```tablegen
let Latency = 3 in
def DefaultWriteLoad : SchedWriteRes<[MemRes]>;
def : InstRW<[DefaultWriteLoad], (instregex "^LD[^i]*$")>;
```

##### 技巧 8：设置正确的 IssueWidth
告诉调度器每周期可发射几条指令，调度器才知道有多少"空槽"可以并行填充 ： [ppl-ai-file-upload.s3.amazonaws](https://ppl-ai-file-upload.s3.amazonaws.com/web/direct-files/attachments/34980942/212fb322-057f-40c9-97ba-fa268cde4890/Chapter18_Instruction_Scheduling.txt)

```tablegen
def MyModel : SchedMachineModel {
    let IssueWidth = 2;  // 每周期可同时发射 2 条指令
}
```

##### 技巧 9：用 DDG Mutation 强制有利顺序
当你知道某两条独立指令的某种顺序对硬件更友好（例如 bank conflict 避免），添加人工约束边 ： [ppl-ai-file-upload.s3.amazonaws](https://ppl-ai-file-upload.s3.amazonaws.com/web/direct-files/attachments/34980942/212fb322-057f-40c9-97ba-fa268cde4890/Chapter18_Instruction_Scheduling.txt)

```cpp
class MyMutation : public ScheduleDAGMutation {
    void apply(ScheduleDAGInstrs *DAG) override {
        // 在 instrB 和 instrC 之间插入依赖边
        // 强制 instrB 先于 instrC 执行
    }
};
```
#### 五、阶段选择类

##### 技巧 10：Pre-RA + Post-RA 双阶段调度
两个阶段各有侧重，叠加使用效果更好 ： [ppl-ai-file-upload.s3.amazonaws](https://ppl-ai-file-upload.s3.amazonaws.com/web/direct-files/attachments/34980942/212fb322-057f-40c9-97ba-fa268cde4890/Chapter18_Instruction_Scheduling.txt)

| 阶段 | 主要收益 |
|------|---------|
| **Pre-RA 调度** | 最大化 ILP，为寄存器分配创造好排列 |
| **Post-RA 调度** | 修复分配后产生的新停顿，精调物理寄存器顺序 |

```cpp
bool enableMachineScheduler() const override { return true; }         // Pre-RA
bool enablePostRAMachineScheduler() const override { return true; }   // Post-RA
```

#### 速查表

| 技巧 | 针对问题 | 实现位置 |
|------|---------|---------|
| 提前调度高延迟指令 | Load/Mul stall | `tryCandidate` |
| 建模转发路径 | 等待周期过长 | `ReadAdvance` 正数值 |
| 关键路径优先 | 长依赖链未覆盖 | 默认启发式已有，可加强 |
| Load/Store 聚簇 | 内存带宽浪费 | `createLoadClusterDAGMutation` |
| 精确延迟标注 | 调度器决策不准 | `InstRW` + `Latency` |
| 设置 IssueWidth | 并行槽未充分利用 | `SchedMachineModel` |
| DDG Mutation | 特定顺序更优 | `ScheduleDAGMutation` |
| 调度方向优化 | Top-down vs Bottom-up | `overrideSchedPolicy` |
| Pre+Post-RA 双阶段 | 单阶段遗漏优化 | `enableMachineScheduler` |
***
