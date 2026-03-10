- print-before-all/print-after-all: Print the IR before (or after, respectively) each pass in the current pipeline.
- print-before/print-after=PassName1[,PassName2]*: Print the IR before (or after, respectively) PassName1, PassName2, and so on, where PassNameI is the CLI option name of the related pass (see Chapter 5 to find out how to find this name). Like all the CLI options that accept comma-separated lists in the LLVM CLI, you can specify this option several times. This is equivalent to providing the concatenated comma-separated list to one instance of the option
- to start producing debug logs in your pass, you must do the following:
  - 1. Include the Debug.h, from the Support library, in your pass.
  - 2. Define the DEBUG_TYPE macro. We recommend sticking to the same name as the one you used for the CLI option name unless it makes sense to group your debug log with other passes.
  - 3. Use the LLVM_DEBUG macro to start performing debug-log-specific tasks such as printing some message or the content of a variable

- The capabilities of llvm-extract are built around the ExtractGVPass and BlockExtractorPass passes from the IPO library. Generally speaking, if you want to extract a region from some IR, this functionality is available in the TransformsUtils library under the CodeExtractor class. This class can be used in the PartialInliner or the LoopExtractor passes.For instance, let’s assume we want to extract the foo and bar functions from some input IR. The following command does this for us:
  ```
  llvm-extract -S -func=bar -func=foo -o - input.ll
  ```
- To extract bb2 from bar, you can run the following command:
  ```
  $ llvm-extract -S -o - input.ll -bb=bar:bb2
  define dso_local void @bar.bb2(i32 %i, ptr %i3.out) {
  <snip>
  bb2:
  %i3 = sdiv i32 3, %i
  store i32 %i3, ptr %i3.out, align 4
  br label %bb6.exitStub
  <snip>
  ```
   The boilerplate code is here to make sure that all the values that are used but not defined in that block are available by adding them as input arguments.
   The capabilities of llvm-extract are built around the ExtractGVPass and BlockExtractorPass passes from the IPO library. Generally speaking, if you want to extract a region from some IR, this functionality is available in the TransformsUtils library under the CodeExtractor class. This class can be used in the PartialInliner or the LoopExtractor passes.
- If the has_bfi.sh script produces a bfi instruction when run on the input.ll file, llvm-reduce will reduce the IR from input.ll such that the final IR (saved in reduced.ll) will still produce this instruction (or more precisely will have has_bfi.sh return zero).
```
# 通过反复简化 aarch64-bit-gen.ll，同时确保简化后的文件仍能使 has_bfi.sh 返回成功（即仍能触发 BFI 相关行为），最终得到一个最小的、仍保留该特征的测试用例
llvm-reduce --test=has_bfi.sh llvm/test/CodeGen/AArch64/aarch64-bit-gen.ll

#!/bin/bash
llc $@ -o - | grep 'bfi'
exit $?
````
- 相同用法
```
bugpoint --compile-command=./has_not_bfi.sh --run-llc --compile-custom input.ll

