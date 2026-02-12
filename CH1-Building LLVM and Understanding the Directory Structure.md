#### 章节目的
<img width="640" height="2132" alt="image" src="https://github.com/user-attachments/assets/c175d1a0-c3b1-4c18-a227-3f804627eb75" />

- Understand the different components that make a compiler
- Build and test the LLVM project
- Navigate LLVM’s directory structure and locate the implementation of different components
- Contribute to the LLVM project
<img width="1086" height="566" alt="image" src="https://github.com/user-attachments/assets/a649e34d-0a34-44d3-af78-1b43b81237df" />

- clang definition:  Clang, is a compiler driver(include frontend and ): it invokes the different tools in the right order and pulls the related dependencies from the standard library to produce the final executable code.
<img width="678" height="782" alt="image" src="https://github.com/user-attachments/assets/749da53d-b1a8-45c0-9861-61faa5e9d5b1" />

- In any case, the focus of this book is LLVM backends, so, why are we spending so much time on Clang?
  The reason is simple: Clang offers a familiar way to interact with LLVM constructs. By using the Clang frontend, you will be able to generate the LLVM intermediate representation (IR) by simply writing C/C++. We believe this is a gentler way to start your journey with LLVM backends.
  ```
  1. Frontend: This validates that the input file is syntactically and semantically correct and
  produces the LLVM IR.
  •Preprocessor: This expands macros (e.g., #include).
  •Sema: This validates the syntax and semantics of the program.
  •Codegen: This produces the LLVM IR.
  2. Backend: This translates the LLVM IR to target specific instructions.
  •Middle-end optimizations: LLVM IR to LLVM IR optimizations.
  •Assembly generation: Target-specific IR to assembly code.
  3. Assembler: This translates assembly code to an object file.
  ```
- Here are the options to inspect their results(逐步深入不同阶段):
  <img width="1200" height="628" alt="image" src="https://github.com/user-attachments/assets/01aabc2b-ea8f-4587-a61a-10fcd288a94d" />

- Building LLVM：如何节省时间
  ```
  cmake -LH ./llvm
  # 查看配置选项
  vi CMakeCache.txt 
  cmake -GNinja -DCMAKE_BUILD_TYPE=Debug -DLLVM_TARGETS_TO_BUILD="X86;AArch64" -DLLVM_OPTIMIZED_TABLEGEN=1 ${LLVM_SRC}/llvm
  # 减少构建时间，指定构建target
  ninja help
  ninja [options] [buildTarget1] [buildTarget2] ...
  ```
- 使能ccache，下回尝试速度有没提升
  ```
  if [ $use_ccache == "1" ]; then
    echo "Build using ccache"
    CMAKE_OPTIONS="$CMAKE_OPTIONS \
                  -DCMAKE_C_COMPILER_LAUNCHER=ccache \
                  -DCMAKE_CXX_COMPILER_LAUNCHER=ccache "
  fi
  ```
- 主要产物:
  ```
  •opt: Build the driver for the LLVM IR to LLVM IR optimizations.
  •llc: Build the driver for the LLVM IR to assembly/object file pipeline.
  •llvm-mc: Build the tool to play with the assembling/disassembling mechanism.
  •check: Run all the core LLVM unit tests. This target will rebuild automatically all the tools that
  are involved in the unit tests, including the aforementioned targets
  ```

- test llvm
  - LLVM uses the Google test infrastructure (gtest) for some of its unit tests.
  - The LLVM Integrated Tester (lit) drives a lot of the LLVM testing infrastructure.
  ```
  In a nutshell, this tester does the following:
  •Discovers the tests to be run
  •Runs them concurrently
  •Prints a summary of failure/success at the end
  ```

- lit 针对不同后缀找过滤指令，比如.ll .mlir:
  ```
  Using these directives, lit determines the following:
  1. Requirements: Does this test need to be run?
  2. Command: How is this test run? 如：
     ; RUN: echo %s %t
  3. Status: At the end of the run, is this result a pass or a failure?
  ```
  <img width="1200" height="600" alt="image" src="https://github.com/user-attachments/assets/4f4c99a6-a40a-4760-b26d-426fda338a62" />
  <img width="1210" height="278" alt="image" src="https://github.com/user-attachments/assets/ae51133f-22f3-4623-9a2a-c4a539a13b5e" />

- Lit Command里的符号替换:
  <img width="866" height="250" alt="image" src="https://github.com/user-attachments/assets/44936571-295b-482b-b0a0-1a73ee39d7e8" />
  ```
  ./bin/llvm-lit test/CodeGen/AArch64/GlobalISel/

  -s Silent: Only print a progress bar and the final report
  -v Verbose: Print the RUN lines and the output of a test on failure
  -a Print all: Same as verbose but for all tests, not just the failing ones
  ```
  
- Command示例：
  ```
  FileCheck --input-file input.txt check-file.txt
  
  input.txt 
  I feel
  great
  today
  How about you?
  This line doesn’t matter
  as well as this one
  I don’t know
  Meh
  The end
  or is it?
  
  check-file.txt
  CHECK: I
  CHECK-SAME: feel
  CHECK: great
  CHECK-NEXT: today
  CHECK: How about you?
  CHECK-DAG: Meh
  CHECK-DAG: I don
  CHECK-NOT: or is it
  CHECK: The end
  CHECK: or is it
  ```
  <img width="882" height="552" alt="image" src="https://github.com/user-attachments/assets/3bf7b80b-951e-4540-b242-0ebe859d50d8" />
  <img width="1210" height="926" alt="image" src="https://github.com/user-attachments/assets/4bc6af7f-0288-4999-8a2d-b0417d28c7b5" />
