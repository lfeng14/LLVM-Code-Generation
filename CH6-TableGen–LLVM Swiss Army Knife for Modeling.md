- TableGen is DSL, generates the table that represents all the registers of a target.
- Running this command will yield the following records(.td represent target)
  ```
  class Person<int age, string name> {
  int _age = age;
  string _name = name;
  }
  def A: Person<23, "A">;
  def B: Person<64, "B">;
  def /*Anonym*/: Person<43, "anonymous">;
  
  $ ${LLVM_INSTALL_DIR}/bin/llvm-tblgen person.td
  :
  def A { // Person
  int _age = 23;
  string _name = "A";
  }
  def B { // Person
  int _age = 64;
  string _name = "B";
  }
  def anonymous_0 { // Person
  int _age = 43;
  string _name = "anonymous";
  }
  ```
- the .inc suffix is used as an indication that the related file has been automatically generated.
 <img width="1306" height="334" alt="image" src="https://github.com/user-attachments/assets/b1cdae86-ece7-4c0a-ba57-570d32872119" />

- Defining multiple records at once
  ```
  multiclass Bundle<string base> {
  def A {
  string name = !strconcat(base, "-"
  int price = 12;
  int weight = 1;
  , "A");
  }
  def B {
  string name = !strconcat(base, "-"
  string tag = "special";
  , "B");
  }
  }
  class ShippingPrice<int arg> {
  int shippingPrice = arg;
  }
  defm valuedBundle : Bundle<"valued">, ShippingPrice<5>;
  ```

  ```
  $ llvm-tblgen -gen-instr-info -I ${LLVM_SRC}/llvm/lib/Target/AArch64 -I${LLVM_
  BUILD}/include -I ${LLVM_SRC}/llvm/include -I ${LLVM_SRC}/llvm/lib/Target
  ${LLVM_SRC}/llvm/lib/Target/AArch64/AArch64.td --write-if-changed -o lib/Target/
  AArch64/AArch64GenInstrInfo.inc -d lib/Target/AArch64/AArch64GenInstrInfo.inc.d
  ```

  ```
  git grep -l 'AArch64GenInstrInfo\.inc"' -- llvm
  llvm/lib/Target/AArch64/AArch64InstrInfo.cpp
  llvm/lib/Target/AArch64/AArch64InstrInfo.h
  llvm/lib/Target/AArch64/Disassembler/AArch64Disassembler.cpp
  llvm/lib/Target/AArch64/MCTargetDesc/AArch64MCTargetDesc.cpp
  llvm/lib/Target/AArch64/MCTargetDesc/AArch64MCTargetDesc.h
  llvm/tools/llvm-exegesis/lib/AArch64/Target.cpp
  llvm/unittests/Target/AArch64/AArch64RegisterInfoTest.cpp
  llvm/unittests/Target/AArch64/AArch64SVESchedPseudoTest.cpp
  ```
#### 附件
- https://llvm.org/docs/TableGen/ProgRef.html#bang-operators.
