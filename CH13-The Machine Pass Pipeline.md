- ir to machine ir：ssa在哪个阶段被删除 phi在哪个阶段被删除 虚拟寄存器哪个阶段被删除，用一句话说明

  <img width="650" height="500" alt="image" src="https://github.com/user-attachments/assets/0e341fc6-f81a-4ca4-91f7-0b37668cd8ab" />
 
- machine IR不同阶段

  <img width="600" height="150" alt="image" src="https://github.com/user-attachments/assets/da3b388d-cf8e-4be2-aa4a-d17e939109da" />

  - PHI 消除：用 COPY 指令替代 PHI，打破 SSA 形式，让虚拟寄存器可以被多次定义。
  - 寄存器分配：把虚拟寄存器映射到物理寄存器，发现 %1 和 %8 可以复用同一个物理寄存器。
  - COPY 消除：既然 %1 和 %8 都在同一个物理寄存器里，中间的 COPY 就变成冗余，被优化掉。
  - 最终结果：原来的 PHI 和 COPY 都消失了，只剩下一条直接操作物理寄存器的 ADDXri 指令。

- This default pipeline is set by the [TargetPassConfig::addMachinePasses](https://github.com/llvm/llvm-project/blob/3b9106149c6867494049d7af4958eebeec5e2393/llvm/lib/CodeGen/TargetPassConfig.cpp#L1115-L1322) method. While you can overload this method so that you have a completely customized Machine pass pipeline, we recommend that you stick with the default implementation

- TargetPassConfig 提供的钩子可在默认编译优化流水线的不同阶段插入自定义 Pass。
  - 若重写 TargetPassConfig::addMachinePasses 默认实现，这些钩子不会被调用。
  - 默认流水线已包含大量通用优化，无需重复添加。
  - 该类通过 addXXX 虚方法提供插入点，如：
  - 重写 addPreRegAlloc：在寄存器分配前插入 Pass；
  - 重写 addPreISel：在指令选择前（仍为 LLVM IR 阶段）插入 Pass。
  - 更多插入点可参考图 13.1 或源码文件 llvm/lib/CodeGen/TargetPassConfig.cpp 中的 addMachinePasses 实现。

- TargetPassConfig 类通过名为 addXXX 的虚方法定义钩子，XXX 对应默认编译流水线的特定阶段。例如：重写该类中与目标相关的 addPreRegAlloc 方法，可在寄存器分配阶段前插入自定义编译分析 / 优化流程。
- Similarly, by overriding the AddPreISel method, you can inject passes right before the instruction selection pass – that is, while the input IR is still using the LLVM IR.
- 类似 LLVM IR  passes 依赖 **TargetTransformInfo** 工作，
  - 部分 Machine IR passes 需借助 **TargetInstrInfo、TargetRegisterInfo、TargetLowering** 类，
  - 同时结合指令自身属性才能完成处理。
- MachineSink 优化：将控制流图（CFG）中可移动的指令下沉到执行频率更低的位置。
  - 移动安全性判断：依据 MachineInstr 实例的属性，如带 mayStore/mayLoad 的内存操作指令被判定为不可移动。
  - 算法驱动：通过 TargetInstrInfo 类的钩子方法（可重写）控制，包括 shouldSink 和 isSafeToSink。

- Start by creating a reduced MIR file with your motivating example and use the run-pass command-line option with the llc tool to quickly iterate on this pass (see Chapter 10 if you need help with that). Remember to use the logging capability of the passes to see why this pass may not perform the optimization you wanted. The logs may highlight what’s missing (see Chapter 10 if you need help with that as well。Next, if the logs aren’t enough, open the file that contains the implementation of the pass. If you can’t find this file, remember that you can leverage the git grep command line to find the source files that contain specific strings
- TII TRI TLI:
  - 在文件中查找 `TargetRegisterInfo`、`TargetInstrInfo`、`TargetLowering` 类的使用，可先搜索常用变量名 `TRI`、`TII`、`TLI`。
  - 查找 `MachineInstr::isXXX` 方法（如 `isAsCheapAsMove`），可搜索 `->is` 或 `.is` 并定位与指令变量 `MI` 相关的调用。
  - 上述操作可明确需在对应目标类中实现的方法，以及指令应具备的属性。
- codegen prepare pass
  - 归属与类型：属于 LLVM CodeGen 库，是**LLVM IR 到 LLVM IR** 的优化遍。
  - 核心作用：弥补传统指令选择器 **SelectionDAG** 的不足，在 IR 层面做转换以生成最优代码。
  - 主要功能：构造合适的指令模式，让指令选择器更高效地利用目标平台**寻址模式**。
  - 实现依赖：大量功能通过目标平台**TargetLowering** 类的重载方法实现，如寻址模式合法性判断 `isLegalAddressingMode`。

- PeepholeOptimizer
  - **作用**：将特定指令模式重写为更高效的指令，其中一类优化是调整类拷贝指令，提升寄存器分配效率。
  - **启用条件**：
    - 在指令描述中添加对应属性：`isBitcast`/`isRegSequenceLike`/`isInsertSubregLike`/`isExtractSubregLike`；
    - 在 `TargetInstrInfo` 类中实现相关方法：`getRegSequenceInputs`、`getInsertSubregInputs`、`getExtractSubregInputs`。

- **MachineCombiner 作用**：提供指令模式替换框架，用新指令替换原有指令组合。
  - **与 PeepholeOptimizer 区别**：
    1. PeepholeOptimizer：**与目标无关**的指令重写，适用于多平台。
    2. MachineCombiner：**面向特定目标平台**，仅在**关键路径更短/等长**且**寄存器压力更低/相等**时才执行替换，有精确收益判断模型。

- 关键路径（Critical Path）
  1. **作用**：用于衡量指令序列调度/执行顺序的优劣。
  2. **定义**：执行一段指令序列所需的**最少处理器时钟周期数**。
  3. **核心特征**：处于关键路径上的指令，一旦延迟，会直接拖慢整个指令序列的执行。
  4. **示例说明**：
      - 指令 `a = b + c` 中，`b` 来自内存加载（需3周期），`c` 由另一指令生成（需1周期）。
      - **b 在关键路径**：延迟最长，无法被并行掩盖；
      - **c 不在关键路径**：可在等待 b 加载时并行完成，不影响总耗时。
- In this chapter, you learned how the Machine pass pipeline is structured and the implications of this structure on the IR. You saw that you start from the Machine IR in SSA form with virtual registers, then go to the non-SSA form but still with virtual registers, and finish with pure physical registers before transitioning to the MC layer

- 核心问题与答案总结（LLVM 后端开发相关）
  - 机器IR（Machine IR）在Lowering过程中的三个主要阶段
     - Machine IR 首先以**带虚拟寄存器的SSA形式**存在，接着转为**带虚拟寄存器的非SSA形式**，最终仅使用**物理寄存器**。
  - 是否应重写 TargetPassConfig::addMachinePasses 方法及原因
     - **不建议重写**：该方法构建了LLVM目标无关代码生成器的核心流水线，重写会重复造轮子，如需调整流水线，应通过“注入pass”的方式而非直接重写。
  - 如何判断通用Pass可在流水线的哪个阶段运行
    - 仅能在特定阶段运行的Pass会重载 `MachineFunctionPass::getRequiredProperties` 方法；
    - 查看该方法的实现，其中列出的“要求”即对应Pass的运行阶段；
    - 默认情况下，Pass可在流水线任意阶段运行。
  - PeepholeOptimizer 与 MachineCombiner 两个Pass的区别
  两者均通过替换指令模式做优化，但：
    - PeepholeOptimizer：应用**目标无关（target-agnostic）** 的指令模式；
    - MachineCombiner：应用**目标特定（target-specific）** 的指令模式（核心是为高频指令序列构建fast path，提升执行效率）。
  - 复用 ExpandPostRAPseudosID 扩展伪指令时，后端需实现的内容
    1. 用 `git grep ExpandPostRAPseudosID` 在LLVM仓库定位该Pass的实现；
    2. 查看其对 TargetXXX 类的调用，识别核心依赖方法：`TargetInstrInfo::expandPostRAPseudo` 和 `TargetInstrInfo::lowerCopy`；
    3. 跟踪这些方法的实现，最终需在后端实现 `TargetInstrInfo::copyPhysReg` 方法。
  - 关键点回顾
    1. Machine IR的Lowering核心是“虚拟寄存器→物理寄存器”+“SSA→非SSA”的逐步转换；
    2. LLVM后端优先复用默认pass流水线，避免重写核心方法，优化可通过“注入pass”实现；
    3. Pass的运行阶段由 `getRequiredProperties` 定义，不同优化Pass的核心差异在于“是否绑定具体目标架构”；
    4. 复用通用Pass的关键是定位其依赖的Target层接口，并在后端实现对应方法。

#### further reading
- https://llvm.org/docs/CodeGenerator.html
