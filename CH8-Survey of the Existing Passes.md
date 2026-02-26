- Let us get started with how you even begin this journey of finding what the middle end has to offer.
- using the description of this pass, you can find where it is implemented by running from your LLVM source directory something such as the following:
  ```
  git grep "the short description of the pass" llvm
  ```
- At this point, you need to use a second command to check which passes are indeed supported by the new pass manager. This command prints the names of all the passes that the new pass manager supports. For instance
  ```
  opt --print-passes
  Module passes:
    always-inline
    annotation2metadata
    attributor
    attributor-light
    called-value-propagation
    canonicalize-aliases
    check-debugify
    constmerge
    coro-cleanup
    coro-early
    cross-dso-cfi
    deadargelim
    debugify
    dfsan
    dot-callgraph
    dxil-upgrade
    elim-avail-extern
    extract-blocks
    forceattrs
    function-import
    globalopt
    globalsplit
    hipstdpar-interpose-alloc
    hipstdpar-select-accelerator-code
    hotcoldsplit
    inferattrs
    inliner-ml-advisor-release
    inliner-wrapper
    inliner-wrapper-no-mandatory-first
    insert-gcov-profiling
    instrorderfile
    instrprof
    internalize
    invalidate<all>
    iroutliner
    jmc-instrumenter
    lower-emutls
    lower-global-dtors
    lower-ifunc
    lowertypetests
    memprof-context-disambiguation
    memprof-module
    mergefunc
    metarenamer
    module-inline
    name-anon-globals
    no-op-module
    objc-arc-apelim
    openmp-opt
    openmp-opt-postlink
    partial-inliner
    pgo-icall-prom
    pgo-instr-gen
    pgo-instr-use
    poison-checking
    print
    print-callgraph
    print-callgraph-sccs
    print-ir-similarity
    print-lcg
    print-lcg-dot
    print-must-be-executed-contexts
    print-profile-summary
    print-stack-safety
    print<inline-advisor>
    print<module-debuginfo>
    pseudo-probe
    pseudo-probe-update
    recompute-globalsaa
    rel-lookup-table-converter
    rewrite-statepoints-for-gc
    rewrite-symbols
    rpo-function-attrs
    sample-profile
    sancov-module
    sanmd-module
    scc-oz-module-inliner
    shadow-stack-gc-lowering
    strip
    strip-dead-debug-info
    strip-dead-prototypes
    strip-debug-declare
    strip-nondebug
    strip-nonlinetable-debuginfo
    synthetic-counts-propagation
    trigger-crash
    trigger-verifier-error
    tsan-module
    verify
    view-callgraph
    wholeprogramdevirt
  Module passes with params:
    asan<kernel>
    cg-profile<in-lto-post-link>
    global-merge<group-by-use;ignore-single-use;max-offset=N;merge-const;merge-external;no-group-by-use;no-ignore-single-use;no-merge-const;no-merge-external;size-only>
    embed-bitcode<thinlto;emit-summary>
    globaldce<in-lto-post-link>
    hwasan<kernel;recover>
    ipsccp<no-func-spec;func-spec>
    loop-extract<single>
    memprof-use<profile-filename=S>
    msan<recover;kernel;eager-checks;track-origins=N>
    print<structural-hash><detailed>
  Module analyses:
    callgraph
    collector-metadata
    inline-advisor
    ir-similarity
    lcg
    module-summary
    no-op-module
    pass-instrumentation
    profile-summary
    stack-safety
    verify
    globals-aa
  Module alias analyses:
    globals-aa
  CGSCC passes:
    argpromotion
    attributor-cgscc
    attributor-light-cgscc
    invalidate<all>
    no-op-cgscc
    openmp-opt-cgscc
  CGSCC passes with params:
    coro-split<reuse-storage>
    function-attrs<skip-non-recursive-function-attrs>
    inline<only-mandatory>
  CGSCC analyses:
    no-op-cgscc
    fam-proxy
    pass-instrumentation
  Function passes:
    aa-eval
    adce
    add-discriminators
    aggressive-instcombine
    alignment-from-assumptions
    annotation-remarks
    assume-builder
    assume-simplify
    bdce
    bounds-checking
    break-crit-edges
    callbrprepare
    callsite-splitting
    chr
    codegenprepare
    consthoist
    constraint-elimination
    coro-elide
    correlated-propagation
    count-visits
    dce
    declare-to-assign
    dfa-jump-threading
    div-rem-pairs
    dot-cfg
    dot-cfg-only
    dot-dom
    dot-dom-only
    dot-post-dom
    dot-post-dom-only
    dse
    dwarf-eh-prepare
    expand-large-div-rem
    expand-large-fp-convert
    expand-memcmp
    fix-irreducible
    flattencfg
    float2int
    gc-lowering
    guard-widening
    gvn-hoist
    gvn-sink
    helloworld
    indirectbr-expand
    infer-address-spaces
    infer-alignment
    inject-tli-mappings
    instcount
    instnamer
    instsimplify
    interleaved-access
    interleaved-load-combine
    invalidate<all>
    irce
    jump-threading
    kcfi
    lcssa
    libcalls-shrinkwrap
    lint
    load-store-vectorizer
    loop-data-prefetch
    loop-distribute
    loop-fusion
    loop-load-elim
    loop-simplify
    loop-sink
    loop-versioning
    lower-constant-intrinsics
    lower-expect
    lower-guard-intrinsic
    lower-widenable-condition
    loweratomic
    lowerinvoke
    lowerswitch
    make-guards-explicit
    mem2reg
    memcpyopt
    memprof
    mergeicmps
    mergereturn
    move-auto-init
    nary-reassociate
    newgvn
    no-op-function
    objc-arc
    objc-arc-contract
    objc-arc-expand
    pa-eval
    partially-inline-libcalls
    pgo-memop-opt
    place-safepoints
    print
    print-alias-sets
    print-cfg-sccs
    print-memderefs
    print-mustexecute
    print-predicateinfo
    print<access-info>
    print<assumptions>
    print<block-freq>
    print<branch-prob>
    print<cost-model>
    print<cycles>
    print<da>
    print<debug-ata>
    print<delinearization>
    print<demanded-bits>
    print<domfrontier>
    print<domtree>
    print<func-properties>
    print<inline-cost>
    print<inliner-size-estimator>
    print<lazy-value-info>
    print<loops>
    print<memoryssa-walker>
    print<phi-values>
    print<postdomtree>
    print<regions>
    print<scalar-evolution>
    print<stack-safety-local>
    print<uniformity>
    reassociate
    redundant-dbg-inst-elim
    reg2mem
    safe-stack
    scalarize-masked-mem-intrin
    scalarizer
    sccp
    select-optimize
    separate-const-offset-from-gep
    sink
    sjlj-eh-prepare
    slp-vectorizer
    slsr
    stack-protector
    strip-gc-relocates
    structurizecfg
    tailcallelim
    tlshoist
    transform-warning
    trigger-verifier-error
    tsan
    typepromotion
    unify-loop-exits
    vector-combine
    verify
    verify<domtree>
    verify<loops>
    verify<memoryssa>
    verify<regions>
    verify<safepoint-ir>
    verify<scalar-evolution>
    view-cfg
    view-cfg-only
    view-dom
    view-dom-only
    view-post-dom
    view-post-dom-only
    wasm-eh-prepare
  Function passes with params:
    cfguard<check;dispatch>
    early-cse<memssa>
    ee-instrument<post-inline>
    function-simplification<O1;O2;O3;Os;Oz>
    gvn<no-pre;pre;no-load-pre;load-pre;no-split-backedge-load-pre;split-backedge-load-pre;no-memdep;memdep>
    hardware-loops<force-hardware-loops;force-hardware-loop-phi;force-nested-hardware-loop;force-hardware-loop-guard;hardware-loop-decrement=N;hardware-loop-counter-bitwidth=N>
    instcombine<no-use-loop-info;use-loop-info;no-verify-fixpoint;verify-fixpoint;max-iterations=N>
    loop-unroll<O0;O1;O2;O3;full-unroll-max=N;no-partial;partial;no-peeling;peeling;no-profile-peeling;profile-peeling;no-runtime;runtime;no-upperbound;upperbound>
    loop-vectorize<no-interleave-forced-only;interleave-forced-only;no-vectorize-forced-only;vectorize-forced-only>
    lower-matrix-intrinsics<minimal>
    mldst-motion<no-split-footer-bb;split-footer-bb>
    print<da><normalized-results>
    print<memoryssa><no-ensure-optimized-uses>
    print<stack-lifetime><may;must>
    separate-const-offset-from-gep<lower-gep>
    simplifycfg<no-forward-switch-cond;forward-switch-cond;no-switch-range-to-icmp;switch-range-to-icmp;no-switch-to-lookup;switch-to-lookup;no-keep-loops;keep-loops;no-hoist-common-insts;hoist-common-insts;no-sink-common-insts;sink-common-insts;bonus-inst-threshold=N>
    speculative-execution<only-if-divergent-target>
    sroa<preserve-cfg;modify-cfg>
    win-eh-prepare<demote-catchswitch-only>
  Function analyses:
    aa
    access-info
    assumptions
    bb-sections-profile-reader
    block-freq
    branch-prob
    cycles
    da
    debug-ata
    demanded-bits
    domfrontier
    domtree
    func-properties
    gc-function
    inliner-size-estimator
    lazy-value-info
    loops
    memdep
    memoryssa
    no-op-function
    opt-remark-emit
    pass-instrumentation
    phi-values
    postdomtree
    regions
    scalar-evolution
    should-not-run-function-passes
    should-run-extra-vector-passes
    ssp-layout
    stack-safety-local
    targetir
    targetlibinfo
    uniformity
    verify
    basic-aa
    objc-arc-aa
    scev-aa
    scoped-noalias-aa
    tbaa
  Function alias analyses:
    basic-aa
    objc-arc-aa
    scev-aa
    scoped-noalias-aa
    tbaa
  LoopNest passes:
    loop-flatten
    loop-interchange
    loop-unroll-and-jam
    no-op-loopnest
  Loop passes:
    canon-freeze
    dot-ddg
    guard-widening
    indvars
    invalidate<all>
    loop-bound-split
    loop-deletion
    loop-idiom
    loop-instsimplify
    loop-predication
    loop-reduce
    loop-reroll
    loop-simplifycfg
    loop-unroll-full
    loop-versioning-licm
    no-op-loop
    print
    print<ddg>
    print<iv-users>
    print<loop-cache-cost>
    print<loopnest>
  Loop passes with params:
    licm<allowspeculation>
    lnicm<allowspeculation>
    loop-rotate<no-header-duplication;header-duplication;no-prepare-for-lto;prepare-for-lto>
    simple-loop-unswitch<nontrivial;no-nontrivial;trivial;no-trivial>
  Loop analyses:
    ddg
    iv-users
    no-op-loop
    pass-instrumentation
  Machine module passes (WIP):
  Machine function passes (WIP):
  Machine function analyses (WIP):
    pass-instrumentation
  ```

