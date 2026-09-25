# LLVM 100-Day Study Plan

Welcome to the 100 Days of LLVM Challenge! This plan breaks down the massive LLVM infrastructure into 100 daily bites. We will start with IR basics, move through the optimization pipeline, dive into CodeGen, explore LTO, and finally look at Clang integration and MLIR.


## Phase 1: Core IR & Basic Optimizations (Days 1 - 10)

### **Day 0: LLVM Infrastructure & The Pass Manager**
* **Concepts:** The New Pass Manager, AnalysisManager, PassBuilder pipelines, and how opt schedules passes.
* **Code Location:** `llvm/include/llvm/IR/PassManager.h`, `llvm/lib/Passes/PassBuilderPipelines.cpp`
* **Status:** Created in [Day-00-LLVM-Infra-Tutorial.md](Day-00-LLVM-Infra-Tutorial.md)

### **Day 1: Constant Folding**
* **Concepts:** Compile-time evaluation of constants, `llvm::Constant`, `ConstantExpr`, and `IRBuilder` folder classes.
* **Code Location:** `llvm/lib/IR/ConstantFold.cpp`, `llvm/lib/Analysis/ConstantFolding.cpp`
* **Status:** Created in [Day-01-Constant-Folding.md](Day-01-Constant-Folding.md)

### **Day 2: Instruction Combining (InstCombine)**
* **Concepts:** Pattern matching for peephole optimizations, simplifying redundant instructions, algebraic simplifications.
* **Code Location:** `llvm/lib/Transforms/InstCombine/`
* **Status:** *Upcoming*

### **Day 3: Control Flow Graphs & Basic Blocks**
* **Concepts:** Iterating over functions, basic blocks, and instructions. Understanding the CFG edges.
* **Code Location:** `llvm/include/llvm/IR/BasicBlock.h`, `llvm/include/llvm/IR/CFG.h`
* **Status:** *Upcoming*

### **Day 4: Dominator Trees**
* **Concepts:** Dominance, strict dominance, post-dominance, and how LLVM uses them to verify variable reachability.
* **Code Location:** `llvm/include/llvm/IR/Dominators.h`, `llvm/lib/IR/Dominators.cpp`
* **Status:** *Upcoming*

### **Day 5: Mem2Reg**
* **Concepts:** Promoting memory allocations to registers, constructing SSA (Static Single Assignment) form, Phi node insertion.
* **Code Location:** `llvm/lib/Transforms/Utils/PromoteMemoryToRegister.cpp`
* **Status:** *Upcoming*

### **Day 6: SROA (Scalar Replacement of Aggregates)**
* **Concepts:** Breaking up `alloca` of structs/arrays into individual variables, removing memory accesses.
* **Code Location:** `llvm/lib/Transforms/Scalar/SROA.cpp`
* **Status:** *Upcoming*

### **Day 7: Dead Code Elimination (DCE & ADCE)**
* **Concepts:** Finding code that has no effect on the program output and removing it. Aggressive DCE vs standard DCE.
* **Code Location:** `llvm/lib/Transforms/Scalar/ADCE.cpp`, `llvm/lib/Transforms/Scalar/DCE.cpp`
* **Status:** *Upcoming*

### **Day 8: Early CSE**
* **Concepts:** Common Subexpression Elimination. A fast, hash-based pass to remove identical computations early in the pipeline.
* **Code Location:** `llvm/lib/Transforms/Scalar/EarlyCSE.cpp`
* **Status:** *Upcoming*

### **Day 9: Reassociate Pass**
* **Concepts:** Algebraic reassociation of commutative/associative operators to enable better constant folding and CSE.
* **Code Location:** `llvm/lib/Transforms/Scalar/Reassociate.cpp`
* **Status:** *Upcoming*

### **Day 10: Jump Threading**
* **Concepts:** Simplifying control flow by analyzing conditions that are known on specific incoming edges to a block.
* **Code Location:** `llvm/lib/Transforms/Scalar/JumpThreading.cpp`
* **Status:** *Upcoming*


## Phase 2: Advanced Scalar Optimizations (Days 11 - 20)

### **Day 11: Global Value Numbering (GVN)**
* **Concepts:** Detecting and eliminating redundant expressions, Value numbering, and Memory Dependence Analysis.
* **Code Location:** `llvm/lib/Transforms/Scalar/GVN.cpp`
* **Status:** *Upcoming*

