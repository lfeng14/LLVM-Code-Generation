- git diff begin_ch16 end_ch16 -- '*.cpp' > ch16.patch
- legalization infrastructure of the generic instruction selection frameworks, namely, the SelectionDAG instruction selection (SDISel) framework and the Global instruction selection (GlobalISel) framework. Fast instruction selection (FastISel) sub-framework has zero support for legalization
- what we call legal code is whatever IR that can be handled by the selection phase

 <img width="650" height="220" alt="image" src="https://github.com/user-attachments/assets/a268c226-3ea8-4443-aaa5-cad5a10ee8fd" />
 <img width="650" height="170" alt="image" src="https://github.com/user-attachments/assets/1ecdc8b8-50c5-42ae-bc72-bf827f057e3c" />
 <img width="650" height="440" alt="image" src="https://github.com/user-attachments/assets/37d968d6-3472-40e0-a629-d10dbaff61d9" />

- 类型合法、操作合法；成本模型，override the TargetLoweringBase::getPreferredVectorAction method to change how particular vector types are legalized. For instance, you can override this method if when you encounter the <2 x i8> type, you would rather legalize it to the <4x i8> type instead of the <2 x i16> type.
```
H2BLBTargetLowering::H2BLBTargetLowering(const TargetMachine &TM,
const H2BLBSubtarget &STI): TargetLowering(TM), Subtarget(STI) {
addRegisterClass(MVT::i16, &H2BLB::GPR16RegClass);
addRegisterClass(MVT::i32, &H2BLB::GPR32RegClass);
...
computeRegisterProperties(Subtarget.getRegisterInfo());
```
- 思考：如果有多个合法的候选指令，如何选择
- example of actions that we use in our H2BLB backend
  - `setOperationAction(ISD::FADD, MVT::f32, LibCall);`
    **语义：** 硬件不支持单精度浮点加法，请调用函数库。
    
    * **`ISD::FADD`**: 浮点加法操作。
    * **`MVT::f32`**: 操作数类型为 32 位浮点数。
    * **`LibCall`**: 告诉 Legalizer，由于硬件没有 `fadd.s` 之类的指令，请将这个节点转换成对运行时库（Runtime Library）的调用（例如调用 `__addsf3`）。
    * **应用场景**：常见于没有 FPU（浮点运算单元）的嵌入式处理器或软浮点（Soft-float）配置。
  
  - `setTruncStoreAction(MVT::i32, MVT::i16, Expand);`
    **语义：** 硬件不支持“截断存储”，请手动拆解。
    
    * **`MVT::i32` / `MVT::i16`**: 从 32 位寄存器中取值，但只存储低 16 位到内存中。
    * **`Expand`**: 告诉 Legalizer，硬件没有那种“自动截断并存储”的指令。
    * **动作**：Legalizer 会将该操作“展开”为多步逻辑。通常是先执行一条指令将 32 位值截断（比如位移或取子寄存器），然后再执行一条标准的 16 位存储指令（`i16 store`）。
  
  - `setOperationAction(ISD::MUL, MVT::i32, Custom);`
    **语义：** 这里的乘法很特殊，我要自己写 C++ 代码来处理。
    
    * **`ISD::MUL`**: 32 位整数乘法。
    * **`Custom`**: 这是最灵活的操作。它告诉编译器：不要用通用的逻辑，请在 Legalize 阶段调用我重写的 `LowerOperation` 函数。
    * **应用场景**：
        * 硬件可能只支持 16x16 乘法，你需要手动用多条指令拼成 32x32。
        * 或者你需要在乘法前后插入特定的硬件 Workaround。
        * **后果**：你必须在 `XXXISelLowering.cpp` 中实现对应的 `LowerMUL` 分支。
  
  - 知识点对比总结
  
    | 动作类型 (Action) | 核心含义 | 最终产出 |
    | :--- | :--- | :--- |
    | **`Legal`** | 硬件原生支持。 | 直接映射为一条机器指令。 |
    | **`LibCall`** | 硬件不支持，需调用函数。 | 产生一个 `bl` 或 `call` 指令指向 `compiler-rt`。 |
    | **`Expand`** | 硬件不支持，请用基础逻辑拆解。 | 自动拆解为多个更简单的 `SDNode`。 |
    | **`Custom`** | 情况复杂，后端手动干预。 | 由后端开发者编写的自定义 C++ 代码决定。 |
    | **`Promote`** | 类型太小，请提升到更大的类型。 | 例如将 `i8` 的加法提升到 `i32` 寄存器里做。 |
  
  - 深度思考：与你之前遇到的 PostgreSQL NaN 问题联系起来
  如果你在处理 `fmax` 指令时出现了 `NaN` 丢失：
  * 如果设为 **`Legal`**，可能是映射到的硬件指令本身就不符合 IEEE 754 的 `NaN` 传播规则。
  * 如果设为 **`Expand`**，LLVM 会用 `CMP + SELECT` 拆解，这时候如果 `CMP` 的条件（如 `OGT` vs `UGT`）选错，就会丢失 `NaN`。
  * 如果设为 **`Custom`**，你就可以在 C++ 代码里显式地插入 `isNaN` 的检查逻辑，从而完美解决 `NaN` 的场景。

