- Before we get to the instruction selection phase, we need to address the **RegBankSelect pass**, which is a mandatory phase for GlobalISel between the legalization and the instruction selection phase.
- 什么是“寄存器库”（Register Bank）？在现代 CPU 中，寄存器并不是一块铁板，而是分成不同的“零件库”：GPR (通用寄存器库)：专门处理整数运算、地址计算（比如循环变量 i）。FPR/VR (浮点/向量寄存器库)：专门处理大规模科学计算、多媒体数据（比如你之前提到的 AlphaFold3 里的向量运算）。问题在于： 这两个库在物理上往往是分开的。如果你想把一个在 GPR 里的整数传给浮点运算单元，CPU 必须执行一条专门的“搬运指令”（Cross-register bank copy）
  ```
  /**
   * 根据比较谓词、目标类型和 NaN 行为要求，获取对应的浮点 Min/Max 通用 Opcode
   */
  unsigned CombinerHelper::getFPMinMaxOpcForSelect(
      CmpInst::Predicate Pred, LLT DstTy,
      SelectPatternNaNBehaviour VsNaNRetVal) const {
    
    // 健壮性检查：必须明确定义 NaN 的预期行为（是返回数字还是返回 NaN）
    assert(VsNaNRetVal != SelectPatternNaNBehaviour::NOT_APPLICABLE &&
           "Expected a NaN behaviour?");
  
    // 根据比较谓词（大于、小于等）进行分支处理
    switch (Pred) {
    default:
      return 0; // 不匹配任何 Min/Max 模式
  
    // --- 场景一：匹配 MAX (最大值) ---
    case CmpInst::FCMP_UGT: // Unordered Greater Than
    case CmpInst::FCMP_UGE: // Unordered Greater Equal
    case CmpInst::FCMP_OGT: // Ordered Greater Than
    case CmpInst::FCMP_OGE: // Ordered Greater Equal
      
      // 1. 如果明确要求“忽略 NaN，返回另一个数字”，对应 FMAXNUM (IEEE 754-2008)
      if (VsNaNRetVal == SelectPatternNaNBehaviour::RETURNS_OTHER)
        return TargetOpcode::G_FMAXNUM;
        
      // 2. 如果明确要求“传播 NaN，只要有 NaN 就返回 NaN”，对应 FMAXIMUM (IEEE 754-2019)
      if (VsNaNRetVal == SelectPatternNaNBehaviour::RETURNS_NAN)
        return TargetOpcode::G_FMAXIMUM;
  
      // 3. 如果没有明确要求，则看硬件（后端）支持哪种：
      // 优先尝试合法的 FMAXNUM
      if (isLegal({TargetOpcode::G_FMAXNUM, {DstTy}}))
        return TargetOpcode::G_FMAXNUM;
      // 其次尝试合法的 FMAXIMUM
      if (isLegal({TargetOpcode::G_FMAXIMUM, {DstTy}}))
        return TargetOpcode::G_FMAXIMUM;
        
      return 0;
  
    // --- 场景二：匹配 MIN (最小值) ---
    case CmpInst::FCMP_ULT: // Unordered Less Than
    case CmpInst::FCMP_ULE:
    case CmpInst::FCMP_OLT:
    case CmpInst::FCMP_OLE:
      
      // 1. 优先满足明确的 NaN 行为要求
      if (VsNaNRetVal == SelectPatternNaNBehaviour::RETURNS_OTHER)
        return TargetOpcode::G_FMINNUM;
      if (VsNaNRetVal == SelectPatternNaNBehaviour::RETURNS_NAN)
        return TargetOpcode::G_FMINIMUM;
  
      // 2. 启发式回退：看后端对该类型的支持情况
      if (isLegal({TargetOpcode::G_FMINNUM, {DstTy}}))
        return TargetOpcode::G_FMINNUM;
        
      // 如果 FMINIMUM 不合法，返回 0 表示不能合并
      if (!isLegal({TargetOpcode::G_FMINIMUM, {DstTy}}))
        return 0;
        
      return TargetOpcode::G_FMINIMUM;
    }
  }
  ```
- 函数的功能就是**根据寄存器类（`TargetRegisterClass`）返回对应的寄存器组（`RegisterBank`）**，用于 GlobalISel 指令选择时约束操作数可用的寄存器组。在你给出的例子中，H2BLB 后端将所有的通用寄存器类（`GPR16`, `GPR32`, `GPR16sp`, `OnlySP`）都映射到同一个寄存器组 `GPRBRegBankID`，表明这些寄存器都属于同一个通用寄存器组。这确保了指令选择阶段，这些寄存器类的操作数都会被分配到该寄存器组中的物理寄存器。
  ```
  const RegisterBank &
  H2BLBRegisterBankInfo::getRegBankFromRegClass(const TargetRegisterClass &RC,
                                                LLT Ty) const {
    switch (RC.getID()) {
    default:
      llvm_unreachable("Register class not supported");
    case H2BLB::GPR16RegClassID:
    case H2BLB::GPR32RegClassID:
    case H2BLB::GPR16spRegClassID:
    case H2BLB::OnlySPRegClassID:
      return getRegBank(H2BLB::GPRBRegBankID);
    }
  }
  ```
- global isel
  ```
  // 全局指令选择 (GlobalISel) 的核心选择函数
  // 作用：把 Generic 指令 转换成 目标架构真正的机器指令
  bool H2BLBInstructionSelector::select(MachineInstr &I) {
    // 获取当前指令的操作码
    unsigned Opc = I.getOpcode();
  
    // 如果这条指令 已经是 目标架构指令
    // 并且不是 PHI / COPY 这种通用指令 → 不需要处理，直接返回 true
    if (!isPreISelGenericOpcode(Opc) && Opc != TargetOpcode::PHI &&
        Opc != TargetOpcode::COPY)
      return true;
  
    ...
  
    switch (Opc) {
    // 处理 全局指令选择里的 G_PHI（伪指令）
    case TargetOpcode::G_PHI:
      // 把 G_PHI 替换成 目标架构真正的 PHI 指令
      I.setDesc(TII.get(TargetOpcode::PHI));
      [[fallthrough]];  // 直接穿透到下面的 PHI 处理逻辑
  
    // 统一处理 PHI / COPY 指令
    case TargetOpcode::PHI:
    case TargetOpcode::COPY:
      // 遍历指令的所有操作数（都是寄存器）
      for (MachineOperand &MO : I.operands()) {
        Register Reg = MO.getReg();
  
        // 如果是物理寄存器，不需要改，跳过
        if (Reg.isPhysical())
          continue;
  
        // 如果寄存器已经分配了寄存器类，跳过
        const TargetRegisterClass *RC = MRI.getRegClassOrNull(Reg);
        if (RC)
          continue;
  
        // 核心：给虚拟寄存器分配正确的寄存器类
        // 16位 → GPR16spRegClass
        // 32位 → GPR32RegClass
        unsigned Size = MRI.getType(Reg).getSizeInBits();
        MRI.setRegClass(Reg, Size == 16 ? &H2BLB::GPR16spRegClass
                                        : &H2BLB::GPR32RegClass);
      }
  ```
