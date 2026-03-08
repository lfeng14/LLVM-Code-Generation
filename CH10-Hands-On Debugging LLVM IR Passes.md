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
#### further
- https://llvm.org/docs/OptBisect.html
- https://llvm.org/docs/CommandGuide/llvm-reduce.htm
- https://clang.llvm.org/docs/AddressSanitizer.html
