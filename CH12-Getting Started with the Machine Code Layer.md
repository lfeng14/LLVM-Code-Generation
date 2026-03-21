- git diff begin_ch12^..end_ch12 > ch12_diff
  
- you can see that the Machine IR gets compiled to the MC representation. Then you can print the MC representation to produce assembly files (.s), in other words, the textual representation of an object file, or assemble it to produce the binary file that is typically called the object file (.o). These are the two main output paths used with MC.

  <img width="400" height="200" alt="image" src="https://github.com/user-attachments/assets/a4d0c06a-fc93-4f78-a772-2f55c46ba640" />

- An assembler takes an assembly file (.s) and produces an object file (.o). The way this works with the LLVM infrastructure is that you need to provide a parser to go from a .s file to the MC representation and then an assembler from MC to the .o file.
- MC层是连接高级中间表示与最终机器代码的桥梁，MC层是编译器输出（汇编文件.s或对象文件.o）的"胶水"层。实现汇编和反汇编。
- 指令编号zm zn 分别是5个bit，0～31寄存器编号；In this figure, you can see that the SME FP16 widening outer product instruction is a 32-bit instruction, as captured by the numbering of the bits in the table. In this table, Zm, Pm, and so on are the operands of the related instruction. Therefore, the operand Zn, for instance, is encoded with 5 bits, starting at the 5th bit in this instruction encoding.
核心结论：指令设计并非凭空创造，而是由目标架构明确决定。先搭建 MC 层的优势：可直接启用汇编器、反汇编器等底层工具。实际价值：无需编译器支持，早期用户即可通过编写汇编代码使用该目标架构。

  <img width="500" height="150" alt="image" src="https://github.com/user-attachments/assets/b4af06cd-833a-4ee1-a75f-e96cb968af32" />

- 如果要将指令转为可读，那么需要知道对应的枚举值表示什么寄存器。反汇编器则需掌握反向解码逻辑，例如知晓寄存器名r3与二进制0b11的互转规则。
- Finally, the assembler must know how this register is encoded in an instruction, meaning which bits it must set in the sequence of bits of the current instruction to indicate to the hardware where to read/write the value from/to. A disassembler needs the same kind of information but in the opposite direction; that is, if the assembler must know how to encode the information, the disassembler must know how to decode it – for instance, knowing that the string r3 translates in 0b11 in binary for the encoding, and the other way around for the decoding.
- 操作数0、1分别对应s0 s1
  ```
  let HwEncoding = 0 in
  def s0 : Register<"s0">;
  let HwEncoding = 1 in
  def s1 : Register<"s1">;
  ```
- llvm-mc command-line tool, which is the LLVM-provided way for developers to test the MC layer of their backend.
  - The human-readable assembly code printer: Represented by the MCInstrPrinter base class, this component is responsible for converting MCInst instances to human-readable strings. This component is represented by the Print arrow in Figure 12.2.
  - The assembly code parser: Represented by the MCTargetAsmParser base class, this component is responsible for converting a human-readable string into an MCInst instance. This component is represented by the Parse arrow in Figure 12.2.
  - The binary assembly code printer: Represented by the MCCodeEmitter base class, this component is responsible for converting MCInst instances to binary code. This component is represented by the Assemble arrow in Figure 12.2.

  <img width="800" height="600" alt="image" src="https://github.com/user-attachments/assets/7812ee2b-e255-4906-860f-dd3a185c4618" />

- MC层的InstructionPrinter和MCCodeEmitter很多代码都是TableGen自动生成的。比如在目标架构的.td文件里定义指令的汇编格式和二进制编码规则，TableGen会根据这些定义生成对应的打印和编码逻辑，不用手动写大量重复的switch-case语句。TableGen 可自动生成 XXXInstPrinter、XXXMCCodeEmitter 类的大部分实现，但无法生成 XXXAsmParser 类。虽可通过回调扩展寄存器、指令描述，但相关解析方法仍需手动实现，XXXAsmParser 大量代码需自行编写。

- 这段代码的核心逻辑总结：
  - 检查 `MCInstPrinter` 的 `PrintAliases` 配置，判断是否需要打印指令别名。
  - 若为 `true`，调用 TableGen 生成的 `printAliasInstr` 打印指令别名；否则调用 `printInstruction` 打印原指令。
  - 最后调用父类方法，打印指令附带的注解（如注释）。
  ```
  void XXXInstPrinter::printInst(const MCInst *MI, uint64_t Address,
  StringRef Annot, const MCSubtargetInfo &STI,
  raw_ostream &O) {
    # 写法挺简单，若PrintAliases为false则打印printInstruction，PrintAliases为true则printAliasInstr；最后都要执行printAnnotation
    if (!PrintAliases || !printAliasInstr(MI, Address, O))
      printInstruction(MI, Address, O);
    printAnnotation(O, Annot);
  }
  ```
