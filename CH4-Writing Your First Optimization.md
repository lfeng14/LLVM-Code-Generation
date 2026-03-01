- 优化考虑两个点：legality and profitability. 合法性和收益
- value: A value is an entity that bears a certain meaning at a given time.
- SSA: is a way to represent values directly in an IR. The idea is straightforward: rename all variables such that each variable holds exactly one value statically. In the LLVM IR and Machine IR levels, phi instructions are grouped together at the beginning of the basic blocks. It is invalid to insert non-phi instructions before phi instructions.

- 节点A 支配 节点B：从程序入口到B的所有路径都必须经过A。这意味着在A处执行的代码，在到达B之前一定会被执行。
- 节点A 后支配 节点B：从B到程序出口的所有路径都必须经过A。这意味着在B之后，所有可能的执行路径最终都会到达A。
- 这里%add有两个使用者：%mul和%sub。如果遍历%add->users()，可能先得到%mul再得到%sub，也可能反过来
  ```
  %add = add i32 %a, %b
  %mul = mul i32 %add, 2
  %sub = sub i32 %add, 1
  ```
- legality: In other words, does this optimization preserve the semantics of the program
- 在数学上，加法满足结合律 (3.0 + a) + 3.0 = a + (3.0 + 6.0)。但在计算机的浮点数运算中，情况完全不同：运算顺序影响结果：浮点数的表示是有限的（如IEEE 754标准），很多小数无法精确表示。中间结果会经历舍入。表达式 (3.0 + a) + 3.0 的运算顺序是确定的：先计算 3.0 + a 得到一个中间结果并舍入，再将这个中间结果与 3.0 相加并再次舍入。
  - `3.0 + a + 3.0` 可变换为 `a + 6.0`。
  - 该变换有效前提：相关指令带有 **reassoc 标志**（reassociation 的缩写）。
  - 作用：允许对数学表达式**重新结合运算顺序**，不考虑结果精度可能受影响。
- No Unsigned Wrap (NUW) and No Signed Wrap (NSW) flags：结果回绕；
  1. 优化思路：将 `x * 2 == 2` 简化为 `x == 1`（两边同除以2）。
  2. 合法前提：`x * 2` 必须带有 **NSW 标记**（无符号溢出）。
  3. 无 NSW 时不合法：
     - 当 `x > INT_MAX / 2`，`x * 2` 会发生溢出环绕。
     - 存在反例：`x = 0x80000001`，满足 `(x * 2) mod 2³² == 2`，但 `x mod 2³² ≠ 1`。
  4. 结论：无 NSW 时该优化会导致结果错误。
  - At the LLVM IR level: bool Instruction::hasNoUnsignedWrap() and bool Instruction::hasNoSignedWrap()
  - At the Machine IR level: bool MachineInstr::getFlag(MIFlag Flag) with the MIFlag::NoUWrap and MIFlag::NoSWrap values
- 内存别名与ssa是什么关系 ？
- The body of a function can contain any kinds of instructions. This means calls to arbitrary functions must be conservatively modeled as having side effects. To avoid unnecessarily constraining optimizations, known functions, such as functions from the standard libraries。
  ```
  bool MemoryEffects::doesNotAccessMemory() tells you whether the memory is accessed at all.
  the Function class exposes a getMemoryEffects() method that describes the memory effects
  ```
-  it would be the right thing to disable inlining to optimize for minimal code size. Indeed, at this point, the code takes more space. However, if you run a later dead code elimination pass, the code shrinks.
-  The main API is InstructionCost TragetTransformInfo::getInstructionCost(const User *U, TargetCostKind CostKind).
   ```
   opt -passes="print<cost model>" -cost-kind=code-size input.ll
   --cost-kind=<value>                                                        - Target cost kind
    =throughput                                                              -   Reciprocal throughput
    =latency                                                                 -   Instruction latency
    =code-size                                                               -   Code size
    =size-latency                                                            -   Code size and latency
   ```
