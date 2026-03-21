- machine IR不同阶段
  <img width="600" height="150" alt="image" src="https://github.com/user-attachments/assets/da3b388d-cf8e-4be2-aa4a-d17e939109da" />
  - PHI 消除：用 COPY 指令替代 PHI，打破 SSA 形式，让虚拟寄存器可以被多次定义。
  - 寄存器分配：把虚拟寄存器映射到物理寄存器，发现 %1 和 %8 可以复用同一个物理寄存器。
  - COPY 消除：既然 %1 和 %8 都在同一个物理寄存器里，中间的 COPY 就变成冗余，被优化掉。
  - 最终结果：原来的 PHI 和 COPY 都消失了，只剩下一条直接操作物理寄存器的 ADDXri 指令。

- This default pipeline is set by the [TargetPassConfig::addMachinePasses](https://github.com/llvm/llvm-project/blob/3b9106149c6867494049d7af4958eebeec5e2393/llvm/lib/CodeGen/TargetPassConfig.cpp#L1115-L1322) method. While you can overload this method so that you have a completely customized Machine pass pipeline, we recommend that you stick with the default implementation

- TargetPassConfig 提供的钩子可在默认编译优化流水线的不同阶段插入自定义 Pass。
  - 若重写 TargetPassConfig::addMachinePasses 默认实现，这些钩子不会被调用。
  - 默认流水线已包含大量通用优化，无需重复添加。
  - 该类通过 addXXX 虚方法提供插入点，如：
  - 重写 addPreRegAlloc：在寄存器分配前插入 Pass；
  - 重写 addPreISel：在指令选择前（仍为 LLVM IR 阶段）插入 Pass。
  - 更多插入点可参考图 13.1 或源码文件 llvm/lib/CodeGen/TargetPassConfig.cpp 中的 addMachinePasses 实现。
  - 
#### further reading
- https://llvm.org/docs/CodeGenerator.html
