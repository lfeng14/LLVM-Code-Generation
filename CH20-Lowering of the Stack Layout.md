# LLVM Codegen 第20章学习笔记：Stack Layout Lowering

## 章节概览

第20章的核心任务是：**把编译器前端/中端使用的抽象栈表示（frame index），最终变成真实的机器指令和栈地址**。这是 codegen pipeline 倒数第二关，完成之后只剩对象文件生成。[1]

整个过程分两个大步骤：
1. **分配 + 布局 Stack Frame**：确定 frame 大小、对齐、各对象位置，生成 prologue/epilogue 指令。[1]
2. **展开 Frame Index**：把所有 MachineInstr 里的 `@Idx` 符号替换成真实的 `SP + offset` 或 `FP + offset` 寻址。[1]

***

## 一、核心概念：Stack Slot 与 Frame Index

### Frame Index 是什么

Frame index 是一个**抽象编号**，代表栈上的某个 slot（内存区域）。你可以把它类比成"虚拟寄存器"，只是它存的是栈地址而非寄存器名——只有等到 PEI pass 才会被解析成真实地址。[1]

- Frame index 由 `MachineFrameInfo` 类管理，维护了 frame index → stack slot 属性（大小、对齐、偏移）的映射。[1]
- 从 `MachineFunction` 实例通过 `getFrameInfo()` 方法获取 `MachineFrameInfo` 实例。[1]

### Fixed Stack Slot vs. 普通 Stack Slot

| 类型 | 创建方法 | 位置约束 | 典型用途 |
|------|----------|----------|----------|
| Fixed Stack Slot | `CreateFixedObject()` | 由 ABI 严格规定，不可优化移动 | 函数参数区、caller/callee 协议位置 |
| 普通 Stack Slot（spill） | `CreateSpillStackObject()` | 编译器自由分配，满足对齐即可 | RA 产生的 spill slot、局部变量 |
| Fixed Spill Slot（特殊） | `CreateFixedSpillStackObject()` | 固定位置的 spill | 要求 callee-saved 在特定位置时 |

[1]

**关键点**：StackSlotColoring pass 仅优化非固定的 stack slot（合并生命周期不重叠的 spill slot），固定 slot 的位置不受影响。[1]

***

## 二、四大组件及分工（必须建立全局视图）

PEI（Prologue/Epilogue Inserter）pass 是整个 stack lowering 的**驱动引擎**，它调用以下四个组件完成工作：[1]

```
PEI Pass（驱动）
  ├── MachineFrameInfo       → 持有 frame index → stack slot 映射表
  ├── TargetFrameLowering    → 生成 prologue/epilogue 指令序列
  ├── TargetRegisterInfo     → 把每个 frame index 展开成真实地址
  └── (RegScavenger)         → 为展开过程提供临时寄存器
```

完成 PEI pass 之后，codegen pipeline 里**不应再有任何 frame index**，全部已展开为真实的寄存器 + 偏移寻址。[1]

***

## 三、Stack Frame 布局详解

### 栈增长方向

- **向下增长（最常见）**：高地址是栈底，SP 向低地址移动（`SP -= frame_size`）。[1]
- 向上增长：低地址是栈底，SP 向高地址增长。
- 增长方向决定 prologue 中 SP 是做加法还是减法，以及 offset 的计算方向。

### 典型 Stack Frame 结构（向下增长示意）

```
高地址 ─────────────────────────────────
       │  caller frame（上一帧）          │
       │    包含传入本函数的 input args    │
       ├─────────────────────────────────┤  ← 本帧起始（FP 指向此处）
       │  callee-saved 寄存器 spill 区   │
       ├─────────────────────────────────┤
       │  局部变量 / 普通 spill slots     │  ← 大小编译期已知
       ├─────────────────────────────────┤
       │  动态分配对象（alloca）          │  ← 运行时大小
       ├─────────────────────────────────┤
       │  next callee 的 input/output args│  ← call frame 区域
低地址 ─────────────────────────────────  ← SP 指向此处
```



