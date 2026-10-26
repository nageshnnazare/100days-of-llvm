# Day 11: Global Value Numbering (GVN)

GVN finds computations that produce the same value and keeps one representative (a *leader*), rewriting the others to use it. In LLVM this is both scalar CSE and a memory-aware load eliminator with optional load PRE. The production pass is `gvn` (`GVNPass` in `GVN.cpp`). A separate experiment, **NewGVN**, uses congruence classes instead of classic RPO leader election; it lives in `NewGVN.cpp` and is not the default `-O2` GVN.

## 1. Why this exists

After inlining, loop opts, and InstCombine, the IR is full of repeated arithmetic and repeated loads of the same address. Local CSE (InstCombine, EarlyCSE) only sees a small window inside a block or along a short chain. GVN walks the whole function with a DominatorTree so a value computed in a dominating block can replace a later congruent computation. Memory-aware numbering extends the same idea to loads: if nothing that may alias has written between two loads of the same pointer, the second load is redundant.

Without GVN, middle-end passes would re-materialize the same `add`/`mul`/`gep` sequences and leave redundant memory traffic that later LICM and vectorization must rediscover. GVN is the workhorse global redundancy pass in the scalar pipeline.

## Diagram

```text
   %a = add %x, 1
   ...
   %b = add %x, 1          GVN: %b and %a are the same
                           value number -> replace %b with %a

   store 7, ptr %p
   %v = load ptr %p        MemoryDependence says the store
                           is the clobber -> replace %v with 7
```

GVN gives two instructions the same value number when their operands have the same numbers and the opcode matches. Loads are included only when a memory dependence query says the loaded address was written by a known store and not clobbered in between. A load is not "the same" as another load just because the address expression matches.

## 2. The idea

Assign every SSA value a *value number*. Two instructions share a number when their opcodes and operand numbers match (and, for memory ops, when the keyed memory state matches). Within a value number, pick a *leader* that dominates the current use site; replace the redundant instruction with that leader.

For loads, ask MemorySSA (via clobber walks and `MemorySSAUpdater` when enabled) whether a prior store or load still reaches. If a must-alias store of a known value reaches, forward the stored value. If an identical load reaches, CSE the load. Load PRE may insert a load on a predecessor edge so the merge block can use a PHI instead of reloading on the hot path.

Calls are deliberately conservative: only effects-free (or read-only) calls with equal arguments share a number. A may-alias store invalidates load availability for that location.

## 3. Where it sits in the -O2/-O3 pipeline

In the default function simplification pipeline, GVN runs after early cleanup (SROA, EarlyCSE, InstCombine rounds) once the CFG and memory are somewhat cleaned, and before late CFG and loop transforms that benefit from fewer loads and denser scalar IR. MemCpyOpt and DSE are nearby memory neighbors; CorrelatedValuePropagation and SimplifyCFG often clean icmps and edges after constants and PHIs appear.

