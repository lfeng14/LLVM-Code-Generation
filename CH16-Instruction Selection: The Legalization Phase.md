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


#### further reading
- AMDGPU legalization: llvm/lib/Target/AMDGPU/AMDGPULegalizerInfo.cpp
- Aarch64 legalization: llvm/lib/Target/AArch64/GISel/AArch64LegalizerInfo.cpp file
