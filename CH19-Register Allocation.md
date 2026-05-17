## 概览：本章解决什么问题

寄存器分配（Register Allocation）是把 LLVM IR 中无限量的**虚拟寄存器**映射到目标机器上有限的**物理寄存器**的过程。当物理寄存器不够用时，分配器需要将部分值溢出（spill）到内存，并在使用前重新载入（reload）。[1]

本章覆盖三个层次：
- **必做**：让你的后端后端支持 spill/reload（实现两个 target hook）
- **理解**：LLVM 寄存器分配 pipeline 的结构和设计哲学（aggressive coalescing + live-range splitting）
- **进阶**：SlotIndex + LiveInterval 两套 liveness 表达体系，以及如何手动维护它们

***

## 一、LLVM 寄存器分配 Pipeline 总览

### 1.1 整体结构

LLVM 的寄存器分配不是一个单一的 monolithic pass，而是分成**两大优化 pass + 两大分析 pass**：[1]

| Pass 类型 | 名称 | 作用 |
|-----------|------|------|
| 优化 pass | Register Coalescer | 合并通过 COPY 连接的虚拟寄存器，消除 COPY 指令 |
| 优化 pass | Register Allocator（Assignment Phase） | 将虚拟寄存器替换为物理寄存器或内存位置 |
| 分析 pass | SlotIndexes | 为函数中每条指令分配唯一的程序点编号 |
| 分析 pass | LiveIntervals | 基于 SlotIndex 建立每个虚拟寄存器的活跃区间信息 |

PHI Elimination（第 13 章）把 Machine IR 从 SSA 形式转为 non-SSA 形式，随后 coalescer 在 non-SSA 的 Machine IR 上运行。在 coalescer 和 allocator 之间还有**预寄存器分配调度（pre-RA scheduling）**，其调度策略会影响寄存器压力，从而影响 allocator 的分配质量。[1]

### 1.2 LLVM 设计的三个"非典型"特点

LLVM 的寄存器分配设计与教科书中的图着色（graph coloring）算法有几处显著不同：[1]

**特点一：Coalescing 与 Assignment 分离**

传统算法通常在一个阶段完成，而 LLVM 将两个阶段分开处理，分别由独立的 pass 负责。[1]

**特点二：Aggressive Coalescing + 事后修正**

- **保守 coalescing（Conservative Coalescing）**：只在确保不增加寄存器数量的前提下才合并，传统算法常见[1]
- **激进 coalescing（Aggressive Coalescing）**：LLVM 的做法——不管寄存器压力如何，尽量消除 COPY 指令；如果产生过多的活跃变量，allocator 阶段会通过 **live-range splitting**（活跃区间切分）来"撤销"过激的 coalescing 决定[1]

```
Coalescer:
  %v1 = ...        →   %v1 = ...
  %v2 = COPY %v1       %v1 used directly (COPY eliminated)
  use %v2

Allocator（如果发现压力过大）:
  re-split %v1 live range，使 %v1 的不同 SSA 值在不同时间段活跃
```

**特点三：Machine IR 是 non-SSA，但 Liveness 仍以 SSA 形式表达**

这是 LLVM 最关键的"怪点"：寄存器分配运行在非 SSA 的 Machine IR 上，但内部的 LiveInterval 仍然用 SSA value 来标注活跃区间。这意味着你可以在 non-SSA 的 Machine IR 上借助 LiveInterval 做类似 SSA 的分析。[1]

***

## 二、为后端启用寄存器分配（必做部分）

### 2.1 连接默认 Pipeline

如果使用 LLVM 默认的 machine pass pipeline（第 13 章），寄存器分配基础设施已经自动就绪。如果自定义 pipeline，需要手动添加：[1]

```cpp
// 手动添加 coalescer
addPass(RegisterCoalescerID);

// 手动添加 allocator（以 Greedy 为例）
addPass(createGreedyRegisterAllocator());
```

> **注意**：`-O0` 时 coalescer 不运行，allocator 使用不同策略（fast allocator）。[1]

### 2.2 必须实现的两个 TargetInstrInfo 方法

只要你的后端不需要 spill，默认 pipeline 就能工作。一旦寄存器不够用，allocator 需要 spill，此时它会调用你后端实现的以下两个方法：[1]

| 方法 | 作用 |
|------|------|
| `storeRegToStackSlot` | 将寄存器的值存到栈上某个 frame index |
| `loadRegFromStackSlot` | 从栈上某个 frame index 载入寄存器 |

**完整代码示例（H2BLB 目标的 `storeRegToStackSlot`）**：[1]

