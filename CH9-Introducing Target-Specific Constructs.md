- 以章节为单位查看提交内容
  ```
  #Using these tags, you can easily see all the changes made in this chapter in one Git diff:
  $ git diff begin_ch9^..end_ch9
  #You can navigate all the commits made for this chapter:
  $ git log begin_ch9^..end_ch9
  #And you see all the changes per commit for this chapter:
  $ git log -p begin_ch9^..end_ch9
  ```
- step:
  - Registering your target in CMake
  - Providing a few required functions
    - void LLVMInitializeH2BLBTargetInfo(): This function registers the instance of the Target
      class, from the MC library, for this backend. We will describe what the Target class is shortly.
    - void LLVMInitializeH2BLBTargetMC(): This function registers the machine code (MC)
      components of this backend in the instance of the Target class of this backend. The MC
      components represent a low-level description of a target, such as the number of registers it
      has or how to produce object files. You will discover more about this as we progress through
      the book.
    - void LLVMInitializeH2BLBTarget(): This function registers the target-specific TargetMachine
      instance, from the Target library, in the instance of the Target class of this backend.
      TargetMachine provides methods to enable and control the various pieces of the LLVM codegen
      infrastructure. We will discuss this in more detail as we progress through the rest of this book
    - **LLVMInitializeH2BLBTargetInfo**：为后端注册MC库中的`Target`类实例。
    - **LLVMInitializeH2BLBTargetMC**：向后端`Target`实例注册机器码（MC）组件，描述寄存器数量、目标文件生成等底层信息。
    - **LLVMInitializeH2BLBTarget**：向后端`Target`实例注册目标相关`TargetMachine`实例，用于启用和控制LLVM代码生成模块。
- 这段代码片段展示了**LLVM 专用 CMake 函数**的使用，核心要点：
  - 1. **add_llvm_component_group**：创建名为 H2BLB 的组件，用于统一聚合该后端的所有依赖。
  - 2. **add_llvm_target**：新建 CMake 构建目标，指定源文件、通过 `LINK_COMPONENTS` 声明依赖库（H2BLBCodeGen、H2BLBDesc、H2BLBInfo、LLVM 通用库 Support），并用 `ADD_TO_COMPONENT` 将目标归入前面定义的组件，同时把当前目录加入头文件搜索路径。
  - 3. 内置函数 **add_subdirectory**：让 CMake 遍历子目录，加载更多 CMakeLists.txt 与构建目标。
  - 4. LLVM 专属 CMake 函数实现位于 `llvm/cmake` 目录下。
  ```
  add_llvm_component_group(H2BLB)
  add_llvm_target(H2BLBCodeGen
  H2BLBTargetMachine.cpp
  LINK_COMPONENTS
  H2BLBDesc
  H2BLBInfo
  
  Support
  ADD_TO_COMPONENT
  H2BLB
  )
  add_subdirectory(MCTargetDesc)
  add_subdirectory(TargetInfo)
  ```

- Adding a new architecture to the Triple class:
  - 1. **Triple类作用**：描述目标的三大特征——架构、厂商、操作系统，新增后端需为其添加架构条目。
  - 2. **核心步骤**
      - 在`llvm/include/llvm/TargetParser/Triple.h`中，给`ArchType`枚举添加新条目。
      - 在`llvm/lib/TargetParser/Triple.cpp`中，为该枚举值补充所有`ArchType`相关`switch`分支（如架构名称、前缀获取函数等）。
  - 3. **开关分支用途**：部分是字符串与枚举值映射，部分用于返回目标属性（如大小端）。
  - 4. **H2BLB目标示例**：32位小端架构，16位指针，兼容COFF、ELF、Mach-O三种目标文件格式。

这段内容解释的是 **LLVM 后端中 `TargetMachine` 构造函数的核心参数**，每个参数都用于精准配置目标硬件和编译行为。以下是逐词通俗解析：

