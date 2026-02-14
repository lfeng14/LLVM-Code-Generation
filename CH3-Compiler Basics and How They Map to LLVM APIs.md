#### compiler jargon
- Target（目标）
   - 名词：程序运行的硬件架构。
   - 动词：为特定架构实现/定制编译器功能（如指令选择适配）。
   ```
     -X archs Build only the specified semi-colon-delimited list of backends(default: "ARM;AArch64;X86").
     -x arch  Build cross-compiling toolchain for the specified target.
   ```
- Host（宿主）
   - 运行编译器的设备。
   - 与Target不同时为交叉编译（如x86电脑编译AArch64手机代码）。
- Lowering（降级/底层化）
   - 编译过程中逐步降低程序抽象层级，从高级表示逼近目标机器汇编。
- Canonical form（规范形式）
   - LLVM中统一、推荐的表示方式，避免等价表达形式爆炸。
   - 不遵循可能影响代码生成质量，LLVM提供API自动规范化。
- build time：构建时间，比如ninja这种工具运行时间
- compile time：编译单个文件的时间
- run time：运行时间
- The target-agnostic part, which happens roughly before the target-specific part：目标无关 目标相关
- ABI:
  ```
   ABIs are extremely important as they provide the glue between functions. In other words, they formalize
   how the handshake happens between the caller and the callee. For instance, the ABI prescribes where
   the input arguments should be set by the caller and, thus, where the callee can expect to find its input
   arguments.
   Then, as long as compilers follow the rules of the ABI, a function compiled with a compiler can talk
   to a function compiled with a different compiler.
  ```