**Frame Pointer vs Stack Pointer**：
- 有动态分配对象（如 `alloca`）时，需要专用寄存器保存帧起始地址 → **Frame Pointer (FP)**。
- 无动态分配对象时，`FP = SP + frame_size`（编译期常量），可以省掉 FP。[1]
- `hasFP()` 方法负责告诉 LLVM 当前函数是否需要 FP。[1]

***

## 四、Reserved Call Frame 模式（重要设计决策）

`TargetFrameLowering::hasReservedCallFrame()` 控制调用栈空间的分配策略，返回值决定两种模式：[1]

### 模式对比

| 模式 | 策略 | 代码大小 | 内存占用 |
|------|------|----------|----------|
| **Reserved（推荐）** | prologue 一次性分配 `local + max(callees)` 的空间 | 更小（无 adj 指令） | 更大（整个生命周期持有最大 call frame） |
| **Non-Reserved** | prologue 只分配 local，每次 call 前后动态 adj | 更大（有 ADJCALLSTACKDOWN/UP） | 更小（精确按需） |

[1]

**结论**：除非目标设备内存极度受限，推荐使用 Reserved 模式，避免每次函数调用都生成额外的栈调整指令。[1]

LLVM 默认行为：当 target 不需要 FP 时自动使用 Reserved 模式。[1]

***

## 五、TargetFrameLowering：必须实现的四个 Hook

### 5.1 hasFP
告诉 PEI 当前函数是否需要 Frame Pointer，影响后续所有 offset 的计算基准。[1]

### 5.2 emitPrologue（核心）

负责在函数入口生成栈空间分配指令，以及保存 callee-saved 寄存器。[1]

**H2BLB 示例实现**：

```cpp
void H2BLBFrameLowering::emitPrologue(MachineFunction &MF,
                                       MachineBasicBlock &MBB) const {
    MachineFrameInfo &MFI = MF.getFrameInfo();
    unsigned NumBytes = MFI.getStackSize();  // 查询需要分配的字节数
    if (NumBytes > 0) {
        const TargetInstrInfo *TII = MF.getSubtarget().getInstrInfo();
        BuildMI(MBB, MBB.begin(), DebugLoc(),
                TII->get(H2BLB::SUBSP), H2BLB::SP)   // SP = SP - NumBytes
            .addReg(H2BLB::SP)
            .addImm(NumBytes)
            .setMIFlag(MachineInstr::FrameSetup);     // 标记为 frame setup 指令
    }
}
```



**细节逻辑**：
- `MFI.getStackSize()` 已经由 LLVM 算好（local slots + spill slots + reserved call frame 大小）。[1]
- 向下增长，所以用**减法**调整 SP。[1]
- `MachineInstr::FrameSetup` flag 是可选的调试标记，方便 dump 时识别 frame 相关指令（epilogue 对应 `FrameDestroy`）。[1]

### 5.3 emitEpilogue

与 prologue 镜像：恢复 callee-saved 寄存器，释放栈空间（`SP += frame_size`）。[1]

### 5.4 eliminateCallFramePseudoInstr

处理 call 周围的 `ADJCALLSTACKDOWN` / `ADJCALLSTACKUP` 伪指令。[1]
- 使用 **Reserved 模式**时：提供空实现即可（prologue/epilogue 已经处理了所有空间）。[1]
- Non-reserved 模式：需要在此展开为真实的 SP 调整指令。[1]

***

## 六、TargetRegisterInfo：eliminateFrameIndex 展开过程

### 6.1 展开的两种场景

展开 frame index 时，核心目标是将 `load @Idx` 替换为 `load SP, offset`：[1]

```
场景A（Good）：指令寻址模式能直接折叠 offset
  原始：$r1 = load @Idx1
  展开：$r1 = load $sp, 32        ← 直接一条指令搞定

场景B（Bad）：offset 超过编码范围，需要临时寄存器
  原始：$r1 = load @Idx1
  展开：? = add $sp, 32           ← 需要临时寄存器 ? 
        $r1 = load ?, 0
```