### **Day 12: Sparse Conditional Constant Propagation (SCCP)**
* **Concepts:** Propagating constants through the CFG while simultaneously evaluating branch conditions to ignore dead blocks.
* **Code Location:** `llvm/lib/Transforms/Scalar/SCCP.cpp`
* **Status:** *Upcoming*

### **Day 13: Bit-Tracking Dead Code Elimination (BDCE)**
* **Concepts:** Tracking known zero/one bits to simplify instructions and remove dead operations.
* **Code Location:** `llvm/lib/Transforms/Scalar/BDCE.cpp`
* **Status:** *Upcoming*

### **Day 14: Correlated Value Propagation (CVP)**
* **Concepts:** Using dominance and range information (from LVI) to propagate facts and simplify branches/switches.
* **Code Location:** `llvm/lib/Transforms/Scalar/CorrelatedValuePropagation.cpp`
* **Status:** *Upcoming*

### **Day 15: Tail Call Elimination**
* **Concepts:** Transforming calls at the end of functions into branches, avoiding stack frame allocation.
* **Code Location:** `llvm/lib/Transforms/Scalar/TailRecursionElimination.cpp`
* **Status:** *Upcoming*

### **Day 16: Branch Weights & Profiling Info**
* **Concepts:** How LLVM represents branch probabilities using metadata (e.g., `!prof`) and the `LowerExpectIntrinsic` pass.
* **Code Location:** `llvm/lib/Transforms/Scalar/LowerExpectIntrinsic.cpp`
* **Status:** *Upcoming*

### **Day 17: Sink Pass**
* **Concepts:** Sinking instructions down to the basic blocks where their results are actually used, to avoid unnecessary computation.
* **Code Location:** `llvm/lib/Transforms/Scalar/Sink.cpp`
* **Status:** *Upcoming*

### **Day 18: SimplifyCFG**
* **Concepts:** Merging basic blocks, removing dead control flow, turning branches into `select` instructions.
* **Code Location:** `llvm/lib/Transforms/Utils/SimplifyCFG.cpp`
* **Status:** *Upcoming*

### **Day 19: Constraint Elimination**
* **Concepts:** Eliminating checks that are mathematically guaranteed to be true based on prior branches (e.g. bounds checks).
* **Code Location:** `llvm/lib/Transforms/Scalar/ConstraintElimination.cpp`
* **Status:** *Upcoming*

### **Day 20: Div/Rem Pairs**
* **Concepts:** Optimizing paired division and remainder operations into a single instruction or better sequences.
* **Code Location:** `llvm/lib/Transforms/Scalar/DivRemPairs.cpp`
* **Status:** *Upcoming*


## Phase 3: Loop Optimizations (Days 21 - 30)

### **Day 21: Loop Simplify**
* **Concepts:** Canonicalizing loops to have preheaders, single latches, and dedicated exits.
* **Code Location:** `llvm/lib/Transforms/Utils/LoopSimplify.cpp`
* **Status:** *Upcoming*

### **Day 22: LCSSA Form**
* **Concepts:** Loop-Closed SSA form. Ensuring all values defined in a loop and used outside pass through a Phi node.
* **Code Location:** `llvm/lib/Transforms/Utils/LCSSA.cpp`
* **Status:** *Upcoming*

### **Day 23: Loop Invariant Code Motion (LICM)**
* **Concepts:** Identifying code inside loops that doesn't change on each iteration and hoisting it into the preheader.
* **Code Location:** `llvm/lib/Transforms/Scalar/LICM.cpp`
* **Status:** *Upcoming*

### **Day 24: Loop Unrolling**
* **Concepts:** Duplicating loop bodies to expose cross-iteration optimizations and reduce loop overhead.
* **Code Location:** `llvm/lib/Transforms/Scalar/LoopUnrollPass.cpp`
* **Status:** *Upcoming*

### **Day 25: Loop Deletion**
* **Concepts:** Removing loops entirely if they are known to be finite and have no side effects.
* **Code Location:** `llvm/lib/Transforms/Scalar/LoopDeletion.cpp`
* **Status:** *Upcoming*

### **Day 26: Loop Rotation**
* **Concepts:** Rotating loops into a do-while style (with a guard condition) to expose LICM opportunities.
* **Code Location:** `llvm/lib/Transforms/Scalar/LoopRotation.cpp`
* **Status:** *Upcoming*

