# Day 14: Correlated Value Propagation (CVP)

CVP rewrites instructions using **LazyValueInfo** facts—especially `ConstantRange` knowledge implied by dominating branches and `llvm.assume`. Pass name: `correlated-propagation` (`CorrelatedValuePropagationPass`). It is CFG-correlated simplification, not a full SCCP-style sparse constant lattice over the whole function.

Deep dive: [Day-14-1-Extra-CVP-DeepDive.md](Day-14-1-Extra-CVP-DeepDive.md). Examples: [Day-14-2-Extra-CVP-Examples.md](Day-14-2-Extra-CVP-Examples.md).

## 1. Why this exists

After a branch `if (x < 10)`, the true block knows `x` is in `[min, 9]`. SCCP only helps when values are fully constant. Many profitable rewrites need **ranges and edge predicates**: fold `icmp`, delete impossible `switch` cases, simplify `select`, prove `urem`/`sdiv` safe, turn `sext` into `zext`, add `nsw`/`nuw` when ranges prove no wrap.

LazyValueInfo (LVI) answers “what is known about `V` at this instruction / on this edge?” CVP is the transform that **consumes** those answers and rewrites IR.

## Diagram

```text
   %c = icmp ult i32 %x, 10
   br i1 %c, label %then, label %else

   then:                         else:
   LVI facts:                    LVI facts:
   %x in [0, 9]                  %x in [10, unsigned max]
   %d = icmp ult i32 %x, 10      %e = icmp ult i32 %x, 5
   --> true                      --> false
```

CVP uses LazyValueInfo, which intersects conditions along the path to the block. Inside `%then` the compiler already knows `%x` is below 10, so a second identical compare is the constant `true`. The information is a range or a predicate, not only a single constant, which is why this catches cases SCCP misses.

## 2. The idea

At each interesting instruction, query LVI for operand ranges (often `getConstantRangeAtUse` with `UndefAllowed=false` for safety). If the range proves the result, replace or reshape the instruction. Knowledge is **correlated** with CFG position: the same SSA value can have different ranges in different blocks.

Current CVP is mostly an **instruction rewriter**—icmp, switch, select, PHI, binops, casts, some intrinsics—not a separate global lattice solver like SCCP.

## 3. Where it sits in the -O2/-O3 pipeline

In the default simplification pipeline, after JumpThreading (and optional DFA jump threading), PassBuilder runs `CorrelatedValuePropagationPass`, then ADCE. SCCP/BDCE/InstCombine typically ran earlier in the same function pipeline; CVP cleans what threading and edge facts expose.

| Item | Value |
|------|--------|
| Pass name | `correlated-propagation` |
| Class | `CorrelatedValuePropagationPass` |
| Analysis | `LazyValueInfo` |
| Debug | `-debug-only=correlated-value-propagation` |

```bash
clang -S -emit-llvm -O1 -Xclang -disable-llvm-passes file.c -o file.ll
opt -passes=correlated-propagation -S file.ll -o file.cvp.ll
```

## 4. Core concepts

### LazyValueInfo

