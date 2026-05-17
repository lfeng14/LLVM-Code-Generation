- instruction scheduling is possible to use this optimization to reduce the register pressure
(how many registers you need to allocate) or increase the instruction-level parallelism (ILP) (the
number of instructions that can be executed in parallel) of your program.

好的！我帮你整理一份完整的学习笔记，包含概念、操作步骤和例子 。 [ppl-ai-file-upload.s3.amazonaws](https://ppl-ai-file-upload.s3.amazonaws.com/web/direct-files/attachments/34980942/212fb322-057f-40c9-97ba-fa268cde4890/Chapter18_Instruction_Scheduling.txt)

***

# LLVM CodeGen 第18章：指令调度（Instruction Scheduling）学习笔记

***

## 一、核心概念

### 1.1 什么是指令调度？
**指令调度**是一种底层优化，通过重新排列指令执行顺序来提升性能，主要目标 ： [ppl-ai-file-upload.s3.amazonaws](https://ppl-ai-file-upload.s3.amazonaws.com/web/direct-files/attachments/34980942/212fb322-057f-40c9-97ba-fa268cde4890/Chapter18_Instruction_Scheduling.txt)
- **降低寄存器压力**：减少同时需要分配的寄存器数量
- **提高指令级并行度（ILP）**：增加可同时执行的指令数

### 1.2 In-order vs Out-of-order 处理器

| 特性 | **In-order（顺序执行）** | **Out-of-order（乱序执行）** |
|------|------------------------|----------------------------|
| **执行方式** | 严格按程序顺序执行  [reddit](https://www.reddit.com/r/explainlikeimfive/comments/3g18ux/eli5_whats_the_difference_between_inorder_vs/) | 硬件动态重排指令  [zh.wikipedia](https://zh.wikipedia.org/wiki/%E4%B9%B1%E5%BA%8F%E6%89%A7%E8%A1%8C) |
| **遇到停顿** | 后续所有指令等待  [ppl-ai-file-upload.s3.amazonaws](https://ppl-ai-file-upload.s3.amazonaws.com/web/direct-files/attachments/34980942/212fb322-057f-40c9-97ba-fa268cde4890/Chapter18_Instruction_Scheduling.txt) | 跳过停顿，执行无依赖指令  [zh.wikipedia](https://zh.wikipedia.org/wiki/%E4%B9%B1%E5%BA%8F%E6%89%A7%E8%A1%8C) |
| **编译器调度重要性** | **极高** - 直接决定性能  [ppl-ai-file-upload.s3.amazonaws](https://ppl-ai-file-upload.s3.amazonaws.com/web/direct-files/attachments/34980942/212fb322-057f-40c9-97ba-fa268cde4890/Chapter18_Instruction_Scheduling.txt) | 中等 - 硬件可部分弥补  [cloud.tencent](https://cloud.tencent.com/developer/article/2547582) |
| **硬件复杂度** | 简单、低功耗 | 复杂、高功耗 |
| **典型应用** | 嵌入式设备（ARM Cortex-A53） | 高性能 CPU（Intel Core、Apple M） |

**关键点**：In-order 处理器无法动态优化，所以编译器调度至关重要 。 [ppl-ai-file-upload.s3.amazonaws](https://ppl-ai-file-upload.s3.amazonaws.com/web/direct-files/attachments/34980942/212fb322-057f-40c9-97ba-fa268cde4890/Chapter18_Instruction_Scheduling.txt)

***

## 二、LLVM 调度框架三大组件

### 2.1 数据依赖图（DDG）- `ScheduleDAGInstrs` 类
**作用**：表示指令间的依赖关系和约束 [ppl-ai-file-upload.s3.amazonaws](https://ppl-ai-file-upload.s3.amazonaws.com/web/direct-files/attachments/34980942/212fb322-057f-40c9-97ba-fa268cde4890/Chapter18_Instruction_Scheduling.txt)

**关键特性**：
- 边的方向：`A → B` 意味着 **B 必须在 A 之前调度**（use-def chain，看起来反直觉） [ppl-ai-file-upload.s3.amazonaws](https://ppl-ai-file-upload.s3.amazonaws.com/web/direct-files/attachments/34980942/212fb322-057f-40c9-97ba-fa268cde4890/Chapter18_Instruction_Scheduling.txt)
- 可以通过 **mutations** 在构建后修改 DDG（例如添加 `load` 聚簇约束） [ppl-ai-file-upload.s3.amazonaws](https://ppl-ai-file-upload.s3.amazonaws.com/web/direct-files/attachments/34980942/212fb322-057f-40c9-97ba-fa268cde4890/Chapter18_Instruction_Scheduling.txt)

**可视化命令**：
```bash
llc --view-misched-dags input.ll
```

### 2.2 调度策略 - `MachineSchedStrategy` 类
**作用**：决定从 ready queue 中选择哪条指令调度 [ppl-ai-file-upload.s3.amazonaws](https://ppl-ai-file-upload.s3.amazonaws.com/web/direct-files/attachments/34980942/212fb322-057f-40c9-97ba-fa268cde4890/Chapter18_Instruction_Scheduling.txt)

**常用基类**：
- `GenericScheduler`：寄存器分配前（Pre-RA）
- `PostGenericScheduler`：寄存器分配后（Post-RA）

**Ready Queue 概念**：
> 所有依赖已满足、可立即调度的指令集合 [ppl-ai-file-upload.s3.amazonaws](https://ppl-ai-file-upload.s3.amazonaws.com/web/direct-files/attachments/34980942/212fb322-057f-40c9-97ba-fa268cde4890/Chapter18_Instruction_Scheduling.txt)
> - **Top-down 调度**：ready queue = 所有后继已调度的指令
> - **Bottom-up 调度**：ready queue = 所有前驱已调度的指令

### 2.3 调度模型 - `TargetSchedModel` 类
**作用**：描述硬件资源和指令特性 [ppl-ai-file-upload.s3.amazonaws](https://ppl-ai-file-upload.s3.amazonaws.com/web/direct-files/attachments/34980942/212fb322-057f-40c9-97ba-fa268cde4890/Chapter18_Instruction_Scheduling.txt)

**包含信息**：
- 指令延迟（latency）：`load` 3 周期、`mul` 2 周期
- 处理单元（Processing Units）：ALU、MemRes、FPU
- 发射宽度（issue width）：每周期可发射指令数
- 资源占用：哪些指令使用哪些处理单元

***

## 三、调度方向详解

### 3.1 三种调度方向对比

```
Top-down（从上到下）        Bottom-up（从下到上）      Bidirectional（双向）
┌─────────────┐            ┌─────────────┐           ┌─────────────┐
│ 已调度区域   │            │             │           │ 已调度区域   │
├─────────────┤            │             │           ├─────────────┤
│ Ready Queue │            │             │           │ Top Queue   │
│ (无后继)    │            │             │           ├─────────────┤
├─────────────┤            │ Ready Queue │           │  待调度区域  │
│  待调度区域  │            │ (无前驱)    │           ├─────────────┤
│             │            ├─────────────┤           │ Bottom Queue│
│             │            │ 已调度区域   │           ├─────────────┤
└─────────────┘            └─────────────┘           │ 已调度区域   │
                                                     └─────────────┘
```

**适用场景** ： [ppl-ai-file-upload.s3.amazonaws](https://ppl-ai-file-upload.s3.amazonaws.com/web/direct-files/attachments/34980942/212fb322-057f-40c9-97ba-fa268cde4890/Chapter18_Instruction_Scheduling.txt)
- **Top-down**：优先调度生产者，适合减少延迟暴露
- **Bottom-up**：优先调度消费者，适合降低寄存器压力
- **Bidirectional**（默认）：动态平衡两者

***

## 四、调度模型实现步骤

### 4.1 核心 TableGen 类

```cpp
// 1. 定义处理单元
def ALURes : ProcResource<1>;      // 1 个 ALU
def MemRes : ProcResource<1>;      // 1 个 Memory Unit
def MemAndALU : ProcResGroup<[MemRes, ALURes]>; // 组合资源

// 2. 定义调度事件
def WriteLoad  : SchedWrite;       // 写事件
def ReadArg0   : SchedRead;        // 读事件

// 3. 绑定资源和延迟
let SchedModel = MyModel in {
  let Latency = 3 in
  def : WriteRes<WriteLoad, [MemRes]>;  // load 使用 MemRes，3 周期
  
  def : ReadAdvance<ReadArg0, -1>;      // 转发路径，减少 1 周期
}

// 4. 装饰指令
def LD : Instruction, Sched<[WriteLoad]>;
```

### 4.2 关键字段说明

**`WriteRes`**：定义写操作的资源占用 [ppl-ai-file-upload.s3.amazonaws](https://ppl-ai-file-upload.s3.amazonaws.com/web/direct-files/attachments/34980942/212fb322-057f-40c9-97ba-fa268cde4890/Chapter18_Instruction_Scheduling.txt)
```cpp
def : WriteRes<WriteWSMUL, [ALURes]> {
  let Latency = 2;  // 2 周期延迟
}
```

**`ReadAdvance`**：建模转发路径（负数延迟） [ppl-ai-file-upload.s3.amazonaws](https://ppl-ai-file-upload.s3.amazonaws.com/web/direct-files/attachments/34980942/212fb322-057f-40c9-97ba-fa268cde4890/Chapter18_Instruction_Scheduling.txt)
```cpp
def : ReadAdvance<ReadArg1, 1>;  // 吸收 1 周期（转发优化）
```

**`InstRW`**（推荐方法）：在调度模型中统一定义 [ppl-ai-file-upload.s3.amazonaws](https://ppl-ai-file-upload.s3.amazonaws.com/web/direct-files/attachments/34980942/212fb322-057f-40c9-97ba-fa268cde4890/Chapter18_Instruction_Scheduling.txt)
```cpp
let Latency = 3 in
def DefaultWriteLoad : SchedWriteRes<[MemRes]>;
def : InstRW<[DefaultWriteLoad], (instregex "^LD[^i]*$")>;  // 匹配所有 load 指令
```

***

## 五、实战示例

### 5.1 In-order 处理器优化案例

**优化前代码**（未调度）：
```assembly
LD  R1, 0(R2)    // 3 周期延迟
ADD R3, R1, R4   // 依赖 R1，必须等待
MUL R5, R6, R7   // 独立指令，但被迫等待
```

**执行时间线**（8 周期）：
```
周期: 1   2   3   4   5   6   7   8
LD   IF  ID  EX  MEM WB
ADD      IF  ID  停  停  EX  WB
MUL          IF  停  停  ID  EX  WB
```

**优化后代码**（调度器重排）：
```assembly
LD  R1, 0(R2)    // 提前发射
MUL R5, R6, R7   // 掩盖 LD 延迟
ADD R3, R1, R4   // 此时 R1 已就绪
```

**执行时间线**（5 周期）：
```
周期: 1   2   3   4   5
LD   IF  ID  EX  MEM WB
MUL      IF  ID  EX  WB
ADD          IF  ID  EX  WB
```

**性能提升**：37.5% 减少执行时间 [ppl-ai-file-upload.s3.amazonaws](https://ppl-ai-file-upload.s3.amazonaws.com/web/direct-files/attachments/34980942/212fb322-057f-40c9-97ba-fa268cde4890/Chapter18_Instruction_Scheduling.txt)

### 5.2 自定义调度策略示例

**场景**：优先调度高延迟指令（load、mul）

```cpp
class MyPreRASchedStrategy : public GenericScheduler {
protected:
  bool tryCandidate(SchedCandidate &Cand, SchedCandidate &TryCand,
                    SchedBoundary *Zone) const override {
    // 先使用默认启发式
    bool BetterCand = GenericScheduler::tryCandidate(Cand, TryCand, Zone);
    
    // 如果只是按原始顺序选择，应用自定义规则
    if (BetterCand && TryCand.Reason != NodeOrder && TryCand.Reason != NoCand)
      return true;
    
    // 优先调度 load 指令（隐藏内存延迟）
    if (TryCand.SU->getInstr()->mayLoad()) {
      TryCand.Reason = Stall;
      return true;
    }
    
    // 优先调度 mul 指令（2 周期延迟）
    if (TryCand.SU->getInstr()->getOpcode() == MyTarget::MUL) {
      TryCand.Reason = Stall;
      return true;
    }
    
    return TryCand.Reason != NoCand;
  }
};
```

***

## 六、启用和测试

### 6.1 启用调度器

```cpp
// 在 TargetSubtargetInfo 子类中
bool enableMachineScheduler() const override {
  return true;  // 启用 Pre-RA 调度
}

bool enablePostRAMachineScheduler() const override {
  return true;  // 启用 Post-RA 调度
}
```

### 6.2 设置调度方向

```cpp
void overrideSchedPolicy(MachineSchedPolicy &Policy,
                         unsigned NumRegionInstrs) const override {
  Policy.OnlyTopDown = true;   // 强制 top-down
  Policy.OnlyBottomUp = false;
}
```

### 6.3 测试工具

```bash
# 1. 可视化 DDG
llc --view-misched-dags input.ll

# 2. 分析微架构性能
llvm-mca input.s --mcpu=cortex-a53

# 3. 查看调度决策
llc -debug-only=machine-scheduler input.ll
```

***

## 七、关键要点总结

### 调度效率评估维度
1. **ILP（指令级并行度）**：同时执行的指令数
2. **寄存器压力**：活跃变量数量
3. **Latency hiding**：是否掩盖了 load/mul 延迟
4. **Resource utilization**：处理单元使用率

### 最佳实践
✅ **In-order 处理器**：重点优化 ILP，提前调度 load [ppl-ai-file-upload.s3.amazonaws](https://ppl-ai-file-upload.s3.amazonaws.com/web/direct-files/attachments/34980942/212fb322-057f-40c9-97ba-fa268cde4890/Chapter18_Instruction_Scheduling.txt)
✅ **Out-of-order 处理器**：重点降低寄存器压力 [cloud.tencent](https://cloud.tencent.com/developer/article/2547582)
✅ 使用 `InstRW` 而非 `SchedRW`（更模块化） [ppl-ai-file-upload.s3.amazonaws](https://ppl-ai-file-upload.s3.amazonaws.com/web/direct-files/attachments/34980942/212fb322-057f-40c9-97ba-fa268cde4890/Chapter18_Instruction_Scheduling.txt)
✅ 设置 `CompleteModel = true` 检测未覆盖指令 [ppl-ai-file-upload.s3.amazonaws](https://ppl-ai-file-upload.s3.amazonaws.com/web/direct-files/attachments/34980942/212fb322-057f-40c9-97ba-fa268cde4890/Chapter18_Instruction_Scheduling.txt)
✅ 用 `llvm-mca` 验证优化效果

### DDG 边方向理解
> **边 A → B 意味着 B 必须在 A 之前调度**  
> 原因：边表示 use-def chain，B 使用 A 的结果，所以 B 要先被调度（从后往前构建） [ppl-ai-file-upload.s3.amazonaws](https://ppl-ai-file-upload.s3.amazonaws.com/web/direct-files/attachments/34980942/212fb322-057f-40c9-97ba-fa268cde4890/Chapter18_Instruction_Scheduling.txt)

***

这份笔记涵盖了第18章的核心内容，你可以直接保存使用！如果有任何概念需要进一步澄清，随时告诉我。
