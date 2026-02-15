#### compiler jargon
- Target（目标）
   - 名词：程序运行的硬件架构。
   - 动词：为特定架构实现/定制编译器功能（如指令选择适配）。
   ```
     -X archs Build only the specified semi-colon-delimited list of backends(default: "ARM;AArch64;X86").
     -x arch  Build cross-compiling toolchain for the specified target.
   ```
- Host（宿主）
   - 运行编译器的设备。
   - 与Target不同时为交叉编译（如x86电脑编译AArch64手机代码）。
- Lowering（降级/底层化）
   - 编译过程中逐步降低程序抽象层级，从高级表示逼近目标机器汇编。
- Canonical form（规范形式）
   - LLVM中统一、推荐的表示方式，避免等价表达形式爆炸。
   - 不遵循可能影响代码生成质量，LLVM提供API自动规范化。
- build time：构建时间，比如ninja这种工具运行时间
- compile time：编译单个文件的时间
- run time：运行时间
- The target-agnostic part, which happens roughly before the target-specific part：目标无关 目标相关
- ABI:
  ```
   ABIs are extremely important as they provide the glue between functions. In other words, they formalize
   how the handshake happens between the caller and the callee. For instance, the ABI prescribes where
   the input arguments should be set by the caller and, thus, where the callee can expect to find its input
   arguments.
   Then, as long as compilers follow the rules of the ABI, a function compiled with a compiler can talk
   to a function compiled with a different compiler.
  ```
- Encoding:
  ```
   Encodings are target-specific and are usually provided by the hardware vendor through their Instruction
   Set Architecture (ISA) document.
  ```
- Module：translation unit (TU) or compilation unit (CU). A module is a container for the program currently being compiled.
- 构建测试
   ```
   mkdir build && cd build
   cmake -DCMAKE_PREFIX_PATH=~/src/jeandle-llvm/jeandle-llvm-install  ../
   make
   ./build_ir 
   ```
- 如何找到llvm的头文件以及库文件路径
   ```
   find_package(LLVM REQUIRED CONFIG) 的作用
   这条命令会查找 LLVM 安装时生成的 LLVMConfig.cmake 配置文件（通常位于 安装前缀/lib/cmake/llvm/）。
   一旦找到并执行该配置文件，CMake 会获得一系列关于 LLVM 的变量，例如：
   LLVM_INCLUDE_DIRS：LLVM 头文件的路径（如 /usr/local/include 或自定义安装路径下的 include）。
   LLVM_LIBRARY_DIRS：库文件路径。
   LLVM_DEFINITIONS：编译 LLVM 所需的预处理器定义。
   ```
- **Context 包含多个 module 包含多个 function 包含 basic block 包含 指令**
- Function:
  ```
   1. 函数本质：具名代码单元，包含计算逻辑，按签名调用，含参数、返回类型、属性（如noinline）等特征。
   2. LLVM 视角：不区分C++中的普通函数、成员函数、lambda等，后端统一视为函数，仅调用方式（调用约定CC）不同。
   3. 前后端分工：前端承担核心工作，如隐式参数生成（this指针）、虚函数多态代码（虚表查找）；后端仅简单处理函数。
   4. 后端唯一区分：函数是本模块内的定义，还是外部模块的声明。
  ```
- 函数声明和实体的区别：Function::empty()
- MachineFunction方式：Function &MachineFunction::getFunction())
  ```
   for(BasicBlock &MyBasicBlock: MyFunction) {
   // Do something with MyBasicBlock.
   }
   for(MachineBasicBlock &MyMachineBB: MyMachineFunction) {
   // Do something with MyMachineBB.
   }
  ```
#### 附件
- MachineFunction.h: https://llvm.org/doxygen/MachineFunction_8h.html
- MacheineFunction class: https://llvm.org/doxygen/classllvm_1_1MachineFunction.html
- https://llvm.org/doxygen/MachineFunction_8h_source.html
  ```
   find jeandle-llvm-install/include/llvm/ -name "*.h" | grep -i MachineFunction
   jeandle-llvm-install/include/llvm/CodeGen/MachineFunction.h
   jeandle-llvm-install/include/llvm/CodeGen/MachineFunctionAnalysis.h
   jeandle-llvm-install/include/llvm/CodeGen/MachineFunctionPass.h
   jeandle-llvm-install/include/llvm/CodeGen/MachineFunctionAnalysisManager.h
  ```
- IRRead.h https://llvm.org/doxygen/llvm_2IRReader_2IRReader_8h.html
- 
