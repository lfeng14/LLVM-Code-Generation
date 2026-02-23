- This cheat sheet is obviously not exhaustive, but we believe it should give you a better grasp of how LLVM IR is structured and how it maps to the related APIs.
  <img width="1076" height="1286" alt="image" src="https://github.com/user-attachments/assets/da39fde7-2a51-4e6b-9ab6-e2ac20226e02" />
- 
  ```
  $ cmake -DCMAKE_PREFIX_PATH=/usr  ../
  $ make VERBOSE=1
  ```
- Another example is noinline, which tells the compiler that this function must not be inlined in any of its callers. Notice how noreturn defines a property of the function and noinline gives direction to the compiler.
  ```
  在了解 LLVM 基础架构后，本部分重点讲解其中间层（middle-end）与中间表示（IR），它是基于 LLVM 的优化编译器核心，包含可复用的优化与分析模块。
  主要围绕四点展开：
  讲解 LLVM IR 及其与高级语言（如 C++）的映射关系
  介绍 LLVM 中间层核心优化技术
  说明目标相关结构在中间层的适配
  教授 LLVM Pass 的调试方法
  学完可掌握 LLVM IR 与 Pass 操作、调试能力，为后续后端（Machine IR）学习奠定基础，对应第 7–10 章内容。
  ```
- The content of a basic block is a list of instructions with a mandatory terminator instruction at the end.
- All instructions map to the same base class called Instruction; A lot of the instructions then use their own derived class; for instance, the br instruction is implemented by the BranchInst class, and the phi instruction is implemented by the PHINode class.
- 找到对应**C++类**通常比较简单。文本IR中的操作码（opcode）与同名C++类**基本一一对应**。举例：`bitcast` 对应 `BitCastInst`，`int_to_ptr` 对应 `IntToPtrInst`。
- LLVM IR supports three main classes of types as base types: integer, floating-point, and pointer types.
- All types, except the label type (see the next section) and void type, can also use the constants undef and poison (e.g., i32 undef and ptr poison).
  ```
  undef versus poison
  undef is a bit of a misnomer because while it represents an undefined value (a random
  pattern of bit), it really means that we do not care about the content of this value. In other
  words, it has nothing to do with the undefined behavior. On the other hand, poison means
  that the value is poisoned, and you should never read it because otherwise the behavior
  is undefined.
  ```
- There are two main aggregate types:
  - The structure type is a collection of fields of different types
  - The array type is a statically sized collection of elements of the same type
#### 附件
- https://llvm.org/docs/LangRef.html#parameter-attributes
