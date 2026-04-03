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

- CCValAssign: Therefore, the original type is i8 but the actual type is i32. You can get this information through the getValVT method for the original type and getLocVT for the actual type
- 使用SDISel降低ABI，需实现**目标专用TargetLowering类**的4个核心方法。
  - 必须实现的方法：**LowerFormalArguments、LowerReturn、LowerCall、CanLowerReturn**。
  - 假设代码如下：
  ```c
  int add(int a, int b) { // <--- Callee 端
      return a + b;
  }
  
  int main() {
      int x = add(10, 20); // <--- Caller 端
      return x;
  }
  ```
  
  - AnalyzeFormalArguments (在 `add` 函数内部使用)
    - 当编译器处理 `add` 函数定义时，它需要知道 `a` 和 `b` 在哪。
    - **作用：** 它分析得到 `a` 应该在 `R0`，`b` 应该在 `R1`。
    - **生成的代码：** 编译器会生成指令把 `R0` 和 `R1` 的值拷贝到函数内部的虚拟寄存器中，供后续加法使用。
  
  - AnalyzeReturn (在 `add` 函数内部使用)
    - 当处理 `return a + b` 时。
    - **作用：** 它分析得到返回值必须放在 `R0`。
    - **生成的代码：** 将加法的结果拷贝到物理寄存器 `R0` 中，然后执行 `ret` 指令。
  
  - AnalyzeCallOperands (在 `main` 函数内部使用)
    - 当处理 `add(10, 20)` 这一行，跳转到 `add` 之前。
    - **作用：** 它分析得到第一个常数 `10` 要放进 `R0`，`20` 要放进 `R1`。
    - **生成的代码：** `MOV R0, #10`, `MOV R1, #20`。
  
  - AnalyzeCallResult (在 `main` 函数内部使用)
    - 当 `add` 函数执行完并返回到 `main` 之后。
    - **作用：** 它分析得到返回值现在就在 `R0` 里。
    - **生成的代码：** 将 `R0` 的值拷贝回 `main` 函数的一个虚拟寄存器（对应变量 `x`）。
  
  - 为什么需要 CCState 和 CCValAssign？这四个方法本质上都是在**填表**。
    - **CCState：** 是一个“账本”，记录了当前寄存器分配的情况（例如：`R0` 是否已被占用？栈已经用了多少字节？）。
    - **AnalyzeXXX：** 是一套“算法”，它会查阅你在 TableGen (`.td`) 中写的 `CCAssignFn`（分配逻辑），然后针对每一个参数算出它的位置。
    - **CCValAssign：** 是“结果单”，每一个参数分析完后都会生成一个 `CCValAssign` 对象。它会告诉你：这个参数是 `isRegLoc`（在寄存器里）还是 `isMemLoc`（在内存里），以及具体的编号或偏移。

- LowerFormalArguments代码注释
  ```
  SDValue H2BLBTargetLowering::LowerFormalArguments(
      SDValue Chain, CallingConv::ID CallConv, bool IsVarArg,
      const SmallVectorImpl<ISD::InputArg> &Ins, const SDLoc &DL,
      SelectionDAG &DAG, SmallVectorImpl<SDValue> &InVals) const {
  
    // --- 第一阶段：准备工作与 ABI 分析 ---
    
    MachineFunction &MF = DAG.getMachineFunction();
    MachineRegisterInfo &RegInfo = MF.getRegInfo();
    
    // ArgLocs 用于存储分析结果：即每个参数最终被分配到了哪个“位置”（寄存器或栈偏移）
    SmallVector<CCValAssign, 16> ArgLocs;
    
    // CCState 是一个状态机，负责根据 H2BLB 的调用约定 (Calling Convention) 来分配资源
    CCState CCInfo(CallConv, IsVarArg, MF, ArgLocs, *DAG.getContext());
    
    // 核心调用：查阅 TableGen 定义的 CC_H2BLB_Common 规则，填充 ArgLocs
    CCInfo.AnalyzeFormalArguments(Ins, CC_H2BLB_Common);
  
    // --- 第二阶段：遍历分析结果并生成代码 ---
  
    for (size_t I = 0; I < ArgLocs.size(); ++I) {
      auto &VA = ArgLocs[I]; // 获取当前参数的分配位置信息
      
      // 情况 A：参数分配到了寄存器
      if (VA.isRegLoc()) {
        // LLVM 目前只支持完整类型的传递，暂不支持半个寄存器传递（需额外处理）
        if (VA.getLocInfo() != CCValAssign::Full)
          report_fatal_error("non-full passing, not yet implemented");
  
        // 获取该位置的类型（如 i32, i16 等）
        EVT RegVT = VA.getLocVT();
        const TargetRegisterClass *DstRC = nullptr;
  
        // 根据位宽选择正确的寄存器类（如 GPR32 或 GPR16）
        // 这里通常会有一个 switch(RegVT.getSizeInBits()) ...
        ... 
  
        // 1. 创建一个虚拟寄存器 (VReg)，用于在函数内部代表这个参数
        Register VReg = RegInfo.createVirtualRegister(DstRC);
        
        // 2. 告诉寄存器分配器：在函数入口处，物理寄存器 (VA.getLocReg()) 的值是有效的（Live-In）
        // 并将其与我们刚创建的虚拟寄存器 VReg 关联起来
        RegInfo.addLiveIn(VA.getLocReg(), VReg);
  
        // 3. 在 DAG 图中创建一个“拷贝”节点：从虚拟寄存器读取值
        // 这个 ArgValue 就是后续 IR 逻辑真正使用的“变量”节点
        SDValue ArgValue = DAG.getCopyFromReg(Chain, DL, VReg, RegVT);
  
        // 4. 将处理好的参数节点放入结果数组中，返回给 LLVM 核心
        InVals.push_back(ArgValue);
  
      } else {
        // 情况 B：参数分配到了栈内存 (Stack)
        // 这部分通常涉及从栈帧 (FrameIndex) 生成 Load 指令
        ... 
      }
    }
  
    // 返回当前的链 (Chain)，以确保指令执行的顺序性
    return Chain;
  }
  ```

