- At the high level, both the legacy and the new pass managers work the same way:
  1. Before running a pass, they check if the analyses that this pass relies on are available:
     - If yes, nothing needs to be done.
     - If no, they run these analyses beforehand.
  2. They run the pass.
  3. If the pass modified the IR, they check what kind of information the pass affects and preserves and invalidate the results of the analyses that are affected and not preserved.
- 旧版 Pass 管理器基于虚函数实现多态，每次调用方法都有运行时开销（虚表查找）。新版采用 CRTP（奇异递归模板模式） 来消除这种开销，CRTP 的原理：
  - 模板基类 P 接受子类作为模板参数：template <typename Derived> class P { ... };
  - 子类继承时将自己传入：class Child : public P<Child> { ... };
  - 在基类 P 中，可以通过 static_cast<Derived*>(this) 将 this 转换为子类指针，从而在编译期确定要调用的子类方法，实现静态多态。
  - 这样，方法分派在编译时通过模板实例化完成，不再需要虚函数表，减少了运行时的间接调用开销。
- legacy pass manager VS new pass manager
  <img width="1322" height="626" alt="image" src="https://github.com/user-attachments/assets/6f0a35ca-051f-4858-8897-68b78bbb730e" />
  <img width="1334" height="484" alt="image" src="https://github.com/user-attachments/assets/db07a505-8c6a-423a-b18e-0c4056d42973" />
  
- you will be able to use the related analysis in the runOnXXX method by calling Pass::getAnalysis</*PassClass*/>().
- legacy pass manager:
  - 生命周期方法：对于 FunctionPass，runOnFunction 和 releaseMemory 会为模块中的每个函数成对调用；而 doInitialization 和 doFinalization 则在整个模块处理前后各调用一次，且不同 Pass 的这些调用可以嵌套（如 PassB 的初始化和收尾在 PassA 的内部执行）。
  - 可选分析访问：getAnalysisIfAvailable<AnalysisType>() 允许在不声明强制依赖的情况下获取分析结果，若分析不可用则返回 nullptr，使 Pass 在缺少信息时仍能运行（只是优化效果可能降低），适用于锦上添花的场景。
 

#### further reading
- Using the New Pass Manager：https://llvm.org/docs/NewPassManager.html#implementing-analysis-invalidation
- Explanation of how to write a pass with the new pass manager: https://llvm.org/docs/WritingAnLLVMNewPMPass.html
- Explanation of how to write a pass with the legacy pass manager: https://llvm.org/docs/WritingAnLLVMPass.html
- Explanation of how the new pass manager works: https://llvm.org/docs/NewPassManager.html