### **Day 27: Loop Idiom Recognize**
* **Concepts:** Detecting manual loops that just clear or copy memory and replacing them with `memset`/`memcpy` intrinsics.
* **Code Location:** `llvm/lib/Transforms/Scalar/LoopIdiomRecognize.cpp`
* **Status:** *Upcoming*

### **Day 28: Loop Unswitch**
* **Concepts:** Hoisting loop-invariant branches outside the loop by duplicating the loop body for each branch outcome.
* **Code Location:** `llvm/lib/Transforms/Scalar/SimpleLoopUnswitch.cpp`
* **Status:** *Upcoming*

### **Day 29: IndVarSimplify**
* **Concepts:** Simplifying induction variables (loop counters), often canonicalizing them to start at 0 and count up.
* **Code Location:** `llvm/lib/Transforms/Scalar/IndVarSimplify.cpp`
* **Status:** *Upcoming*

### **Day 30: Loop Interchange**
* **Concepts:** Swapping the order of nested loops to improve cache locality and memory access patterns.
* **Code Location:** `llvm/lib/Transforms/Scalar/LoopInterchange.cpp`
* **Status:** *Upcoming*


## Phase 4: Analyses & Memory (Days 31 - 40)

### **Day 31: Alias Analysis (BasicAA)**
* **Concepts:** Determining if two pointers can point to the same memory location to enable memory optimizations.
* **Code Location:** `llvm/lib/Analysis/BasicAliasAnalysis.cpp`
* **Status:** *Upcoming*

### **Day 32: Type-Based Alias Analysis (TBAA)**
* **Concepts:** Using frontend type metadata to prove that pointers of different types don't alias (e.g., `int*` and `float*`).
* **Code Location:** `llvm/lib/Analysis/TypeBasedAliasAnalysis.cpp`
* **Status:** *Upcoming*

### **Day 33: Memory Dependence Analysis**
* **Concepts:** Finding which memory instruction provides the value for a load, or which store overwrites another.
* **Code Location:** `llvm/lib/Analysis/MemoryDependenceAnalysis.cpp`
* **Status:** *Upcoming*

### **Day 34: Scalar Evolution (SCEV)**
* **Concepts:** A framework for analyzing the formulas of integer variables as they evolve through loop iterations.
* **Code Location:** `llvm/lib/Analysis/ScalarEvolution.cpp`
* **Status:** *Upcoming*

### **Day 35: SCEV & Loop Trip Counts**
* **Concepts:** Using SCEV to calculate exactly how many times a loop will execute before it terminates.
* **Code Location:** `llvm/lib/Analysis/ScalarEvolution.cpp (computeBackedgeTakenCount)`
* **Status:** *Upcoming*

### **Day 36: Dependence Analysis**
* **Concepts:** Analyzing data dependencies between loop iterations (read-after-write, etc.) for vectorization/parallelization.
* **Code Location:** `llvm/lib/Analysis/DependenceAnalysis.cpp`
* **Status:** *Upcoming*

### **Day 37: MemorySSA Form**
* **Concepts:** An SSA-like representation specifically for memory, allowing O(1) dependency queries.
* **Code Location:** `llvm/lib/Analysis/MemorySSA.cpp`
* **Status:** *Upcoming*

### **Day 38: Demanded Bits Analysis**
* **Concepts:** Determining which bits of an integer value are actually used, allowing simplification of upper bits.
* **Code Location:** `llvm/lib/Analysis/DemandedBits.cpp`
* **Status:** *Upcoming*

### **Day 39: TargetLibraryInfo**
* **Concepts:** Information about which C standard library functions are available and how to fold them (e.g. `strlen("abc") == 3`).
* **Code Location:** `llvm/lib/Analysis/TargetLibraryInfo.cpp`
* **Status:** *Upcoming*

### **Day 40: TargetTransformInfo (TTI)**
* **Concepts:** Target-specific cost models used by IR passes to decide if an optimization (like unrolling) is profitable.
* **Code Location:** `llvm/lib/Analysis/TargetTransformInfo.cpp`
* **Status:** *Upcoming*


## Phase 5: Interprocedural Optimizations (IPO) (Days 41 - 50)

### **Day 41: Call Graph Analysis**
* **Concepts:** Building a graph of which functions call which other functions, used for bottom-up/top-down traversal.
* **Code Location:** `llvm/lib/Analysis/CallGraph.cpp`
* **Status:** *Upcoming*