- The developer tool named opt acts as a driver for all the LLVM-IR-to-LLVM-IR passes. As such, it covers everything that exists in the middle end. Therefore, to get an overview of everything that is available there, you can simply leverage the help message offered by this tool. But there is a caveat. This tool allows you to use both the legacy pass manager and the new one. Therefore, the default help message (available through the --help option) covers both managers indiscriminately. In other words, what is printed may go beyond the middle end since the legacy pass manager also drives the backends
- Therefore, the default help message (available through the --help option) covers both managers indiscriminately. This command describes how to use the opt tool and lists all the passes you can run, but it does not tell you which one runs with each pass manager

- These subdirectories, as identified by the name after each bullet point, contain, respectively:
  - AggressiveInstCombine: A more aggressive version of instcombiner; see the InstCombine point that follows.
  - Coroutines: Transformations around the lowering of coroutines. A coroutine is a function that can be resumed or suspended without locking the current thread. The lowering is usually highly specific to a source language.
  - HipStdPar: Transformations to enable the HIP C++ standard parallelism support.
  - IPO: Interprocedural optimizations, or IPOs, are transformations that apply across procedures, for instance, inlining, which takes the IR of the callee function and puts it in the IR of its caller.
  - InstCombine: Transformations that apply simple rewrite patterns that are beneficial for all targets. Instrumentation: Transformations used for instrumentation purposes. These transformations add constructs to the IR to maintain specific data structures next to the original IR. These are used to collect information for performance purposes, such as program-guided optimization (PGO), or for debugging purposes, like what a sanitizer would need to diagnost use-after-free issues or undefined behaviors.
  - Scalar: Transformations that are non-vector-related. This includes many different optimizations, of which the loop optimizations are unrelated to vectorization (see the Vectorize bullet point below).
  - Utils: Generally useful passes or helper functions used to transform the LLVM IR.
  - Vectorize: Transformations that produce vectorized code, meaning code that follows a singleinstruction multiple-data (SIMD) model. In other words, passes contained in this directory produce LLVM IR instructions that use vector types.