- 打印操作数，判断寄存器类型、立即数
  ```
  void XXXInstPrinter::printOperand(const MCInst *MI, unsigned OpNo,
  raw_ostream &O) {
      const MCOperand &Op = MI->getOperand(OpNo);
      if (Op.isReg()) { // 
        unsigned Reg = Op.getReg();
        O << getRegisterName(Reg);
      } else if (Op.isImm()) {
        O << formatImm(Op.getImm());
      } else {
        assert(Op.isExpr() && "unknown operand kind in printOperand");
        Op.getExpr()->print(O, &MAI);
      }
  }
  ```
- Implementing your own MCInstPrinter class
  - **声明函数**：在 `XXXInstPrinter.h` 文件中声明 `createXXXMCInstPrinter` 函数。
  - **实现函数**：在 `XXXInstPrinter.cpp`（或多数后端选择的 `XXXMCTargetDesc.cpp`）中实现该函数。
  - **注册函数**：在 `LLVMInitializeXXXTargetMC` 函数（位于 `XXXMCTargetDesc.cpp`）内，调用 `TargetRegistry::RegisterMCInstPrinter` 注册上述创建的函数。
- code emit
  - 代码实现前做了两个简化假设：目标指令采用**32位编码**、为**小端序**。
  - 实现逻辑：调用 TableGen 生成的 `getBinaryCodeForInstr` 方法获取指令二进制码。
  - 调用支持库的 `endian::write` 函数，按正确参数将编码写入 `CB` 对应的字节数组中。
  ```
  void XXXMCCodeEmitter::encodeInstruction(const MCInst &MI,
    SmallVectorImpl<char> &CB,
    SmallVectorImpl<MCFixup> &Fixups,
    const MCSubtargetInfo &STI) const {
      uint64_t Encoding = getBinaryCodeForInstr(MI, Fixups, STI);
      support::endian::write<uint32_t>(CB, Encoding, llvm::endianness::little);
  }
  ```
- asm parser: 写汇编 Parser 从可视化汇编转 MC，首先得明确目标架构的指令集，比如 x86 或 ARM，因为不同架构的指令格式和编码规则不一样。然后需要拆解步骤：先做词法分析，把汇编语句拆分成操作码、寄存器、立即数等 token；再做语法分析，根据指令语法规则判断语句是否合法；最后根据架构的编码表，把合法的语法结构转换成二进制机器码。中间还要处理伪指令、标签跳转的地址计算这些细节。we’ll provide an example of how to write a basic assembly parser in the companion repository of this book at the basic-asm-parser_ch12 tag.
- llvm-mc --show-encoding
  ```
  $ cat test.c
  int main () {
    int a = 1 + 1;
    return 0;
  }

  $ clang test.c -S -o test.s
  
  $ llvm-mc --show-encoding test.s
  	.text
  	.file	"test.c"
  	.globl	main
  	.p2align	2
  	.type	main,@function
  main:
  	.cfi_startproc
  	sub	sp, sp, #16                     // encoding: [0xff,0x43,0x00,0xd1]
  	.cfi_def_cfa_offset 16
  	mov	w0, wzr                         // encoding: [0xe0,0x03,0x1f,0x2a]
  	str	wzr, [sp, #12]                  // encoding: [0xff,0x0f,0x00,0xb9]
  	mov	w8, #2                          // encoding: [0x48,0x00,0x80,0x52]
                                          // =0x2
  	str	w8, [sp, #8]                    // encoding: [0xe8,0x0b,0x00,0xb9]
  	add	sp, sp, #16                     // encoding: [0xff,0x43,0x00,0x91]
  	.cfi_def_cfa_offset 0
  	ret                                     // encoding: [0xc0,0x03,0x5f,0xd6]
  .Lfunc_end0:
  	.size	main, .Lfunc_end0-main
  	.cfi_endproc
  	.ident	"Ubuntu clang version 18.1.3 (1ubuntu1)"
  	.section	".note.GNU-stack","",@progbits
  	.addrsig
  ```
- h2blb asm parser
  - 该后端支持的语法比指令章节中定义MC层的语法更简单。
  - 不使用赋值运算符，采用更通用的汇编语法。
  - 格式类似：`opcode def0,def1,src0,src1`。
  - 所有操作数跟在操作码后，定义先于参数。
- 语法解析完成后，需将解析器与 TargetMachine 实例关联：
  - 为后端添加 AsmParser 库。
  - 必须实现 `LLVMInitializeXXXAsmParser` 函数。
  - 该函数由 `InitializeAllAsmParsers` 调用，供需要汇编解析功能的 LLVM 工具使用。
- 只有需要传递到**MC层**的指令才必须进行编码。原因：机器层可能会使用**伪指令**（如 PHI、COPY 指令），以配合后端通用部分工作。
- MCCodeEmitter class? The MCCodeEmitter class is responsible for translating an MCInst instance into a sequence of bytes. In other words, it is responsible for encoding the instructions
- R格式：寄存器操作，如ADD X0, X1, X2；I格式：立即数操作，如ADDI X0, X1, #10；D格式：加载/存储，如LDR X0, [X1, #8]
- 
#### further reading
- https://developer.arm.com/documentation/ddi0487/latest/
```

