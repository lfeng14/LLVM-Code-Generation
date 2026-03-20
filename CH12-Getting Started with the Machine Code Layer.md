- git diff begin_ch12^..end_ch12 > ch12_diff
  
- you can see that the Machine IR gets compiled to the MC representation. Then you can print the MC representation to produce assembly files (.s), in other words, the textual representation of an object file, or assemble it to produce the binary file that is typically called the object file (.o). These are the two main output paths used with MC.
  <img width="856" height="456" alt="image" src="https://github.com/user-attachments/assets/a4d0c06a-fc93-4f78-a772-2f55c46ba640" />

 这一章教你如何通过TableGen描述来增强寄存器和指令的描述，从而构建MC层的基础。具体包括：
-描述汇编语法
-描述指令编码

主要专业术语解释

1.MC层

LLVM中用于处理目标架构最终指令表示的层次。MC层是编译器输出（汇编文件.s或对象文件.o）的"胶水"层。

2.TableGen

LLVM的域特定语言（DSL），用于描述目标架构的寄存器、指令等信息。TableGen通过后端工具自动生成C++代码。

3.MCInst/MCOperand

-MCInst：MC层中表示单个指令的类
-MCOperand：表示指令操作数的类（寄存器、立即数、符号等）

4.ISA(InstructionSetArchitecture)

指令集架构。文档中每个目标架构的ISA文档会列出所有支持的指令、它们的编码方式以及汇编语法。

5.编码

将指令及其操作数转换为机器可以理解的二进制位序列。例如，一条32位指令需要指定每个操作数在哪些位上编码。

6.汇编语法

人类可读的指令文本表示形式。例如：addr0,r1,r2

7.关键TableGen字段

-AsmString：指令的汇编语法字符串格式（如"$dst0,$dst1=myinstr$src0,$src1"）
-AsmName：寄存器的汇编名称
-HwEncoding：寄存器在硬件中的编码值
-Inst：指令的编码位模式

8.MCInstPrinter

负责将MCInst实例转换为人类可读的汇编字符串。对应图中的"Print"箭头。

9.MCTargetAsmParser

负责将人类可读的汇编字符串解析为MCInst实例。对应图中的"Parse"箭头。

10.MCCodeEmitter

负责将MCInst实例转换为二进制代码（字节序列）。对应图中的"Assemble"箭头。

11.伪指令

在Machine层使用的虚拟指令，不会到达MC层。例如PHI和COPY指令。它们通常设置IsPseudo或IsCodeGenOnly属性。

12.字节序

-小端序：最低有效字节存储在最低地址
-大端序：最高有效字节存储在最低地址

13.MCFixup

表示汇编代码中需要在链接时或后续阶段修复的位置（如符号地址的占位符）。

相关背景知识补充

LLVM编译流程中的MC层位置

LLVMIR→MachineIR→MC层→汇编文件(.s)/对象文件(.o)

为什么需要MC层？

1.独立测试：即使没有完整的编译器支持，用户也可以通过汇编代码直接使用目标架构
2.工具支持：提供汇编器、反汇编器等底层工具
3.编码验证：可以早期测试指令编码的正确性

指令编码的注意事项

-位序问题：Inst{31-22}表示从第31位到第22位（高位到低位），而Inst{22-31}含义不同
-变量连接：编码中使用的变量名（如dst0,src0）必须与OutOperandList和InOperandList中的操作数变量名一致

TableGen后端工具

本章使用了三个TableGen后端来生成MC层代码：
1.生成MCInstPrinter相关代码
2.生成MCCodeEmitter相关代码
3.生成MCTargetAsmParser相关代码

实践意义

通过这一章的学习，你可以：
-在不需要完整编译器支持的情况下，为目标架构启用基本的汇编/反汇编工具
-提前验证指令编码的正确性
-使用llvm-mc工具测试汇编语法和生成编码

这一章是构建完整LLVM后端的重要基础，MC层是连接高级中间表示与最终机器代码的桥梁