- Using the list of files returned by this command, you can invoke lit to re-run a specific test or subset of the tests and see the pass in action by looking at the input IR and the output IR
  ```
  git grep -l 'RUN: .*always-inline' llvm/test
  ```
- In this section, you will find out how to do the following:
  - Determine whether a pass is part of the middle end
  - Get a brief description of all the passes
  - Find the implementation of the passes
  - Get some more information on what a pass does
  - Find the unit tests of a pass and play with the input IR to refine your understanding of what a pass does
- verify: opt --passes=verify input.ll -o input2.ll
- Printer pass 再每个pass 前后执行
  ```
  -print-before-all/-print-after-all: Print the IR before/after each pass.
  -print-before/-print-after=passname: Print the IR before/after the pass named passname,
  as identified by the name used in the registration process in the related pass manager (see Chapter 5).
  ```
- 旧版 Pass 管理器对外的非分析类 Pass，多在匿名命名空间实现，无法直接用构造函数实例化。需通过对应 createXXXPass 函数创建，XXX 为旧版 Pass 名称。
- Doing this translates into the following snippet for the legacy pass manager:
  ```
  TargetTransformInfo &TTI = getAnalysis<TargetTransformInfoWrapperPass>().getTTI(F);
  ```
  In this snippet, the call to getAnalysis happens from your pass (which must inherit one of the derived Pass classes) and F is an instance of the Function class. Similarly, with the new pass manager, this translates into the following snippet:
  ```
  TargetTransformInfo &TTI = FAM.getResult<TargetIRAnalysis>(F);
  ```
  we get the result of the TargetIRAnalysis pass from an instance of the FunctionAnalysisManager class, called FAM