```cpp
void H2BLBInstrInfo::storeRegToStackSlot(
    MachineBasicBlock &MBB,
    MachineBasicBlock::iterator MBBI,
    Register SrcReg, bool isKill, int FI,
    const TargetRegisterClass *RC,
    const TargetRegisterInfo *TRI,
    Register VReg,
    MachineInstr::MIFlag Flags) const {

  MachineFunction &MF = *MBB.getParent();
  MachineFrameInfo &MFI = MF.getFrameInfo();

  // 1. 构造内存元信息（MachineMemOperand），描述 store 操作的地址和大小
  MachinePointerInfo PtrInfo = MachinePointerInfo::getFixedStack(MF, FI);
  MachineMemOperand *MMO =
      MF.getMachineMemOperand(PtrInfo, MachineMemOperand::MOStore,
                               MFI.getObjectSize(FI), MFI.getObjectAlign(FI));

  // 2. 根据寄存器大小选择合适的 store 指令
  //    TRI->getSpillSize(*RC) 返回字节数，2 => 16-bit, 4 => 32-bit
  unsigned Opc = TRI->getSpillSize(*RC) == 2 ? H2BLB::STRSP16 : H2BLB::STRSP32;

  // 3. 确认 stack ID
  MFI.setStackID(FI, TargetStackID::Default);

  // 4. 使用 BuildMI 在指定位置插入 store 指令
  BuildMI(MBB, MBBI, DebugLoc(), get(Opc))
      .addReg(SrcReg, getKillRegState(isKill))  // 源寄存器
      .addFrameIndex(FI)                          // 目标栈槽
      .addImm(0)                                  // 偏移量
      .addMemOperand(MMO);                        // 内存元数据
}
```

**逻辑细节**：
- `MachinePointerInfo::getFixedStack` 创建一个代表"固定栈位置"的指针描述[1]
- `getSpillSize(*RC)` 让代码对不同寄存器大小自动适应，而不是硬编码[1]
- `getKillRegState(isKill)` 正确设置 kill flag，告知后续分析该寄存器在这条指令后是否不再活跃[1]

### 2.3 Rematerialization（重计算代替溢出）

某些值重算比溢出更便宜（如常量、立即数）。启用 trivial remat 只需在 TableGen 中设置：[1]

```tablegen
def MY_CONST_INST : Instruction {
  let isReMaterializable = 1;
  // ...
}
```

- LLVM 只支持 trivial rematerialization（操作数都是常量）[1]
- 复杂情况可 override `TargetInstrInfo` 中的 remat 相关虚函数[1]

### 2.4 可选的调优钩子

实现 `storeRegToStackSlot` 和 `loadRegFromStackSlot` 后，pipeline 就完整了。以下是可选的深度调优点：[1]

| 调优目标 | 方法/字段 |
|----------|-----------|
| 控制是否合并两个虚拟寄存器 | `TargetRegisterInfo::shouldCoalesce` |
| 控制物理寄存器的尝试顺序 | RegisterClass TableGen 中的 `AltOrders` 字段 |
| 给虚拟寄存器设置"偏好物理寄存器" | `TargetRegisterInfo::getRegAllocationHints` |
| 启用子寄存器级别的 liveness 追踪 | `TargetSubtargetInfo::enableSubRegLiveness` |

***

## 三、SlotIndex：程序点的精确编号

### 3.1 什么是 SlotIndex

SlotIndex 是对 `MachineFunction` 中所有指令的**单调递增编号**，跨 basic block 连续，用来唯一标识一个"程序点"。[1]

**示例 dump**（在 Machine IR 打印中出现）：

```
0B  bb.0 (%ir-block.1):
...
16B     %3:gpr32 = COPY $w0
...
224B    B %bb.1
240B bb.1 (%ir-block.5):
256B    ADJCALLSTACKDOWN 0, 0, ...
```

- 编号**不是连续的**（每条指令间隔 16），这是刻意留下的空隙，以便中间插入新指令而无需重编号整个函数[1]
- 如果插入了太多指令，会触发重编号（renumbering）——代价较大[1]

### 3.2 四个 Slot（执行阶段）

每个 SlotIndex 不只是一个数字，还关联一个 **slot**，代表该指令执行过程中的一个子阶段。每个指令的 SlotIndex 对应四个 slot，按序排列：[1]

```
Index  Slot  说明
32B    B     block  —— 基本块入口标记，无实际操作
32e    e     early-clobber  —— early-clobber 定义开始于此
32r    r     register  —— 正常定义/最后一次使用 发生于此
32d    d     dead  —— dead def 的终止 slot
```

**为什么需要 slot？** 因为寄存器分配需要知道"def 和 use 是否能用同一个物理寄存器"，而这取决于它们的活跃区间是否重叠：[1]

**场景一：Normal def（可以和 use 共用寄存器）**