- Pass name for `opt`: `gvn`
- Class: `GVNPass`
- Related (not default): `newgvn` → [NewGVN.cpp](https://github.com/llvm/llvm-project/blob/main/llvm/lib/Transforms/Scalar/NewGVN.cpp)

```bash
clang -S -emit-llvm -O1 -Xclang -disable-llvm-passes file.c -o file.ll
opt -passes=gvn -S file.ll -o file.gvn.ll
```

## 4. Core concepts

**Value number / Expression.** Nested class `GVNPass::ValueTable` hashes an instruction into an `Expression` (opcode, type, operand value numbers, and optionally a memory-state number). Congruent expressions map to one integer.

**Leader election.** For each number, GVN tracks available instances across blocks and picks a dominating leader when rewriting. The DominatorTree decides whether replacement is legal in SSA.

**MemorySSA and MemDep.** Modern GVN integrates MemorySSA: clobber checks for load elimination go through MemorySSA walkers, and updates use `MemorySSAUpdater`. A parallel path still exists via `MemoryDependenceAnalysis` (`-enable-gvn-memdep` / `-enable-gvn-memoryssa` control which machinery is active). AAResults refine may-alias vs must-alias.

**Load PRE.** Controlled by `-enable-load-pre` (enabled by default). When a loaded value is available on some predecessors but not others, GVN may split critical edges, insert loads where missing, and PHI-merge so the consumer block does not reload.

**Other analyses.** `AssumptionCache` feeds known bits / assume facts that interact with availability; DominatorTree is mandatory for leaders and PRE.

## 5. The algorithm

1. Fetch DominatorTree, AAResults, AssumptionCache, and the configured memory analysis (MemorySSA and/or MemDep).
2. Number values while walking in an order that respects dominance (RPO-style processing in `GVNPass`).
3. For a scalar instruction, `ValueTable::lookupOrAdd` builds an `Expression`. If a congruent leader already dominates, replace this instruction with the leader and erase it.
4. For loads: compute reaching memory. If a store of identical bytes dominates with no intervening clobber, replace the load with the stored value. If a congruent load is available, CSE.
5. If load PRE is enabled and availability is partial across predecessors, attempt critical-edge splits, insert loads on unavailable preds, and build a PHI at the merge — only when speculation / EH rules allow.
6. For calls: `lookupOrAddCall` only shares numbers for safe memory effects with equal args; otherwise assign a fresh number.
7. When a store may alias, subsequent loads fail availability or receive distinct memory-keyed numbers so CSE does not fire incorrectly.
8. Preserve/update MemorySSA through `MemorySSAUpdater` when that path is enabled so later passes see a consistent MSSA.

## 6. Where to read the code

- [llvm/lib/Transforms/Scalar/GVN.cpp](https://github.com/llvm/llvm-project/blob/main/llvm/lib/Transforms/Scalar/GVN.cpp) — `GVNPass`, `ValueTable`, load elim / PRE
- [llvm/include/llvm/Transforms/Scalar/GVN.h](https://github.com/llvm/llvm-project/blob/main/llvm/include/llvm/Transforms/Scalar/GVN.h) — options and public API
- [llvm/lib/Transforms/Scalar/NewGVN.cpp](https://github.com/llvm/llvm-project/blob/main/llvm/lib/Transforms/Scalar/NewGVN.cpp) — congruence-class alternative
- [llvm/include/llvm/Analysis/MemorySSA.h](https://github.com/llvm/llvm-project/blob/main/llvm/include/llvm/Analysis/MemorySSA.h) — MemorySSA / walker concepts
- MemorySSAUpdater usages inside `GVN.cpp` when rewriting memory ops

Search symbols: `GVNPass::run`, `ValueTable::lookupOrAdd`, `ValueTable::lookupOrAddCall`, `ValueTable::addMemoryStateToExp`, `processLoad`, load PRE / `AnalyzeLoadAvailability`-style helpers.

## 7. Worked before/after

**C** (`gvn_cse.c`):

```c
int add_twice(int a, int b) {
  int x = a + b;
  int y = a + b;
  return x + y;
}

int load_cse(int *p) {
  int a = *p;
  int b = *p;
  return a + b;
}

int blocked(int *p, int *q) {
  int a = *p;
  *q = 1;          /* may alias p */
  int b = *p;
  return a + b;
}
```

**Commands:**

```bash
clang -S -emit-llvm -O1 -Xclang -disable-llvm-passes gvn_cse.c -o gvn_cse.ll
opt -passes=gvn -S gvn_cse.ll -o gvn_cse.gvn.ll
```

**`add_twice` before:**

```llvm
define i32 @add_twice(i32 %a, i32 %b) {
  %x = add i32 %a, %b
  %y = add i32 %a, %b
  %r = add i32 %x, %y
  ret i32 %r
}
```

**After GVN:**

```llvm
define i32 @add_twice(i32 %a, i32 %b) {
  %x = add i32 %a, %b
  %r = add i32 %x, %x
  ret i32 %r
}
```

**`load_cse` after GVN** (no clobber):

```llvm
define i32 @load_cse(ptr %p) {
  %a = load i32, ptr %p, align 4
  %r = add i32 %a, %a
  ret i32 %r
}
```

**`blocked`:** the second load of `%p` must remain if `%q` may alias `%p`; GVN keeps both loads.

## 8. Reading order for today

1. `GVN.h` — option knobs (scalar PRE, load PRE, MemorySSA).
2. `ValueTable::createExpr` / `lookupOrAdd` in `GVN.cpp`.
3. `lookupOrAddCall` — call CSE policy.
4. One full load-elimination path (availability + clobber).
5. File header / class overview of `NewGVN.cpp` for contrast only.

## 9. Hands-on commands

```bash
clang -S -emit-llvm -O1 -Xclang -disable-llvm-passes gvn_cse.c -o gvn_cse.ll
opt -passes=gvn -S gvn_cse.ll -o out.ll
opt -passes=gvn -debug-only=gvn -S gvn_cse.ll 2>&1 | less
opt -passes=gvn -enable-load-pre=false -S gvn_cse.ll -o out.nopre.ll
opt -passes=newgvn -S gvn_cse.ll -o out.newgvn.ll
```

Use an assertions build for denser `-debug-only=gvn` logs.

## 10. Pitfalls and invariants

- `clang -O0` adds `optnone`; `opt` skips the function. Prefer `-O1 -Xclang -disable-llvm-passes`.
- May-alias stores block load CSE; `noalias`, TBAA, and `invariant.load` change results.
- Ordinary calls are not CSE’d; need readnone/readonly (or equivalent effect attrs) plus equal args.
- Load PRE must not introduce faulting loads on paths that did not load before.
- Opaque pointers: congruence is on `ptr` SSA values and load types.
- Do not confuse `gvn` with `newgvn`.

## 11. Neighbors

- **Before:** EarlyCSE, InstCombine, SROA feed canonical scalar IR.
- **Uses:** DominatorTree, AAResults, AssumptionCache, MemorySSA/MemDep.
- **After:** InstCombine / SimplifyCFG fold PRE PHIs; LICM and DSE see fewer redundant loads.
- **Invalidation:** aliasing stores kill load leaders; MemorySSA must stay coherent via updater APIs.

## 12. Check yourself

**Q1.** What does a value number identify?
**A.** Congruent computations (opcode, operands, and memory key), not merely textual IR clones.

**Q2.** Why is DominatorTree required?
**A.** Leaders may replace a use only when they dominate that use (SSA).

**Q3.** When can identical calls CSE?
**A.** When effects allow (readnone/readonly-style) and arguments share value numbers.

**Q4.** What does `-enable-load-pre` do?
**A.** Allows inserting loads on preds / splitting critical edges so a PHI replaces a reload.

**Q5.** How does NewGVN differ?
**A.** Congruence-class approach in `NewGVN.cpp`; not the default pipeline pass.

**Q6.** Debug flag?
**A.** `-debug-only=gvn`.

> Tip: Implementation details are in `Day-11-1-Extra-GVN-DeepDive.md`. Runnable cases are in `Day-11-2-Extra-GVN-Examples.md`.


## Appendix A: Value numbering vs hash-consing

Think of `ValueTable` as hash-consing IR operations over SSA names. Constants and arguments are leaves. Each instruction is a node keyed by opcode and the numbers of its children. Dominator constraints mean you cannot freely replace with any congruent node — only with one that dominates. That is the difference between a pure DAG hash-cons and SSA-aware GVN.

When MemorySSA keys are folded into the expression, the “children” include a memory token. That token is not an IR `Value` the user wrote; it is an analysis artifact. Two loads of `%p` separated by a non-aliasing store of `%q` can still share a memory key if AA proves no alias; a may-alias store allocates a different key or fails availability.

## Appendix B: Critical edges and PRE

A critical edge runs from a block with multiple successors to a block with multiple predecessors. Inserting a load directly on that edge is impossible without a new block. GVN’s load PRE path splits such edges when profitable and legal, then inserts the compensating load in the new block. After PRE, InstCombine and SimplifyCFG may fold trivial PHIs.

If PRE looks “broken” on a toy example, dump the CFG before and after and look for new blocks on formerly critical edges. `-enable-load-pre=false` is the control experiment.

## Appendix C: Interaction with EarlyCSE

EarlyCSE does local value numbering inside a block (and some simple memory VN). GVN is the global pass. Redundant work between them is acceptable; EarlyCSE is cheap insurance early in the pipeline, GVN cleans what crosses blocks after more transforms. When reading `-O2` pass lists, you will see both.

## Appendix D: Atomic and volatile loads

GVN must respect `load atomic` ordering and `volatile`. A volatile load is an observable side effect; eliminating it would change the abstract machine. Ordered atomics participate in the concurrency model — CSE across potential races is unsafe unless language/IR rules say otherwise. When teaching, always include one volatile counterexample next to the happy-path load CSE.

Unordered/`monotonic` cases still go through AA and MemorySSA; do not assume “atomic ⇒ never CSE”. The pass encodes the precise rules; read the load handler rather than guessing.

## Appendix E: Opaque pointers reminder

All examples use `ptr`. Older textbooks show typed pointer syntax. On LLVM 15+, typed pointers are gone from IR. GEP type is supplied on the instruction (`getelementptr inbounds i32, ptr %p, i64 %i`), not on the pointer type. GVN expressions include the load’s result type so `load i32` and `load i64` from the same address never share a number.

## Appendix F: One-line summary

GVN = dominance-aware hash-consing of scalar ops + memory-aware load CSE/PRE via `ValueTable` leaders, with NewGVN as a non-default congruence-class cousin.



