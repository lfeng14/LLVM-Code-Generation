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
  ```
   $ cat input.c
   extern int baz();
   extern void bar(int);
   
   void foo(int a, int b) {
     int var = a + b;
     if (var == 0xFF) {
       bar(var);
       var = baz();
     }
     bar(var);
   }
   
   $ cat input.ll
   ; ModuleID = 'input.c'
   source_filename = "input.c"
   target datalayout = "e-m:e-i8:8:32-i16:16:32-i64:64-i128:128-n32:64-S128"
   target triple = "aarch64-unknown-linux-gnu"
   
   ; Function Attrs: noinline nounwind optnone uwtable
   define dso_local void @foo(i32 noundef %0, i32 noundef %1) #0 {
     %3 = alloca i32, align 4
     %4 = alloca i32, align 4
     %5 = alloca i32, align 4
     store i32 %0, ptr %3, align 4
     store i32 %1, ptr %4, align 4
     %6 = load i32, ptr %3, align 4
     %7 = load i32, ptr %4, align 4
     %8 = add nsw i32 %6, %7
     store i32 %8, ptr %5, align 4
     %9 = load i32, ptr %5, align 4
     %10 = icmp eq i32 %9, 255
     br i1 %10, label %11, label %14
   
   11:                                               ; preds = %2
     %12 = load i32, ptr %5, align 4
     call void @bar(i32 noundef %12)
     %13 = call i32 @baz()
     store i32 %13, ptr %5, align 4
     br label %14
   
   14:                                               ; preds = %11, %2
     %15 = load i32, ptr %5, align 4
     call void @bar(i32 noundef %15)
     ret void
   }
   
   $ llc -print-after-all -filter-print-funcs=foo input.ll -o -
   	.text
   	.file	"input.c"
   	.globl	foo                             // -- Begin function foo
   	.p2align	2
   	.type	foo,@function
   foo:                                    // @foo
   	.cfi_startproc
   // %bb.0:
   	sub	sp, sp, #32
   	.cfi_def_cfa_offset 32
   	stp	x29, x30, [sp, #16]             // 16-byte Folded Spill
   	add	x29, sp, #16
   	.cfi_def_cfa w29, 16
   	.cfi_offset w30, -8
   	.cfi_offset w29, -16
   	stur	w0, [x29, #-4]
   	str	w1, [sp, #8]
   	ldur	w8, [x29, #-4]
   	ldr	w9, [sp, #8]
   	add	w8, w8, w9
   	str	w8, [sp, #4]
   	ldr	w8, [sp, #4]
   	cmp	w8, #255
   	b.ne	.LBB0_2
   // %bb.1:
   	ldr	w0, [sp, #4]
   	bl	bar
   	bl	baz
   	str	w0, [sp, #4]
   .LBB0_2:
   	ldr	w0, [sp, #4]
   	bl	bar
   	.cfi_def_cfa wsp, 32
   	ldp	x29, x30, [sp, #16]             // 16-byte Folded Reload
   	add	sp, sp, #32
   	.cfi_def_cfa_offset 0
   	.cfi_restore w30
   	.cfi_restore w29
   	ret
   .Lfunc_end0:
   	.size	foo, .Lfunc_end0-foo
   	.cfi_endproc
                                           // -- End function
   	.ident	"Ubuntu clang version 18.1.3 (1ubuntu1)"
   	.section	".note.GNU-stack","",@progbits
  ```
- basic block: （**注意这里是bb块内的单个路径只有一个入口一个出口,只能从块的最后一条指令离开**）A basic block is a single-entry single-exit (SESE) region of code where the entry point is at the beginning of the region and the exit point at the end.
  ;basic blocks are formed such that they are maximal;
  ```
   This implies that when you
   hit an instruction, by definition, all the instructions before this instruction within the same block have
   been executed, and all the instructions remaining in this basic block will be executed.
  ```
  ```
   1. 基本块（BasicBlock）中：
      - 用 `BasicBlock::getFirstNonPHI()` 获取**首条非PHI指令**
      - 用 `BasicBlock::getTerminator()` 获取**终结指令**
   2. 对基本块做局部修改时，应遍历 **`getFirstNonPHI()` 到 `getTerminator()`** 之间的指令。
  ```
