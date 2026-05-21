# LLVM 第17章知识点梳理：指令选择（Selection Phase）及后续

## 章节总览

第17章是整个指令选择流水线的最后阶段，负责将框架使用的通用合法中间表示（IR）翻译为目标机器特定的 `MachineIR`。本章涵盖三大选择框架（SDISel、FastISel、GlobalISel）的 Pattern 匹配机制，以及选择后的流水线完成工作。[1]

全章结构如下：

| 主题模块 | 核心内容 | 重要程度 |
|---------|---------|---------|
| Register Bank Selection | GlobalISel 专有，解决跨 bank 拷贝 | ⭐⭐⭐ |
| Selection Patterns | TableGen 描述指令匹配模式 | ⭐⭐⭐⭐⭐ |
| Advanced Patterns | PatFrag、SDNodeXForm、ComplexPattern | ⭐⭐⭐⭐⭐ |
| SDISel / FastISel / GlobalISel 集成 | 如何将 Pattern 注入各框架 | ⭐⭐⭐⭐ |
| Finalizing（流水线收尾） | Custom Inserter、finalizeLowering | ⭐⭐⭐⭐ |
| Optimizations（DAGCombiner / GICombiner） | 流水线内注入自定义优化 | ⭐⭐⭐⭐ |
| Debugging | 如何调试 Match Table | ⭐⭐⭐⭐⭐ |

***

## 一、Register Bank Selection（GlobalISel 专有）

### 1.1 概念与动机

Register Bank Selection（`RegBankSelect` Pass）是 GlobalISel 流水线中合法化（Legalization）之后、指令选择之前的**强制阶段**。其目标是为所有虚拟寄存器分配寄存器 bank，从而在分配前消除**跨 Bank 拷贝（cross-register bank copies）**。[1]

SDISel 无此优化：类型直接绑定寄存器类，无法避免跨 bank 拷贝。GlobalISel 通过扫描指令使用方式，可将向量操作标量化来完全消除这类代价高昂的拷贝。[1]

### 1.2 TableGen 描述寄存器 Bank

```tablegen
def GPRBRegBank : RegisterBank<"GPRB", [GPR32, GPR16sp]>;
```

- `RegisterBank` 类接受名称和所覆盖的寄存器类列表[1]
- 不需要列出所有子类，只需列出最大的超集，TableGen 会自动推导子类

生成代码需要在头文件中引入 `GET_REGBANK_DECLARATIONS` 和 `GET_TARGET_REGBANK_CLASS`，在 `.cpp` 文件中引入 `GET_TARGET_REGBANK_IMPL`。[1]

### 1.3 RegisterBankInfo 类实现

需要实现两个核心方法：

**`getInstrMapping`**：最核心的方法，返回 `InstructionMapping` 实例。关键类层次如下：[1]

```
InstructionMapping
  └── ValueMapping (每个操作数一个)
        └── PartialMapping[] (描述 bits 如何映射到 bank)
              └── {起始bit, 位宽, RegisterBank}
```

通过预先定义编译期常量的 `PartialMapping` 和 `ValueMapping` 数组，可显著降低编译时开销。[1]

**`getRegBankFromRegClass`**：将寄存器类映射到寄存器 bank（通常是一个大 switch）。[1]

***

## 二、Selection Patterns（核心重点）⭐⭐⭐⭐⭐

### 2.1 Pattern 的两种表达方式

**方式一：Instruction 类的 `Pattern` 字段（一对一模式）**

```tablegen
let Pattern = [(set GPR16:$dst, (add GPR16:$src0, GPR16:$src1))] in
def ADDi16rr : H2BLBBinaryInstruction<"addi16", ...
```

- `set` 关键字后接目标变量名，再描述模式[1]
- 操作码来自 `SDNode` 实例的名称（定义在 `TargetSelectionDAG.td` 中）
- 寄存器类 → 类型映射来自 `.td` 文件中的 `RegisterClass` 定义

**方式二：`Pat` 类（输入/输出分离，支持多指令折叠）**

```tablegen
def : Pat<(i32 (mul (sext GPR16:$src0), (sext GPR16:$src1))),
            (WIDENING_SMUL GPR16:$src0, GPR16:$src1)>;
```

- 第一个 dag 是待匹配 IR，第二个 dag 是产生的指令序列[1]
- `Pat` 类无需 `set` 关键字，可以在一个 Pattern 里产生多条指令