- H2BLB Target Machine：这些参数共同定义了 LLVM 如何为特定硬件生成精准、高效的代码，是编译器后端开发的核心配置项
  -  1. **T（Target）**  目标后端的**单例对象**（Singleton），代表 LLVM 对特定硬件架构（如 X86、ARM）的整体抽象，是后端所有逻辑的入口。
  -  2. **TT（Triple）**  **目标三元组**（Target Triple），格式通常为 `架构-厂商-操作系统`（如 `x86_64-pc-linux-gnu`）。  
     - 作用：根据三元组的值，LLVM 会针对不同的**目标 OS、ABI 或架构变体**做差异化处理（例如 Windows 和 Linux 的调用约定不同）。
  -  3. **CPU**  **具体 CPU 型号字符串**（如 `skylake`、`haswell`）。  
     - 作用：同一架构下不同代次的 CPU 支持的指令集特性不同（例如 Skylake 支持 AVX-512，而 Haswell 不支持）。LLVM 会据此调整 `TargetMachine`，生成适配该 CPU 的最优代码。
  -  4. **FS（Feature String）**  **特性开关字符串**，用于显式启用（`+`）或禁用（`-`）特定硬件特性。  
     - 例子：`+sse2,-sse` 表示启用 SSE2 指令集，同时禁用基础 SSE 指令集。  
     - 对应 Clang 命令：通过 `-Xclang -target-feature` 传递（如 `-Xclang -target-feature -Xclang +sse2`）。
  -  5. **Options**  **目标默认行为选项**，控制编译时的底层语义。  
     - 例子：数学指令的处理方式（如是否允许“快速数学”优化，牺牲精度换速度）。
  -  6. **RM（Relocation Model）**  **重定位模型**，决定二进制中符号地址的解析方式。  
     - **静态模式**：函数/变量地址在编译时固定，直接调用。  
     - **动态模式**：地址在运行时通过重定位表动态解析（如动态库）。  
     - 对应 Clang 命令：`-mrelocation-model`（如 `-mrelocation-model pic` 生成位置无关代码）。
  -  7. **CM（Code Model）**  **代码模型**，告诉编译器最终二进制的预期大小，影响寻址方式。  
     - **小模式（默认）**：程序和符号需放在低 2GB 内存，可用相对寻址（如 PC 相对偏移）。  
     - **大模式**：需完整计算地址，支持更大的二进制。  
     - 对应 Clang 命令：`-mcmodel`（如 `-mcmodel large`）。
  -  8. **OL（Optimization Level）**  **优化级别**，对应 Clang 的 `-O0`（无优化）、`-O1`/`-O2`/`-O3`（不同程度优化）。
  -  9. **JIT**  **即时编译标志**，表示是否在 JIT（Just-In-Time）场景下运行。作用：JIT 模式下，LLVM 可能会简化优化流水线，以换取更快的编译速度（优先保证“即时生成代码”的效率）。
  ```
  llc --version
  ...
  Registered Targets:
  h2blb - How to build an LLVM backend by example
  ```
- Plumbing your Target through Clang:
  - 该代码片段用于声明 clang::TargetInfo 类的目标特定实现，核心需完成两点：一是在构造函数中通过 resetDataLayout 设置目标的数据布局字符串；二是实现基类的纯虚函数（如    getTargetDefines/getTargetBuiltins），这些函数为 Clang 提供目标专属信息（例：H2BLB 后端新增 __H2BLB__ 预处理宏）。
  - 接入 Clang 时只需实现最小可用版本（如 getTargetBuiltins 可返回 std::nullopt），可参考仓库中 H2BLBTargetInfo.cpp 的实现方式。
  - 需更新 clang/lib/Basic/CMakeLists.txt 中 clangBasic 库的 .cpp 文件列表，并在 Targets.cpp 的 AllocateTarget 函数开关语句中添加目标分支（如 llvm::Triple::h2blb），同时引入对应头文件，完成后端与 Clang 中 Triple 的关联。
  ```
  clang --print-targets
  Registered Targets:
  ...
  h2blb - How to build an LLVM backend by example

  clang -target h2blb -O0 -o - -emit-llvm -S input.c
  target datalayout = “e-p:16:16:16-n16:32-i32:32:32-i16:16:16-i1:8:8-
  f32:32:32-v32:32:32”
  target triple = “h2blb”
  ...
  ```