```
use 的 live range:  ... 32r)   ← 在 32r 结束（不含）
def 的 live range:  

**场景二：Early-clobber def（不能和输入 use 共用寄存器）**

```
use 的 live range:  ... 32r)   ← 在 32r 结束
def 的 live range:  

**场景三：Dead def（定义了但立即死亡）**

```
dead def 的 live range: 

### 3.3 SlotIndexes API（维护映射关系时必用）

`SlotIndexes` 类维护 `SlotIndex <-> MachineInstr` 的映射表。在你的 pass 中修改指令时必须同步更新它[1]：

```cpp
// 当插入一条新指令时
SlotIndexes->insertMachineInstrInMaps(*NewMI);

// 当删除一条指令时
SlotIndexes->removeMachineInstrFromMaps(*OldMI);

// 辅助查询
SlotIndexes->getMBBFromIndex(SI);     // 找到 SlotIndex 所在的 MBB
SlotIndexes->getMBBStartIdx(MBB);     // 找到 MBB 起始的 SlotIndex
```

**Debug 技巧**：在 pass 中打印带 SlotIndex 的 MachineFunction[1]：

```cpp
MF.print(llvm::dbgs(), getAnalysisIfAvailable<SlotIndexes>());
```

---

## 四、LiveInterval：活跃区间与 SSA-in-Disguise

### 4.1 LiveInterval 的结构

`LiveInterval` 是某个虚拟寄存器的活跃信息，由两部分组成[1]：

**Segments（段）**：虚拟寄存器活跃的区间，格式为 `

**VNInfo（值编号信息）**：记录每个 SSA value 的定义点，格式为 `N@SlotIndex` 或 `N@SlotIndex-phi`

### 4.2 完整示例解读

以下是一段 Machine IR 和对应 `%10` 的 LiveInterval[1]：

```
Machine IR:                          Live Interval of %10:
0B  bb.0:                            Segments:
    ...                                

**逐点解析**：

- `[80r,320r:0)` 表示 SSA value 0（定义于 `COPY $w0`）在 slot 80r 到 320r（不含）之间活跃，横跨 bb.0 和 bb.1[1]
- `2@368B-phi` 是一个**抽象 phi 定义**——Machine IR 中 bb.2 入口没有 phi 指令（已被消除），但 LiveInterval 仍然记录这里有一个 phi 汇合点，并分配了新的 SSA value[1]
- 这就是"Machine IR 是 non-SSA，但 liveness 维护 SSA"的体现[1]

### 4.3 利用 LiveInterval 查找 Reaching Definitions（替代数据流分析）

由于 Machine IR 已经不是 SSA，传统方法需要跑一遍 reaching-definition 数据流分析。借助 LiveInterval，可以避免这个开销：[1]

**步骤如下**：

```
步骤 1：获取 LiveIntervals 分析对象
  - 新 pass manager：依赖 LiveIntervalsAnalysis
  - 旧 pass manager：依赖 LiveIntervalsWrapperPass

步骤 2：getInterval(VirtReg) → 得到该虚拟寄存器的 LiveInterval

步骤 3：把目标 use 的 MachineInstr 映射到 SlotIndex
  - 直接用 SlotIndexes，或用 LiveIntervals 提供的 wrapper

步骤 4：getVNInfoAt(SlotIndex) → 得到该 use 点对应的 VNInfo

步骤 5a：如果 VNInfo.isPHIDef() == false
    → 这就是 reaching definition
    → 通过 VNInfo.def 找到定义的 SlotIndex，再映射回 MachineInstr

步骤 5b：如果 VNInfo.isPHIDef() == true
    → 遍历该 phi 所在块的所有 predecessor blocks
    → 对每个 pred block 的末尾 SlotIndex 递归执行步骤 4
    → 直到找到非 phi 的 VNInfo
```

**代码示意**：

```cpp
LiveIntervals &LIS = getAnalysis<LiveIntervalsWrapperPass>().getLIS();
LiveInterval &LI = LIS.getInterval(VirtReg);

SlotIndex UseIdx = LIS.getInstructionIndex(*UseMI).getRegSlot();
VNInfo *VNI = LI.getVNInfoAt(UseIdx);

if (!VNI->isPHIDef()) {
  // 直接是 reaching def
  MachineInstr *DefMI = LIS.getInstructionFromIndex(VNI->def);
} else {
  // 是 phi，需要递归到前驱块末尾查
  for (MachineBasicBlock *Pred : MBB->predecessors()) {
    SlotIndex PredEnd = LIS.getMBBEndIdx(Pred);
    VNInfo *PredVNI = LI.getVNInfoAt(PredEnd.getPrevSlot());
    // 继续递归...
  }
}
```

### 4.4 常用 LiveInterval / LiveIntervals API

```cpp
// 检查 liveness
LI.liveAt(SlotIndex)         // 某虚拟寄存器在该程序点是否活跃？
LI.overlaps(OtherLI)         // 两个 LiveInterval 是否有干涉（不能同一物理寄存器）？