- sdisel custom legalizetion：SDISel will automatically update all the uses of the original node with your new node; therefore
  ```
  SDValue H2BLBTargetLowering::lowerMUL(SDValue Op, SelectionDAG &DAG) const {
  assert(Op.getOpcode() == ISD::MUL);
    ... // Check if we can custom lower ISD::MUL.
    if (PlainLHS.getValueType() != MVT::i16 ||
    PlainRHS.getValueType() != MVT::i16)
    return SDValue();
    unsigned Opcode =
    isSigned ? H2BLBISD::WIDENING_SMUL : H2BLBISD::WIDENING_UMUL;
    SDValue NewVal = DAG.getNode(Opcode, SDLoc(Op), ValTy, PlainLHS, PlainRHS);
    return NewVal;
  }
  ```
- you learned that SDISel breaks down the legalization phase into two phases: the type legalization and the operation legalization. The first phase allows you to narrow down the types that you need to support, and in the second phase, you need to provide legalization actions for all the operations to produce legal operations.
- legalizetion in global isel: we create an H2BLB**LegalizerInfo** instance, our implementation of the LegalizerInfo class, in the constructor of our Subtarget class and we provide it in the overridden getter. LegalizerInfo是GlobalISel的指令合法化核心类，通过**程序化定义「操作+类型」配对规则**替代SDISel的固定合法类型，适配LLVM超大类型空间，最终通过`computeTables()`生效合法化规则。

  - GlobalISel 与 SDISel 的差异点（面试/笔记重点）

  | 特性 | SelectionDAG ISel (SDISel) | GlobalISel |
  | :--- | :--- | :--- |
  | **阶段划分** | **两阶段**：先类型合法化，再操作合法化。 | **单阶段/一体化**：通过 `LegalizerInfo` 统一处理类型和操作。 |
  | **类型系统** | **MVT (Machine Value Type)**：强依赖硬件预定义的类型。 | **LLT (Low-Level Type)**：更灵活，可以表示任意位宽（如 `s17`）。 |
  | **配置方式** | **命令式 (Imperative)**：调用大量的 `setOperationAction` 枚举。 | **声明式 (Declarative)**：使用 `RuleSet` 和谓词（Predicate）构建规则。 |
  | **粒度控制** | 较粗：通常针对 (Opcode, Type) 对进行配置。 | **极细**：可以使用 Lambda 表达式根据内存属性、对齐等动态决定。 |
  | **转换逻辑** | 相对固定（Expand/LibCall/Promote/Custom）。 | 极其灵活：`lower`、`libcall`、`widenScalar`、`narrowScalar` 等动作可链式调用。 |