### 2.2 类型歧义处理

当一个寄存器类支持多种类型时，必须显式指定类型：

```tablegen
(set (i32 GPR32:$dst), ...)
```

类型不匹配会报 `Type set is empty for each HW mode` 这类难以阅读的错误。[1]

***

## 三、Advanced Selection Patterns（重点难点）⭐⭐⭐⭐⭐

### 3.1 PatFrag / ImmLeaf：子模式与立即数范围

`PatFrag` 允许定义可重用的 DAG 子模式，配合 C++ 谓词进行约束匹配。`ImmLeaf` 是 `PatFrag` 的常用形式，用于限定立即数范围：[1]

```tablegen
def uimm7 : ImmLeaf<i16, [{return Imm >= 0 && Imm < 128;}]>;

def LD16imm7 : H2BLBInstruction<...> {
    let Pattern = [(set GPR16:$dst, uimm7:$imm7)];
}
```

> **要点**：`PatFrag` 的 C++ 谓词运行在 `XXXTargetLowering` 类的上下文中。[1]

### 3.2 SDNodeXForm：SDNode 类型转换

用于在匹配过程中将一种 `SDNode` 转换为另一种（例如，将通用 `frameindex` 转换为目标特定的 `targetframeindex`）：

```tablegen
def to_tframeindex : SDNodeXForm<frameindex, [{
  return CurDAG->getTargetFrameIndex(N->getIndex(), N->getValueType(0));
}]>;

def : Pat<(i16 (frameindex:$ptr)), (MOVFROMSP (i16 (to_tframeindex $ptr)))>;
```

> **⚠️ 注意**：`SDNodeXForm` 绑定到 `SDValue`/SDISel，**不能**被 GlobalISel 直接导入。需要改用纯 C++ 方式处理。[1]

### 3.3 ComplexPattern：自定义 C++ 匹配函数

`ComplexPattern` 是处理目标特定寻址模式（addressing mode）的标准手段：

```tablegen
def addrmode : ComplexPattern<iPTR, 2, "selectAddrMode", []>;

def : Pat<(v2i16 (load (addrmode GPR16:$addr, uimm4:$offset))),
            (LDR32 $addr, $offset)>;
```

在 C++ 中实现 `selectAddrMode` 方法（位于 `XXXDAGToDAGISel` 类）：
- 第一个参数是待匹配的 `SDValue`
- 其余参数是产生的值（按引用传递）
- 返回 `true` 表示匹配成功[1]

**Pattern 优先级控制**：当多个 Pattern 可以匹配同一 IR 时，使用 `AddedComplexity` 字段控制优先级，值越高越先尝试。[1]

***

## 四、三大框架中的指令选择集成

### 4.1 SDISel 集成

SDISel 在 Chapter 14 完成大部分基础设施搭建后，所有 TableGen Pattern 会自动注入到生成的 `SelectCode` 方法中。自定义 C++ 选择逻辑在 `XXXISelDAGToDAG::Select` 方法中实现。[1]

### 4.2 FastISel 集成

简单 Pattern 会自动注入到 `selectOperator` 方法，但**不支持** `PatFrag`、`SDNodeXForm`、`ComplexPattern` 等高级构造。[1]

手动实现需要重写 `fastSelectInstruction` 方法，可使用以下辅助方法：

| 方法系列 | 含义 |
|---------|------|
| `fastEmitInst_x` | 直接用目标特定 opcode 产生 `MachineInstr` |
| `fastEmit_x` | 基于 TableGen Pattern 通过类型+ISD opcode 推断 opcode |

> `fastEmit_x` 可能失败，必须检查返回的寄存器是否为 `MCRegister::NoRegister`。[1]

另外，FastISel 可回退到 SDISel，需要调用 `updateValueMap` 维护 LLVM IR 值到虚拟寄存器的映射。[1]

### 4.3 GlobalISel 集成（InstructionSelect Pass）

**Step 1**：在 `CMakeLists.txt` 中调用 `gen-global-isel` TableGen backend：[1]

```cmake
tablegen(LLVM H2BLBGenGlobalISel.inc -gen-global-isel)
```

**Step 2**：在 `XXXInstructionSelector` 类中按位置引入以下宏：[1]