// 子寄存器 liveness（需要 enableSubRegLiveness）
LI.hasSubRanges()            // 是否有子寄存器 live range？
for (auto &SR : LI.subranges()) { ... }  // 遍历子寄存器 live range
```

***

## 五、维护 LiveIntervals（在 regalloc pipeline 中写 pass 时必须）

### 5.1 何时需要手动维护

- **不需要**：如果你的 pass 不依赖 LiveIntervals，pass manager 会在你修改 IR 后自动重算[1]
- **必须手动维护**：你的 pass 运行在 regalloc pipeline 中，且：
  - 既修改了 Machine IR，又依赖 LiveIntervals 的正确性
  - 或者你不想每次都重建 LiveIntervals（重建代价很高）[1]

### 5.2 维护流程（两步走）

**第一步：更新 SlotIndexes 映射**（见第三节 API）

**第二步：更新 LiveIntervals**：[1]

```cpp
// 场景1：新建了一个虚拟寄存器，但还没有 def/use
LIS.createEmptyInterval(NewVirtReg);

// 场景2：新建了虚拟寄存器，并且 def/use 都已经建好
LIS.createAndComputeVirtRegInterval(NewVirtReg);

// 场景3：删除了某个 use
LIS.shrinkToUses(&LI);

// 场景4：插入了新的 use
LIS.extendToIndices(LI, NewIndices);
```

### 5.3 验证手段

**务必开启 machine verifier** 来捕捉 SlotIndex / LiveInterval 维护遗漏的问题：[1]

```bash
llc -verify-machineinstrs your_file.ll
```

或者在 debug build 中使用：

```bash
llc -debug-only=regalloc your_file.ll
```

`-debug-only=regalloc` 会 dump 所有 regalloc 相关 pass 的调试信息，是排查分配问题的首选工具。[1]

***

## 六、知识点关联图

```
PHI Elimination（第13章）
        ↓
   Machine IR（non-SSA）
        ↓
Register Coalescer ─────────────── SlotIndexes（分析pass）
  • Aggressive coalescing                  ↓
  • 消除 COPY 指令          LiveIntervals（分析pass）
        ↓                    • 维护 SSA value 视角的 live range
Pre-RA Scheduling（第18章） • VNInfo 记录 def 和 phi 定义点
  • 影响寄存器压力
        ↓
Register Assignment（Allocator）
  • 将 virtual reg → physical reg
  • 当压力过大时 live-range split（撤销过激 coalescing）
  • 当无法分配时 spill → 调用 storeRegToStackSlot / loadRegFromStackSlot
        ↓
VirtReg Rewriter
  • 将 virtual reg 替换为 physical reg（或 spill slot）
        ↓
   Final Machine IR（全是物理寄存器）
```

***

## 七、常见面试 / 深度问题

**Q1：LLVM 为什么用 aggressive coalescing 而不是 conservative coalescing？**

Aggressive coalescing 能消除更多 COPY 指令，减少 move 开销。即使有时因此增加了寄存器压力，allocator 阶段可以通过 live-range splitting 来修正，相比 conservative coalescing 更灵活，总体效果更好。[1]

**Q2：Live interval 为什么要用 SSA value 来表达，而不是直接用 non-SSA 的虚拟寄存器？**

因为同一个虚拟寄存器在不同程序点可能持有不同的"值"（来自不同的 def 路径）。SSA value 粒度的 liveness 让你能精确知道某个 use 的 reaching definition 是谁，避免了需要额外 dataflow 分析的开销。[1]

**Q3：Slot index 为什么要设计成间隔为 16，而不是连续编号？**

留有空隙（holes）是为了支持在两条已编号指令之间插入新指令而不需要重新编号整个函数。如果连续编号，每次插入都会触发高代价的全局重编号。[1]

**Q4：什么情况下 rematerialization 比 spilling 更优？**

当值可以通过廉价指令重新计算（如加载常量、地址计算），remat 比 spill 省去了 store 和 load 两条内存指令的开销。[1]

***

## 八、扩展阅读

- Matthias Braun 2017 LLVM Developers Meeting 演讲（强烈推荐，尤其是第二部分专讲 regalloc 和 liveness 表达）：https://llvm.org/devmtg/2017-10/slides/Braun-Welcome%20to%20the%20Back%20End.pdf[1]
- 视频录像：https://www.youtube.com/watch?v=objxlZg01D0[1]
- 实际代码参考：companion repository tag `regalloc-hooks_ch19`（https://github.com/PacktPublishing/LLVM-Code-Generation-by-example）[1]