### **Day 42: Inliner & InlineCost Analysis**
* **Concepts:** Interprocedural Optimization (IPO): replacing a function call with the function body itself.
* **Code Location:** `llvm/lib/Transforms/IPO/Inliner.cpp`, `llvm/lib/Analysis/InlineCost.cpp`
* **Status:** *Upcoming*

### **Day 43: Interprocedural SCCP (IPSCCP)**
* **Concepts:** Propagating constants across function boundaries (e.g., if a function is always called with a constant arg).
* **Code Location:** `llvm/lib/Transforms/IPO/SCCP.cpp`
* **Status:** *Upcoming*

### **Day 44: Dead Argument Elimination**
* **Concepts:** Removing arguments from internal functions if the caller always passes unused or constant values.
* **Code Location:** `llvm/lib/Transforms/IPO/DeadArgumentElimination.cpp`
* **Status:** *Upcoming*

### **Day 45: Argument Promotion**
* **Concepts:** Promoting pointer arguments to pass the actual value by register if the function only loads from the pointer.
* **Code Location:** `llvm/lib/Transforms/IPO/ArgumentPromotion.cpp`
* **Status:** *Upcoming*

### **Day 46: Global DCE**
* **Concepts:** Removing global variables and internal functions that are never referenced.
* **Code Location:** `llvm/lib/Transforms/IPO/GlobalDCE.cpp`
* **Status:** *Upcoming*

### **Day 47: Global Optimizer (GlobalOpt)**
* **Concepts:** Optimizing globals (e.g., making them constant if they are only initialized once).
* **Code Location:** `llvm/lib/Transforms/IPO/GlobalOpt.cpp`
* **Status:** *Upcoming*

### **Day 48: Function Attributes Deduction (Attributor)**
* **Concepts:** A powerful framework to deduce properties like `readnone`, `nonnull`, `noalias` across the call graph.
* **Code Location:** `llvm/lib/Transforms/IPO/Attributor.cpp`
* **Status:** *Upcoming*

### **Day 49: Cross-Module Inlining**
* **Concepts:** Understanding how LLVM prepares functions to be inlined across different compilation units via LTO.
* **Code Location:** `llvm/lib/Transforms/IPO/CrossDSOCFI.cpp (and ThinLTO logic)`
* **Status:** *Upcoming*

### **Day 50: OpenMP/Parallel optimizations (OpenMPOpt)**
* **Concepts:** Deducing properties and optimizing OpenMP runtime calls in parallel regions.
* **Code Location:** `llvm/lib/Transforms/IPO/OpenMPOpt.cpp`
* **Status:** *Upcoming*


## Phase 6: Vectorization (Days 51 - 60)

### **Day 51: Loop Vectorizer - Legality**
* **Concepts:** Checking if it is mathematically and legally safe to vectorize a loop based on data dependencies.
* **Code Location:** `llvm/lib/Transforms/Vectorize/LoopVectorize.cpp (LoopVectorizationLegality)`
* **Status:** *Upcoming*

### **Day 52: Loop Vectorizer - Cost Model**
* **Concepts:** Estimating the performance cost of the scalar vs vectorized loop to decide the Vectorization Factor (VF).
* **Code Location:** `llvm/lib/Transforms/Vectorize/LoopVectorize.cpp (LoopVectorizationCostModel)`
* **Status:** *Upcoming*

### **Day 53: VPlan Infrastructure**
* **Concepts:** The explicit model for vectorization plans, separating analysis from IR transformation.
* **Code Location:** `llvm/lib/Transforms/Vectorize/VPlan.cpp`
* **Status:** *Upcoming*

### **Day 54: SLP Vectorizer**
* **Concepts:** Superword-Level Parallelism: finding straight-line code (like `a.x+b.x; a.y+b.y`) and combining it into vector ops.
* **Code Location:** `llvm/lib/Transforms/Vectorize/SLPVectorizer.cpp`
* **Status:** *Upcoming*

### **Day 55: Load Store Vectorizer**
* **Concepts:** Combining sequential scalar loads and stores into single wide vector loads/stores.
* **Code Location:** `llvm/lib/Transforms/Vectorize/LoadStoreVectorizer.cpp`
* **Status:** *Upcoming*

### **Day 56: Vector Combine**
* **Concepts:** Peephole optimizations specifically for vector operations (like shuffles, extracts, inserts).
* **Code Location:** `llvm/lib/Transforms/Vectorize/VectorCombine.cpp`
* **Status:** *Upcoming*

