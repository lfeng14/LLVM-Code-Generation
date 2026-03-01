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
