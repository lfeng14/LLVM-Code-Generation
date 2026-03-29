- part 4 总结：
  - 机器中间表示（IR）的原理已揭示，接下来将学习如何从 LLVM IR 生成它。
  - 核心是**LLVM IR 到机器 IR 的转换**，该过程称为**指令选择**，LLVM 提供多种对应框架。
  - 将学习不同指令选择框架的使用方法，以及提供所需的**目标特定信息**，包括目标ABI、支持指令、LLVM IR到目标指令的映射。
  - 学完后可实现完整可用的指令选择功能，为后续机器级优化奠定基础。
  - 本部分包含第14–17章，依次介绍指令选择入门、IR构建阶段、合法化阶段、选择及后续阶段。

- git diff begin_ch14^..end_ch14 > ch14_diff
- LLVM 提供**两套半**指令选择（ISel）框架：
  - **SelectionDAG（SDISel）**：传统指令选择器。
  - **GlobalISel（GISel）**：新一代指令选择器。
  - **FastISel**：属于 SDISel 的子框架。
- Instruction Selection 工作机制总结
  - **SDISel 与 GlobalISel 通用流程**
      - **IR builder**：将 LLVM IR 转换为选择器框架的中间表示；
      - **Legalization**：把后端指令集不支持的 IR 结构改写为支持形式；
      - **Selection**：将选择器框架 IR 替换为后端机器指令 IR；
      - 阶段间可穿插优化，灵活度因框架而异。
  - **FastISel 特点**
      - 是 SDISel 内部的子选择器；
      - 一站式完成 IR 转换、合法化、选择，直接从 LLVM IR 生成机器指令 IR；
      - 编译速度更快，因此得名 FastISel。

- 图1- 1流程要点总结
  - 菱形代表**条件判断**（是/否）。
  - 后端选用**GlobalISel**时，先由其选择LLVM IR函数。
  - 若**GlobalISel失败**，通过`ResetMachineFunction`重置状态，转由**SDISel**处理。
  - **SDISel**逐块处理基本块：优先用**FastISel**，失败则由SDISel补全未选中部分。
  - 无对应选择器时，**回退路径失效**（如无FastISel则不尝试）。

  <img width="650" height="500" alt="image" src="https://github.com/user-attachments/assets/62b0771d-536d-40b8-a519-6cad542c1970" />

- 如果后端未提供两个主要选择器之一，代码生成器的 Pass 管理器将无法实例化 Pass 流水线，编译会立即失败。就像之前sme编译失败的
- **指令选择（Instruction Selection）** 的 **回退机制**。
  - 简单理解：LLVM 现在有两套指令选择框架——老的 `SelectionDAG`（SDISel）和新的 `GlobalISel`。当使用 GlobalISel 时，如果遇到它当前无法处理的情况（比如某些复杂操作或尚未实现的后端特性），编译器可以 **自动回退** 到 SelectionDAG 来完成这部分代码的指令选择。
    - **回退可能静默发生**  
       如果你没留意，可能代码实际走的是回退路径，而你写的 GlobalISel 选择器代码根本没被调用；或者即便被调用，后续经过 `ResetMachineFunction` 这类 pass 后状态被重置，导致你调试时找不到原因。  
       → 所以要清楚自己的指令选择流水线到底跑的是哪套机制。
    - **利用回退机制做渐进迁移**  
       把旧后端从 SelectionDAG 移植到 GlobalISel 时，可以先让 GlobalISel 只处理自己能处理的部分，其余回退到旧的 SelectionDAG。这样能逐步迁移，保证功能完整。  
       → 文本说后面会介绍如何用 API 和命令行选项来控制这套流水线。
    总之，这个回退机制是为了保证兼容性和可迁移性，但需要开发者清楚它何时被触发，避免调试时产生困惑。
- 编译时三种指令选择器总结
  - **FastISel**
      - 目标：**极快编译速度**
      -  trade-off：通用性差，常需回退到 SDISel。
  - **SDISel**
      - 目标：生成**最优指令序列**
      -  trade-off：编译耗时显著更高，常占后端总编译时间约 20%。
  - **GlobalISel**
      - 定位：**FastISel 与 SDISel 之间的平衡方案**
      - 特点：基础模式轻量快速，可叠加优化提升代码质量（对应 O0～O3 不同级别）；
      - 性能：Apple GPU 后端表现与 SDISel 相当且快约 2 倍；AArch64 无回退时仅比 FastISel 慢 - 1～- 2 倍，速度竞争力强。
- 这段内容核心要点总结：
  - 示例中 `%i3`、`%i4` 是 `%arg1`、`%arg2` 经符号扩展（sext）得到，在 `bb` 块生成，却在 `bb5`、`bb6` 块使用。
  - 后端支持**宽位乘法**，可直接完成16位到32位的符号扩展，理想情况是直接匹配 `mul(sext i16 to i32, ...)` 生成对应指令。
  - **SDISel 局限**：受基本块作用域限制，无法跨块折叠该模式；且基于DAG，不能表示循环、处理PHI节点，需靠预处理等迂回方案，仍存在无法解决的场景。
  - **GlobalISel 优势**：可访问整个函数作用域，无需此类变通方法。
  - 本节概述了LLVM选择器的核心流程（IR构建、合法化、选择）、回退机制及三大设计维度（编译时间、模块化可测性、作用域），后续将给出选择器实现建议。
  ```
  define i32 @smul_widen(i32 %arg, i16 %arg1, i16 %arg2) {
    bb:
      %i = icmp eq i32 %arg, 0
      %i3 = sext i16 %arg1 to i32
      %i4 = sext i16 %arg2 to i32
      br i1 %i, label %bb6, label %bb5
    bb5:
      tail call void @bar(i32 %i3, i32 %i4)
      br label %bb6
    bb6:
      %i7 = mul nsw i32 %i4, %i3
      ret i32 %i7
  }
  ```