[llvm/lib/Analysis/LazyValueInfo.cpp](https://github.com/llvm/llvm-project/blob/main/llvm/lib/Analysis/LazyValueInfo.cpp) caches lattice/range facts at program points. CVP calls into LVI rather than re-deriving branch conditions itself.

### ConstantRange

Bounds on integer values. Predicates on edges refine ranges. Assumes add constraints. CVP uses ranges to fold compares, narrow divides, and prove nowrap regions via `ConstantRange::makeGuaranteedNoWrapRegion`.

### Instruction families CVP touches

- `icmp` / cmps — fold or flip signedness
- `switch` — remove impossible cases
- `select` — propagate known operands
- PHI — common value / edge values
- `urem`/`srem`/`udiv`/`sdiv` — nonzero denom, signed→unsigned, narrowing
- `and` of masks; `ashr`→`lshr`; `sext`→`zext`; sitofp→uitofp
- Overflow / saturating / abs / minmax intrinsics
- Add `nsw`/`nuw` on binops when ranges guarantee no wrap

### Nowrap and poison

Setting `nsw`/`nuw` means overflow is **poison**. Ranges must justify that. Queries often use `UndefAllowed=false` so undef-lattice fuzziness does not justify poison flags. Historically `-cvp-dont-add-nowrap-flags` disabled adding these flags for experimentation; modern trees may not expose that flag—treat **nowrap inference as must-be-poison-correct**, and verify with Alive2-style reasoning when unsure.

## 5. The algorithm

`runImpl` in [CorrelatedValuePropagation.cpp](https://github.com/llvm/llvm-project/blob/main/llvm/lib/Transforms/Scalar/CorrelatedValuePropagation.cpp):

1. Fetch LVI, DominatorTree, and related analyses in `CorrelatedValuePropagationPass::run`.
2. Walk blocks / instructions; dispatch `processSelect`, `processCmp`, `processBinOp`, `processUDivOrURem`, `processSDivOrSRem`, `processSwitch` on terminators, cast handlers, intrinsic handlers, etc.
3. On change, keep going as the implementation structures (per-block changed flags); LVI may be updated/invalidated as IR mutates—read the pass’s LVI touch points.
4. Return preserved analyses accordingly.

Mental model: **local rewrite driven by LVI queries at use sites**, repeated over the function.

## 6. Where to read the code

- [llvm/lib/Transforms/Scalar/CorrelatedValuePropagation.cpp](https://github.com/llvm/llvm-project/blob/main/llvm/lib/Transforms/Scalar/CorrelatedValuePropagation.cpp)
- [llvm/include/llvm/Transforms/Scalar/CorrelatedValuePropagation.h](https://github.com/llvm/llvm-project/blob/main/llvm/include/llvm/Transforms/Scalar/CorrelatedValuePropagation.h)
- [llvm/lib/Analysis/LazyValueInfo.cpp](https://github.com/llvm/llvm-project/blob/main/llvm/lib/Analysis/LazyValueInfo.cpp)
- [llvm/include/llvm/Analysis/LazyValueInfo.h](https://github.com/llvm/llvm-project/blob/main/llvm/include/llvm/Analysis/LazyValueInfo.h)
- [llvm/include/llvm/IR/ConstantRange.h](https://github.com/llvm/llvm-project/blob/main/llvm/include/llvm/IR/ConstantRange.h)

Key helpers: `processICmp`, `constantFoldCmp`, `processSwitch`, `processSelect`, `processBinOp`, `computeNoWrapFlags`, `inferNoWrapFromUses`, `processUDivOrURem`, `processSDivOrSRem`.

## 7. Worked before/after

**C** (`cvp_range.c`):

```c
int bounded(int x) {
  if (x < 0 || x > 7)
    return -1;
  /* here 0 <= x <= 7 */
  if (x < 8)
    return x;
  return 99; /* unreachable given the range */
}
```

**Commands:**

```bash
clang -S -emit-llvm -O1 -Xclang -disable-llvm-passes cvp_range.c -o cvp_range.ll
opt -passes=correlated-propagation -S cvp_range.ll -o cvp_range.out.ll
```

**IR sketch after guarding:**

```llvm
; in the block dominated by 0..7 facts:
;   %cmp = icmp slt i32 %x, 8   ; LVI proves true → fold
```

**Expected:** the inner `icmp slt %x, 8` becomes constant true (or the dead return path is simplified away in concert with later CFG passes). CVP folds the compare using the range at that use.

## 8. Reading order for today

1. `CorrelatedValuePropagationPass::run` / `runImpl` dispatch switch.
2. `processCmp` + `processICmp`.
3. `processSwitch`.
4. `processBinOp` / `computeNoWrapFlags` (poison care).
5. Skim LVI’s `getConstantRangeAtUse` in `LazyValueInfo`.
6. Deep dive + examples.

## 9. Hands-on commands

```bash
clang -S -emit-llvm -O1 -Xclang -disable-llvm-passes cvp_range.c -o cvp_range.ll
opt -passes=correlated-propagation -S cvp_range.ll -o out.ll
opt -passes=correlated-propagation -debug-only=correlated-value-propagation -S cvp_range.ll
opt -passes='jump-threading,correlated-propagation' -S cvp_range.ll -o out.jt.ll
opt -passes='sccp,correlated-propagation' -S cvp_range.ll -o out.both.ll
```

## 10. Pitfalls and invariants

- CVP ≠ SCCP: ranges and edge correlation vs sparse global constants.
- `UndefAllowed=false` on many queries: do not let undef inflate nowrap facts.
- `nsw` from ranges must respect **poison** semantics.
- Division rewrites need nonzero / sign-domain proofs.
- LVI results are point-sensitive; same value, different blocks, different facts.
- Opaque pointers: CVP’s headline power is on **integer** ranges; pointer nonnull is a smaller side story (`NumNonNull` style stats).

## 11. Neighbors

- **Before:** JumpThreading exposes useful edges; SCCP/InstCombine create constants.
- **Analysis:** LazyValueInfo, DominatorTree.
- **After:** ADCE / SimplifyCFG remove dead cases CVP proved impossible.
- **Related:** IPSCCP ranges (IPO), PredicateInfo in SCCP—different clients of related ideas.

## 12. Check yourself

**Q1.** Pass name for `opt`?
**A1.** `correlated-propagation`.

**Q2.** Primary analysis?
**A2.** LazyValueInfo.

**Q3.** Name three instruction classes CVP simplifies.
**A3.** icmp, switch, select (also urem/sdiv, and/or of icmps, binop nowrap, …).

**Q4.** Why is CVP not “just SCCP”?
**A4.** It rewrites using CFG-correlated ranges, not only a global constant lattice fixed point.

**Q5.** Why is nowrap inference delicate?
**A5.** `nsw`/`nuw` make overflow poison; ranges must truly guarantee no wrap.

**Q6.** Debug type string?
**A6.** `correlated-value-propagation`.

> Tip: Implementation walk-through is in `Day-14-1-Extra-CVP-DeepDive.md`. Runnable cases are in `Day-14-2-Extra-CVP-Examples.md`.

## Appendix A: Edge vs block facts

LVI can answer at an instruction (block-constrained) or along a specific edge (PHI incoming). `getValueOnEdge` / PHI processing uses edge facts so a merge block can still simplify when each incoming edge carries a strong fact—even if the merged range is weaker.

## Appendix B: Signedness flips

When ranges prove values are non-negative, signed predicates and signed divides can become unsigned equivalents (`NumSICmps`, `NumSDivs`, `NumSRems`). That unlocks further InstCombine and better codegen for some targets.

## Appendix C: Common rewrite catalog

| Query proves… | Typical rewrite |
|---------------|-----------------|
| icmp always true/false at point | replace with constant i1 |
| switch case outside range | erase case |
| select condition constant | replace with taken arm |
| `x` in `[0,…]` and signed icmp | unsigned icmp / udiv |
| denom ≠ 0 | legal urem/udiv reshape |
| add operands in nowrap region | set `nsw`/`nuw` |
| sext source nonnegative | `zext` |

Use this table while grepping `process*` in the CVP cpp file.

## Appendix D: JumpThreading partnership

JumpThreading duplicates blocks / redirects edges so that correlated facts become local. CVP then fires more icmp folds. When a hand example “should fold” but does not under `correlated-propagation` alone, try:

```bash
opt -passes='jump-threading,correlated-propagation' -S t.ll
```

Threading is not mandatory for all CVP wins—dominating branches already on the CFG often suffice.

## Appendix E: LVI cache discipline

LVI caches answers per value/program-point. After CVP deletes or rewrites an instruction, stale cache entries are dangerous. The pass notifies LVI when values disappear. If you hack on CVP, every erase should consider LVI invalidation—the existing `process*` helpers show the pattern.

## Appendix F: One-line summary

CVP = LVI-driven instruction rewriting from CFG-correlated ConstantRanges, with careful poison-aware nowrap flags.

## Appendix G: SCCP then CVP

SCCP proves full constants and kills edges. CVP proves ranges on surviving edges. Running both is common in the real pipeline (SCCP earlier, CVP after jump threading). On a single test file:

```bash
opt -passes='sccp,jump-threading,correlated-propagation,simplifycfg' -S t.ll
```

Expect SCCP to erase some paths entirely and CVP to finish icmps that only have range—not constant—information.

## Appendix H: Debug-only reminder

```bash
opt -passes=correlated-propagation -debug-only=correlated-value-propagation -S t.ll
```