- 别名分析
  ```
  程序采用两种别名分析：一种基于指向值类型（严格别名规则，浮点与整型指针不互斥），另一种基于范围分析（保守估算指针可访问内存大小）。
  类型分析开销更低，若能判断指针是否重叠，就不启用范围分析。
  ```
- basic block frequency
  ```
  opt -passes='print<block-freq>' input.ll
  WARNING: You're attempting to print out a bitcode file.
  This is inadvisable as it may cause display problems. If
  you REALLY want to taste LLVM bitcode first-hand, you
  can force output with the `-f' option.
  
  Printing analysis results of BFI for function 'main':
  block-frequency-info: main
   - : float = 1.0, int = 562949953421312
   - : float = 32.0, int = 18014398509481984
   - : float = 31.0, int = 17451448556060672
   - : float = 31.0, int = 17451448556060672
   - : float = 1.0, int = 562949953421312
  
  $ cat test.c
  int main() {
    for (int i = 0; i < 100; i++) {
      ;
    }
    return 0;
  }
  $ cat input.ll
  ; ModuleID = 'test.c'
  source_filename = "test.c"
  target datalayout = "e-m:e-i8:8:32-i16:16:32-i64:64-i128:128-n32:64-S128"
  target triple = "aarch64-unknown-linux-gnu"
  
  ; Function Attrs: noinline nounwind optnone uwtable
  define dso_local i32 @main() #0 {
    %1 = alloca i32, align 4
    %2 = alloca i32, align 4
    store i32 0, ptr %1, align 4
    store i32 0, ptr %2, align 4
    br label %3
  3:                                                ; preds = %7, %0
    %4 = load i32, ptr %2, align 4
    %5 = icmp slt i32 %4, 100
    br i1 %5, label %6, label %10
  6:                                                ; preds = %3
    br label %7
  7:                                                ; preds = %6
    %8 = load i32, ptr %2, align 4
    %9 = add nsw i32 %8, 1
    store i32 %9, ptr %2, align 4
    br label %3, !llvm.loop !6
  10:                                               ; preds = %3
    ret i32 0
  }
  ```
- Domtree：This analysis is part of the IR library and not the Analysis library because it is deeply connected with the SSA properties
  ```
  $ opt --passes=dot-cfg input.ll -disable-output
  $ dot -Tpng .main.dot -o cfg.png
  $ opt -passes='print<domtree>' input.ll
  WARNING: You're attempting to print out a bitcode file.
  This is inadvisable as it may cause display problems. If
  you REALLY want to taste LLVM bitcode first-hand, you
  can force output with the `-f' option.
  
  DominatorTree for function: main
  =============================--------------------------------
  Inorder Dominator Tree: DFSNumbers invalid: 0 slow queries.
    [1] %0 {4294967295,4294967295} [0]
      [2] %3 {4294967295,4294967295} [1]
        [3] %6 {4294967295,4294967295} [2]
          [4] %7 {4294967295,4294967295} [3]
        [3] %10 {4294967295,4294967295} [2]
  Roots: %0 
  ```
  <img width="442" height="663" alt="image" src="https://github.com/user-attachments/assets/40713100-1dfe-4bba-9cff-c8377ec837c8" />
