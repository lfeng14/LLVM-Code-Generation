# 第21章：LLVM 汇编器入门学习笔记

## 概述

本章是《LLVM Code Generation by Example》的最后一章，目标是帮助读者理解如何利用 LLVM 基础设施构建一个**汇编器（Assembler）**——即将文本格式的汇编代码转化为二进制目标文件（`.o`）的工具。[1]

汇编器处于编译器后端与二进制工具链的交界处，其核心挑战在于：**如何将文本指令编码为字节流，同时正确处理符号地址的解析问题**。[1]

***

## 一、核心概念速查

| 概念 | 解释 |
|------|------|
| **汇编器（Assembler）** | 将文本汇编文件（`.s`）转化为二进制目标文件（`.o`）的工具 |
| **Fixup（修正占位符）** | 指令编码中尚未填入的地址位，用 `xxxx` 占位，等待解析 |
| **Relaxation（指令松弛）** | 将编码空间不足的指令替换为语义相同但编码更长的指令 |
| **Relocation（重定位）** | 在目标文件中写入的「待修补指令」，告诉链接器运行时如何填入正确地址 |
| **MCStreamer** | LLVM 的抽象输出类，决定输出文本汇编还是二进制目标文件 |
| **MCCodeEmitter** | 负责指令编码，并在遇到符号引用时发出 fixup |
| **MCAsmBackend** | 负责处理 fixup、松弛过程，并转化为 relocation |
| **MCObjectTargetWriter** | 负责将 fixup 翻译为 relocation，写入目标文件的重定位表 |
| **MCAssembler** | 持有符号、节（section）等信息，协调三个目标特定类协同工作 |

***

## 二、汇编流程：从文本到二进制

### 2.1 整体流程

汇编器需完成两件核心事情：[1]
1. **指令编码**：将助记符翻译成机器码（在第12章已实现）
2. **符号地址解析**：处理跳转目标、函数调用地址等，这才是汇编的**主要难点**

### 2.2 地址解析的三阶段流程

以 `call fctA` 指令为例，地址解析走以下路径：[1]

```
call fctA
    │
    ▼
[阶段1] 发出 Fixup
    - 为符号 fctA 创建一个占位符 xxxx
    - 记录：需要填入的位数（如11位）
    │
    ▼
[阶段2] 尝试解析符号地址
    ├─ 可解析（fctA 定义在同文件）
    │       │
    │       ▼
    │   地址是否能放入可用位数？
    │       ├─ 能放入 → 直接编码，完成！
    │       │   例：opcode=0110, address=0010
    │       └─ 放不下 → 进入 Relaxation
    │               │
    │               ▼
    │           找到语义相同但编码更长的指令？
    │               ├─ 找到（如 longerCall）→ 重新检查
    │               └─ 无法再松弛 → 进入阶段3
    │
    └─ 不可解析（外部函数）→ 进入阶段3
            │
            ▼
[阶段3] 记录 Relocation
    - 在目标文件的 relocation table 中写入一条记录
    - 告诉链接器："PC 处的指令需要被 fctA 的地址填充"
```

**关键结论**：Fixup 是汇编器内部的临时概念；Relocation 是写入目标文件、留给链接器处理的最终结果。[1]

***

## 三、LLVM 汇编架构解析

### 3.1 MCStreamer：统一输出抽象

LLVM 使用同一条 `AsmPrinter` 流水线产生文本汇编和二进制目标文件，区别仅在于 `MCStreamer` 的子类：[1]

```
AsmPrinter (target-specific)
        │
        ▼
    MCStreamer（抽象基类）
       ├── MCAsmStreamer     → 输出 .s 文本文件
       ├── MCELFStreamer     → 输出 ELF 二进制 (.o)
       ├── MCXCoffStreamer   → 输出 COFF 二进制
       └── ...（其他格式）
```

### 3.2 MCObjectStreamer 的扩展机制

`MCObjectStreamer` 通过 `MCAssembler` 引入三个**目标特定类**来完成二进制生成：[1]

```
MCObjectStreamer
       │
       └── MCAssembler
               ├── MCCodeEmitter     ← 编码指令 + 发出 fixup
               ├── MCAsmBackend      ← 应用 fixup + 松弛 + 生成 relocation
               └── MCObjectTargetWriter ← 将 fixup 转化为 relocation 记录
```

**三者协作流程**：
1. `MCCodeEmitter` 在无法直接编码符号地址时，调用 `Fixups.push_back(...)` 发出 fixup
2. `MCAsmBackend::applyFixup()` 尝试将已知地址直接 patch 进字节流
3. 未解析的 fixup 传递给 `MCObjectTargetWriter::recordRelocation()`，写入 `.o` 的重定位表