### 6.2 H2BLB 示例实现（eliminateFrameIndex）

**Step 1：获取 offset（frame pointer 视角）**

```cpp
MachineOperand &FIOp = MI.getOperand(FIOperandNum);
int Index = FIOp.getIndex();
int64_t Offset = MFI.getObjectOffset(Index);  // 相对于 FP 的偏移
```



**Step 2：转换为 SP 视角**

由于不使用 FP，需要加上 frame 大小把偏移从 FP 视角转换到 SP 视角：[1]

```cpp
Offset += MFI.getStackSize();   // FP_offset → SP_offset
```

**Step 3：按 opcode 展开（switch 分发）**

```cpp
switch (MI.getOpcode()) {
case H2BLB::LDRSP16:
    // 把 frame index 操作数替换为 SP 寄存器
    FIOp.ChangeToRegister(H2BLB::SP, /*IsDef=*/false);
    // 合并现有 const offset 和计算出的 frame offset
    Offset += MI.getOperand(2).getImm();
    // 断言 offset 在编码范围内（否则需要 RegScavenger 介入）
    assert(Offset >= -64 && Offset < 63 && "Offset must fit 7 bits");
    MI.getOperand(2).setImm(Offset);
    break;
// ... 其他 opcode
}
```



**Step 4：返回值**

返回 `bool`，表示传入的 `II` 迭代器是否失效（即是否在展开中删除了原指令）。大多数情况返回 `false`。[1]

### 6.3 需要实现 eliminateFrameIndex 的指令范围

只需支持**会产生 frame index 的指令**，主要来源：[1]
1. ABI lowering 阶段（Chapter 15）生成的指令（参数传递相关）。
2. 寄存器分配器生成的 spill/reload 指令。

***

## 七、Register Scavenger 机制（处理 Bad Scenario）

### 7.1 问题背景

展开 frame index 时可能需要临时寄存器（如场景 B 中的 `?`）。但此时 RA 已完成，没有虚拟寄存器可分配——需要在已分配的物理寄存器里**找一个当前程序点死掉（dead）的**来临时使用。[1]

### 7.2 RegScavenger 工作原理

```
enterBasicBlockEnd()   → 以 live-out 集初始化活跃寄存器集合
backward(II)           → 反向迭代更新 liveness（必须逆序扫描到关注的程序点）
FindUnusedReg() /
getRegsAvailable()     → 查询当前程序点可用寄存器
scavengeRegisterBackwards()  → 强制拿一个寄存器（可能触发 emergency spill）
```



**关键**：在 `eliminateFrameIndex` 中，PEI pass 已经帮你维护了 `RegScavenger` 实例，并通过 `RS` 参数传给你，**不需要自己管理生命周期**。[1]

### 7.3 Emergency Spill Slot（保底机制）

当 `scavengeRegisterBackwards()` 在当前程序点找不到任何空闲寄存器时，需要把某个寄存器**临时 spill 到栈上**，用完再恢复——这个专用的栈 slot 就是 **emergency spill slot**。[1]

**预留时机**：在 `determineCalleeSaves` 或 `processFunctionBeforeFrameFinalized` 方法里（这两个方法有 `RegScavenger` 参数）。[1]

**示例逻辑**：

```cpp
void MyFrameLowering::processFunctionBeforeFrameFinalized(
    MachineFunction &MF, RegScavenger *RS) const {
    MachineFrameInfo &MFI = MF.getFrameInfo();
    // 只在真的有 stack object 时才预留（避免空函数也有 prologue）
    if (MFI.hasStackObjects()) {
        // 为临时寄存器预留一个 slot（大小 = 目标寄存器大小，如 4 bytes）
        int FI = MFI.CreateStackObject(4, Align(4), false);
        RS->addScavengingFrameIndex(FI);  // 注册到 scavenger
    }
}
```



