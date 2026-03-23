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
- 
#### further reading
- 2015、2017、2019年LLVM开发者大会中**GlobalISel**相关的核心分享资源与内容脉络，是理解LLVM中SDISel与GlobalISel工作机制的重要参考，具体要点如下：
  -  **资源指向**：明确了三届大会中GlobalISel主题的演讲幻灯片官方链接，且推荐搭配对应的YouTube录制视频观看，从多视角理解该技术。
  -  **2015年（第九届，圣何塞）**：由苹果Quentin Colombet提出**GlobalISel设计提案**，核心针对传统SDISel（SelectionDAGISel）编译速度慢、作用域仅基本块、设计单一等缺陷，规划全新的全局指令选择框架以解决问题并提升代码生成能力。
  -  **2017年（第十一届，圣何塞）**：Quentin Colombet与Ahmed Bougacha分享GlobalISel**发展现状与未来规划**，介绍了框架设计与基础设施的优化、性能特征，以及后续开发的重点方向和社区协作需求。
  -  **2019年（圣何塞）**：SandersKeles带来《Generating Optimized Code with GlobalISel》分享，聚焦**基于GlobalISel生成优化代码**的相关实践与技术要点。
  -  **资源价值**：这些大会分享不仅讲解SDISel和GlobalISel的工作原理，还呈现了LLVM社区对该技术的研发演进、实际落地思路，对相关技术学习和研究具有重要参考意义。
