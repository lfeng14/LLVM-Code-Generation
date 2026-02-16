- value:
- SSA: is a way to represent values directly in an IR. The idea is straightforward: rename all variables such that each variable holds exactly one value statically. In the LLVM IR and Machine IR levels, phi instructions are grouped together at the beginning of the basic blocks. It is invalid to insert non-phi instructions before phi instructions.

- 节点A 支配 节点B：从程序入口到B的所有路径都必须经过A。这意味着在A处执行的代码，在到达B之前一定会被执行。
- 节点A 后支配 节点B：从B到程序出口的所有路径都必须经过A。这意味着在B之后，所有可能的执行路径最终都会到达A。
- 这里%add有两个使用者：%mul和%sub。如果遍历%add->users()，可能先得到%mul再得到%sub，也可能反过来
  ```
  %add = add i32 %a, %b
  %mul = mul i32 %add, 2
  %sub = sub i32 %add, 1
  ```
- legality: al? In other words, does this optimization preserve the semantics of the program
- 在数学上，加法满足结合律 (3.0 + a) + 3.0 = a + (3.0 + 6.0)。但在计算机的浮点数运算中，情况完全不同：运算顺序影响结果：浮点数的表示是有限的（如IEEE 754标准），很多小数无法精确表示。中间结果会经历舍入。表达式 (3.0 + a) + 3.0 的运算顺序是确定的：先计算 3.0 + a 得到一个中间结果并舍入，再将这个中间结果与 3.0 相加并再次舍入。
- No Unsigned Wrap (NUW) and No Signed Wrap (NSW) flags：结果回绕；
  1. 优化思路：将 `x * 2 == 2` 简化为 `x == 1`（两边同除以2）。
  2. 合法前提：`x * 2` 必须带有 **NSW 标记**（无符号溢出）。
  3. 无 NSW 时不合法：
     - 当 `x > INT_MAX / 2`，`x * 2` 会发生溢出环绕。
     - 存在反例：`x = 0x80000001`，满足 `(x * 2) mod 2³² == 2`，但 `x mod 2³² ≠ 1`。
  4. 结论：无 NSW 时该优化会导致结果错误。
- 内存别名与ssa是什么关系 ？
#### 附件
- 论文：Simple and Efficient Construction of Static Single Assignment Form by Braun et al. published in Compiler Construction in 2013
- value class：https://llvm.org/doxygen/classllvm_1_1Value.html
