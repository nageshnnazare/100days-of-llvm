# Glossary of Core LLVM Optimization Passes

A short reference for passes you will see in LLVM's `-O2` and `-O3` pipelines, with before/after sketches. Each entry says why the pass is in the compiler, not only what it rewrites. The day-by-day reading plan, with source walks and examples, is the [100 Days of LLVM](README.md) index.

The sequences below were printed by Homebrew LLVM 22.1.4:

```bash
opt -passes='default<O2>' -print-pipeline-passes -disable-output </dev/null
```

Substitute `O1`, `O3`, `Os`, or `Oz`. The order changes between LLVM releases. A pass that is missing from the printout is not "implied"; it runs only if you name it, except `loop-simplify` and `lcssa`, which the loop pass manager inserts itself (see [Where a pass runs](#where-a-pass-runs)).

## Where a pass runs

`clang -O2` and `opt -passes='default<O2>'` build the same style of pipeline. It is not one flat list. Four managers nest:

```text
  Module          ipsccp, globalopt, globaldce, deadargelim, always-inline
      |
      +-- once per function, early
      |     lower-expect, simplifycfg, sroa, early-cse, mem2reg, instcombine
      |
      +-- CGSCC, bottom-up on the call graph
      |     inline, then the scalar and loop passes, then coro-split
      |     (a callee is optimized before its caller is inlined)
      |
      +-- once per function, late
            loop-vectorize, slp-vectorizer, loop-unroll, div-rem-pairs
```

Inside every `loop(...)` and `loop-mssa(...)` adaptor, LLVM first runs `loop-simplify` and `lcssa`. The printed pipeline does not spell those two names. The loop pass manager adds them so that LICM, indvars, and the vectorizer see a preheader, one latch, and a single-input phi at each exit. Loop passes then walk innermost loop first.

Several passes are repeated on purpose. InstCombine runs after inlining, after the loop passes, and after vectorization, because each of those creates the patterns it folds. SimplifyCFG runs whenever a pass has left an empty block or a constant branch. SROA runs again after inlining because the inlined body brings new allocas.

### The `-O2` order

This is `default<O2>` with the angle-bracket options removed. The nesting is the real order. The sentence after each step is why that step is required there, before the next one.

1. **Module prelude.** `memprof-remove-attributes`, `annotation2metadata`, `forceattrs`, `inferattrs`, `coro-early`.
   Required so later passes see attributes and coroutine intrinsics instead of frontend markers. A pass cannot honor `noinline` or a forced attribute that is still sitting in an annotation.

2. **Early function cleanup.** `lower-expect`, `simplifycfg`, `sroa`, `early-cse`.
   Required before any serious analysis. Branch weights have to exist before a later pass reads them, aggregates have to be split before they can be promoted, and the obvious duplicate adds have to go so the inliner and IPSCCP are not costing the unsimplified body.

3. **Module constants, before inlining.** `openmp-opt`, `ipsccp`, `called-value-propagation`, `globalopt`.
   Required before the inliner decides what to copy. If a callee argument is the constant `7`, the inliner should see that; discovering it only after the call has been left in place is too late for this round.

4. **SSA and a first combine.** `mem2reg`, `instcombine`, `simplifycfg`.
   Required so the inliner copies SSA, not allocas. Inlining a function that is still loads and stores spreads that form into every caller. Promoting first means the copied body is already values and phis.

5. **`always-inline`.**
   Required before the cost-model inliner. Functions the frontend marked `alwaysinline` must be copied even when they look expensive. Doing it here means the CGSCC inliner never reconsiders them.

6. **CGSCC, bottom-up.** For each strongly connected component: `inline`, `function-attrs`, `openmp-opt-cgscc`, then the function pipeline in step 7, then `function-attrs`, `coro-split`, `coro-annotation-elide`.
   Required in this order because a caller can only be optimized across a call after the callee has already been simplified. Bottom-up means the callee's SCC runs step 7 before the caller inlines it. `function-attrs` then records facts (`readonly`, `nocapture`) the next inline decision uses.

7. **Scalar and loop pipeline (inside that CGSCC, per function).** `sroa`, `early-cse` with MemorySSA, `speculative-execution`, `jump-threading`, `correlated-propagation`, `simplifycfg`, `instcombine`, `aggressive-instcombine`, `libcalls-shrinkwrap`, `tailcallelim`, `simplifycfg`, `reassociate`, `constraint-elimination`, then a MemorySSA loop nest: `loop-instsimplify`, `loop-simplifycfg`, `licm` (no speculation), `loop-rotate`, `licm` (speculation allowed), trivial `simple-loop-unswitch`. Then `simplifycfg`, `instcombine`, then a loop nest: `loop-idiom`, `indvars`, `extra-simple-loop-unswitch-passes`, `loop-deletion`, `loop-unroll-full`. Then `sroa`, `vector-combine`, `mldst-motion`, `gvn`, `sccp`, `bdce`, `instcombine`, `jump-threading`, `correlated-propagation`, `adce`, `memcpyopt`, `dse`, `move-auto-init`, `licm`, `coro-elide`, `simplifycfg`, `instcombine`.
   Required as a sequence, not a bag of passes. Threading and reassociation run before the loop nest so LICM and unswitch see a cleaner CFG. Rotation sits between the two LICM runs because the second one wants a rotated latch. Idiom recognition, indvars, deletion, and full unroll come after rotation because they need a canonical induction variable and a trip count. GVN, SCCP, and DSE come after the loops because the loops are what create the redundant loads and the dead stores. Each InstCombine is required where it sits: it canonicalizes whatever the pass immediately before it just produced.

8. **After the call graph has been inlined.** `deadargelim`, `coro-cleanup`, `globalopt`, `globaldce`, `elim-avail-extern`, `rpo-function-attrs`.
   Required only once inlining has finished changing who calls whom. An argument that became unused, or an internal function that no longer has a caller, cannot be removed earlier without breaking a call the inliner had not copied yet.

9. **Vectorize, then clean up.** `float2int`, `lower-constant-intrinsics`, `loop-rotate`, `loop-deletion`, `loop-distribute`, `loop-vectorize`, `loop-load-elim`, `instcombine`, `simplifycfg`, `slp-vectorizer`, `vector-combine`, `instcombine`, `loop-unroll`, `sroa`, `instcombine`, `licm`, `alignment-from-assumptions`, `loop-sink`, `instsimplify`, `div-rem-pairs`, `tailcallelim`, `simplifycfg`.
   Required late because vectorization wants the scalar loop passes to have already deleted, rotated, and canonicalized the loop. Distribution is immediately before the vectorizer so a loop the vectorizer would reject can be split first. SLP is after the loop vectorizer so it sees the scalar remainder. Unroll is after both, so it unrolls what was not vectorized. `div-rem-pairs` and the last `tailcallelim` are last because unrolling is what creates the div/rem pairs and the new tail calls.

10. **Module epilogue.** `globaldce`, `constmerge`, `cg-profile`, `rel-lookup-table-converter`, `annotation-remarks`, `verify`.
    Required so the bitcode handed to codegen does not still contain functions step 9 made unused, and so identical constants are one object. `verify` is the check that the pipeline did not break the IR.

`gvn` in step 7 is the pass in section 11. `newgvn` is a different pass and is not in this pipeline.

### What changes at `-O1`, `-O3`, `-Os`, and `-Oz`

`-O1` is a shorter pipeline, not the same passes with a lower cost. `-O3` adds a few passes and turns nontrivial unswitching on. `-Os` follows `-O2` and drops passes that grow code. `-Oz` follows `-Os` and also refuses header duplication and ordinary vectorization.

| | `-O1` | `-O2` | `-O3` | `-Os` | `-Oz` |
| --- | --- | --- | --- | --- | --- |
| Jump threading, CVP, GVN, DSE, constraint elimination, aggressive InstCombine | no | yes | yes | yes | yes |
| Early `tailcallelim` (before the loop passes) | no | yes | yes | yes | yes |
| Late `tailcallelim` (after `div-rem-pairs`) | yes | yes | yes | yes | yes |
| `libcalls-shrinkwrap` | yes | yes | yes | no | no |
| `openmp-opt-cgscc` | no | yes | yes | no | no |
| Loop unswitch | trivial only | trivial, plus a later extra pass | trivial and nontrivial | trivial, plus the extra pass | trivial, plus the extra pass |
| `loop-rotate` | duplicates the header | duplicates the header | duplicates the header | duplicates the header | does not duplicate the header |
| `loop-vectorize` | only loops marked to force it | yes | yes | yes | only loops marked to force it |
| `slp-vectorizer` | no | yes | yes | yes | no |
| Runtime `loop-unroll` threshold | `O1` | `O2` | `O3` | `O2` | `O2` |
| `argpromotion` | no | no | yes, inside the CGSCC | no | no |
| `callsite-splitting` | no | no | yes, in the early function pipeline | no | no |
| `chr` (control-height reduction) | no | no | yes, just before vectorization | no | no |

Full unroll of a tiny constant-trip loop (`loop-unroll-full`) still runs at every level, inside the loop nest in step 7. The level changes the later runtime unroll, not that full unroll.

These are real passes, and they are not in `default<O1>` through `default<Oz>` on this LLVM. You add them by name: `sink` (the pipeline runs `loop-sink` instead), `newgvn`, `loop-interchange`, `loop-fusion`, `loop-unroll-and-jam`, `irce`, `loop-predication`, `hotcoldsplit`, `partial-inliner`, `load-store-vectorizer`. `dce` as its own pass is also absent; trivial dead instructions are deleted inside other passes, and the pipeline's dead-code pass is `adce`.

---

## 1. SROA (Scalar Replacement of Aggregates)
**Why:** Clang emits one `alloca` for a whole struct or array. InstCombine and mem2reg do not split that object. SROA breaks it into scalar slots so the rest of the pipeline can promote them. It runs before the inliner and again after, because inlining brings in new allocas.

**Full Name:** `SROAPass`
**What it does:** Breaks down aggregate types (like structs or arrays) allocated on the stack (via `alloca`) into individual scalar variables if accessed directly. This is crucial for allowing other passes to put those variables into registers.

**Example:**
*Before SROA (C-level conceptual):*
```c
struct Point { int x; int y; };
void foo() {
    struct Point p;  // alloca'd on the stack
    p.x = 10;
    p.y = 20;
    return p.x + p.y;
}
```
*After SROA (C-level conceptual):*
```c
void foo() {
    int p_x = 10;    // Now just a scalar integer, can live in a register
    int p_y = 20;    // Now just a scalar integer
    return p_x + p_y;
}
```

---

## 2. Mem2Reg (Promote Memory to Register)
**Why:** A value that lives in an `alloca` is invisible to InstCombine, GVN, and LICM: they see a load and a store, not the add that produced it. Mem2reg rebuilds SSA so those passes can work. It runs immediately after the early SROA and simplifycfg.

**Full Name:** `PromotePass`
**What it does:** Converts `alloca` instructions (stack memory) into SSA registers (`%var`) by inserting Phi nodes. This pass requires a Dominator Tree to place Phi nodes at join points in the control flow graph. It is usually run immediately after SROA to lift the broken-down scalars into registers.

**Example:**
*Before Mem2Reg (Unoptimized IR):*
```llvm
entry:
  %x = alloca i32, align 4
  store i32 5, ptr %x, align 4
  %val = load i32, ptr %x, align 4
  ret i32 %val
```
*After Mem2Reg:*
```llvm
entry:
  ; The alloca, load, and store are completely removed
  ret i32 5
```

---

## 3. EarlyCSE (Early Common Subexpression Elimination)
**Why:** The same add, or the same load, is often repeated a few instructions apart. GVN can remove it too, but GVN is heavier and runs much later. EarlyCSE is the cheap first pass so later passes do not keep reanalyzing duplicates. Inside the inliner pipeline it runs again with MemorySSA, which can forward a load across a call that does not alias.

**Full Name:** `EarlyCSEPass`
**What it does:** A fast, hash-based pass that searches for redundant computations (expressions calculated multiple times with the exact same inputs) and eliminates all but the first one, replacing later uses with the first computed value.

**Example:**
*Before EarlyCSE:*
```llvm
  %a = add i32 %x, %y
  %b = add i32 %x, %y  ; Exact same computation!
  %c = mul i32 %a, %b
```
*After EarlyCSE:*
```llvm
  %a = add i32 %x, %y
  ; %b is deleted
  %c = mul i32 %a, %a  ; Reuses %a
```

---

## 4. InstCombine (Instruction Combining)
**Why:** Every other pass leaves IR that is correct but not in the one spelling later patterns match (`x+0`, `mul` by 2, a compare that is already known). InstCombine is that canonical cleanup, which is why it is repeated after inlining, after the loop passes, and after vectorization. One run is not enough: each of those passes creates new patterns.

**Full Name:** `InstCombinePass`
**What it does:** A massive peephole optimizer that pattern-matches combinations of instructions and replaces them with simpler, more efficient instructions (e.g., algebraic simplifications, bitwise tricks, canonicalizing IR).

**Example:**
*Before InstCombine:*
```llvm
  %a = add i32 %x, 0      ; x + 0 is just x
  %b = mul i32 %a, 2      ; a * 2 is a shift left by 1
```
*After InstCombine:*
```llvm
  %b = shl i32 %x, 1
```

---

## 5. SimplifyCFG (Simplify Control Flow Graph)
**Why:** Passes delete instructions and fold conditions, and they leave empty blocks and branches to the next block. Those blocks hide a preheader from LICM and waste every later CFG walk. SimplifyCFG puts the CFG back. It runs at the start, after inlining, and at the end, with a stricter bonus threshold than a single cleanup would use.

**Full Name:** `SimplifyCFGPass`
**What it does:** Cleans up the flow of execution. It merges basic blocks that have unconditional branches to each other, removes unreachable blocks, and converts simple conditional branches into `select` instructions to avoid branch prediction penalties.

**Example:**
*Before SimplifyCFG:*
```llvm
entry:
  br label %next
next:
  %x = add i32 %a, %b
  ret i32 %x
```
*After SimplifyCFG:*
```llvm
entry:
  %x = add i32 %a, %b
  ret i32 %x
```

---

## 6. Jump Threading
**Why:** A condition is often already known on one incoming edge, but the branch still tests it. The dead side then survives and blocks deletion and unswitching. Threading rewrites that edge so the dead side has no predecessor. It is in the `-O2` function pipeline, twice: once before the loop passes and once after GVN. `-O1` does not run it.

**Full Name:** `JumpThreadingPass`
**What it does:** If a block branches to another block based on a condition, and that condition was already proven true or false on a specific incoming path, Jump Threading redirects the incoming edge directly to the correct destination, bypassing the redundant check.

**Example:**
*Before Jump Threading (C-level conceptual):*
```c
  if (x == 5) {
      y = 10;
  }
  // Jump Threading realizes that if the first block ran, x IS 5.
  if (x == 5) {
      z = 20;
  }
```
*After Jump Threading:*
The compiler threads the execution path so that if the first block runs, it jumps directly to `z = 20` without evaluating `x == 5` a second time.

---

## 7. LICM (Loop Invariant Code Motion)
**Why:** An add of two loop-invariant values inside a loop runs once per iteration and does not need to. Hoisting it to the preheader runs it once. LICM is in the loop pipeline twice at `-O2`: before rotation, without speculating, and after rotation, with speculation allowed. Rotation is what gives the second run a latch it can reason about.

**Full Name:** `LICMPass`
**What it does:** Identifies computations inside a loop that produce the exact same result on every iteration. It "hoists" these computations out into the loop's preheader so they are executed only once.

**Example:**
*Before LICM:*
```c
for (int i=0; i<100; i++) {
    int max = x + y; // x and y do not change in this loop!
    arr[i] = max;
}
```
*After LICM:*
```c
int max = x + y;     // Hoisted out of the loop
for (int i=0; i<100; i++) {
    arr[i] = max;
}
```

---

## 8. Loop Unrolling
**Why:** A loop's backedge, compare, and induction update are real instructions, and a rolled body hides cross-iteration folds from InstCombine and SLP. Full unroll of a tiny constant trip count runs inside the loop nest at every optimization level. A second, threshold-limited unroll runs after vectorization; the threshold is `loop-unroll<O1>`, `<O2>`, or `<O3>` depending on the level.

**Full Name:** `LoopUnrollPass`
**What it does:** Duplicates the body of a loop multiple times to reduce the overhead of the branch/counter instructions and to expose instructions across iterations to passes like InstCombine and SLP Vectorization.

**Example:**
*Before Loop Unrolling:*
```c
for(int i=0; i<4; i++) {
    a[i] = 0;
}
```
*After Loop Unrolling (Fully Unrolled):*
```c
// The loop structure is completely removed
a[0] = 0;
a[1] = 0;
a[2] = 0;
a[3] = 0;
```

---

## 9. Loop Deletion
**Why:** A loop that computes nothing still occupies a branch and a header, and it blocks SimplifyCFG from joining the blocks around it. Deletion removes that. It also implements `mustprogress`: an infinite side-effect-free loop in such a function may be removed. It runs inside the loop nest, after indvars has made the trip count obvious, and again late, after another rotation.

**Full Name:** `LoopDeletionPass`
**What it does:** Removes a loop that cannot affect observable behavior. A finite loop whose body has no side effects goes away. An infinite loop with no side effects is removed only when the function is `mustprogress`; without that attribute the loop is kept, because deleting it would let the code after the loop run.

**Example:**
*Before Loop Deletion:*
```c
for (int i=0; i<1000; i++) {
    int x = i * 2;
    // x is never used outside, no memory is written
}
```
*After Loop Deletion:*
The entire loop is deleted by the compiler, leaving nothing.

---

## 10. Inliner
**Why:** A call is a barrier: the caller cannot see the callee's loads, and the callee cannot see the caller's constants. Inlining copies the body so SROA, InstCombine, and IPSCCP's results apply across the call. The cost-model inliner runs bottom-up on the call-graph SCC, after `always-inline` has already copied functions the frontend marked as mandatory.

**Full Name:** `InlinerPass`
**What it does:** Replaces a function call with the actual body of the called function. This eliminates function call overhead (saving registers, pushing stack frames) and exposes the function's internal logic to the caller's optimization passes.

**Example:**
*Before Inlining:*
```c
int add(int a, int b) { return a + b; }

int main() {
    return add(5, 5);
}
```
*After Inlining:*
```c
int main() {
    return 5 + 5; // Will be immediately constant folded to 10
}
```

---

## 11. GVN (Global Value Numbering)
**Why:** EarlyCSE only hashes a straight-line scope. Values that are equal across a branch, and loads that are redundant across a block, wait for GVN. It sits after the loop passes have already simplified induction variables and deleted dead loops, so it numbers the IR those passes produced. `-O1` does not run it; `-O2` and above do (`gvn`, not `newgvn`).

**Full Name:** `GVNPass`
**What it does:** Assigns a unique "value number" to expressions. If two instructions compute the same value number, the second is deleted. It handles redundant memory loads heavily using Memory Dependence Analysis, which EarlyCSE cannot do.

**Example:**
*Before GVN:*
```llvm
  %v1 = load i32, ptr %p
  ; ... some complex code that doesn't write to %p ...
  %v2 = load i32, ptr %p  ; Redundant load!
  %sum = add i32 %v1, %v2
```
*After GVN:*
```llvm
  %v1 = load i32, ptr %p
  ; ... some complex code ...
  %sum = add i32 %v1, %v1 ; Second load is eliminated
```

---

## 12. SCCP (Sparse Conditional Constant Propagation)
**Why:** After inlining, many arguments are constants and many branches are constant, but the dead arm is still in the function. SCCP folds the constants and refuses to walk the dead arm, so later passes do not spend compile time there. The interprocedural version, IPSCCP, runs earlier, before the inliner; this pass is the function-local run after GVN.

**Full Name:** `SCCPPass`
**What it does:** Propagates constants through the program. If it proves a branch condition is always false based on constant propagation, it ignores the dead block completely, never propagating values through it.

**Example:**
*Before SCCP:*
```c
int x = 5;
if (x > 10) {
    // SCCP proves 5 > 10 is false.
    complex_math(); 
}
return x;
```
*After SCCP:*
```c
return 5; // The if-statement and complex math are completely deleted
```

---

## 13. Correlated Value Propagation (CVP)
**Why:** SCCP's lattice is a single constant. A branch `x > 100` proves `x > 0` on that path, and that fact is a range, not a constant. CVP is what consumes the range, so a redundant bounds check can go away. It runs next to jump threading at `-O2` and above, and not at `-O1`.

**Full Name:** `CorrelatedValuePropagationPass`
**What it does:** Uses Lazy Value Info (LVI) to track ranges of variables based on previous branch conditions. It can eliminate switch cases, bounds checks, or redundant conditional branches.

**Example:**
*Before CVP:*
```c
if (x > 100) {
    if (x > 0) { // This must be true if x > 100!
        do_something();
    }
}
```
*After CVP:*
```c
if (x > 100) {
    do_something(); // The redundant check is eliminated
}
```

---

## 14. ADCE (Aggressive Dead Code Elimination)
**Why:** Folding a branch often makes an entire region control-dependent on nothing the function returns or stores. DCE deletes an instruction with no uses; it will not delete the branch that selects between two empty paths. ADCE starts from live side effects and deletes everything that cannot reach one. It is the dead-code pass the default pipeline actually schedules; a separate `dce` pass is not in `default<O1>` through `default<Oz>`.

**Full Name:** `ADCEPass`
**What it does:** Removes instructions that do not contribute to the final observable output of the function. Works backwards from returns/stores, marking "live" instructions. Deletes everything unmarked.

**Example:**
*Before ADCE:*
```llvm
entry:
  br i1 %c, label %t, label %f
t:
  %dead = add i32 %a, 1    ; nothing live depends on this, or on the branch
  br label %join
f:
  br label %join
join:
  ret i32 0
```
*After ADCE:*
```llvm
entry:
  ret i32 0
```
The unused `add` alone is DCE's job (the next section). ADCE is what proves the branch itself is dead, because neither successor reaches a store, a return of a computed value, or any other live side effect.

---

## 15. GlobalOpt (Global Optimizer)
**Why:** A global that is stored once, with a constant, is still a load at every use until something promotes it. GlobalOpt does that promotion at module scope, so IPSCCP and InstCombine see a constant instead of a memory access. It runs before the inliner and again after, once dead functions have been removed.

**Full Name:** `GlobalOptPass`
**What it does:** Optimizes global variables. If a global is only ever written to once, it makes it a constant. If a global is never used, it deletes it. It can also localize globals to functions if they aren't accessed externally.

**Example:**
*Before GlobalOpt:*
```c
int G = 0; // Global variable
void init() { G = 5; }
int read() { return G; }
```
*After GlobalOpt:*
The compiler may realize `G` is only ever 5 after initialization, turning it into a constant and just returning `5` from `read()`.

---

## 16. Loop Vectorizer
**Why:** A scalar loop uses one lane of a vector register and pays the loop overhead once per element. The vectorizer is the main thing `-O2` adds over a forced-only `-O1`: at `-O1` and `-Oz` it runs only when the loop is marked to force vectorization, and at `-O2`, `-O3`, and `-Os` it runs on any loop its cost model accepts. It is late, after loop distribution, so a loop that had to be split is already split.

**Full Name:** `LoopVectorizePass`
**What it does:** Analyzes loops for data dependencies. If mathematically safe, it converts scalar instructions into wide vector instructions (SIMD like AVX or NEON) to process multiple loop iterations at the exact same time.

**Example:**
*Before Loop Vectorizer:*
```c
for(int i=0; i<4; i++) {
    a[i] = b[i] + c[i];
}
```
*After Loop Vectorizer (Conceptual):*
```llvm
  ; Loads 4 elements of b and c at once into a 128-bit vector register
  %vec_b = load <4 x i32>, ptr %b
  %vec_c = load <4 x i32>, ptr %c
  ; Does a single vector addition!
  %vec_sum = add <4 x i32> %vec_b, %vec_c
  ; Stores 4 elements to a
  store <4 x i32> %vec_sum, ptr %a
```

---

## 17. SLP Vectorizer (Superword-Level Parallelism)
**Why:** Not every parallel group is a loop. Adjacent scalar adds and loads in one block will not be seen by the loop vectorizer. SLP packs those after loop vectorization, so it also sees the scalar remainder the loop vectorizer left behind. `-O1` and `-Oz` skip it; code size is the reason at `-Oz`.

**Full Name:** `SLPVectorizerPass`
**What it does:** Looks for straight-line scalar code that performs identical operations on adjacent memory locations, and bundles them into a single vector operation.

**Example:**
*Before SLP Vectorizer:*
```c
a.x = b.x + c.x;
a.y = b.y + c.y;
```
*After SLP Vectorizer (Conceptual):*
The compiler bundles `x` and `y` into a vector `[x, y]` and performs a single vector addition:
```llvm
  %vec_b = load <2 x i32> ...
  %vec_c = load <2 x i32> ...
  %vec_a = add <2 x i32> %vec_b, %vec_c
```

---

The entries below use the same before/after sketches. **Pass** is the name you give `opt -passes=`. The sketch is the transform, not a promise that one run of `opt` prints identical names: thresholds, target costs, and surrounding attributes change what fires. Each entry points at the day that walks the source.

## 18. DCE (Dead Code Elimination)
**Why:** Instructions with no uses still take space in every analysis. Deleting them is required for compile time as much as for the generated code. Other passes do this internally as they go. The standalone `dce` pass is what you run when you want only that deletion; the default pipelines schedule `adce` instead, because by then the interesting deadness is control-dependent.

**Pass:** `dce`
**Full Name:** `DCEPass`
**Day:** [Day 7](Day-07-0-DCE-and-ADCE.md)
**What it does:** Deletes an instruction that has no uses and no side effect. It does not delete stores, and it does not delete branches. That is the difference from ADCE.

**Example:**
*Before DCE:*
```llvm
  %dead = mul i32 %x, 500
  %live = add i32 %x, 1
  ret i32 %live
```
*After DCE:*
```llvm
  %live = add i32 %x, 1
  ret i32 %live
```

---

## 19. Reassociate
**Why:** InstCombine and EarlyCSE match one shape of an expression. `(a+1)+b` and `(a+b)+1` are the same arithmetic and different IR, so CSE misses them. Reassociation writes one order, constants at the end, before the loop passes and the second InstCombine. It drops `nsw` on purpose: a reassociated add must not invent overflow poison.

**Pass:** `reassociate`
**Full Name:** `ReassociatePass`
**Day:** [Day 9](Day-09-0-Reassociate.md)
**What it does:** Rewrites associative chains into one order, with constants at the right, so later CSE and folding see the same expression. Integer reassociation drops `nsw`: the new form must not invent overflow poison.

**Example:**
*Before:*
```llvm
  %t = add i32 %a, 1
  %s = add i32 %t, %b
```
*After:*
```llvm
  %t = add i32 %a, %b
  %s = add i32 %t, 1
```

---

## 20. BDCE (Bit-Tracking Dead Code Elimination)
**Why:** InstCombine deletes an instruction whose value is unused. It does not delete an instruction whose value is used only in bits that a later `and` throws away. BDCE tracks demanded bits so those instructions go away before codegen emits them. It runs after SCCP in the function pipeline, at every level including `-O1`.

**Pass:** `bdce`
**Full Name:** `BDCEPass`
**Day:** [Day 13](Day-13-0-BDCE.md)
**What it does:** Tracks which bits of a value any root (a return, a store, a branch) actually reads. An instruction whose bits are demanded by nobody is deleted, even when a use exists, if that use also ignores the bits.

**Example:**
*Before:*
```llvm
  %a = or i32 %x, 1
  %b = and i32 %a, -2     ; clears bit 0, so the `or 1` did nothing
  ret i32 %b
```
*After:*
```llvm
  %b = and i32 %x, -2
  ret i32 %b
```

---

## 21. Tail Call Elimination
**Why:** A self-recursive call that is really a loop grows the stack by one frame per iteration and is invisible to LICM and the vectorizer. Turning it into a backedge makes it a loop those passes can handle. `-O2` and above run this once before the loop pipeline and once more at the very end, after unrolling and `div-rem-pairs` have created new tail calls. `-O1` runs only the late one.

**Pass:** `tailcallelim`
**Full Name:** `TailCallElimPass`
**Day:** [Day 15](Day-15-0-TailCallElim.md)
**What it does:** Turns a self-recursive call in tail position into a loop, so the stack does not grow. A call is in tail position only when its result is returned with nothing left to do. The factorial multiply has to become an accumulator argument before the call is in tail position.

**Example:**
*Before (the call is not yet tail, because of the multiply):*
```llvm
  %r = call i32 @fact(i32 %n1)
  %m = mul i32 %r, %n
  ret i32 %m
```
*After the accumulator form, the recursive call is a branch back to the header, not a `call`.*

---

## 22. Sink
**Why:** An instruction in the entry block that is used only on a cold path keeps its operands live across the hot path and costs a register. Sinking moves it to the cold block. The general `sink` pass is not in `default<O2>`; the pipeline's version is `loop-sink`, late, after vectorization and unrolling, when live ranges are at their worst.

**Pass:** `sink`
**Full Name:** `SinkingPass`
**Day:** [Day 17](Day-17-0-Sink.md)
**What it does:** Moves an instruction from a block into a successor that actually uses it, when the operands are available there. The value then lives for a shorter stretch.

**Example:**
*Before:*
```llvm
entry:
  %s = add i32 %a, %b      ; used only in %cold
  br i1 %c, label %hot, label %cold
cold:
  ret i32 %s
```
*After:*
```llvm
entry:
  br i1 %c, label %hot, label %cold
cold:
  %s = add i32 %a, %b
  ret i32 %s
```

---

## 23. Constraint Elimination
**Why:** A loop often rechecks a condition the code before the loop already established. LICM cannot hoist a check that is not invariant, and the vectorizer will refuse a loop that still contains it. Constraint elimination folds the check using the inequalities that dominate it. It runs after reassociation and before the loop nest, at `-O2` and above, not at `-O1`.

**Pass:** `constraint-elimination`
**Full Name:** `ConstraintEliminationPass`
**Day:** [Day 19](Day-19-0-ConstraintElimination.md)
**What it does:** Records the inequalities a dominating branch established, and folds a later compare that those inequalities already prove.

**Example:**
*Before, inside the block reached when `%x` is unsigned-below `%n`:*
```llvm
  %c1 = icmp ult i32 %x, %n
  br i1 %c1, label %in, label %out
in:
  %c2 = icmp uge i32 %x, %n    ; contradicts %c1
  br i1 %c2, label %never, label %ok
```
*After:* the compare in `%in` is the constant `false`, and the `%never` edge can be deleted by SimplifyCFG.

---

## 24. Div/Rem Pairs
**Why:** A `udiv` and a `urem` of the same values are two IR instructions and, on targets with a combined divide, one machine instruction. If they sit in different blocks, selection will not see that. This pass moves them together. It is last among the arithmetic cleanups, after unrolling, because unrolling is what often duplicates a div/rem pair into the same region.

**Pass:** `div-rem-pairs`
**Full Name:** `DivRemPairsPass`
**Day:** [Day 20](Day-20-0-DivRemPairs.md)
**What it does:** When the target can produce a quotient and a remainder from one divide (`TTI::hasDivRemOp`), moves a matching `udiv`/`urem` or `sdiv`/`srem` into the same block so instruction selection emits one divide.

**Example:**
*Before:* the divide and the remainder sit in different blocks.
*After:* both are in one block. SelectionDAG then builds one `UDIVREM` or `SDIVREM` node. If the target has no combined divide, the pair stays split.

---

## 25. Loop Rotate
**Why:** LICM, indvars, and the vectorizer are written for a rotated loop: the first iteration's test is a guard, and the latch is a backedge out of the body. Clang emits while-loops the other way around. Rotation runs inside the loop nest, between the two LICM runs, so the second LICM sees the rotated form. At `-Oz` the pass is built with `no-header-duplication`, because copying the header grows code.

**Pass:** `loop-rotate`
**Full Name:** `LoopRotatePass`
**Day:** [Day 26](Day-26-0-LoopRotate.md)
**What it does:** Turns a while-loop into a guarded do-while. The condition is checked once before the loop, and again at the latch. Induction-variable simplification and the vectorizer want that shape.

**Example:**
*Before:* the header tests `%i < %n` and only then runs the body.
*After:* a guard in the preheader skips the loop when the trip count is zero, and the latch branches back on a copy of the test. The body is no longer behind the header test.

---

## 26. Loop Idiom Recognition
**Why:** A loop that only zeroes memory is larger and slower than `llvm.memset`, and it is a poor input to the vectorizer. Recognizing it here, inside the loop nest after rotation, deletes the loop before indvars and the vectorizer spend work on it. The memset is also what DSE and MemCpyOpt already know how to reason about.

**Pass:** `loop-idiom`
**Full Name:** `LoopIdiomRecognizePass`
**Day:** [Day 27](Day-27-0-LoopIdiom.md)
**What it does:** Replaces a loop that is really a memory primitive with `llvm.memset` or `llvm.memcpy`.

**Example:**
*Before:*
```c
for (int i = 0; i < n; ++i)
  a[i] = 0;
```
*After:*
```llvm
  call void @llvm.memset.p0.i64(ptr %a, i8 0, i64 %bytes, i1 false)
```
A loop that stores a non-invariant value is not a memset and is left alone.

---

## 27. Simple Loop Unswitch
**Why:** An invariant branch inside a loop stops the vectorizer and keeps LICM from moving the instructions on each side. Unswitching lifts the branch and duplicates the loop so each copy is straight-line. Trivial unswitch (one side is empty, or the cost is tiny) runs at every level. Nontrivial unswitch, which duplicates a real body, is an `-O3` choice (`simple-loop-unswitch<nontrivial>`). `-O2` uses a separate extra unswitch pass later in the same loop nest instead.

**Pass:** `simple-loop-unswitch`
**Full Name:** `SimpleLoopUnswitchPass`
**Day:** [Day 28](Day-28-0-SimpleLoopUnswitch.md)
**What it does:** Hoists a branch whose condition does not change inside the loop, and duplicates the loop into the two arms. Each copy then has straight-line control where the branch used to be.

**Example:**
*Before:*
```c
for (int i = 0; i < n; ++i)
  if (flag)
    a[i] = 1;
  else
    a[i] = 2;
```
*After:* one loop that stores `1`, and one loop that stores `2`, with `if (flag)` outside both. The pass refuses when copying the body would exceed its size threshold.

---

## 28. IndVarSimplify
**Why:** The vectorizer, unroll, and deletion all ask SCEV for a trip count, and a mess of several induction variables makes that count fail or look unprofitable. Indvars rewrites them onto one canonical IV first. It runs in the loop nest after idiom recognition and before deletion and full unroll, so those three see the rewritten form.

**Pass:** `indvars`
**Full Name:** `IndVarSimplifyPass`
**Day:** [Day 29](Day-29-0-IndVarSimplify.md)
**What it does:** Rewrites induction variables onto one canonical IV, turns `i * scale` into an add recurrence, and widens a narrow IV when the extra bits cannot change the result.

**Example:**
*Before:* a 32-bit phi stepping by 1, used only as a GEP index on a 64-bit target.
*After:* a 64-bit phi, so the GEP no longer sign-extends on every iteration. A wrapping IV stays narrow.

---

## 29. Loop Interchange
**Why:** A nest that walks the wrong index streams memory with a stride, and the vectorizer's cost model then rejects it or emits gathers. Interchange swaps the loops when the dependences allow it, so the inner loop is contiguous. It is not in `default<O1>` through `default<Oz>` on LLVM 22.1.4. You run it with `-passes=loop-interchange` when you want it.

**Pass:** `loop-interchange`
**Full Name:** `LoopInterchangePass`
**Day:** [Day 30](Day-30-0-LoopInterchange.md)
**What it does:** Swaps two tightly nested loops so the inner one walks memory with unit stride, when dependence analysis says the swap is legal.

**Example:**
*Before:* the inner index is the column of a row-major array.
*After:* the inner index is the row, and consecutive iterations touch consecutive addresses.

---

## 30. Dead Store Elimination
**Why:** A store whose bytes are overwritten before any read still becomes a stack slot and a memory operation in codegen. GVN does not delete stores. DSE does, after MemCpyOpt has already turned copy loops into `memcpy`, so a memcpy to a dead buffer can be removed too. `-O2` and above run it; `-O1` does not.

**Pass:** `dse`
**Full Name:** `DSEPass`
**Day:** [Day 106](Day-106-0-DSE.md)
**What it does:** Deletes a store whose bytes are overwritten before any read, using MemorySSA. It does not delete volatile or atomic stores.

**Example:**
*Before:*
```llvm
  store i32 1, ptr %p
  store i32 2, ptr %p
  ret void
```
*After:*
```llvm
  store i32 2, ptr %p
  ret void
```
A load of `%p` between the two stores keeps the first store.

---

## 31. MemCpyOpt
**Why:** A field-by-field copy is a long chain of loads and stores. Codegen will not turn that chain into the target's memcpy, and DSE has a harder time seeing that the destination is dead. MemCpyOpt does the recognition. `-O1` runs it inside the function pipeline; `-O2` runs it next to DSE, after ADCE has cleared unrelated dead code.

**Pass:** `memcpyopt`
**Full Name:** `MemCpyOptPass`
**Day:** [Day 107](Day-107-0-MemCpyOpt.md)
**What it does:** Turns a sequence of loads and stores that copy bytes into `llvm.memcpy` or `llvm.memset`, and deletes a copy whose destination is never observed. The source file is `MemCpyOptimizer.cpp`.

**Example:**
*Before:* two integer loads from `%src` and two stores to `%dst`, covering the whole object.
*After:* one `llvm.memcpy` of that size. If `%dst` is an alloca that dies without being read, the copy can disappear entirely.

---

## 32. IPSCCP (Interprocedural SCCP)
**Why:** The inliner's cost model and InstCombine both want to know that an argument is the constant 7 before they run. IPSCCP proves that at module scope, before `always-inline` and the CGSCC inliner. On this LLVM its default options also specialize: a callee that is only sometimes passed a constant is cloned, so the constant caller can be optimized without forcing the other caller onto the same body. Every level runs `ipsccp`.

**Pass:** `ipsccp`
**Full Name:** `IPSCCPPass`
**Day:** [Day 43](Day-43-0-IPSCCP.md), specialization in [Day 110](Day-110-0-Specialization-and-Splitting.md)
**What it does:** SCCP across the call graph. A parameter that is the same constant at every call becomes that constant in the callee. With function specialization enabled (`ipsccp<func-spec>`, the default in the `-O3` pipeline on current LLVM), a callee that is *sometimes* passed a constant is cloned, and only the clone sees the constant.

**Example:**
*Before:*
```llvm
  call void @callee(i32 7)
define void @callee(i32 %x) {
  %y = mul i32 %x, 2
  ret void
}
```
*After, when this is the only call:* the multiply is `mul i32 7, 2`, and InstCombine folds it to 14. Specialization is the case with two callers, one passing `7` and one passing a runtime value: IPSCCP clones `@callee` for the constant caller instead of giving up.

---

## 33. Dead Argument Elimination
**Why:** After inlining, an internal function often has parameters nothing reads. They still occupy registers and stack at the call. Dead-argument elimination rewrites the signature once the CGSCC pipeline has finished changing who calls whom. It runs at every level, after `coro-cleanup` and before the second `globalopt`. An external function keeps the argument; the ABI is the reason.

**Pass:** `deadargelim`
**Full Name:** `DeadArgumentEliminationPass`
**Day:** [Day 44](Day-44-0-DeadArgElim.md)
**What it does:** Removes a parameter the function never reads, and a return value no caller reads, when every call site can be rewritten. An externally visible function keeps its ABI; the unused argument stays.

**Example:**
*Before, `@f` is `internal`:*
```llvm
define internal i32 @f(i32 %a, i32 %b) {
  ret i32 %a
}
```
*After:*
```llvm
define internal i32 @f(i32 %a) {
  ret i32 %a
}
```
The call drops the second argument. If `@f` were `external`, both arguments would remain.

---

## 34. Argument Promotion
**Why:** A pointer argument that the callee only loads is an unnecessary memory dependency in the caller and an unnecessary register of pointer type. Promoting it to the scalar lets the caller fold the load and lets InstCombine see the value. That costs compile time and can grow code, so only `-O3` runs `argpromotion`, inside the CGSCC pipeline.

**Pass:** `argpromotion`
**Full Name:** `ArgumentPromotionPass`
**Day:** [Day 45](Day-45-0-ArgumentPromotion.md)
**What it does:** Rewrites an internal function's pointer argument into the scalar it loads, and moves that load to the caller. The pointer must not escape, and the memory must not change between the call and the load.

**Example:**
*Before:*
```llvm
define internal i32 @f(ptr %p) {
  %v = load i32, ptr %p
  ret i32 %v
}
```
*After:*
```llvm
  %v = load i32, ptr %obj
  %r = call i32 @f(i32 %v)
define internal i32 @f(i32 %v) {
  ret i32 %v
}
```

---

## 35. GlobalDCE
**Why:** Inlining and dead-argument elimination leave internal functions and globals with no remaining users. Emitting them costs binary size and, for a large internal function, compile time in the backend. GlobalDCE removes them. It runs after the inliner and again at the epilogue, so anything the late function passes made unused is also removed.

**Pass:** `globaldce`
**Full Name:** `GlobalDCEPass`
**Day:** [Day 46](Day-46-0-GlobalDCE.md)
**What it does:** Deletes functions and globals that nothing live can reach. Externally visible symbols are roots. An `internal` function with no callers is deleted even if its body contains stores.

**Example:**
*Before:* `@main` calls `@used`. `@unused` is `internal` and has no callers.
*After:* `@unused` is gone. `@main` remains, because another module might call it.

---

## 36. Loop Predication
**Why:** A bounds check inside a loop prevents vectorization even when the trip count already proves the check. Predication lifts that check to a loop-invariant test so the vector body can drop it. It is not in `default<O2>` on LLVM 22.1.4. You add `-passes=loop-predication`. IRCE, next, is a different answer to the same problem.

**Pass:** `loop-predication`
**Full Name:** `LoopPredicationPass`
**Day:** [Day 109](Day-109-0-LoopPredication-IRCE.md)
**What it does:** Pulls a range check that SCEV can relate to the induction variable out to a loop-invariant predicate, often so an implicit guard runs once instead of every iteration.

**Example:**
*Before:* each iteration checks `i < length` before `a[i]`.
*After:* one check before the loop covers the whole trip, and the in-loop check is gone. The pass does nothing when the trip count is not an affine function of the same induction variable.

---

## 37. IRCE (Inductive Range Check Elimination)
**Why:** Some range checks cannot be lifted as a single invariant predicate, but they can be removed from the middle of the trip by running a checked preloop and postloop around an unchecked main loop. That split is what makes the main loop vectorizable. IRCE is not in the default pipelines; `-passes=irce` adds it.

**Pass:** `irce`
**Full Name:** `IRCEPass`
**Day:** [Day 109](Day-109-0-LoopPredication-IRCE.md)
**What it does:** Splits a loop into a preloop, a main loop, and a postloop so the main loop can drop a range check. This is a different pass from loop predication: the check stays on the pre and post loops, and disappears only from the middle.

**Example:**
*Before:* one loop, `if (i < n) use(a[i]);` inside.
*After:* a short preloop that still checks, a main loop with no check, and a short postloop that still checks.

---

## 38. Hot/Cold Splitting
**Why:** A large cold error path in an otherwise small function blows the instruction cache on the hot path and defeats layout. Splitting outlines the cold region. `hotcoldsplit` is not in `default<O1>` through `default<Oz>` on this LLVM. Profiles and an explicit pass are how it gets turned on.

**Pass:** `hotcoldsplit`
**Full Name:** `HotColdSplittingPass`
**Day:** [Day 110](Day-110-0-Specialization-and-Splitting.md)
**What it does:** Outlines a cold region (a block marked cold, or reached only through a cold path) into a separate function so the hot body stays small in the instruction cache.

**Example:**
*Before:* `@handle` contains the fast path and a large error path in one function.
*After:* the error path is a call to an outlined cold function. The fast path no longer contains that code.

---

## 39. Partial Inliner
**Why:** The inliner either takes the whole callee or leaves the call. A callee whose entry is hot and whose tail is a large cold region then loses either way: inlining bloats the caller, and not inlining pays a call on the hot check. The partial inliner splits that difference. It is not in the default pipelines; `-passes=partial-inliner` adds it.

**Pass:** `partial-inliner`
**Full Name:** `PartialInlinerPass`
**Day:** [Day 110](Day-110-0-Specialization-and-Splitting.md)
**What it does:** Inlines the hot prefix of a callee and leaves the cold tail as an outlined function. The caller pays for the prefix without absorbing the tail.

**Example:**
*Before:* `@caller` calls `@parse`, and `@parse` does a small check then a large failure path.
*After:* the check is in `@caller`. The failure path is a separate function, called only when the check fails.

---

## 40. Loop Distribute
**Why:** The loop vectorizer rejects a whole loop when one statement in it has a carried dependence, even if the other statement is a clean vector add. Distribution splits them so the clean one can be vectorized. It runs immediately before `loop-vectorize` at every level, including `-O1`, which is why a forced vector loop still benefits from the split.

**Pass:** `loop-distribute`
**Full Name:** `LoopDistributePass`
**Day:** [Day 117](Day-117-0-LoopDistribute-Fuse.md)
**What it does:** Splits one loop into two when a dependence stops the vectorizer from widening the whole body, and one of the pieces is vectorizable on its own.

**Example:**
*Before:* a loop stores `a[i] = b[i] + 1` and also `c[i] = c[i - 1] + a[i]`. The second statement carries a dependence.
*After:* one loop for the independent store, which the vectorizer can widen, and one loop for the carried recurrence.

---

## 41. AggressiveInstCombine
**Why:** InstCombine refuses patterns that are expensive to match or that are not the canonical form it wants to produce. A narrow use of a wide expression is the usual miss. AggressiveInstCombine runs once, after the first InstCombine in the `-O2` function pipeline, to catch those. `-O1` does not run it.

**Pass:** `aggressive-instcombine`
**Full Name:** `AggressiveInstCombinePass`
**What it does:** Peepholes that InstCombine will not do because they are more expensive or less canonical. The usual case is narrowing a chain of wide operations when only the low bits are used, or recognizing a pattern that spans more than a couple of instructions.

**Example:**
*Before:* a sequence of shifts and masks that implements a truncate to 16 bits across several `i32` operations.
*After:* a `trunc` to `i16` (or a narrower chain) when every later use only reads those bits.

---

## 42. Lower Expect
**Why:** `__builtin_expect` is not a CPU hint. It is an intrinsic, and nothing in SimplifyCFG, unroll, or block placement reads that intrinsic. `lower-expect` turns it into `!prof` branch weights at the very start of the function pipeline, before the first SimplifyCFG, so every later pass that looks at edge frequency sees the weights.

**Pass:** `lower-expect`
**Full Name:** `LowerExpectIntrinsicPass`
**Day:** [Day 16](Day-16-0-BranchWeights.md)
**What it does:** Turns `llvm.expect` (Clang's lowering of `__builtin_expect`) into `!prof` branch-weight metadata. Block placement later lays the heavy edge out as fall-through. The intrinsic itself does not hint the CPU.

**Example:**
*Before:*
```llvm
  %c = call i1 @llvm.expect.i1(i1 %cond, i1 false)
  br i1 %c, label %cold, label %hot
```
*After:*
```llvm
  br i1 %cond, label %cold, label %hot, !prof !{!"branch_weights", i32 1, i32 2000}
```

---

## 43. Loop Simplify
**Why:** LICM has nowhere to hoist if the header has two outside predecessors. The vectorizer will not describe a loop that has two latches. Loop simplify exists so those passes do not each contain their own CFG repair. The loop pass manager runs it, together with LCSSA, on entry to every `loop` and `loop-mssa` pipeline. That is why the printed `default<O2>` string does not contain the name `loop-simplify`.

**Pass:** `loop-simplify`
**Full Name:** `LoopSimplifyPass`
**Day:** [Day 21](Day-21-0-LoopSimplify.md)
**What it does:** Gives a natural loop one preheader, one latch, and dedicated exits. LICM, indvars, and the vectorizer assume that shape. It does not change how many times the body runs.

**Example:**
*Before:* two blocks outside the loop both branch to the header, so there is no place to hoist into.
*After:* both edges go through one `%preheader` block, and an exit that was also reached from outside the loop is split so only the loop reaches the dedicated exit.

---

## 44. LCSSA
**Why:** When indvars rewrites an induction variable, every use outside the loop would otherwise have to be updated one by one. A single-input phi at the exit gives those uses one place to read. The loop pass manager inserts LCSSA before every loop pass and keeps it there. Like loop simplify, it will not appear as its own name in `-print-pipeline-passes`.

**Pass:** `lcssa`
**Full Name:** `LCSSAPass`
**Day:** [Day 22](Day-22-0-LCSSA.md)
**What it does:** Inserts a single-input phi at the loop exit for each value defined in the loop and used outside it. The phi does not change the program. Later passes edit that phi instead of hunting for uses outside the loop.

**Example:**
*Before:* `%i` is defined in the header and used after the loop.
*After:*
```llvm
exit:
  %i.lcssa = phi i32 [ %i, %latch ]
  ; uses outside the loop read %i.lcssa
```

---

## 45. Loop Fusion
**Why:** Two adjacent loops with the same trip count pay two compares and two induction updates, and they reload values the other loop just wrote. Fusing them runs the bodies together. Fusion is not in `default<O2>` on LLVM 22.1.4, because a wrong dependence answer is a miscompile and the profitability is uneven. `-passes=loop-fusion` adds it.

**Pass:** `loop-fusion`
**Full Name:** `LoopFusePass`
**Day:** [Day 117](Day-117-0-LoopDistribute-Fuse.md)
**What it does:** Merges two adjacent loops into one when they have the same trip count and dependence analysis says the bodies may run together. One loop pays one set of increment and branch instructions.

**Example:**
*Before:*
```c
for (int i = 0; i < n; ++i)
  a[i] = b[i];
for (int i = 0; i < n; ++i)
  c[i] = a[i] + 1;
```
*After:* one loop that does both the copy and the add. Fusion is refused when the second loop reads a value the first loop has not written yet on this iteration, or writes a value the first loop still needs.

---

## 46. Loop Unroll and Jam
**Why:** An inner loop that reloads an outer-loop value on every iteration is a missed register reuse. Unroll-and-jam unrolls the outer loop and fuses the inner copies so that value stays live. It is not in the default pipelines. The runtime unroll after vectorization is `loop-unroll`, which unrolls one loop and does not jam a nest.

**Pass:** `loop-unroll-and-jam`
**Full Name:** `LoopUnrollAndJamPass`
**Day:** [Day 117](Day-117-0-LoopDistribute-Fuse.md)
**What it does:** Unrolls the outer loop of a nest and fuses the resulting inner loops, so the inner body reuses values loaded by the outer iteration. Legality is a dependence question, the same kind as fusion.

**Example:**
*Before:* `for i` around `for j`, and the inner loop reloads `b[i]` on every `j`.
*After:* two (or more) copies of the inner body jammed into one inner loop, with `b[i]` and `b[i+1]` live across the inner trip.

---

## 47. Vector Combine
**Why:** The vectorizer and InstCombine leave extract/insert pairs that are scalar operations on one lane of a vector. Those pairs do not lower to a single vector instruction, and they confuse the cost of later passes. VectorCombine cleans them up. `-O2` runs it inside the CGSCC pipeline and again after SLP. `-O1` runs it only in the late pipeline.

**Pass:** `vector-combine`
**Full Name:** `VectorCombinePass`
**Day:** [Day 56](Day-56-0-VectorCombine.md)
**What it does:** Cleans up vector IR after the vectorizers. It folds an extract, a scalar op, and an insert back into a vector op, or the other way when a scalar is cheaper.

**Example:**
*Before:*
```llvm
  %v = load <4 x i32>, ptr %p
  %e = extractelement <4 x i32> %v, i64 0
  %s = add i32 %e, 1
```
*After:* a vector add, or a scalar load of the one lane, whichever the target cost model says is cheaper. The pass does not discover a new loop to widen.

---

## 48. Load/Store Vectorizer
**Why:** On targets where a wide load is much cheaper than several narrow ones, consecutive loads should be one instruction even when there is no arithmetic to vectorize. SLP looks for the arithmetic; this pass looks only at the memory ops. It is not in `default<O2>` on this LLVM. GPU targets add it in their own pipelines; `-passes=load-store-vectorizer` adds it by hand.

**Pass:** `load-store-vectorizer`
**Full Name:** `LoadStoreVectorizerPass`
**Day:** [Day 55](Day-55-0-LoadStoreVectorizer.md)
**What it does:** Merges consecutive loads or stores in one block into one wider access. It does not look for arithmetic to widen; SLP does that.

**Example:**
*Before:*
```llvm
  %a = load i32, ptr %p
  %b = load i32, ptr %q     ; %q is %p+4, and nothing may-alias sits between
```
*After:*
```llvm
  %w = load <2 x i32>, ptr %p
```
A call or a possibly aliasing store between the loads blocks the group.

---

## 49. NewGVN
**Why:** GVN misses some equalities that depend on a phi or on a branch predicate. NewGVN is a second algorithm for the same job, kept so those cases can be caught when you ask for it. `default<O2>` runs `gvn`, not `newgvn`. Use `-passes=newgvn` when you want the other one; do not expect both in the default pipeline.

**Pass:** `newgvn`
**Full Name:** `NewGVNPass`
**Day:** [Day 11](Day-11-0-GVN.md)
**What it does:** The same job as GVN, value numbering, on a different algorithm that understands more predicate and phi cases. It is not a replacement you pass by default in every pipeline; some pipelines still run `gvn`. When you see both names, they are two implementations of "this instruction computes a value we already have."

**Example:** The GVN load example in section 11 is the shape. NewGVN's extra wins are where a phi or a branch condition makes two expressions equal only on one path. GVN's memory forwarding still applies: a load is not the same as another load just because the address expression matches.