#!/bin/bash
llc $@ -o - | grep 'bfi'
! exit $?
```
- To enable sanitizers, assuming you’re using Clang as the compiler toolchain, you only need to use -fsanitize=typeOfSanitizer, where typeOfSanitizer is one of the following:
  - address: Detects invalid address accesses – for instance, use-after-free, out-of-bounds access, and so on
  - memory: Detects access to valid addresses that haven’t been uninitialized yet
  - thread: Detects race conditions
  - undefined: Detects pieces of code that rely on undefined behavior
  - leak: Detects memory leaks
- optimization disabled (-O0) and the debug information enabled (-g). This is exactly what the Debug CMake build type is about。
- The main advantage of the debugger compared to the logging mechanism presented earlier is that you don’t have to re-compile your program each time you want to inspect something that hasn’t been logged yet.
  - Know where you are in the program
  - Stop at specific locations
  - Inspect the state of the program
  ```
  $ lldb
  (lldb) target create ./myExec
  (lldb) run arg0 arg1 arg2
  ```
  ```
  $ lldb -- ./myExec arg0 arg1 arg2
  (lldb) run
  ```
- There are mainly two ways to specify when to stop a program:
  - Use breakpoints to stop the program when you reach a specific location.
  - Use watchpoints to stop the program when some data is read and/or written
- To set a breakpoint, we can use the b command, followed by one of the given patterns:
  - filename:lineNumber: For instance, main.cpp:12 to stop at the 12th line in the main.cpp file
  - functionName: For instance, main to stop at the beginning of the main function (more precisely, after the main function’s prologue)
  - 0xRawAddress: For instance, 0xf00ba4 to stop at the given program counter, which is represented by a raw address, in hexadecimal format, within the program being debugged
- LLDB provides extensive documentation of its commands through the help command. Simply run help <command> (for example, help breakpoint) within lldb to get the help for the given family of commands, then help <command> <subcommand> (for example, help breakpoint set) for the help message of the specific subcommand. For instance, you can view the exhaustive list of the supported ways to set a breakpoint with the b command by using help b.
- lldb支持复杂断点表示，break modify
  ```
  (lldb) b BinaryOperator::BinaryOperator
  Breakpoint 8: 2 locations.
  (lldb) break modify 8 -c 'iType == llvm::Instruction::BinaryOps::SDiv && S2-
  >getValueID() == llvm::Value::ValueTy::ConstantIntVal && ((ConstantInt*)S2)-
  >getZExtValue() == 5'
  ```
- 这个命令是在 LLDB 调试器中设置一个写入监视点，用于监控指定变量 MyVar 在内存中的修改操作。当程序执行过程中对该变量进行写入（赋值）时，调试器会暂停程序，并提示监视点命中
  ```
  (lldb) watchpoint set variable -w write MyVar
  ```
- here are the main LLDB commands you can use to control the execution:
  - next (or n): Executes the next statement of the program.
  - step (or s): This is like the next command but if the statement is a function call, it stops at the first statement in the target function. In contrast, the next command simply executes the whole function call.
  - finish (or fin): This executes the program until it finishes executing the current frame. In other words, this command allows us to exit the current function call by executing whatever is left to be executed in this function.
  - thread until <lineNumber>: This executes the program until the specified lineNumber is reached. If the normal execution of the program doesn’t go through the specified line number, LLDB stops the program when returning from the current frame.
  - thread return <value>: This overrides the program’s execution by directly returning from the current frame with the provided value. This command illustrates how powerful debuggers are; you can alter the execution of the program however you want. Be aware that the state of the program may be incorrect since you may have skipped a lot of code that’s normally executed.
- 例子：For instance, consider the following snippet and assume that the next statement to be executed is the one with the --> symbol:
  ```
  int foo(int a, int b) {
  --> int c = bar(a);
  return c + b;
  }
  ```
  Now, let’s describe what would happen if we executed one of the following commands at this point:
  - next: The program will execute until the return c + b statement.
  - step: The program will execute until it reaches the first statement in bar.
  - finish: The program will execute until it exits foo – that is, it stops at the first statement after the call to foo in the caller function.
  - thread return 12: This bypasses the normal execution of the program and returns a value of 12 straight away. The program stops at the same point that the finish command would have yielded.

- 钩子函数
  ```
  (lldb) b populate_function.cpp:42
  Breakpoint 3: <snip>
  (lldb) break command add 3
  Enter your debugger command(s). Type 'DONE' to end.
  > p &Foo
  > DONE
  ```
- 一些技巧
  - print 不仅支持打印变量还支持表达式：p MyVar = fct(OtherVar) + 3
  - 使用 `print` 命令可随时执行程序中的任意函数，并可与断点结合使用。
  - 场景示例：需观察特定输入下的函数执行、未设断点却发现函数异常时，可先设断点，再用 `print` 调用该函数。
  - 默认情况下，`print` 表达式执行会忽略所有断点。
  - 可通过 LLDB 命令 `setting set target.process.ignore-breakpoints-in-expressions false` 关闭该默认行为。
- LLDB中一个实用命令：**apropos**。
- 使用方式：`apropos + 关键词`，可列出所有相关的 LLDB 命令。
- 示例：`apropos stack` 会显示与栈相关的命令，如打印回溯、跳出当前帧、修改栈帧显示格式等。
#### further
- https://llvm.org/docs/OptBisect.html
- https://llvm.org/docs/CommandGuide/llvm-reduce.htm
- https://clang.llvm.org/docs/AddressSanitizer.html
- https://llvm.org/docs/CommandGuide/llvm-reduce.html for the llvm-reduce CLI
- https://llvm.org/docs/CommandGuide/bugpoint.html for the bugpoint CLI and https://llvm.org/docs/Bugpoint.html for its fancier use cases
- https://llvm.org/docs/CommandGuide/llvm-extract.html for the llvm-extract CLI
- https://llvm.org/docs/OptBisect.html for an explanation of how to use the opt-bisect-limit command-line option
- For the sanitizers offered by Clang, please look at the related documentation pages:
  - The address sanitizer: https://clang.llvm.org/docs/AddressSanitizer.html
  - The thread sanitizer: https://clang.llvm.org/docs/ThreadSanitizer.html
  - The memory sanitizer: https://clang.llvm.org/docs/MemorySanitizer.html
  - The undefined behavior sanitizer: https://clang.llvm.org/docs/UndefinedBehaviorSanitizer.html
  - The dataflow sanitizer: https://clang.llvm.org/docs/DataFlowSanitizer.html
  - The leak sanitizer: https://clang.llvm.org/docs/LeakSanitizer.html
- https://lldb.llvm.org/use/tutorial.html