- GlobalISel 如何实现合法化？
  - **A. 建立规则集 (The RuleSet)**
  你通过 `getActionDefinitionsBuilder` 选定要处理的操作码（如 `G_LOAD`, `G_STORE`）。这会返回一个 `LegalizeRuleSet` 对象，你所有的“法律条文”都将写在这里。
  - **B. 定义“合法”的边界 (Predicates)**
  代码通过多种方式定义什么是合法的：
    - **硬性指定 (`legalForTypesWithMemDesc`)**：直接列出支持的组合。例如 `{s16, p0, s8, 8}` 告诉编译器：*“如果把 8 位内存数据加载到 16 位寄存器（符号扩展/零扩展加载），且地址空间是 0，对齐是 8，这是合法的。”*
    - **范围约束 (`clampScalar`)**：这是一个非常强大的工具。`clampScalar(0, s16, s32)` 相当于划了一条红线：索引为 0 的类型（通常是结果类型）必须在 16 位到 32 位之间。
      *   如果遇到 `s8`，它会自动**提升 (Promote)** 到 `s16`。
      *   如果遇到 `s64`，它会尝试**拆分 (Split/Narrow)** 到 `s32`。
  - **C. 动态逻辑 (Predicates with Lambdas)**
  这是 GlobalISel 的杀手锏。你可以通过 `lowerIf` 或 `legalIf` 传入一个 **Lambda 表达式**。比如你的代码中：如果类型是标量且寄存器类型与内存类型不一致，就执行 `lower`（将其展开为更基础的指令）。这种基于逻辑判断的合法化在 SDISel 中很难写得这么优雅。

 ```
 // H2BLB 架构合法化器的构造函数
 H2BLBLegalizerInfo::H2BLBLegalizerInfo(const H2BLBSubtarget &ST) : ST(ST) {
   
   // 定义指针类型 p0：地址空间为 0，宽度为 16 位（暗示该架构地址线为 16 位）
   const LLT p0 = LLT::pointer(0, 16);
   // 假设上下文中已定义 s8, s16, s32 分别为 8, 16, 32 位的标量类型
 
   // 1. 获取 G_LOAD (加载) 和 G_STORE (存储) 指令的规则构建器
   getActionDefinitionsBuilder({TargetOpcode::G_LOAD, TargetOpcode::G_STORE})
     
     // 2. 显式声明硬件原生支持的组合：{寄存器类型, 指针类型, 内存类型, 对齐位宽}
     .legalForTypesWithMemDesc({
       {s8, p0, s8, 8},   // 8位加载/存储：寄存器 s8 <-> 内存 8bit
       {s16, p0, s8, 8},  // 扩展加载/截断存储：寄存器 s16 <-> 内存 8bit
       {s16, p0, s16, 8}, // 16位加载/存储：寄存器 s16 <-> 内存 16bit (字节对齐即可)
       {s32, p0, s32, 8}  // 32位加载/存储：寄存器 s32 <-> 内存 32bit
     })
 
     // 3. 约束标量范围：强制要求索引 0 的类型（寄存器类型）必须在 s16 到 s32 之间
     // 如果不在范围内（如 s8 或 s64），框架会自动尝试 Widen (提升) 或 Narrow (拆分)
     .clampScalar(0, s16, s32)
 
     // 4. 动态降级规则：如果寄存器类型与内存类型不匹配（且未在 legal 中定义）
     // 例如：用 s32 寄存器去读内存中的 s16，但上面没写这条规则，则执行 lower (拆解)
     .lowerIf([=](const LegalityQuery &Query) {
       return Query.Types[0].isScalar() &&
              Query.Types[0] != Query.MMODescrs[0].MemoryTy;
     })
 
     // 5. 动态合法性判断：再次确认，只要位宽是 16 或 32 位，就视为合法
     .legalIf([=](const LegalityQuery &Query) {
       TypeSize Size = Query.Types[0].getSizeInBits();
       return Size == 16 || Size == 32;
     })
 
     // 6. 向量平铺：如果传入的是向量类型（如 <2 x s16>），将其拆解为多个标量操作
     .scalarize(0)
 
     // 7. 兜底方案：如果以上所有规则都不匹配，则将该指令执行 lower (展开/降级)
     .lower();
 
   // 核心步骤：将上述流式声明的规则编译成内部查找表，优化编译器的匹配效率
   getLegacyLegalizerInfo().computeTables();
 }
 ```

 | 方法名 | 核心功能 (What it does) | 典型应用场景 |
 | :--- | :--- | :--- |
 | **`legalForTypesWithMemDesc`** | **精确匹配合法性**：根据“寄存器类型、指针类型、内存类型、对齐方式”这四个维度直接定义合法组合。 | 定义 Extending Load (s8->s16) 或 Truncating Store。 |
 | **`clampScalar`** | **范围约束**：强制指定某个类型索引（Type Index）的位宽必须在给定的最小值和最大值之间。 | 硬件只支持 16~32 位运算，遇到 s8 则提升，遇到 s64 则拆分。 |
 | **`legalIf`** | **条件合法化**：传入一个谓词（Predicate），如果返回 `true`，则将当前操作标记为合法。 | 根据动态条件（如指令的某个属性）判断是否允许直接执行。 |
 | **`lowerIf`** | **条件降级**：传入一个谓词，如果返回 `true`，则将当前指令替换为一组更简单的指令序列。 | 当操作数类型组合不支持时，手动触发展开逻辑。 |
 | **`scalarize`** | **标量化**：如果指定的类型索引是向量（Vector），则将其拆解为多个标量操作。 | 硬件不支持向量指令，需化整为零。 |
 | **`lower`** | **直接降级**：无条件触发指令的展开逻辑（具体展开成什么由操作码本身决定）。 | 将复杂的取模 `G_UREM` 展开为除法和减法的组合。 |