- LowerFormalArguments AnalyzeFormalArguments 这两个函数有什么区别
  - LowerFormalArguments：执行者 (The Doer)
    - 它是 `TargetLowering` 类中的一个 **虚函数**，你需要为你的特定硬件（如 NPU）重写它。
    - **它的任务：** 真正地修改 **SelectionDAG**（选择图）。它负责生成 `CopyFromReg`（从寄存器拷贝）或者 `Load`（从栈加载）这些 DAG 节点。
    - **输入：** 抽象的 `ISD::InputArg` 列表。
    - **输出：** 返回一个新的 `Chain`（链），并填充 `InVals` 向量（告诉 LLVM 每个参数对应的 DAG 节点是谁）。
    - **本质：** 它是后端降低（Lowering）流程的一部分，处理的是 **“如何把值拿进函数”**。

  - AnalyzeFormalArguments：决策者 (The Planner)
    - 它是 `CCState` 类的一个 **成员方法**，通常由 LLVM 核心库提供，你直接调用它。
    - **它的任务：** 纯粹的 **逻辑计算**。它查阅你用 TableGen 定义的调用约定（ABI 规则），计算出：“第 1 个参数应该在 R0，第 2 个在 R1，第 3 个在栈偏移 8 的位置”。
    - **输入：** `ISD::InputArg` 列表和你定义的分配函数（如 `CC_H2BLB_Common`）。
    - **输出：** 它不生成任何指令，它只是往 `CCState` 里的 `ArgLocs` 列表里填入一堆 `CCValAssign` 对象。
    - **本质：** 它是 ABI 规则的 **翻译官**，只负责计算位置，不负责搬运数据。

  - 协作流程图，你可以把这个过程想象成物流公司：
    - **LowerFormalArguments** 开始工作。
    - 它先创建一个“记账本” `CCState`。
    - 它调用 **AnalyzeFormalArguments**。
    - **AnalyzeFormalArguments** 查阅“规章制度”（ABI），在记账本上写下：“参数 A 在 R0，参数 B 在栈”。
    - **LowerFormalArguments** 拿回记账本，看到“参数 A 在 R0”，于是动手写下一行代码（生成 `CopyFromReg` 节点）；看到“参数 B 在栈”，于是又写下一行代码（生成 `Load` 节点）。


  - 核心对比表
    | 特性 | LowerFormalArguments | AnalyzeFormalArguments |
    | :--- | :--- | :--- |
    | **所属类** | `TargetLowering` (需子类实现) | `CCState` (直接调用) |
    | **主要工作** | 创建 DAG 节点 (`SDValue`) | 计算分配位置 (`CCValAssign`) |
    | **是否修改 DAG** | **是** | **否** (只读/计算) |
    | **逻辑来源** | 手写 C++ 代码 | 调用 TableGen 生成的函数 |
    | **调用时机** | SelectionDAG 建立初期 | 在 `LowerFormalArguments` 内部 |

---

- 为什么要分开？
  - 这种设计是为了 **解耦**。
  - **统一逻辑：** 不同的硬件后端（X86, ARM, NPU）在“如何搬运数据”（Lowering）上可能有很大差异，但“如何计算位置”（Analyze）的逻辑通常是由一套通用的 TableGen 规则定义的。
  - **代码复用：** 比如 `AnalyzeCallOperands` 和 `AnalyzeFormalArguments` 可能会共享同一套调用约定规则（`CCAssignFn`），这样就能保证调用者（Caller）放参数的位置和被调用者（Callee）取参数的位置完全一致。

- **总结：**
  - 如果你想改变 **参数放在哪个寄存器**，去改 `.td` 文件里的调用约定；
  - 如果你想改变 **如何从特殊内存里读取参数**，去改 `LowerFormalArguments` 里的 `Load` 指令生成逻辑。
  - 你是在为你自研的 NPU 定义新的寄存器类吗？如果是的话，你可能需要先修改 TableGen 文件，然后再去 `LowerFormalArguments` 里处理这些新寄存器的拷贝逻辑。

#### further reading
- aarch64调用约定：https://github.com/jeandle/jeandle-llvm/blob/main/llvm/lib/Target/AArch64/AArch64CallingConvention.td
- aapcs64调用约定：https://github.com/ARM-software/abi-aa/releases/download/2024Q3/aapcs64.pdf
- LowerFormalArguments：https://github.com/llvm/llvm-project/blob/97dbf38c9c495ce9fb958137957cb7794ef3285b/llvm/lib/Target/AArch64/AArch64ISelLowering.cpp#L8818
- 像 Linux 内核的 Documentation 目录里，从架构细节到驱动开发都有详细文档，还有 Git 的官方文档，从基础命令到内部实现原理都讲得很透。另外，PostgreSQL 的文档也很全面，从 SQL 语法到数据库内核架构都有覆盖。你平时更关注编译器相关的项目，还是其他领域的？那可以试试看看 Redis 的文档，它把数据结构、持久化机制这些核心原理讲得很通俗，还有 WebKit 的文档，对浏览器渲染引擎的细节解释得很到位，这两个项目的文档和 LLVM 一样，都属于 “能让自学党啃明白” 的类型。