### **Day 57: Vector Predication**
* **Concepts:** Handling vectorization with active masks, useful for architectures like SVE or AVX512.
* **Code Location:** `llvm/lib/Transforms/Vectorize/VPlanRecipes.cpp`
* **Status:** *Upcoming*

### **Day 58: Interleaved Access Passes**
* **Concepts:** Optimizing struct-of-arrays (SoA) and array-of-structs (AoS) memory accesses during vectorization.
* **Code Location:** `llvm/lib/CodeGen/InterleavedAccessPass.cpp`
* **Status:** *Upcoming*

### **Day 59: Target-Specific Vector Lowering**
* **Concepts:** How generic IR vectors map to specific instruction sets via TTI and ISel.
* **Code Location:** `llvm/lib/CodeGen/SelectionDAG/LegalizeVectorOps.cpp`
* **Status:** *Upcoming*

### **Day 60: Matrix Multiply Intrinsics**
* **Concepts:** Optimizing dense matrix math using LLVM's matrix intrinsics and lowering.
* **Code Location:** `llvm/lib/Transforms/Scalar/LowerMatrixIntrinsics.cpp`
* **Status:** *Upcoming*


## Phase 7: Code Generation - SelectionDAG (Days 61 - 70)

### **Day 61: SelectionDAG Intro**
* **Concepts:** Converting LLVM IR into the SelectionDAG graph representation for CodeGen.
* **Code Location:** `llvm/lib/CodeGen/SelectionDAG/SelectionDAGBuilder.cpp`
* **Status:** *Upcoming*

### **Day 62: Legalize Types**
* **Concepts:** Converting illegal IR types (like `i123` or `v3f32`) into types supported by the target CPU.
* **Code Location:** `llvm/lib/CodeGen/SelectionDAG/LegalizeTypes.cpp`
* **Status:** *Upcoming*

### **Day 63: Legalize Operations**
* **Concepts:** Emulating unsupported operations (like 64-bit divide on a 32-bit machine) via library calls or expansion.
* **Code Location:** `llvm/lib/CodeGen/SelectionDAG/LegalizeDAG.cpp`
* **Status:** *Upcoming*

### **Day 64: DAG Combine**
* **Concepts:** Pattern matching and peephole optimization on the DAG (like InstCombine, but for target nodes).
* **Code Location:** `llvm/lib/CodeGen/SelectionDAG/DAGCombiner.cpp`
* **Status:** *Upcoming*

### **Day 65: Instruction Selection (ISel)**
* **Concepts:** Matching DAG patterns against TableGen descriptions to emit target MachineInstrs.
* **Code Location:** `llvm/lib/CodeGen/SelectionDAG/SelectionDAGISel.cpp`
* **Status:** *Upcoming*

### **Day 66: FastISel**
* **Concepts:** A fast, naive instruction selector used for `-O0` debug builds to compile quickly.
* **Code Location:** `llvm/lib/CodeGen/SelectionDAG/FastISel.cpp`
* **Status:** *Upcoming*

### **Day 67: GlobalISel Intro**
* **Concepts:** The modern, IR-based replacement for SelectionDAG, eliminating the DAG overhead.
* **Code Location:** `llvm/lib/CodeGen/GlobalISel/GlobalISel.cpp`
* **Status:** *Upcoming*

### **Day 68: GlobalISel - IRTranslator**
* **Concepts:** Translating LLVM IR into generic Machine IR (GMIR) for GlobalISel.
* **Code Location:** `llvm/lib/CodeGen/GlobalISel/IRTranslator.cpp`
* **Status:** *Upcoming*

### **Day 69: Machine IR (MIR)**
* **Concepts:** Understanding LLVM's representation for physical registers, virtual registers, and opcodes.
* **Code Location:** `llvm/include/llvm/CodeGen/MachineInstr.h`
* **Status:** *Upcoming*

### **Day 70: Two-Address Instruction Pass**
* **Concepts:** Converting 3-operand instructions (A = B + C) to 2-operand destructive forms (A = A + C) for architectures like x86.
* **Code Location:** `llvm/lib/CodeGen/TwoAddressInstructionPass.cpp`
* **Status:** *Upcoming*


## Phase 8: Code Generation - Machine Passes (Days 71 - 80)

### **Day 71: Register Allocation (Greedy)**
* **Concepts:** The default `-O2`/`-O3` register allocator, handling graph coloring, spilling, and live ranges.
* **Code Location:** `llvm/lib/CodeGen/RegAllocGreedy.cpp`
* **Status:** *Upcoming*