| 宏 | 位置 |
|----|------|
| `GET_GLOBALISEL_PREDICATE_BITSET` | 类声明前 |
| `GET_GLOBALISEL_PREDICATES_DECL` | 类声明内 |
| `GET_GLOBALISEL_TEMPORARIES_DECL` | 类声明内 |
| `GET_GLOBALISEL_IMPL` | 类声明后 |
| `GET_GLOBALISEL_PREDICATES_INIT` | 构造函数初始化列表 |

**Step 3**：实现 `select` 方法，先跳过已选择的指令，再调用 TableGen 生成的 `selectImpl`。[1]

#### Pattern 导入注意事项

GlobalISel 自动从 SDISel Pattern 导入，但以下情况需要手动处理：[1]

1. **未指定类型**：会报 `unsupported type for Src operand`，修复方法是显式加 `(i16 0)` 等类型注解
2. **输出 Pattern 缺少寄存器类**：会报 `Could not infer class for ... operand`，修复方法是在输出 Pattern 中重复寄存器类
3. **ComplexPattern**：必须定义 `GIComplexOperandMatcher` + `GIComplexPatternEquiv` 桥接[1]
4. **自定义 SDNode**：必须用 `GINodeEquiv` 映射 SDNode → G_XXX opcode[1]

#### GlobalISel ComplexPattern 桥接

```tablegen
def gi_addrmode :
    GIComplexOperandMatcher<p0, "selectAddrMode">,
    GIComplexPatternEquiv<addrmode>;
```

在 `XXXInstructionSelector` 中实现 `selectAddrMode`，返回**渲染器函数（renderer lambdas）**而非直接传引用：[1]

```cpp
return {{
    [=](MachineInstrBuilder &MIB) { MIB.addReg(BaseReg); },
    [=](MachineInstrBuilder &MIB) { MIB.addImm(Offset); },
}};
```

#### PHI / COPY 的特殊处理

GlobalISel 流水线中，PHI 和 COPY 指令可能缺少寄存器类，需要在 `select` 方法中为其补充赋值。[1]

***

## 五、Finalizing the Selection Pipeline（收尾阶段）

`FinalizeISel` Pass 负责为后续 Pass 准备好 `MachineFunction`。有两个主要自定义点：

### 5.1 Custom Inserter（伪指令展开）

在 `.td` 文件中标记需要展开的伪指令：

```tablegen
let usesCustomInserter = true in
def LD16imm16 : H2BLBPseudoInstruction<...
```

然后在 `TargetLowering::EmitInstrWithCustomInserter` 中实现展开逻辑。这适合无法用普通 Pattern 描述的复杂指令展开（如根据立即数值动态选择指令序列）。[1]

### 5.2 finalizeLowering 的坑

> **⚠️ 重要陷阱**：GlobalISel 流水线中 `finalizeLowering` 会被调用**两次**（一次在指令选择阶段，一次在 FinalizeISel Pass）。[1]

正确写法：用 `Selected` 属性检查是否已执行：

```cpp
void H2BLBTargetLowering::finalizeLowering(MachineFunction &MF) const {
    if (MF.getProperties().hasProperty(
            MachineFunctionProperties::Property::Selected))
        return;
    // ... do your things
    TargetLowering::finalizeLowering(MF);  // 必须调用父类！
}
```

***

## 六、Optimizations（指令选择流水线内优化）

### 6.1 SDISel：DAGCombiner 框架

SDISel 提供四个固定 hook 点进行 DAG Pattern 重写：[1]

| CombineLevel 值 | 时机 |
|----------------|------|
| `BeforeLegalizeTypes` | 合法化前 |
| `AfterLegalizeTypes` | 标量类型合法化后、向量合法化前 |
| `AfterLegalizeVectorOps` | 向量合法化后、操作合法化前 |
| `AfterLegalizeDAG` | 全部合法化完成后 |

使用步骤：
1. 在 `TargetLowering` 构造函数中调用 `setTargetDAGCombine(ISD::XXX)`
2. 重写 `TargetLowering::PerformDAGCombine`，返回 `SDValue()` 表示无重写，否则返回新节点

> **⚠️ 死循环风险**：自定义 rewrite 可能撤销通用优化，导致 DAGCombiner 无法达到不动点而陷入死循环。解决方案：引入自定义 SDNode 令通用优化"看不见"该节点。[1]

