- This cheat sheet is obviously not exhaustive, but we believe it should give you a better grasp of how LLVM IR is structured and how it maps to the related APIs.
  <img width="600" height="800" alt="image" src="https://github.com/user-attachments/assets/da39fde7-2a51-4e6b-9ab6-e2ac20226e02" />
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
- type: single type、label type、aggregate type
  
  <img width="600" height="800" alt="image" src="https://github.com/user-attachments/assets/148f1b59-7e45-46f0-b51a-0b6cb411da32" />

  ```
  define i32 @foo(i32, i32, i32 %arg) {
    entry:
      %myid = add i32 %0, %1
      %31 = mul i32 %myid, 2
      %45 = shl i32 %31, 5
      %"00~random~00" = udiv i32 %45, %arg
      br label %46  # %46 label type在哪里，指下一行
    
      br label %47
    47:
     ret i32 %"00~random~00"
  }
  ```
- 没有entry label，则隐式命令为%2，所以下面的IR：
  - 第一种改法为 后续从%3开始；
  - 第二种改法为 显式命令为bb
  ```
  define i32 @add(i32 %0, i32 %1) {
    %2 = add i32 %0, %1
    ret i32 %2
  }
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
    - {i32, {ptr, half}, {i32, i1, i1}}；获取half成员 %addr_half_field = getelementptr inbounds %my.type, ptr %dst, i64 0, i32 1, i32 1
  - The array type is a statically sized collection of elements of the same type

- All the LLVM IR types derive from the Type class：IntegerType, FunctionType, StructType, VectorType, PointerType,
  ```
  bool isVectorOfIntV2(Instruction &Add) {
  Type *Ty = Add.getType();
  return Ty->isVectorTy() &&
  Ty->getScalarType()->isIntegerTy();
  }
  ```
- intrinsic 区分：generic intrinsic(llvm.vector.reduce.add.xxx); target-specific(isTargetIntrinsic: llvm.aarch64.ldxr)
  1. 判断LLVM中`CallInst`（`CallBase`子类）是否为内部函数调用，需先获取调用目标。
  2. 通过`CallBase::getCalledFunction()`获取被调用函数。
  3. 两种方式判断是否为通用内部函数：
     - 直接用`Function::isIntrinsic()`；
     - 用`Function::getIntrinsicID()`，判断结果非`Intrinsic::not_intrinsic`。
  4. 两种方式判断是否为目标相关内部函数：
     - 直接用`Function::isTargetIntrinsic()`；
     - 仅持有`Intrinsic::ID`时，使用静态方法`Function::isTargetIntrinsic(Intrinsic::ID)`。
- triple
  - The target architecture, for instance, x86_64 or aarch64
  - The vendor, for instance, Apple or Nvidia
  - The OS, for instance, macOS or iOS
- data layout是triple的体现，可以这么说两者是一体的：影响对齐 todo；
  ```
  target datalayout = "e-m:e-i8:8:32-i16:16:32-i64:64-i128:128-n32:64-S128"
  target triple = "aarch64-unknown-linux-gnu"
  ```
- IR分两个格式，文本格式、位码格式。文本格式如果是旧版本编译器生成，可以通过先转为位码格式，然后喂给新版本编译器。bitcodereader class from bitreader library(bitcode reader)；autoupgrade 的命令，它已经融入到 LLVM 工具链的血液中了。只需用最新版的工具处理旧文件即可

- GetElementPtr
  ```
  GEP 只算地址，不碰内存。
  第二个操作数是必须存在的指针，且后面必须跟索引。
  索引个数必须与类型结构严格匹配，不能多余。
  末尾的零索引不影响地址，但影响类型；开头的零索引同时影响地址语义（剥离顶层）和类型，因此都不能省略。
  ```
- Are vector types aggregate types?
   ```
    No, they are not.
    Vector types are single-value types because of the following:
    They cannot use non-single value types as element types
    They usually map to simple low-level constructs such as a plain register used in a SIMD
    instruction
    See the Single-value types section for more details.
   ```
- 怎么去除隐式变量：
  ```
  cat > input.ll
  define i32 @add(i32 %0, i32 %1) {
  bb:
  %2 = add i32 %0, %1
  ret i32 %2
  }
  $ 
  $ opt --passes=instnamer -S input.ll -o -
  ; ModuleID = 'input.ll'
  source_filename = "input.ll"
  
  define i32 @add(i32 %arg, i32 %arg1) {
  bb:
    %i = add i32 %arg, %arg1
    ret i32 %i
  }
  ```
#### 附件
- https://llvm.org/docs/LangRef.html#parameter-attributes
- type类：https://llvm.org/doxygen/classllvm_1_1Type.html
- datalayout：https://llvm.org/docs/LangRef.html#data-layout
- vector-type：https://llvm.org/docs/LangRef.html#vector-type
- https://llvm.org/doxygen/BitcodeReader_8h_source.html
- GetElementPtr：https://llvm.org/docs/GetElementPtr.html
- https://llvm.org/devmtg/2020-09/slides/Lee-UndefPoison.pdf
