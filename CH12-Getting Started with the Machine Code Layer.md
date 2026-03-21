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

- MC层的InstructionPrinter和MCCodeEmitter很多代码都是TableGen自动生成的。比如在目标架构的.td文件里定义指令的汇编格式和二进制编码规则，TableGen会根据这些定义生成对应的打印和编码逻辑，不用手动写大量重复的switch-case语句。ableGen 可自动生成 XXXInstPrinter、XXXMCCodeEmitter 类的大部分实现，但无法生成 XXXAsmParser 类。虽可通过回调扩展寄存器、指令描述，但相关解析方法仍需手动实现，XXXAsmParser 大量代码需自行编写。

#### further reading
- https://developer.arm.com/documentation/ddi0487/latest/
```