### 6.2 GlobalISel：GICombiner 框架（TableGen 驱动）

GlobalISel 使用 `gen-global-isel-combiner` TableGen backend 生成 `MachineFunctionPass` 类实现。[1]

**TableGen 定义**：用 `GICombineRule` 描述规则，用 `GIDefMatchData` 携带 match/apply 之间的数据：

```tablegen
def registers_matchinfo: GIDefMatchData<"SmallVector<Register>">;
def insertvectorelt_to_build_vector : GICombineRule<
    (defs root:$root, registers_matchinfo:$matchinfo),
    (match (wip_match_opcode G_INSERT_VECTOR_ELT):$root,
           [{ return matchInsertVectorElt(*${root}, ${matchinfo}); }]),
    (apply [{ applyInsertVectorElt(*${root}, ${matchinfo}); }])>;
```

**CMakeLists.txt 调用**：

```cmake
tablegen(LLVM H2BLBGenMandatoryPreLegalizeGICombiner.inc
         -gen-global-isel-combiner
         -combiners="H2BLBMandatoryPreLegalizerCombiner")
```

**代码集成宏顺序**：[1]

| 宏 | 位置 |
|----|------|
| `GET_GICOMBINER_DEPS` | `.cpp` 文件顶部 |
| `GET_GICOMBINER_TYPES` | 匿名 namespace 内 |
| `GET_GICOMBINER_CLASS_MEMBERS` | `XXXImpl` 类声明内 |
| `GET_GICOMBINER_IMPL` | 类声明外 |
| `GET_GICOMBINER_CONSTRUCTOR_INITS` | 构造函数初始化列表 |

***

## 七、Debugging the Selectors（调试重点）⭐⭐⭐⭐⭐

调试是实际工程中最高频的技能，务必熟练掌握。

### 7.1 SDISel 调试

**DAG 可视化**：使用 `view-*-dags` 系列选项输出 `.dot` 文件：[1]

```bash
dot -Tpdf /path/to/file.dot -o mygraph.pdf
```

常用选项：`view-isel-dags`（选择前）、`view-legalize-dags`（合法化前），可用 `filter-view-dags` 过滤特定函数。

**DAG 节点结构**：矩形圆角代表 `SDNode`，蓝色虚线箭头=chain，红色粗箭头=glue，黑色实线=数据依赖。[1]

**Match Table 调试**：使用 `-debug-only=isel` 选项：[1]

```
ISEL: Starting pattern match
  Initial Opcode index to 31041
  Match failed at index 31046
  Continuing at 158464
```

数字对应 `XXXGenDAGISel.inc` 文件中的 match table 索引，TableGen 会在文件中以注释标注。[1]

### 7.2 GlobalISel 调试

使用 `-debug-only=instruction-select,<debug_type>` 选项，其中 `<debug_type>` 是 `XXXInstructionSelector` 类中 `getName()` 返回的值（通常是文件的 `DEBUG_TYPE` 宏）。[1]

GlobalISel match table 调试技巧：在 `.inc` 文件中查找 `@ annotations`（例如 `// Label 147: @3085`），Try-block 条目通常有对应注释，用调试输出的索引减1可以定位。[1]

> **已知缺陷**：GlobalISel 的 match table 注释覆盖不如 SDISel 完善，LLVM GitHub issue #119177 跟踪此改进。[1]

***

## 知识点关联图

```
Chapter 17 核心体系
│
├── 【前置】RegBankSelect（GlobalISel only）
│     └── PartialMapping → ValueMapping → InstructionMapping
│
├── 【核心】Selection Patterns（三框架共用）
│     ├── 基础: Pattern field / Pat class
│     ├── 进阶: PatFrag / ImmLeaf（立即数范围）
│     ├── 进阶: SDNodeXForm（类型转换，仅 SDISel）
│     └── 进阶: ComplexPattern（寻址模式，需桥接 GlobalISel）
│
├── 【集成】各框架注入
│     ├── SDISel → 自动 via SelectCode（Chapter 14 已搭建）
│     ├── FastISel → 简单 Pattern 自动 + fastSelectInstruction 手工
│     └── GlobalISel → gen-global-isel + 导入检查 + 桥接机制
│
├── 【收尾】FinalizeISel
│     ├── Custom Inserter（伪指令展开）
│     └── finalizeLowering（注意双调用陷阱）
│
├── 【优化】Combiner 框架
│     ├── SDISel: DAGCombiner（4个 hook 点）
│     └── GlobalISel: GICombiner（TableGen backend + MachineFunctionPass）
│
└── 【调试】Match Table 调试
      ├── SDISel: view-*-dags + -debug-only=isel
      └── GlobalISel: -debug-only=instruction-select,xxx
```