**注意事项**：[1]
- 需要几个同时使用的临时寄存器，就预留几个 emergency spill slot（不是共享）。
- 不建议无条件预留（会让每个函数都产生不必要的 prologue/epilogue）。
- 用 `MFI.hasStackObjects()` 做条件判断是个不错的实践折中。
- 参考实现：SystemZ 的 `processFunctionBeforeFrameFinalized`，AArch64 的 `determineCalleeSaves`。

***

## 八、完整数据流：从 IR 到汇编的栈处理链

```
LLVM IR (alloca / stack object)
          │
          ▼
  指令选择阶段（Chapter 15）
  ├── ABI 参数 → CreateFixedObject()   → Fixed Frame Index
  └── 局部变量 → CreateStackObject()  → 普通 Frame Index
          │
          ▼
  寄存器分配（RA）
  └── spill → CreateSpillStackObject() → Spill Frame Index
          │
          ▼
  StackSlotColoring Pass
  └── 合并不重叠的非固定 spill slot
          │
          ▼
  PEI Pass（本章主角）
  ├── 调用 TargetFrameLowering::emitPrologue()
  │     └── 生成 SUB SP, SP, #frame_size 等指令
  ├── 调用 TargetFrameLowering::emitEpilogue()
  │     └── 生成恢复指令
  └── 对每个 frame index → 调用 TargetRegisterInfo::eliminateFrameIndex()
        ├── 简单场景：直接 ChangeToRegister(SP) + setImm(offset)
        └── 复杂场景：通过 RegScavenger 拿临时寄存器，多条指令展开
          │
          ▼
  输出：纯物理寄存器 + 立即数偏移的 MachineInstr（无 frame index）
          │
          ▼
  汇编打印（Chapter 21：对象文件生成）
```

***

## 九、常见错误与调试

| 错误 | 原因 | 解决方案 |
|------|------|----------|
| `Cannot scavenge register without an emergency spill slot` | frame index 展开需要临时寄存器，但未预留 emergency spill slot | 在 `processFunctionBeforeFrameFinalized` 或 `determineCalleeSaves` 中预留 slot 并 `addScavengingFrameIndex()` |
| `Offset must fit N bits` assertion 触发 | frame 过大，offset 超出指令编码范围 | 用 RegScavenger 拿一个临时寄存器，先 materialize 大偏移 |
| frame index 展开后地址错误 | FP-to-SP offset 转换遗漏了 `getStackSize()` | 确认 `Offset += MFI.getStackSize()` 被执行（没有 FP 时必须做此转换） |
| prologue/epilogue 出现在不该出现的地方 | 无条件预留了 emergency spill slot 导致所有函数都有 frame | 用 `MFI.hasStackObjects()` 判断是否真需要 |

[1]

***

## 十、章节测验答案（自检用）

1. **Fixed stack slot 代表什么？** → 必须在栈上有确定地址的对象，通常由 ABI 规定，caller/callee 双方依赖此位置交换数据。[1]

2. **四个主要组件？** → `MachineFrameInfo`、`TargetFrameLowering`、`TargetRegisterInfo`、PEI pass。[1]

3. **Reserved call frame 的优缺点？** → 优点：代码序列短、高效；缺点：函数整个生命周期内栈空间占用更大。[1]

4. **为什么需要 scavenge 寄存器？** → frame index 展开时可能需要额外指令组装大 offset，需要临时寄存器持有中间计算结果，但此时 RA 已完成，无法分配虚拟寄存器。[1]

5. **需要自己维护 RegScavenger 实例吗？** → 不需要，PEI pass 已经维护并通过参数传给你。[1]

6. **Emergency spill slot 是什么？** → 提前预留的栈 slot，在 scavenger 找不到空闲寄存器时，用于临时 spill 某个寄存器以腾出空间给 frame index 展开。[1]