- There are a couple of differences compared to the BasicBlock class:
  ```
   • APIs: begin(), end(), and getParent()
   • MachineBasicBlock instances can have zero or several terminator instructions.
   • MachineBasicBlock instances offer a direct API to traverse their predecessors and successors through predXXX and succXXX methods. 
  ```
- llvm instruction method: moveBefore, moveAfter, insertBefore gerParent
- llvm machine instruction method: MachineInstr::defs()) and arguments (MachineInstr::uses())
- CFG: The first node executed in a CFG is called the entry point, and the last possible ones are called exit points
- LLVM offers an API, named ReversePostOrderTraversal, to use RPO on your functions out of the box. This API is part of the ADT library.A reverse post-order (RPO) traversal defines an order in which the nodes of a CFG are visited.
  ```
   Here is an example of using an RPO traversal in the LLVM IR:
   ReversePostOrderTraversal<Function *> RPOT(&MyFunction);
   For (BasicBlock *BB: RPOT) {
   // Do something in topological order.
   }
   The following snippet shows the use of an RPO traversal in the Machine IR:
   ReversePostOrderTraversal<MachineFunction *> RPOT(&MyMachineFunction);
   For (MachineBasicBlock *MBB: RPOT) {
   // Do something in topological order.
   }
  ```
- 【作业】自己写一个RPO示例：逆后续遍历，就是后续遍历的反向；那什么是后续遍历，先返回子节点在访问根节点；
  ```
   // clang++ -std=c++14 -o printBBs printBBs.cpp $(llvm-config --cxxflags --ldflags --libs core irreader)
   #include "llvm/IR/LLVMContext.h"
   #include "llvm/IR/Module.h"
   #include "llvm/IR/Function.h"
   #include "llvm/IR/BasicBlock.h"
   #include "llvm/IR/CFG.h"               // 提供 GraphTraits<Function*> 特化
   #include "llvm/ADT/PostOrderIterator.h"
   #include "llvm/Support/SourceMgr.h"
   #include "llvm/Support/raw_ostream.h"
   #include "llvm/IRReader/IRReader.h"
   #include <memory>
   
   using namespace llvm;
   
   void printBasicBlocksInReversePostOrder(Function &F) {
       // 使用 ReversePostOrderTraversal 对函数 F 的基本块进行逆后序遍历
       ReversePostOrderTraversal<Function*> RPOT(&F);
   
       outs() << "Function: " << F.getName() << " (Reverse Post Order)\n";
   
       for (BasicBlock *BB : RPOT) {
           if (BB->hasName()) {
               outs() << "  " << BB->getName() << "\n";
           } else {
               outs() << "  BB@" << BB << "\n";
           }
           // 可选：打印基本块中的指令
           for (Instruction &I : *BB) {
                outs() << "    " << I << "\n";
            }
       }
   }
   
   int main(int argc, char **argv) {
       if (argc != 2) {
           errs() << "Usage: " << argv[0] << " <IR file>\n";
           return 1;
       }
   
       LLVMContext Context;
       SMDiagnostic Err;
       std::unique_ptr<Module> MyModule = parseIRFile(argv[1], Err, Context);
       if (!MyModule) {
           Err.print(argv[0], errs());
           return 1;
       }
   
       for (Function &F : *MyModule) {
           if (!F.isDeclaration()) {
               printBasicBlocksInReversePostOrder(F);
           }
       }
   
       return 0;
   }
  ```
- Backedge: a backedge is a control flow edge that goes backward. In other words, this is an edge that takes you back in the execution of your program; identifying backedges is important to identify loops.
   ```
   To put things together, identifying backedges is important to identify loops. Identifying loops is
   important since loops are where most programs spend most of their runtime, hence, they are the
   primary focus for compiler optimizations.
   ```
- Critical edge: At the IR level, you can check if an edge is critical using the isCriticalEdge function from the IR library.
#### 附件
- MachineFunction.h https://llvm.org/doxygen/MachineFunction_8h.html
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
- MachineBasicBlock https://llvm.org/doxygen/classllvm_1_1MachineBasicBlock.html
