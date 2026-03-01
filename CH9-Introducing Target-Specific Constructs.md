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