***

## 重点攻克建议

作为编译器工程师，建议按以下优先级重点掌握：

1. **Selection Patterns（最高优先级）**：这是三大框架的共同基础，写好 TableGen Pattern 直接影响代码质量和开发效率。重点掌握 `Pat` 类的多指令折叠和 `ComplexPattern` 的寻址模式处理。[1]

2. **GlobalISel 完整流水线**：RegBankSelect → Legalize → InstructionSelect 的设计理念和 Pattern 导入机制是现代 LLVM backend 开发的方向，务必理解 GIComplexPatternEquiv 桥接和 GICombiner 框架。[1]

3. **Match Table 调试技能**：在实际开发中，Pattern 匹配失败是最常见的问题。能够读懂 `-debug-only=isel` 输出并在 `.inc` 文件中定位 match table 条目是必备技能。[1]

4. **Custom Inserter + finalizeLowering**：这两个机制是处理"Pattern 无法覆盖"场景的最后手段，`finalizeLowering` 的双调用陷阱是面试中经典的踩坑题。[1]

***
- Before we get to the instruction selection phase, we need to address the **RegBankSelect pass**, which is a mandatory phase for GlobalISel between the legalization and the instruction selection phase.
- 什么是“寄存器库”（Register Bank）？在现代 CPU 中，寄存器并不是一块铁板，而是分成不同的“零件库”：GPR (通用寄存器库)：专门处理整数运算、地址计算（比如循环变量 i）。FPR/VR (浮点/向量寄存器库)：专门处理大规模科学计算、多媒体数据（比如你之前提到的 AlphaFold3 里的向量运算）。问题在于： 这两个库在物理上往往是分开的。如果你想把一个在 GPR 里的整数传给浮点运算单元，CPU 必须执行一条专门的“搬运指令”（Cross-register bank copy）
  ```
  /**
   * 根据比较谓词、目标类型和 NaN 行为要求，获取对应的浮点 Min/Max 通用 Opcode
   */
  unsigned CombinerHelper::getFPMinMaxOpcForSelect(
      CmpInst::Predicate Pred, LLT DstTy,
      SelectPatternNaNBehaviour VsNaNRetVal) const {
    
    // 健壮性检查：必须明确定义 NaN 的预期行为（是返回数字还是返回 NaN）
    assert(VsNaNRetVal != SelectPatternNaNBehaviour::NOT_APPLICABLE &&
           "Expected a NaN behaviour?");
  
    // 根据比较谓词（大于、小于等）进行分支处理
    switch (Pred) {
    default:
      return 0; // 不匹配任何 Min/Max 模式
  
    // --- 场景一：匹配 MAX (最大值) ---
    case CmpInst::FCMP_UGT: // Unordered Greater Than
    case CmpInst::FCMP_UGE: // Unordered Greater Equal
    case CmpInst::FCMP_OGT: // Ordered Greater Than
    case CmpInst::FCMP_OGE: // Ordered Greater Equal
      
      // 1. 如果明确要求“忽略 NaN，返回另一个数字”，对应 FMAXNUM (IEEE 754-2008)
      if (VsNaNRetVal == SelectPatternNaNBehaviour::RETURNS_OTHER)
        return TargetOpcode::G_FMAXNUM;
        
      // 2. 如果明确要求“传播 NaN，只要有 NaN 就返回 NaN”，对应 FMAXIMUM (IEEE 754-2019)
      if (VsNaNRetVal == SelectPatternNaNBehaviour::RETURNS_NAN)
        return TargetOpcode::G_FMAXIMUM;
  
      // 3. 如果没有明确要求，则看硬件（后端）支持哪种：
      // 优先尝试合法的 FMAXNUM
      if (isLegal({TargetOpcode::G_FMAXNUM, {DstTy}}))
        return TargetOpcode::G_FMAXNUM;
      // 其次尝试合法的 FMAXIMUM
      if (isLegal({TargetOpcode::G_FMAXIMUM, {DstTy}}))
        return TargetOpcode::G_FMAXIMUM;
        
      return 0;
  
    // --- 场景二：匹配 MIN (最小值) ---
    case CmpInst::FCMP_ULT: // Unordered Less Than
    case CmpInst::FCMP_ULE:
    case CmpInst::FCMP_OLT:
    case CmpInst::FCMP_OLE:
      
      // 1. 优先满足明确的 NaN 行为要求
      if (VsNaNRetVal == SelectPatternNaNBehaviour::RETURNS_OTHER)
        return TargetOpcode::G_FMINNUM;
      if (VsNaNRetVal == SelectPatternNaNBehaviour::RETURNS_NAN)
        return TargetOpcode::G_FMINIMUM;
  
      // 2. 启发式回退：看后端对该类型的支持情况
      if (isLegal({TargetOpcode::G_FMINNUM, {DstTy}}))
        return TargetOpcode::G_FMINNUM;
        
      // 如果 FMINIMUM 不合法，返回 0 表示不能合并
      if (!isLegal({TargetOpcode::G_FMINIMUM, {DstTy}}))
        return 0;
        
      return TargetOpcode::G_FMINIMUM;
    }
  }
  ```