### **Day 72: Live Intervals Analysis**
* **Concepts:** Calculating exactly when a virtual register is defined and when its last use occurs.
* **Code Location:** `llvm/lib/CodeGen/LiveIntervals.cpp`
* **Status:** *Upcoming*

### **Day 73: Register Coalescing**
* **Concepts:** Removing copy instructions by assigning the source and destination to the same physical register.
* **Code Location:** `llvm/lib/CodeGen/RegisterCoalescer.cpp`
* **Status:** *Upcoming*

### **Day 74: Machine Instruction Scheduling**
* **Concepts:** Reordering instructions to avoid CPU pipeline stalls and hide latency.
* **Code Location:** `llvm/lib/CodeGen/MachineScheduler.cpp`
* **Status:** *Upcoming*

### **Day 75: Prolog/Epilog Insertion (PEI)**
* **Concepts:** Setting up the stack frame, saving callee-saved registers, and resolving abstract frame indices.
* **Code Location:** `llvm/lib/CodeGen/PrologEpilogInserter.cpp`
* **Status:** *Upcoming*

### **Day 76: Machine LICM & Machine CSE**
* **Concepts:** Performing LICM and CSE again, but this time on target-specific MachineInstructions.
* **Code Location:** `llvm/lib/CodeGen/MachineLICM.cpp`
* **Status:** *Upcoming*

### **Day 77: Branch Folding & Tail Duplication**
* **Concepts:** Optimizing the layout of machine basic blocks to eliminate redundant branches.
* **Code Location:** `llvm/lib/CodeGen/BranchFolding.cpp`
* **Status:** *Upcoming*

### **Day 78: Machine Block Placement**
* **Concepts:** Ordering blocks using branch probability info to maximize fall-throughs and improve instruction cache locality.
* **Code Location:** `llvm/lib/CodeGen/MachineBlockPlacement.cpp`
* **Status:** *Upcoming*

### **Day 79: Post-RA Scheduling**
* **Concepts:** A second pass of scheduling after physical registers are assigned, resolving anti-dependencies.
* **Code Location:** `llvm/lib/CodeGen/PostRASchedulerList.cpp`
* **Status:** *Upcoming*

### **Day 80: MC Layer**
* **Concepts:** Machine Code emission: turning MachineInstrs into binary object files (ELF/Mach-O) or assembly strings.
* **Code Location:** `llvm/lib/MC/MCAsmStreamer.cpp`, `llvm/lib/MC/MCObjectStreamer.cpp`
* **Status:** *Upcoming*


## Phase 9: Link Time Optimization & PGO (Days 81 - 90)

### **Day 81: LTO Overview**
* **Concepts:** Link-Time Optimization: combining all modules into one massive IR module for cross-file optimization.
* **Code Location:** `llvm/lib/LTO/LTO.cpp`
* **Status:** *Upcoming*

### **Day 82: ThinLTO - Module Summaries**
* **Concepts:** How ThinLTO creates lightweight summaries (call edges, sizes) to allow parallel LTO.
* **Code Location:** `llvm/lib/Analysis/ModuleSummaryAnalysis.cpp`
* **Status:** *Upcoming*

### **Day 83: ThinLTO - Indexing & Importing**
* **Concepts:** The link stage of ThinLTO: deciding which functions to import into which modules based on the summary index.
* **Code Location:** `llvm/lib/Transforms/IPO/FunctionImport.cpp`
* **Status:** *Upcoming*

### **Day 84: PGO - Instrumentation**
* **Concepts:** Profile Guided Optimization: inserting counters into the IR to track edge execution frequencies.
* **Code Location:** `llvm/lib/Transforms/Instrumentation/PGOInstrumentation.cpp`
* **Status:** *Upcoming*

### **Day 85: PGO - Profile Usage**
* **Concepts:** Reading `.profdata` files to annotate the IR with exact branch weights for block placement and inlining.
* **Code Location:** `llvm/lib/Transforms/Instrumentation/PGOInstrumentation.cpp (Use phase)`
* **Status:** *Upcoming*

### **Day 86: SamplePGO (AutoFDO)**
* **Concepts:** Using hardware performance counters (like Linux `perf`) instead of instrumentation for zero-overhead profiling.
* **Code Location:** `llvm/lib/Transforms/IPO/SampleProfile.cpp`
* **Status:** *Upcoming*

