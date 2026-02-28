- 章节目的
  - Building a compiler
    - What is a compiler?
    - Opening Clang's hood
    - Building Clang
    - Experimenting with Clang
    
  - Building LLVM
    - Configuring the build system
    - Crash course on Ninja
    - Building the core LLVM project
    
  - Testing a compiler
    - Crash course on the Google test infrastructure
    - Crash course on the LLVM Integrated Tester
    - Crash course on FileCheck
    - LLVM unit tests
    - LLVM functional tests
    
  - Understanding the directory structure
    - High-level directory structure
    - Focusing on the core LLVM project
    
  - A word on the include files
    - Private headers
    - What is the deal with <project>/include/<project>?
    - What is include/<project>-c?
    
  - Overview of some of the LLVM components
    - Generic LLVM goodness
    - Working with the LLVM IR
    - Generic backend infrastructure
    - Target-specific constructs

- Understand the different components that make a compiler
- Build and test the LLVM project
- Navigate LLVM’s directory structure and locate the implementation of different components
- Contribute to the LLVM project
<img width="500" height="250" alt="image" src="https://github.com/user-attachments/assets/a649e34d-0a34-44d3-af78-1b43b81237df" />

- clang definition:  Clang, is a compiler driver(include frontend and ): it invokes the different tools in the right order and pulls the related dependencies from the standard library to produce the final executable code.
<img width="340" height="390" alt="image" src="https://github.com/user-attachments/assets/749da53d-b1a8-45c0-9861-61faa5e9d5b1" />

- In any case, the focus of this book is LLVM backends, so, why are we spending so much time on Clang?
  The reason is simple: Clang offers a familiar way to interact with LLVM constructs. By using the Clang frontend, you will be able to generate the LLVM intermediate representation (IR) by simply writing C/C++. We believe this is a gentler way to start your journey with LLVM backends.
  ```
  1. Frontend: This validates that the input file is syntactically and semantically correct and
  produces the LLVM IR.
    • Preprocessor: This expands macros (e.g., #include).
    • Sema: This validates the syntax and semantics of the program.
    • Codegen: This produces the LLVM IR.
  2. Backend: This translates the LLVM IR to target specific instructions.
    • Middle-end optimizations: LLVM IR to LLVM IR optimizations.
    • Assembly generation: Target-specific IR to assembly code.
  3. Assembler: This translates assembly code to an object file.
  ```
