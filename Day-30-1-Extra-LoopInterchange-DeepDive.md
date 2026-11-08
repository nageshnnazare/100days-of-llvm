# Day 30 Extra: Loop Interchange — Deep Dive

Source: [LoopInterchange.cpp](https://github.com/llvm/llvm-project/blob/main/llvm/lib/Transforms/Scalar/LoopInterchange.cpp). Overview: [Day-30-0-LoopInterchange.md](Day-30-0-LoopInterchange.md).

## Diagram

```text
   for i:                         for j:
     for j:            --->         for i:
       A[j][i] = ...                  A[j][i] = ...

   row-major A[j][i] is contiguous
   in j, not in i. The inner loop
   should walk j.

   legality: every dependence
   distance must stay legal after
   the interchange (DependenceAnalysis)
```

Loop interchange swaps two perfectly nested loops so the inner one walks memory with unit stride. The swap is legal only when dependence analysis says the direction vectors still hold. A loop that carries a dependence on the outer index, or that is not tightly nested, is left alone. The profitability check asks which order is actually contiguous.

## Pass registration

In [PassRegistry.def](https://github.com/llvm/llvm-project/blob/main/llvm/lib/Passes/PassRegistry.def) it is a `LOOPNEST_PASS`. That means it sees a loop nest, not a single innermost loop in isolation—necessary because the transform’s subject is a pair of levels.

`DEBUG_TYPE` is `"loop-interchange"`.

## Structural legality (before dependences)

Remark IDs in the file teach the checklist:

| Remark theme | Meaning |
|--------------|---------|
| `UnsupportedLoop` / nest depth | Nest shape not handled |
| `NotTightlyNested` | Code between outer and inner |
| `ExitingNotLatch` | Exit structure too complex |
| `UnsupportedPHIOuter` / `Inner` | IV/reduction PHIs not in the supported pattern |
| `UnsupportedInnerLatchPHI` | Latch PHI issues |
| `UnsupportedExitPHI` | Exit SSA form issues |
| `CallInst` | Calls present |
| `UnsupportedInnerReduction` | Reductions that break interchange assumptions |

Read the functions that emit these—each is a concrete IR shape test. Interchange only proceeds if all pass.

## Dependence legality

The pass queries `DependenceInfo` for pairs of memory accesses in the nest. For each dependence, direction vector entries at outer/inner levels are examined under the **permuted** loop order.

Intuition:

- A flow dependence with direction `<` on the outer level and `=` on the inner may or may not survive swap—matrix permutation rules from classic dependence theory apply.
- Direction `ALL` or problematic GT/LT combinations that reverse causality cause `Dependence` missed remarks.

Banerjee/GCD details live in Day 36; here you only need: **interchange is legal iff after swapping levels no dependence is violated.**

## Profitability engine

Near the top, an enum lists profitability rule kinds. A manager applies heuristics in order—**cache locality** (stride of inner accesses), sometimes **vectorization** potential (would the new inner vectorize better?), using TTI costs when available.

Important teaching point: **legality ≠ profitability**. Lit tests may force interchange ignoring profitability; production pipelines do not.

If a rule can decide “yes/no,” later rules may short-circuit (`InterchangeNotProfitable` remark).

## The rewrite

Once allowed, the transform:

1. Swaps which loop is outer vs inner in LoopInfo’s nest.
2. Rewrites header PHIs so the new outer IV updates at the new outer latch.
3. Moves bound compares to the correct headers.
4. Fixes LCSSA at exits for values defined in either loop.
5. Updates DT and related analyses.

This is delicate IR surgery—when reading, follow one lit test’s CHECK lines against the transform function end-to-end.

## Interaction with vectorization cost

Comments mention considering vectorized costs of both orders. That does not mean LoopVectorize runs inside interchange; it means cost models approximate whether the nest would vectorize better after the swap. Actual LV still runs separately.

## What LoopAccessAnalysis has to do with this

LoopVectorize primarily uses **LoopAccessAnalysis** (runtime checks, dependence distances for vectorization), not this pass’s DependenceInfo client path. Interchange is one of the few transforms that still leans on classical `DependenceAnalysis`. Keep the split clear (Day 36 expands it).

## Debugging

```bash
opt -debug-only=loop-interchange \
    -passes='loop-simplify,lcssa,loop-interchange' -S nest.ll
```

Also enable optimization remarks:

```bash
opt -passes='loop-simplify,lcssa,loop-interchange' \
    -pass-remarks=loop-interchange \
    -pass-remarks-missed=loop-interchange -S nest.ll
```

Missed remarks usually name the exact structural or dependence failure—use them before diving into the dependence matrix by hand.

## Reading path

1. Profitability enum + cl::opt (policy).
2. Tight nesting / PHI checks (structure).
3. Dependence legality function (math).
4. Transform/rewrite function (mechanics).
5. One positive and one negative lit test.

## Remark-driven debugging workflow

1. Run with `-pass-remarks-missed=loop-interchange`.
2. Note the remark id (`CallInst`, `Dependence`, `NotTightlyNested`, …).
3. Jump to the `OptimizationRemarkMissed(DEBUG_TYPE, "ThatId"` site in LoopInterchange.cpp.
4. Read the IR condition immediately above the remark.

This is faster than reading the whole file linearly.

## Nest pass mechanics

As a `LOOPNEST_PASS`, the pass receives a top-level nest rather than only the innermost loop. Candidate pair discovery walks the nest tree looking for adjacent outer/inner pairs that pass structural filters. Understanding LoopNest / LoopInfo parent links helps when the IR has more than two depths.

## Vectorization cost as profitability input

Comments about comparing vectorized costs of both loop orders mean: estimate whether LV would like the swapped nest better. That estimate can be wrong, but it shows TTI (Day 40) feeding interchange policy. Legal + unprofitable for scalar locality might still be profitable if the new inner vectorizes—heuristics try to capture that.

## DependenceInfo lifetime

The pass requires DependenceInfo to be available in the nest analysis results. If your custom pipeline forgets `da`, interchange cannot legally prove swaps—and should not proceed. Register analyses properly when building toy pipelines.

### Additional study points (1)

- 1. Run with `-pass-remarks-missed=loop-interchange`.
- 2. Note the remark id (`CallInst`, `Dependence`, `NotTightlyNested`, …).
- 3. Jump to the `OptimizationRemarkMissed(DEBUG_TYPE, "ThatId"` site in LoopInterchange.cpp.
- 4. Read the IR condition immediately above the remark.
- This is faster than reading the whole file linearly.
As a `LOOPNEST_PASS`, the pass receives a top-level nest rather than only the innermost loop. Candidate pair discovery walks the nest tree looking for adjacent outer/inner pairs that pass structural filters. Understanding LoopNest / LoopInfo parent links helps when the IR has more than two depths.
Comments about comparing vectorized costs of both loop orders mean: estimate whether LV would like the swapped nest better. That estimate can be wrong, but it shows TTI (Day 40) feeding interchange policy. Legal + unprofitable for scalar locality might still be profitable if the new inner vectorizes—heuristics try to capture that.
The pass requires DependenceInfo to be available in the nest analysis results. If your custom pipeline forgets `da`, interchange cannot legally prove swaps—and should not proceed. Register analyses properly when building toy pipelines.

```bash
# revisit with debug (session 1)
opt -passes='loop-simplify,lcssa' -S in.ll -o /tmp/in.simp.ll
```

### Additional study points (2)

- 1. Run with `-pass-remarks-missed=loop-interchange`.
- 2. Note the remark id (`CallInst`, `Dependence`, `NotTightlyNested`, …).
- 3. Jump to the `OptimizationRemarkMissed(DEBUG_TYPE, "ThatId"` site in LoopInterchange.cpp.
- 4. Read the IR condition immediately above the remark.
- This is faster than reading the whole file linearly.
As a `LOOPNEST_PASS`, the pass receives a top-level nest rather than only the innermost loop. Candidate pair discovery walks the nest tree looking for adjacent outer/inner pairs that pass structural filters. Understanding LoopNest / LoopInfo parent links helps when the IR has more than two depths.
Comments about comparing vectorized costs of both loop orders mean: estimate whether LV would like the swapped nest better. That estimate can be wrong, but it shows TTI (Day 40) feeding interchange policy. Legal + unprofitable for scalar locality might still be profitable if the new inner vectorizes—heuristics try to capture that.
The pass requires DependenceInfo to be available in the nest analysis results. If your custom pipeline forgets `da`, interchange cannot legally prove swaps—and should not proceed. Register analyses properly when building toy pipelines.

```bash
# revisit with debug (session 2)
opt -passes='loop-simplify,lcssa' -S in.ll -o /tmp/in.simp.ll
```

## Rectangular bound checklist

Interchange wants outer and inner bounds that SCEV can treat as rectangular: inner trip count independent of the outer IV (or otherwise specially handled). Triangle nests (`j < i`), jagged arrays, and pointer-chasing outers fail early. Print SCEV for both IVs and their limits before expecting a swap.

## IR hygiene for experiments

Always generate IR with `clang -S -emit-llvm -O1 -Xclang -disable-llvm-passes` so functions are not marked `optnone`. Opaque pointers (`ptr`) are mandatory in modern IR examples; do not revive typed pointer syntax in new notes.

When comparing before/after, prefer `diff -u` on files produced by the same opt pipeline with one pass removed. That isolates the transform better than comparing against clang -O3, which runs dozens of passes.

## Verifier and analysis invalidation

After a transform, `opt -passes='verify'` (or the verifier embedded in pipelines) should still succeed. Loop passes must preserve LCSSA and simplified form when they advertise those preservations. Analysis managers invalidate SCEV, AA, and MemorySSA according to the preserved-analyses set returned by the pass—reading `PreservedAnalyses` in the pass’s `run` method teaches what clients may keep.

## Reading upstream tests

For any pass, open `llvm/test/Transforms/<PassName>/` or `llvm/test/Analysis/<Name>/` and skim the smallest lit test. The `RUN:` lines show the canonical `opt` invocation, including required simplify/lcssa partners. Copying a RUN line into your shell is the fastest way to get a known-good baseline before editing IR by hand.