- 函数的功能就是**根据寄存器类（`TargetRegisterClass`）返回对应的寄存器组（`RegisterBank`）**，用于 GlobalISel 指令选择时约束操作数可用的寄存器组。在你给出的例子中，H2BLB 后端将所有的通用寄存器类（`GPR16`, `GPR32`, `GPR16sp`, `OnlySP`）都映射到同一个寄存器组 `GPRBRegBankID`，表明这些寄存器都属于同一个通用寄存器组。这确保了指令选择阶段，这些寄存器类的操作数都会被分配到该寄存器组中的物理寄存器。
  ```
  const RegisterBank &
  H2BLBRegisterBankInfo::getRegBankFromRegClass(const TargetRegisterClass &RC,
                                                LLT Ty) const {
    switch (RC.getID()) {
    default:
      llvm_unreachable("Register class not supported");
    case H2BLB::GPR16RegClassID:
    case H2BLB::GPR32RegClassID:
    case H2BLB::GPR16spRegClassID:
    case H2BLB::OnlySPRegClassID:
      return getRegBank(H2BLB::GPRBRegBankID);
    }
  }
  ```
- global isel
  ```
  // 全局指令选择 (GlobalISel) 的核心选择函数
  // 作用：把 Generic 指令 转换成 目标架构真正的机器指令
  bool H2BLBInstructionSelector::select(MachineInstr &I) {
    // 获取当前指令的操作码
    unsigned Opc = I.getOpcode();
  
    // 如果这条指令 已经是 目标架构指令
    // 并且不是 PHI / COPY 这种通用指令 → 不需要处理，直接返回 true
    if (!isPreISelGenericOpcode(Opc) && Opc != TargetOpcode::PHI &&
        Opc != TargetOpcode::COPY)
      return true;
  
    ...
  
    switch (Opc) {
    // 处理 全局指令选择里的 G_PHI（伪指令）
    case TargetOpcode::G_PHI:
      // 把 G_PHI 替换成 目标架构真正的 PHI 指令
      I.setDesc(TII.get(TargetOpcode::PHI));
      [[fallthrough]];  // 直接穿透到下面的 PHI 处理逻辑
  
    // 统一处理 PHI / COPY 指令
    case TargetOpcode::PHI:
    case TargetOpcode::COPY:
      // 遍历指令的所有操作数（都是寄存器）
      for (MachineOperand &MO : I.operands()) {
        Register Reg = MO.getReg();
  
        // 如果是物理寄存器，不需要改，跳过
        if (Reg.isPhysical())
          continue;
  
        // 如果寄存器已经分配了寄存器类，跳过
        const TargetRegisterClass *RC = MRI.getRegClassOrNull(Reg);
        if (RC)
          continue;
  
        // 核心：给虚拟寄存器分配正确的寄存器类
        // 16位 → GPR16spRegClass
        // 32位 → GPR32RegClass
        unsigned Size = MRI.getType(Reg).getSizeInBits();
        MRI.setRegClass(Reg, Size == 16 ? &H2BLB::GPR16spRegClass
                                        : &H2BLB::GPR32RegClass);
      }
  ```