***

## 四、实现细节：逐类解析

### 4.1 MCCodeEmitter：发出 Fixup

**核心职责**：编码指令，并对无法直接编码的符号引用发出 fixup。

**步骤一**：声明目标特定 Fixup 枚举：[1]

```cpp
// H2BLBMCFixups.h
enum FixupKind {
    FK_H2BLB_PCRel_11 = FirstTargetFixupKind,
    LastTargetFixupKind,
    NumTargetFixupKinds = LastTargetFixupKind - FirstTargetFixupKind
};
```

**命名逻辑解析**：
- `FK` = FixupKind 缩写
- `H2BLB` = 目标后端名
- `PCRel` = PC 相对寻址（运行时地址 = 当前 PC + offset）
- `11` = 指令中为地址预留的位数

**步骤二**：在 `getMachineOpValue` 中发出 fixup：[1]

```cpp
unsigned H2BLBMCCodeEmitter::getMachineOpValue(
    const MCInst &MI, const MCOperand &MO,
    SmallVectorImpl<MCFixup> &Fixups,
    const MCSubtargetInfo &STI) const {

    // 处理非符号操作数（见第12章）...

    // 遇到 call 指令的符号引用时，发出 fixup
    assert(MO.isExpr());
    const MCExpr *Expr = MO.getExpr();
    if (MI.getOpcode() == H2BLB::CALL) {
        Fixups.push_back(
            MCFixup::create(
                0,                          // offset: fixup 从第0位开始
                Expr,                       // 符号表达式
                (MCFixupKind)H2BLB::FK_H2BLB_PCRel_11  // fixup 类型
            )
        );
    }
    return 0; // 编码的位先置零
}
```

**为什么 offset=0**：因为 `call` 指令中，11位地址字段从第0位（最低有效位）开始。[1]

***

### 4.2 MCAsmBackend：处理 Fixup

**核心职责**：将已解析的 fixup 应用（patch）到字节流中，未解析的转交给 `MCObjectTargetWriter`。

**必须实现的三个方法**：[1]

| 方法 | 作用 |
|------|------|
| `getNumFixupKinds()` | 返回目标特定 fixup 的数量 |
| `getFixupKindInfo()` | 描述每种 fixup 的特征（大小、偏移等） |
| `applyFixup()` | 将 fixup 值 patch 进字节流 |
| `createObjectTargetWriter()` | 返回对应格式的 `MCObjectTargetWriter` |

**`applyFixup` 实现逻辑**：[1]

```cpp
void H2BLBAsmBackend::applyFixup(
    const MCAssembler &Asm, const MCFixup &Fixup,
    const MCValue &Target, MutableArrayRefhar> Data,
    uint64_t Value, bool IsResolved,
    const MCSubtargetInfo *STI) const {

    // 值为零则无需修改字节
    if (!Value) return;

    unsigned Kind = Fixup.getKind();
    // 如果是 relocation（非 fixup），跳过
    if (Kind >= FirstLiteralRelocationKind) return;

    // 获取 fixup 描述（位宽、偏移等）
    MCFixupKindInfo Info = getFixupKindInfo(Fixup.getKind());

    // 将 Value 左移以对齐目标位位置
    Value <<= Info.TargetOffset;

    unsigned NumBytes = (Info.TargetSize + 7) / 8;
    uint32_t Offset = Fixup.getOffset();

    // 逐字节写入（小端序）
    for (unsigned i = 0; i != NumBytes; ++i)
        Data[Offset + i] |= static_cast<uint8_t>((Value >> (i * 8)) & 0xff);
}
```

**小端序写入的原因**：H2BLB 是小端目标架构，低字节在低地址。[1]

***

### 4.3 MCObjectTargetWriter：记录 Relocation

**核心职责**：对于无法在汇编时解析的 fixup，将其转化为 **relocation 记录**写入目标文件。

H2BLB 后端选择 **Mach-O** 格式，因此继承 `MCMachObjectTargetWriter`。[1]

**`recordRelocation` 的关键字段**：[1]

```cpp
// Mach-O 的 relocation_info 结构（来自系统头文件）
struct relocation_info {
    int32_t  r_address;                // fixup 相对于 section 的字节偏移
    uint32_t r_symbolnum : 24,         // 符号表索引 或 section 编号
             r_pcrel     : 1,          // 是否为 PC 相对重定位
             r_length    : 2,          // 编码大小（log2 字节数）
             r_extern    : 1,          // 是否引用符号表
             r_type      : 4;          // 重定位类型（目标特定）
};
```

**实现中的两种情况**：[1]

