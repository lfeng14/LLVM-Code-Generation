- The first phase of the instruction selection pipeline consists of lowering the LLVM intermediate representation (IR) down to the generic IR of the instruction selection framework that you chose. This phase is called the IR building phase.
- 对 SDISel：一条 LLVM IR 指令对应一个或多个 SDNode 实例；
- However, you must provide the target hooks that implement the lowering of the ABI for your selector. Meaning that each selector, including FastISel, has its own target hooks to lower the ABI.

  <img width="650" height="200" alt="image" src="https://github.com/user-attachments/assets/958efa9c-b8e7-41f0-8080-28304c4a7913" />

- 调用约定说简单点：传参数使用哪个寄存器，读取参数使用哪个寄存器
- 结构体返回值降级，涉及调用约定（修改函数参数形式）：What sret demotion does is, if a value cannot be returned through the physical registers by the callee (typically, this happens when a structure is too large), then the caller creates a stack slot for the value and passes it to the callee as an additional argument. The callee then uses the stack slot to write the return values

  <img width="500" height="160" alt="image" src="https://github.com/user-attachments/assets/9e0b911a-885f-4514-9ede-031b2810118e" />
  
  the callee function returns a presumably large structure. If that structure cannot be returned via the dedicated physical registers (as defined by the ABI of the target), the signature of the callee function is changed so that it takes the address of the structure where it must write the result

- 后端通常需支持多种调用约定，不同操作系统、输入语言（如Swift、C++）会有不同转换规则。
  - 调用约定与名称关联，例如ARM架构AAPCS标准分32位aapcs32和64位aapcs64变体。
  - 建议以调用约定名称为核心标识，如在TableGen记录前添加对应名称前缀，便于区分归属。
 
- 函数入参、函数返回、csr调用约定：补丁为LLVM新增H2BLB目标平台的调用约定定义：
  - 定义通用调用约定`CC_H2BLB_Common`，将`v2i16`按`i32`处理，`i16/f16`分配至`R1-R3`，`i32/f32`分配至`D1`，剩余参数入栈（2字节对齐）。
  - 定义返回值约定`RetCC_H2BLB_Common`，规则与参数传递一致。
  - 声明被调用者保存寄存器`CSR`为`D2`、`D3`。
  - let Entry = 1 表示llvm调用空间
  ```
  --- /dev/null
  +++ b/llvm/lib/Target/H2BLB/H2BLBCallingConvention.td
  @@ -0,0 +1,17 @@
  +let Entry = 1 in
  +def CC_H2BLB_Common : CallingConv<[
  +  CCIfType<[v2i16], CCBitConvertToType<i32>>,
  +  CCIfType<[i16,f16], CCAssignToReg<[R1, R2, R3]>>,
  +  CCIfType<[i32,f32], CCAssignToReg<[D1]>>,
  +
  +  CCAssignToStack<2, 2>
  +]>;
  +
  +let Entry = 1 in
  +def RetCC_H2BLB_Common : CallingConv<[
  +  CCIfType<[v2i16], CCBitConvertToType<i32>>,
  +  CCIfType<[i16,f16], CCAssignToReg<[R1, R2, R3]>>,
  +  CCIfType<[i32,f32], CCAssignToReg<[D1]>>
  +]>;
  +
  +def CSR : CalleeSavedRegs<(add D2, D3)>;
  ```
- When you call a function, you need three pieces of information: where its arguments need to be, where its results need to be, and which registers are preserved by this function call, meaning which registers can live through the call of the function without being clobbered.
#### further reading
- aarch64调用约定：https://github.com/jeandle/jeandle-llvm/blob/main/llvm/lib/Target/AArch64/AArch64CallingConvention.td
- aapcs64调用约定：https://github.com/ARM-software/abi-aa/releases/download/2024Q3/aapcs64.pdf
- 像 Linux 内核的 Documentation 目录里，从架构细节到驱动开发都有详细文档，还有 Git 的官方文档，从基础命令到内部实现原理都讲得很透。另外，PostgreSQL 的文档也很全面，从 SQL 语法到数据库内核架构都有覆盖。你平时更关注编译器相关的项目，还是其他领域的？那可以试试看看 Redis 的文档，它把数据结构、持久化机制这些核心原理讲得很通俗，还有 WebKit 的文档，对浏览器渲染引擎的细节解释得很到位，这两个项目的文档和 LLVM 一样，都属于 “能让自学党啃明白” 的类型。