- Creating your own intrinsics: Intrinsics是 C/C++ 等语言层面暴露的特殊函数，可使用目标平台特定指令（如 AArch64/X86 的 AES 硬件加密指令）以加速算法。本节主要讲解如何为后端自定义 Intrinsics，包括让 Clang 识别并将其注册为合法的 LLVM IR 内置函数。Intrinsics 与汇编代码的关联会在后续章节介绍，需在 Machine IR 层搭建相关基础架构。先探讨核心问题：何时需要使用 Intrinsics。
  ```
  Intrinsics are special functions that are exposed at the source language level, such as C and C++, that
  allow to use of target-specific constructs. For instance, the Advanced Encryption Standard (AES)
  instruction set of the AArch64 or X86 backend exposes hardware-accelerated cryptographic instructions
  that you can leverage to speed up related algorithms.
  ```

- In the rest of this section, we will show you how to connect two intrinsics for our H2BLB backend：
  - 1. 目标XXX的专用固有函数定义在**IntrinsicsXXX.td**中，并被包含到`llvm/include/llvm/IR/Intrinsics.td`。
  - 2. 该文件经**gen-intrinsic-enums TableGen**处理，生成`IntrinsicsXXX.h`。
  - 3. 头文件在`llvm/lib/IR/Intrinsics.cpp`中引用，供LLVM IR层使用。
  - 实现自定义固有函数需四步：创建并填写`.td`文件、引入到总文件、对接TableGen后端、在`.cpp`中使用新生成头文件。

- What you see on the first line of this snippet is a sort of namespace, called TargetPrefix. If you remember your TableGen syntax from Chapter 6, you will know that this effectively sets a TargetPrefix field for all the records (that is, the instances identified with the def keyword) within this block.
  - 内置函数名以 int_ 开头 + 目标前缀 + 功能名，命名规则可自定义；
  - Intrinsic 类仅返回类型参数必填，其余参数控制优化 / 指令选择属性；
  - 支持预定义 / 自定义类型，示例函数为 16 位入参、32 位返回值。
  ```
  let TargetPrefix = “h2blb” in {
  def int_h2blb_widening_smul :
    # only the first argument, which describes the return type of the intrinsic, is mandatory.
    # return a 32-bit integer, and take two 16-bit integers as input.
    Intrinsic<[llvm_i32_ty], [llvm_i16_ty, llvm_i16_ty]>;
  def int_h2blb_widening_umul :
    Intrinsic<[llvm_i32_ty], [llvm_i16_ty, llvm_i16_ty]>;
  } // end TargetPrefix = “h2blb”
  ```