```
target 符号存在 Base symbol？
├── 有 Base：
│   RelSymbol = Base
│   如果 Symbol != Base → 调整 Value 偏移
└── 无 Base（符号在某 section 内）：
    Index = section 编号（从1开始）
    Value = 符号的绝对地址
    如果是 PC-relative → 减去当前 PC + fixup偏移 + 编码大小
```

**最终发射 relocation**：[1]

```cpp
Type = unsigned(MachO::ARM64_RELOC_BRANCH26); // 使用已有类型（mock）
FixedValue = Value;                           // addend 写回给 applyFixup 用

MachO::any_relocation_info MRE;
MRE.r_word0 = FixupOffset;                   // r_address
MRE.r_word1 =
    (Index << 0)    |   // r_symbolnum (bits 0-23)
    (IsPCRel << 24) |   // r_pcrel (bit 24)
    (Log2Size << 25)|   // r_length (bits 25-26)
    (Type << 28);       // r_type (bits 28-31)

Writer->addRelocation(RelSymbol, Fragment->getParent(), MRE);
```

***

## 五、注册与验证

### 5.1 注册 MCAsmBackend

完成三个类的实现后，需在 `LLVMInitializeH2BLBTargetMC()` 函数中注册：[1]

```cpp
// 在 H2BLBMCTargetDesc.cpp 中
LLVMInitializeH2BLBTargetMC() {
    ...
    RegisterMCAsmBackend<H2BLBAsmBackend> X(getTheH2BLBTarget());
}
```

### 5.2 使用 llvm-objdump 验证

```bash
# 汇编并检查 relocation 表
llvm-mc -triple=h2blb --filetype=obj input.s -o output.o
llvm-objdump -r output.o  # 查看重定位表
```

***

## 六、三大核心类职责对比

| 类 | 触发时机 | 核心方法 | 输出 |
|----|---------|---------|------|
| `MCCodeEmitter` | 遇到符号引用操作数 | `getMachineOpValue()` | Fixup 列表 + 部分编码字节 |
| `MCAsmBackend` | 汇编完成后处理 fixup | `applyFixup()` | Patch 后的字节流 |
| `MCObjectTargetWriter` | Fixup 无法解析时 | `recordRelocation()` | 目标文件中的 relocation 表项 |

***

## 七、关键逻辑：Fixup vs Relocation 的本质区别

- **Fixup** 是汇编器内存中的临时概念，表示「这里有个洞，我现在填不上」[1]
- **Relocation** 是最终写入 `.o` 文件的持久记录，表示「链接器，你来填这个洞」[1]
- **Relaxation** 是在两者之间的尝试：当洞太小装不下地址时，把指令换成有更大洞的版本[1]

```
Fixup → [能解析且位数够?] → 直接 applyFixup() → 字节流中已填入
     → [位数不够?]       → Relaxation → 换更长编码 → 重试
     → [无法解析?]       → recordRelocation() → 写入 .o relocation table
```

***

## 八、延伸阅读

- **MC 层设计理念**：Chris Lattner 2010 年博客 [https://blog.llvm.org/2010/04/intro-to-llvm-mc-project.html](https://blog.llvm.org/2010/04/intro-to-llvm-mc-project.html)[1]
- **集成汇编器实现教程**：Simon Cook 的 *Implementing LLVM Integrated Assembler* ([https://www.embecosm.com/appnotes/ean10/ean10-howto-llvmas-1.0.html](https://www.embecosm.com/appnotes/ean10/ean10-howto-llvmas-1.0.html))[1]
- **Mach-O 格式规范**：[https://github.com/aidansteele/osx-abi-macho-file-format-reference](https://github.com/aidansteele/osx-abi-macho-file-format-reference)[1]
- **伴随仓库**：完整代码在 `begin_ch21` 到 `end_ch21` Git tag 之间，见 [https://github.com/PacktPublishing/LLVM-Code-Generation-by-example](https://github.com/PacktPublishing/LLVM-Code-Generation-by-example)[1]

***

## 九、章末测验解析

**Q1：地址解析的三个主要步骤？**
→ ① 插入 Fixup ② 尝试 Relaxation ③ 记录 Relocation[1]

**Q2：汇编目标文件所需的三个目标特定组件？**
→ `MCCodeEmitter`、`MCAsmBackend`、`MCObjectTargetWriter`[1]

**Q3：`MCAsmBackend` 的职责？**
→ 处理 fixup 的应用、松弛过程，以及提供 `MCObjectTargetWriter` 实例[1]

**Q4：驱动文本/二进制输出的目标特定类？**
→ `AsmPrinter`（target-specific 子类）[1]

**Q5：决定输出类型的 MC 组件？**
→ `MCStreamer`：`MCAsmStreamer` 输出 `.s`，`MCObjectStreamer` 子类输出 `.o`[1]