- 在 GlobalISel 的设计语境下，它并不直接说指令是“合法”还是“不合法”，而是给指令贴上一个动作标签（Action）。
 - 逻辑拆解
    - **检查条件**：是一个 32 位的标量乘法吗？（`!isVector() && Size == 32`）
    - **如果是 (True)**：把这个指令标记为 **`Custom`**。
    - **后续动作**：一旦被标记为 `Custom`，通用合法化工具（LegalizerHelper）就会停下手中的活，转而调用你重写的 `legalizeCustom` 函数。
   
  - 这不是“不合法”，而是“我要亲自打磨”
    在 `LegalizerInfo` 中，有几种常见的标签：
    - **`Legal`**：硬件直接支持，直接过。
    - **`Lower`**：硬件不支持，请编译器用更简单的指令（如 `ADD`/`SUB`）帮我拆了。
    - **`LibCall`**：硬件不支持，请帮我生成一个函数调用（如 `__mulsi3`）。
    - **`Custom`**：**“离远点，这事儿只有我（后端开发者）知道怎么弄最快/最对。”**
  
  - 为什么 32 位乘法经常需要 `Custom`？
    在高性能计算（HPC）或特定的嵌入式架构中，32 位乘法经常是“特例”：
    - **结果溢出**：硬件可能提供一个指令，同时产出高 32 位和低 32 位，这需要你手动分配两个寄存器。
    - **特定的优化**：比如你的 `H2BLB` 架构可能在特定条件下（如其中一个操作数是常量）使用更高效的 `Shift + Add` 序列，而不是通用的乘法。
    - **NaN 处理（呼应你之前的问题）**：如果你需要对乘法结果进行严苛的 `NaN` 检查，`Custom` 逻辑允许你在生成乘法指令后，紧接着插入一段检查逻辑。
  ```
   getActionDefinitionsBuilder(TargetOpcode::G_MUL)
   .customIf([=](const LegalityQuery &Query) {
    const auto &DstTy = Query.Types[0];
    return !DstTy.isVector() && DstTy.getSizeInBits() == 32;
   });
   bool H2BLBLegalizerInfo::legalizeCustom(
   LegalizerHelper &Helper, MachineInstr &MI,
   LostDebugLocObserver &LocObserver) const {
     MachineIRBuilder &MIRBuilder = Helper.MIRBuilder;
     MachineRegisterInfo &MRI = *MIRBuilder.getMRI();
     GISelChangeObserver &Observer = Helper.Observer;
     switch (MI.getOpcode()) {
     ...
     case TargetOpcode::G_MUL:
     return legalizeMul(MI, MRI, MIRBuilder, Observer);
     }
     llvm_unreachable("expected switch to return");
   }
  getActionDefinitionsBuilder(TargetOpcode::G_MUL)
  .customIf([=](const LegalityQuery &Query) {
   const auto &DstTy = Query.Types[0];
   return !DstTy.isVector() && DstTy.getSizeInBits() == 32;
  });
  bool H2BLBLegalizerInfo::legalizeCustom(
  LegalizerHelper &Helper, MachineInstr &MI,
  LostDebugLocObserver &LocObserver) const {
    MachineIRBuilder &MIRBuilder = Helper.MIRBuilder;
    MachineRegisterInfo &MRI = *MIRBuilder.getMRI();
    GISelChangeObserver &Observer = Helper.Observer;
    switch (MI.getOpcode()) {
    ...
    case TargetOpcode::G_MUL:
    return legalizeMul(MI, MRI, MIRBuilder, Observer);
    }
    llvm_unreachable("expected switch to return");
  }
  // 自定义处理 G_MUL 指令的合法化函数
  bool H2BLBLegalizerInfo::legalizeMul(MachineInstr &MI, MachineRegisterInfo &MRI,
                                      MachineIRBuilder &MIRBuilder,
                                      GISelChangeObserver &Observer) const {
    // 确保传入的确实是通用乘法指令 G_MUL
    assert(MI.getOpcode() == TargetOpcode::G_MUL);
  
    // 获取左操作数（LHS）的虚拟寄存器
    Register LHS = MI.getOperand(1).getReg();
    Register PlainLHS;
    bool isSigned = false; // 默认视为无符号乘法
  
    // ... (省略部分逻辑)
  
    // 核心逻辑：模式匹配 (Instruction Matching)
    // 检查操作数是否是通过 G_SEXT (带符号扩展) 得到的
    // 如果两个操作数都是从较窄类型扩展而来的，我们可以使用硬件的“加宽乘法”指令
    if (mi_match(LHS, MRI, m_GSExt(m_Reg(PlainLHS))) &&
        mi_match(RHS, MRI, m_GSExt(m_Reg(PlainRHS))))
      isSigned = true; // 匹配到带符号扩展，标记为有符号
  
    // ... (省略部分逻辑)
  
    // 获取目标指令信息 (TII) 以访问硬件指令描述
    const TargetInstrInfo &TII = *ST.getInstrInfo();
  
    // 根据匹配结果选择硬件指令：有符号加宽乘法 (SMUL) 或 无符号加宽乘法 (UMUL)
    unsigned Opcode = isSigned ? H2BLB::WIDENING_SMUL : H2BLB::WIDENING_UMUL;
  
    // --- 开始修改指令 ---
    // 必须通知观察者：我们要开始修改这条指令了 (GISel 增量更新要求)
    Observer.changingInstr(MI);
  
    // 将通用的 G_MUL 变更为目标机器特有的 Opcode
    // 这实际上是将“合法化”和“指令选择”的一部分提前合并完成了
    MI.setDesc(TII.get(Opcode));
  
    // ... (针对 Opcode 调整操作数等逻辑)
  
    // 约束寄存器：确保该指令的虚拟寄存器符合目标硬件的 Register Bank 和寄存器类要求
    constrainSelectedInstRegOperands(MI, TII, *MRI.getTargetRegisterInfo(),
                                     *ST.getRegBankInfo());
  
    // 通知观察者：指令修改完成，触发后续的 Worklist 更新
    Observer.changedInstr(MI);
  
    return true; // 返回 true 表示自定义合法化成功，不再执行默认流程
  }
  ```
#### further reading
- AMDGPU legalization: llvm/lib/Target/AMDGPU/AMDGPULegalizerInfo.cpp
- Aarch64 legalization: llvm/lib/Target/AArch64/GISel/AArch64LegalizerInfo.cpp file