### **Day 87: Context-Sensitive PGO (CS-PGO)**
* **Concepts:** Distinguishing profile data for a function based on who called it, for more accurate inlining.
* **Code Location:** `llvm/lib/Transforms/IPO/SampleContextTracker.cpp`
* **Status:** *Upcoming*

### **Day 88: Post-Link Optimization (BOLT)**
* **Concepts:** Optimizing the final linked binary (reordering functions and blocks) outside of the standard compiler pipeline.
* **Code Location:** `llvm-project/bolt/ (Subproject)`
* **Status:** *Upcoming*

### **Day 89: Whole Program Devirtualization**
* **Concepts:** Converting virtual C++ method calls into direct calls when all derived classes are known at link time.
* **Code Location:** `llvm/lib/Transforms/IPO/WholeProgramDevirt.cpp`
* **Status:** *Upcoming*

### **Day 90: Control Flow Integrity (CFI)**
* **Concepts:** Security passes that prevent ROP attacks by verifying indirect call targets against a known hierarchy.
* **Code Location:** `llvm/lib/Transforms/IPO/LowerTypeTests.cpp`
* **Status:** *Upcoming*


## Phase 10: Infrastructure, Clang, and MLIR (Days 91 - 100)

### **Day 91: The New Pass Manager**
* **Concepts:** The modern infrastructure for scheduling, caching analyses, and running IR passes (`PassBuilder`).
* **Code Location:** `llvm/lib/Passes/PassBuilder.cpp`
* **Status:** *Upcoming*

### **Day 92: Clang CodeGen**
* **Concepts:** How the C++ frontend (Clang) traverses its AST to emit LLVM IR.
* **Code Location:** `clang/lib/CodeGen/CodeGenModule.cpp`
* **Status:** *Upcoming*

### **Day 93: Writing an Out-of-Tree Pass**
* **Concepts:** Setting up a CMake project to load your own custom optimization pass dynamically.
* **Code Location:** `llvm/docs/WritingAnLLVMNewPMPass.rst`
* **Status:** *Upcoming*

### **Day 94: TableGen (.td) Basics**
* **Concepts:** Understanding the declarative language LLVM uses to define registers, instructions, and DAG patterns.
* **Code Location:** `llvm/utils/TableGen/TableGen.cpp`
* **Status:** *Upcoming*

### **Day 95: TableGen for InstrInfo**
* **Concepts:** How to read and understand target instruction definitions (e.g., `X86InstrInfo.td`).
* **Code Location:** `llvm/lib/Target/X86/X86InstrInfo.td`
* **Status:** *Upcoming*

### **Day 96: Sanitizers (ASan, TSan)**
* **Concepts:** How LLVM instruments code to catch memory leaks, out-of-bounds accesses, and data races.
* **Code Location:** `llvm/lib/Transforms/Instrumentation/AddressSanitizer.cpp`
* **Status:** *Upcoming*

### **Day 97: ORC JIT**
* **Concepts:** On-Request Compilation: LLVM's modern library for building Just-In-Time compilers.
* **Code Location:** `llvm/lib/ExecutionEngine/Orc/Core.cpp`
* **Status:** *Upcoming*

### **Day 98: Debug Info & DWARF**
* **Concepts:** How LLVM preserves source code locations using `llvm.dbg.value` intrinsics and metadata.
* **Code Location:** `llvm/lib/CodeGen/AsmPrinter/DwarfDebug.cpp`
* **Status:** *Upcoming*

### **Day 99: MLIR Introduction**
* **Concepts:** Multi-Level IR: LLVM's newer subproject for domain-specific representations (like TensorFlow or Polyhedral loops).
* **Code Location:** `mlir/lib/IR/MLIRContext.cpp`
* **Status:** *Upcoming*

### **Day 100: The Full Pipeline Review**
* **Concepts:** Bringing it all together: Building a custom compiler frontend that lowers to LLVM IR, optimizes, and JITs.
* **Code Location:** `llvm-project/llvm/examples/Kaleidoscope/`
* **Status:** *Upcoming*

---

## How to use this plan
1. Every day, create or read the corresponding `.md` file in this directory.
2. Open the source code in `llvm-project` and trace through the main entry points.
3. Write small C/C++ examples and compile them using `clang -O0 -S -emit-llvm` vs `clang -O3 -S -emit-llvm` to see the optimizations in action!