- value tracking
  ```
  /// Determine which bits of V are known to be either zero or one and return
  /// them in the KnownZero/KnownOne bit sets.
  ///
  /// This function is defined on values with integer type, values with pointer
  /// type, and vectors of integers.  In the case
  /// where V is a vector, the known zero and known one values are the
  /// same width as the vector element, and the bit is set only if it is true
  /// for all of the elements in the vector.
  LLVM_ABI void computeKnownBits(const Value *V, KnownBits &Known,
                                 const DataLayout &DL,
                                 AssumptionCache *AC = nullptr,
                                 const Instruction *CxtI = nullptr,
                                 const DominatorTree *DT = nullptr,
                                 bool UseInstrInfo = true, unsigned Depth = 0);
  ```

- instcombine pass优化后，指令数增加，但是为后续优化：https://llvm.org/doxygen/InstCombineAddSub_8cpp_source.html
  ```
  For instance, consider the following input IR from that file:
  %b = load i64, ptr %x
  %c = inttoptr i64 %b to ptr
  This snippet loads a 64-bit value and casts it to a pointer type. Since the target only supports 32-bit
  pointers, this means this conversion implicitly truncates the 64-bit value to 32-bit.
  When running instcombine on this example, the truncate is explicitly exposed:
  %b = load i64, ptr %x
  %b32 = trunc i64 %b to i32
  %c = inttoptr i32 %b32 to ptr

  # 第二个例子：
  Consider the following snippet in an LLVM IR:
  %res = xor i64 %x, %x
  This snippet does a bitwise logical exclusive of both operands, which in this case is twice the %x value.
  This will always return 0 and instcombine catches this pattern and replaces all the values of %res
  with the constant 0.
  ```
  - 相关用例：llvm/test/Transforms/InstCombine; Now, realistically, we recommend using it without going deeply into what it does. If you find that there are patterns that it does not optimize, consider whether this is beneficial for all targets (or a **canonical** way to represent something), and if yes, try to contribute it back to open source. If not, you must do that in your own target-specific pass.
    ```
    # git grep -l 'RUN: .*instcombine' llvm/test
    define i32 @test13(i32 %A) {
    ; CHECK-LABEL: @test13(
    ; CHECK-NEXT:    ret i32 8
    ;
      %B = or i32 %A, 12
      ; Always equal to 8
      %C = and i32 %B, 8
      ret i32 %C
    }
    ```
    instcombine是一类pass
    ```
    llvm/lib/Target/AMDGPU/AMDGPUInstCombineIntrinsic.cpp
    llvm/lib/Target/X86/X86InstCombineIntrinsic.cpp
    llvm/lib/Transforms/AggressiveInstCombine
    llvm/lib/Transforms/AggressiveInstCombine/TruncInstCombine.cpp
    llvm/lib/Transforms/AggressiveInstCombine/AggressiveInstCombineInternal.h
    llvm/lib/Transforms/AggressiveInstCombine/AggressiveInstCombine.cpp
    llvm/lib/Transforms/InstCombine/InstCombineAtomicRMW.cpp
    llvm/lib/Transforms/InstCombine/InstCombineCompares.cpp
    llvm/lib/Transforms/InstCombine/InstCombineCasts.cpp
    llvm/lib/Transforms/InstCombine/InstCombineInternal.h
    llvm/lib/Transforms/InstCombine/InstCombineAddSub.cpp
    llvm/lib/Transforms/InstCombine/InstCombineLoadStoreAlloca.cpp
    llvm/lib/Transforms/InstCombine/InstCombineMulDivRem.cpp
    llvm/lib/Transforms/InstCombine/InstCombineSelect.cpp
    llvm/lib/Transforms/InstCombine/InstCombineSimplifyDemanded.cpp
    llvm/lib/Transforms/InstCombine/InstCombineVectorOps.cpp
    llvm/lib/Transforms/InstCombine/InstCombineAndOrXor.cpp
    llvm/lib/Transforms/InstCombine/InstCombineNegator.cpp
    llvm/lib/Transforms/InstCombine/InstCombinePHI.cpp
    llvm/lib/Transforms/InstCombine/InstCombineCalls.cpp
    llvm/lib/Transforms/InstCombine/InstCombineShifts.cpp
    ```
  - mem2reg: llvm/lib/Transforms/Utils/PromoteMemoryToRegister.cpp
    ```
    git grep -l 'RUN: .*mem2reg' llvm/test
    llvm/test/Analysis/TypeBasedAliasAnalysis/argument-promotion.ll
    llvm/test/DebugInfo/Generic/assignment-tracking/mem2reg/phi.ll
    llvm/test/DebugInfo/Generic/assignment-tracking/mem2reg/single-block-alloca.ll
    llvm/test/DebugInfo/Generic/assignment-tracking/mem2reg/single-store-alloca.ll
    llvm/test/DebugInfo/Generic/assignment-tracking/mem2reg/store-to-part-of-alloca.ll
    llvm/test/DebugInfo/Generic/assignment-tracking/sroa/split-pre-fragmented-store-2.ll
    llvm/test/DebugInfo/Generic/assignment-tracking/sroa/split-pre-fragmented-store.ll
    llvm/test/DebugInfo/Generic/dbg-value-lower-linenos.ll
    llvm/test/DebugInfo/Generic/mem2reg-promote-alloca-1.ll
    llvm/test/DebugInfo/Generic/mem2reg-promote-alloca-2.ll
    llvm/test/DebugInfo/Generic/mem2reg-promote-alloca-3.ll
    llvm/test/DebugInfo/Generic/volatile-alloca.ll
    llvm/test/DebugInfo/X86/mem2reg_fp80.ll
    llvm/test/DebugInfo/debugify.ll
    llvm/test/Other/new-pm-print-pipeline.ll
    llvm/test/Other/printer.ll
    llvm/test/Transforms/ArgumentPromotion/basictest.ll
    llvm/test/Transforms/ArgumentPromotion/profile.ll
    llvm/test/Transforms/InstCombine/2004-09-20-BadLoadCombine.ll
    llvm/test/Transforms/InstCombine/2004-09-20-BadLoadCombine2.ll
    llvm/test/Transforms/InstCombine/2007-02-01-LoadSinkAlloca.ll
    llvm/test/Transforms/InstCombine/2007-12-28-IcmpSub2.ll
    llvm/test/Transforms/JumpThreading/and-and-cond.ll
    llvm/test/Transforms/JumpThreading/and-cond.ll
    llvm/test/Transforms/Mem2Reg/2002-03-28-UninitializedVal.ll
    llvm/test/Transforms/Mem2Reg/2002-05-01-ShouldNotPromoteThisAlloca.ll
    llvm/test/Transforms/Mem2Reg/2003-04-10-DFNotFound.ll
    llvm/test/Transforms/Mem2Reg/2003-04-18-DeadBlockProblem.ll
    llvm/test/Transforms/Mem2Reg/2003-04-24-MultipleIdenticalSuccessors.ll
    llvm/test/Transforms/Mem2Reg/2003-06-26-IterativePromote.ll
    llvm/test/Transforms/Mem2Reg/2003-10-05-DeadPHIInsertion.ll
    llvm/test/Transforms/Mem2Reg/2005-06-30-ReadBeforeWrite.ll
    llvm/test/Transforms/Mem2Reg/2005-11-28-Crash.ll
    llvm/test/Transforms/Mem2Reg/ConvertDebugInfo.ll
    llvm/test/Transforms/Mem2Reg/ConvertDebugInfo2.ll
    llvm/test/Transforms/Mem2Reg/PromoteMemToRegister.ll
    llvm/test/Transforms/Mem2Reg/UndefValuesMerge.ll
    llvm/test/Transforms/Mem2Reg/alloca_addrspace.ll
    llvm/test/Transforms/Mem2Reg/atomic.ll
    llvm/test/Transforms/Mem2Reg/crash.ll
    llvm/test/Transforms/Mem2Reg/dbg-inline-scope-for-phi.ll
    llvm/test/Transforms/Mem2Reg/dbg_declare_to_value_conversions.ll
    llvm/test/Transforms/Mem2Reg/debug-alloca-phi-2.ll
    llvm/test/Transforms/Mem2Reg/debug-alloca-phi.ll
    llvm/test/Transforms/Mem2Reg/debug-alloca-vla-1.ll
    llvm/test/Transforms/Mem2Reg/debug-alloca-vla-2.ll
    llvm/test/Transforms/Mem2Reg/ignore-droppable.ll
    llvm/test/Transforms/Mem2Reg/ignore-lifetime.ll
    llvm/test/Transforms/Mem2Reg/opaque-ptr.ll
    llvm/test/Transforms/Mem2Reg/optnone.ll
    llvm/test/Transforms/Mem2Reg/pr24179.ll
    llvm/test/Transforms/Mem2Reg/pr37632-unreachable-list-of-stores.ll
    llvm/test/Transforms/Mem2Reg/preserve-nonnull-load-metadata.ll
    llvm/test/Transforms/Mem2Reg/single-store.ll
    llvm/test/Transforms/Mem2Reg/undef-order.ll
    llvm/test/Transforms/SimplifyCFG/hoist-from-addresstaken-block.ll
    llvm/test/Transforms/SimplifyCFG/inline-asm-sink.ll
    ```
 - The memory to register rewriter:mem2reg 为什么它是优化的起点？
   - 经过该 Pass 后，IR 变成了真正的 SSA 形式，后续优化得以高效、精确地进行：
   - def-use 链清晰：每个值只有唯一的定义点，使用点可直接追踪。
   - 支配信息可用：可以轻松判断某个定义是否支配其使用，这对代码移动、循环优化等至关重要。
   - 别名分析负担减轻：大多数局部变量不再涉及内存，别名查询范围缩小。
   - 经典优化立即生效：如常数传播、死代码消除、值合并、循环不变代码外提等，都基于 SSA 实现。