- Here are the options to inspect their results(逐步深入不同阶段):
  <img width="600" height="300" alt="image" src="https://github.com/user-attachments/assets/01aabc2b-ea8f-4587-a61a-10fcd288a94d" />

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
    • opt: Build the driver for the LLVM IR to LLVM IR optimizations.
    • llc: Build the driver for the LLVM IR to assembly/object file pipeline.
    • llvm-mc: Build the tool to play with the assembling/disassembling mechanism.
    • check: Run all the core LLVM unit tests. This target will rebuild automatically all the tools that are involved in the unit tests, including the aforementioned targets
  ```

- test llvm
  - LLVM uses the Google test infrastructure (gtest) for some of its unit tests.
  - The LLVM Integrated Tester (lit) drives a lot of the LLVM testing infrastructure.
  
  ```
  There are two kinds of unit tests in LLVM that are logically separated into two different folders of your build directory
  • unittests: This contains the tests that are directly generated from the related directory of the LLVM source (${LLVM_SRC}/llvm/unittests). These tests are written using gtest.
  • test: This contains the output scripts used by lit. The actual tests live in the related directory of the LLVM source (${LLVM_SRC}/llvm/test). 比如 make check clang
  In a nutshell, this tester does the following:
    • Discovers the tests to be run
    • Runs them concurrently
    • Prints a summary of failure/success at the end
  ```

- lit 针对不同后缀找过滤指令，比如.ll .mlir:
  
  <img width="370" height="270" alt="image" src="https://github.com/user-attachments/assets/8ea31ecb-0123-44b8-a1dd-3e9c7aa21f42" />
  
  ```
  Using these directives, lit determines the following:
  1. Requirements: Does this test need to be run?
  2. Command: How is this test run? 如： ; RUN: echo %s %t
  3. Status: At the end of the run, is this result a pass or a failure?
  ```
  <img width="600" height="300" alt="image" src="https://github.com/user-attachments/assets/4f4c99a6-a40a-4760-b26d-426fda338a62" />
  <img width="600" height="140" alt="image" src="https://github.com/user-attachments/assets/ae51133f-22f3-4623-9a2a-c4a539a13b5e" />

- Lit Command里的符号替换:
  
  <img width="430" height="125" alt="image" src="https://github.com/user-attachments/assets/44936571-295b-482b-b0a0-1a73ee39d7e8" />
  
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
  
  <img width="400" height="250" alt="image" src="https://github.com/user-attachments/assets/3bf7b80b-951e-4540-b242-0ebe859d50d8" />
  <br>
  <img width="800" height="400" alt="image" src="https://github.com/user-attachments/assets/0e6bbc70-93d2-4e82-bb1f-f429c9fc35f0" />
   
- 有check-label后，划分匹配区域，label后面的check仅匹配当前块
    ```
    define %struct.C* @C_ctor_base(...) {
    entry:
    ; CHECK-LABEL: C_ctor_base:
    ; CHECK: mov [[SAVETHIS:r[0-9]+]], r0
    ; CHECK: bl A_ctor_base
    ; CHECK: mov r0, [[SAVETHIS]]
      ...
    }
    
    define %struct.D* @D_ctor_base(...) {
    entry:
    ; CHECK-LABEL: D_ctor_base:
    ```
- FileCheck支持选项
  
  <img width="600" height="460" alt="image" src="https://github.com/user-attachments/assets/4bc6af7f-0288-4999-8a2d-b0417d28c7b5" />

- CMAKE使用再详细介绍
- 功能行测试失败：1、编译失败 2、编译没问题二进制有功能问题
  ```
  If you are lucky, the bug will fail the
  compilation. For instance, it will crash the compiler, and in this case, you are back to debugging a
  regular C++ application except the application is your compiler. If you are less fortunate, you may
  have to deal with a miscompile: the compilation process worked fine, but the application fails its
  correctness test.
  ```

- 调试编译器误编译问题步骤为：
  1. 缩小差异范围  
     - 明确一个能工作与一个不能工作的编译器版本（或选项组合），差异越小越易定位。  
     - 理想情况：通过独立开关精准控制可疑改动。
  2. 锁定目标文件
     - 用 MD5 对比编译产物，但需排除因路径、时间戳等引起的非逻辑差异（如调试信息）。  
     - 通过自定义链接或修改构建系统，**二分筛选**出首个产生差异的目标文件。
  3. 细粒度隔离与复现
     - 若已知罪魁改动，可增加环境变量/选项控制其应用次数，并单线程确定性构建，进行二次二分。  
     - 避免Ninja等多线程环境带来的不确定性。
  4. 借助工具与规范 
     - 使用 sanitizers（如 UB、Address）辅助诊断；  
     - 警惕应用程序自身缺陷——你的改动可能仅是触发了已有问题；  
     - 语义分歧时以语言标准为最终依据。
  至此，你已掌握 LLVM 的构建、测试流程，熟悉 gtest、lit、FileCheck 等工具，能独立运行单元测试并复现问题，并对深入调试误编译有了清晰的方法论储备。

- LLVM目录结构独立，比如llvm clang lldb lld，所以多不要紧找一个想看的目录看下去，看测试用例
- llvm c c++ api
  ```
  build/bin/llvm-config --includedir
  drwxr-xr-x 54 franz staff          4096 Jan  6 03:25 llvm/
  drwxr-xr-x  3 franz staff          4096 Jan  6 03:25 llvm-c/
  ```
#### 思考
- 假如我要写openjdk java用例，你建议使用llvm-lit + FileCheck这一套来处理吗 ？
  - 我理解了，函数单元测试建议用jtreg+断言，而llvm的lit测试用来黑盒测试，匹配二进制执行输出，两者功能有差异
    ```
    JTreg + 断言（或 JUnit） 是白盒测试，直接验证程序内部状态、返回值、异常等运行时行为，测的是“代码逻辑对不对”。
    lit + FileCheck 是黑盒测试，通过命令行驱动程序，只验证输出文本是否符合预期模式，测的是“工具链输出对不对”。
    两者功能互补、场景隔离，不存在谁替代谁。在 OpenJDK 开发中，JTreg 是官方主力；而在 LLVM 生态中，lit+FileCheck 是标准配置。把正确的工具用在正确的地方，这正是工程经验的体现。
  
    另外补充一个小点：lit+FileCheck 不仅限于测试二进制执行输出，它可以测试任何命令行工具生成的标准输出/错误，包括 javac --help、clang -v、opt -passes 等纯文本输出。它本质上是一个文本契约测试工具。
    ```
- 作者想表达：单元测试只是编译器验证的第一道防线，通过了不代表编译器真的能生成正确的可执行代码；你必须学会如何运行生成的程序，并验证它的输出是否符合预期，这才是真正的功能测试。
#### 更多详细文档
- 本书配套仓库 https://github.com/PacktPublishing/LLVM-Code-Generation/tree/main/ch1/FileCheckExamples/ex3
- LLVM FileCheck https://llvm.org/docs/CommandGuide/FileCheck.html
- LLVM BUILDING https://llvm.org/docs/CMake.html
- CMAKE 11 CHS https://cmake.org/cmake/help/latest/guide/tutorial/index.html
- llvm-lit https://llvm.org/docs/TestSuiteGuide.html