- TargetLoweringInfo::isFunctionVectorizable 判断库函数（如 libm 中的 cosf）是否支持指定向量长度，能否并行处理多元素。
- Datatype properties – DataLayout：TypeSize DataLayout::getTypeSizeInBits(Type *Ty)
- Register pressure：The idea behind register pressure is to keep track of all resources that may reside in the register and make sure that this number does not exceed the number of physical registers.
- BasicBlockFrequency: By default, the block frequencies are heuristically computed. For instance, a block before an if-then-else statement would have a frequency of 1, the then and else blocks a frequency of 0.5, and the block after the if-then-else-statement would have a frequency of 1. Profile-guided information means that you compile your program once with some instrumentations enabled. This instrumentation collects the frequencies of the basic blocks of your program while you run it on representative examples.
This information can then be fed back to the compiler to improve the accuracy of the cost models/heuristics.
- More precise instruction properties – scheduling model and instruction description:
  - 代码编译**lowering 过程**中，越接近最终可执行代码，优化转换可利用的目标平台信息越多。
  - **Machine IR 及更低层级**可通过 `MCInstrDesc` 结构体获取指令底层信息：指令类型、代码大小、调度类 ID 等。
  - 可通过 `MachineInstr::getDesc()` 或 `TargetInstrInfo` 直接获取 `MCInstrDesc`。
  - 目标**调度模型**由 `MCSchedModel` 表示，包含指令延迟、吞吐量等信息，可从 `MachineFunction` 逐级获取。
  - 用指令的调度类 ID 可从模型中查到对应的 `MCSchedClassDesc` 详细信息。
  - 调度模型细节将在第15章展开，本节最后会介绍优化领域常用术语。
    
- Instcombine: taking a = b * 2 and rewriting it into a = b << 1 is a sort of instcombine.
- Fixed point：A value is said to be alive at a given program point when its definition reaches this point, and use of
that value is still reachable from this point.
- Hoisting：The term hoisting refers to a transformation that moves something up in the CFG. For instance, if you pull up an invariant outside of a loop, you are hoisting this invariant outside of the loop.
- Sinking：The term sinking refers to the opposite transformation of hoisting
- Loop：LLVM offers loop-related information through the derived classes of the LoopBase class. LoopBase::getLoopPreheader()), the loop header (LoopBase::getHeader()
- constant propagation: simplify computations by replacing variables with constants and combining the constants to produce fewer computations
  - Only integer types are constant propagated.
  - Constant propagation is always legal and profitable.
  - We give up on constants when the constant type changes; for instance, zero extension (for example, unsigned a = -3; long long b = a;).
- Constant class派生类 ConstantInt；Value::replaceAllUsesWith(Value *NewVal).
- LLVM explicitly disabled all RTTI support. Instead, it has its own RTTI support that essentially assigns an embedded unique identifier to each class and uses this identifier to statically (that is, without RTTI support) check if an instance is of a certain type. What you need to remember is that you will need to use LLVM’s rolled-out RTTI constructs (isa<typename>(Obj), cast<typename>(Obj), and dyn_cast<typename>(Obj)
- 常量传播也存在增加开销的情况，举例：
  ```
  cst = load <low_part(262144)>   // 指令1
  cst |= load <high_part(262144)> // 指令2
  a = cst + b                      // 指令3
  c = cst + 3                      // 指令4
  e = c + d                        // 指令5
  # 这里常量 cst 只被构建了一次，然后被重复使用。如果进行常量传播，将 cst 替换到所有使用处：
  text
  cst = load <low_part(262144)>
  cst |= load <high_part(262144)>
  a = cst + b
  c = load <low_part(262144 + 3)>   // 新构建 (262144+3)
  c |= load <high_part(262144 + 3)>
  e = c + d
  ```
- ConstantHoisting.cpp 核心总结：
  - 筛选出无法折叠到指令中、构造开销超过TCC_BASIC阈值的整数常量及相关表达式；
  - 将分散在多个基本块的相似常量合并，通过 “基常量 + 偏移” 的形式统一构造；
  - 把基常量提升到支配所有使用点的位置，避免 SelectionDAG 中重复生成常量；
  - 隐藏提升后的常量，防止被后续转换生成新的高开销常量。

- What is the difference between a Use object and a User object in LLVM ?
  ```
  A Use object represents the relationship between a definition and an operand of one of its users;
  that is the entity that uses that value. A user is represented with an instance of the User class.
  Put differently, a Use object represents the edge between a definition and a User object, and a
  definition can have several Use instances for one User object. For example, a = b + b: b has
  one user (the computation that produces a) and two uses (the first and second operands of the
  computation that produces a).
  ```
#### 附件
- 论文：Simple and Efficient Construction of Static Single Assignment Form by Braun et al. published in Compiler Construction in 2013
- value class：https://llvm.org/doxygen/classllvm_1_1Value.html
- Loop：https://llvm.org/docs/LoopTerminology.html
- 常量上提： https://llvm.org/doxygen/ConstantHoisting_8cpp_source.html