- FastISel 与 SDISel、GlobalISel要点总结
  - FastISel
    - 优点：编译速度快，适合输入 LLVM IR 范围窄的场景；代码质量可通过投入优化做到最优；是完整的 C++ 指令选择器实现。
    - 缺点：自主实现成本高，仅靠它难以满足全部需求，通常必须搭配完整 SDISel 使用。
  - SDISel
    - 优点：LLVM 长期主流指令选择器，生态完善、稳定性强，经多年迭代优化，上手初期效率高。
    - 缺点：优化逻辑可控性差、灵活性不足；编译开销大，单元测试难；优化范围有局限，需额外前后置 Pass 弥补。
  - GlobalISel
    - **设计目标**：旨在消除 SDISel 的僵化与局限，是理想的指令选择框架。
    - **现存问题**：缺少实用功能（如适配函数作用域的领域专用语言、专用 TableGen 后端），优化能力不如 SDISel 丰富，存在使用短板。
    - **使用建议**：问题不构成阻碍，但使用体验仍不完善；追求**面向未来**且能接受不完美可选 GlobalISel，还可参与开源完善；追求**成熟稳定**则选 SDISel。
- SDNode：表示 SDISel 中间表示（IR）中的一条指令。 SDValue：表示一条指令的其中一个结果，是对 SDNode 及其结果列表中对应索引的封装。
- 核心结构：SDISel 采用 DAG 表示，节点为操作，有向边代表依赖关系。三类依赖关系
  - 数据依赖：源节点读取目标节点生成的值，为使用 - 定义链（与常规定义 - 使用链方向相反），是最常见依赖。
  - 调度依赖：DAG 线性化时源节点需先于目标节点执行，用于约束指令执行顺序（如无法判定别名的加载、存储指令）。
  - 粘连依赖：DAG 线性化时源、目标节点需紧邻，用于压缩指令序列、缩短物理寄存器活跃范围。
  - 实现类：DAG 由SelectionDAG类实现，节点为SDNode及其子类，边为节点间指针，依赖类型由源节点值类型决定。

- selection dag object：
  ```
  SelectionDAG has 9 nodes:
    t0: ch,glue = EntryToken
        t2: i16,ch = CopyFromReg t0, Register:i16 %0
        t4: i16,ch = CopyFromReg t0, Register:i16 %1
      t5: i16 = add t2, t4
    # 该节点将加法结果t5存入物理寄存器 $r1，并输出新的链和 glue。其中glue是为了让后续使用该寄存器的节点（如下一行的 H2BLBISD::RETURN_GLUE）能够正确地与当前节点形成顺序依赖
    t7: ch,glue = CopyToReg t0, Register:i16 $r1, t5
    # 依赖t7.0 glue
    t8: ch = H2BLBISD::RETURN_GLUE t7, Register:i16 $r1, t7:1
  ```
  - 通用操作码：llvm/include/llvm/CodeGen/ISDOpcodes.h
- 节点会尽力缩进以直观展示父子依赖关系，子节点缩进层级高于父节点。
  - 该缩进仅为尽力实现：若一个节点存在多个父节点，因节点只打印一次，缩进仅对首个打印的父节点有效。
  - 注意：SDISel 中父子关系方向反转，使用者（父节点）指向其定义（子节点）。

- EVT class represents your types, the SDNode class your instructions, and the SDValue class a particular result of your instruction。
- MachineInstr实例的操作码可使用带**G_前缀**的通用Machine IR操作码，易于识别。例：通用加法指令对应**G_ADD**操作码。此类操作码定义在文件：**llvm/include/llvm/Target/GenericOpcodes.td**。
  - %0:gpr32 → 普通虚拟寄存器，已指定寄存器类 gpr32。
  - %1:gpr32(s32) → 带标量类型（32位）的虚拟寄存器。
  - %2:_(p0) → 通用虚拟寄存器，无寄存器类/组，类型为地址空间0的指针。
  - %3:gprb(<2 x s16>) → 使用寄存器组 gprb，类型为 <2 x s16> 向量。

<img width="1326" height="754" alt="image" src="https://github.com/user-attachments/assets/a112f445-fef5-486e-8c0b-da4e6b9c72ce" />

- If you need help finding the method to override or anything else, feel free to look at the example at the commit tagged connect-tgtinfo-apis_ch14 in the companion repository
- respectively, the global-isel and the fast-isel command-line options. For instance, the -fast-isel=0 option will disable FastISel altogether.
- You can change this setting using the global-isel-abort command-line option with 0 to fall back silently, 1 to abort on fallbacks, and 2 to emit the warning.
- You learned the specificities of the generic IR used by each selector, namely the SelectionDAG representation for SDISel and the generic Machine IR for GlobalISel, and discovered which APIs to use to manipulate these IRs

#### further reading
- https://www.llvm.org/devmtg/2015-10/slides/Colombet-GlobalInstructionSelection.pdf
- https://llvm.org/devmtg/2017-10/slides/Bougacha-Colombet-GlobalISel.pdf
- https://llvm.org/devmtg/2019-10/slides/SandersKeles-GeneratingOptimizedCodewithGlobalISel.pdf