- intrinsic包含各个架构：https://github.com/llvm/llvm-project/blob/main/llvm/include/llvm/IR/Intrinsics.td
- llvm/include/llvm/IR/CMakeLists.txt
  ```
  tablegen(LLVM IntrinsicsH2BLB.h -gen-intrinsic-enums -intrinsic-prefix=h2blb)
  This line tells the build system to run the LLVM TableGen tool, ${BUILD_DIR}/bin/llvm-tblgen (the
  first argument), to produce IntrinsicsH2BLB.h (the second argument), using the -gen-intrinsic-
  enums -intrinsic-prefix=h2blb command-line options (the remaining arguments). 
  ```
 - intrinsic前端使用的文件
   
  <img width="610" height="200" alt="image" src="https://github.com/user-attachments/assets/41107fad-8124-4157-b795-987485ae3522" />
 
  - 1. 手动编写内置函数原型与属性字符串繁琐且后期难维护。
  - 2. gen-clang-builtins 的 TableGen 功能对 LLVM 编译器开发者十分实用。
  - 3. 本节将讲解新 TableGen 语法及构建系统中启用对应后端的方法，但不介绍全部功能。
  - 4. 相关类与定义在 `BuiltinsBase.td`，通用内置函数在 `Builtins.td`（含使用示例）。
  - 5. 定义 H2BLB 内置函数不再用 `.def` 文件，改为编写 `BuiltinsH2BLB.td`。
  ```
  int widening_signed_multiply(short a, short b) {
  return __builtin_h2blb_widening_smul(a, b); # 这个是前端生成的
  }
  # 转为ir
  define i32 @widening_signed_multiply(i16 signext %a, i16 signext %b) {
  entry:
  ret i32 %0
  %0 = tail call i32 @llvm.h2blb.widening.smul(i16 %a, i16 %b)
  }
  ```
- In this section, you learned how to create target-specific intrinsics at the LLVM IR level and how to connect them, all the way up to a source language level, through Clang. This offers your users a unique way to interact with your target.
- In the next section, we show you how to set up your TargetTransformInfo class. This class allows you to influence optimizers by providing custom cost modeling
- Introducing target-specific costs
  - 1. 在`TargetTransformInfo`类中引入**目标特定成本**，只需在`BasicTTIImplBase`的目标特定实现里重写对应方法。
  - 2. 需重写的方法由想要影响的**优化器**决定，可查看优化器实现来确定依赖的`TargetTransformInfo`接口。
  - 3. 可优先重写最实用的API，后续再逐步补全所有方法。
  - 4. 示例：若要控制**加载/存储向量化器**行为（如仅支持`<2 x i16>`加载），需在`H2BLBTTIImpl`类中新增并实现`getLoadVectorFactor`方法。
  ```
  # TargetTransformInfo（TTI）实现的一个成员，用于指导向量化器（如循环向量化器）在进行加载操作向量化时，决定实际可用的向量宽度因子
  unsigned H2BLBTTIImpl::getLoadVectorFactor(unsigned VF, unsigned LoadSize,
  unsigned ChainSizeInBytes, VectorType *VecTy) const {
    unsigned ElemSize = VecTy->getScalarSizeInBits();
    if (ElemSize != 16)
    return 0;
    return std::min(VF, 2u);
  }
  ```
  ```
  opt -passes=load-store-vectorizer %s -S -mtriple=h2blb input.ll -o -
  ```
- passmanager pipeline
  - 1. **注册自定义Pass**：需在`llvm/lib/Target/H2BLB/H2BLBPassRegistry.def`文件中，通过`FUNCTION_PASS`宏注册目标Pass，指定其命令行选项名（如`h2blb-simple-cst-prop`）和构造方式。
  - 2. **调用自定义Pass**：确保目标架构已注册（指定正确Triple值），可通过`opt`命令调用，示例：`opt -mtriple=h2blb -passes=h2blb-simple-cst-prop %s -S input.ll -o –`。
  - 3. **注入默认Pass流水线**：在`TargetMachine::registerPassBuilderCallbacks`中，通过`PassBuilder`的`registerXXXEPCallback`方法（如`registerPeepholeEPCallback`）将Pass插入默认流水线，插入位置取决于Pass功能。
  - 自定义Pass需先在指定`.def`文件中通过宏完成注册，明确命令行调用名和构造逻辑；
  - 可通过`opt`命令结合目标Triple值直接调用已注册的Pass；
  - 若要让Pass融入默认流水线，需在指定回调函数中通过`PassBuilder`的扩展点方法配置，插入位置需匹配Pass用途。
  ```
  opt -O1 -mtriple=h2blb -debug-pass-manager %s -S -o /dev/null
  ```
#### 附件
- https://github.com/llvm/llvm-project/blob/main/llvm/lib/TargetParser/Triple.cpp
