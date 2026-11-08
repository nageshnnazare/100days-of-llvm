# Day 30: Loop Interchange (`loop-interchange`)

Loop interchange swaps two perfectly nested loops to improve locality (and sometimes enable vectorization of a better inner loop). Legality comes from dependence analysis; profitability from cache / TTI heuristics. Deep dive: [Day-30-1-Extra-LoopInterchange-DeepDive.md](Day-30-1-Extra-LoopInterchange-DeepDive.md). Examples: [Day-30-2-Extra-LoopInterchange-Examples.md](Day-30-2-Extra-LoopInterchange-Examples.md).

## Why this exists

Row-major C arrays walked column-first thrash the cache:

```c
for (int i = 0; i < N; ++i)
  for (int j = 0; j < M; ++j)
    A[j][i] += 1; /* inner stride is huge */
```

Interchanging to `for j` outer / `for i` inner (when legal) makes the inner walk contiguous. The transform is classic HPC; LLVM’s implementation is deliberately conservative.

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

## The idea

Given perfectly nested loops `L_outer` containing only `L_inner` (tight nesting):

1. **Legality:** dependence direction vectors between memory accesses must remain valid after swapping loop order. A dependence that requires “inner iterates slower in a certain direction” may become illegal if that direction reverses incorrectly.
2. **Profitability:** estimate whether the new inner loop has better stride / vectorization / cost (TTI, cache heuristics).
3. **Rewrite:** swap the loops’ header/latch structure, adjust inductive PHIs and bounds, keep LCSSA.

## Where it sits in the pipeline

| Item | Value |
|------|--------|
| Pass name | `loop-interchange` |
| Kind | Loop **nest** pass (`LOOPNEST_PASS` in PassRegistry.def) |
| File | [llvm/lib/Transforms/Scalar/LoopInterchange.cpp](https://github.com/llvm/llvm-project/blob/main/llvm/lib/Transforms/Scalar/LoopInterchange.cpp) |
| Debug | `-debug-only=loop-interchange` |
| Needs | LoopSimplify, LCSSA, DependenceInfo, ScalarEvolution, DT, LI |

```bash
clang -S -emit-llvm -O1 -Xclang -disable-llvm-passes nest.c -o nest.ll
opt -passes='loop-simplify,lcssa,loop-interchange' -S nest.ll
```

## Core concepts

### Perfect nesting

The outer body should essentially be the inner loop (plus trivial glue). Extra statements between outer header and inner loop, or multiple sibling inners, fail `NotTightlyNested`-style remarks.

### Rectangular bounds / simple IVs

Bounds should be analyzable (often invariant outer/inner extents). Exotic induction (nonlinear, pointer-chasing) is unsupported.

### Dependence legality

Using DependenceAnalysis results (`DependenceInfo`), direction vectors (LT/EQ/GT/ALL) for each loop level are checked after a conceptual swap. **Negative directions that would reverse incorrectly** block interchange. Remarks like `Dependence` appear on failure.

### No calls (tight restriction)

Call instructions in the nest often make the pass give up (`CallInst` remark)—both for dependence opacity and exception/side-effect reasons.

### Profitability

Even legal interchanges may be skipped if heuristics say locality does not improve. Options can list profitability rules or force interchange for testing (search profitability cl::opt in the .cpp—names vary; one theme is ignoring profitability for lit tests).

## The algorithm

1. Identify a candidate inner/outer pair in the nest.
2. Structural checks: tight nesting, exiting=latch patterns, PHI shapes, no bad calls.
3. Build dependence matrix between loads/stores; verify swap legality.
4. Run profitability heuristics (cache, vectorize potential via TTI).
5. Perform CFG/IV rewrite swapping the loops; update analyses; emit remark `Interchanged`.

## Where to read the code

1. [LoopInterchange.cpp](https://github.com/llvm/llvm-project/blob/main/llvm/lib/Transforms/Scalar/LoopInterchange.cpp) — legality helpers, profitability manager, transform.
2. DependenceAnalysis (Day 36) — direction vectors.
3. Tests: `llvm/test/Transforms/LoopInterchange/`.

## Worked before / after

Before: outer `i`, inner `j`, access `A[i][j]` already good—may **not** interchange.  
Before: outer `i`, inner `j`, access `A[j][i]`—may interchange to outer `j`, inner `i`.

After a successful swap, the inner loop’s IV corresponds to the contiguous dimension.

## Reading order

1. Perfect nesting constraints.
2. Dependence legality section in the .cpp (remark strings).
3. Profitability enum/heuristics near the top of the file.
4. Extras + Day 36.

## Hands-on

```bash
cat > nest.c <<'EOF'
#define N 64
void bad_locality(int A[N][N]) {
  for (int i = 0; i < N; ++i)
    for (int j = 0; j < N; ++j)
      A[j][i]++;
}
EOF
clang -S -emit-llvm -O1 -Xclang -disable-llvm-passes nest.c -o nest.ll
opt -passes='loop-simplify,lcssa,loop-interchange' -S nest.ll -o nest.out.ll
opt -debug-only=loop-interchange -passes='loop-simplify,lcssa,loop-interchange' -S nest.ll
```

## Pitfalls

- **Expecting clang -O3 magic on every nest** — restrictions are severe; many real nests fail nesting or dependence checks.
- **Confusing with LoopVectorize** — interchange is a nest reshape; LV is separate.
- **Calls / I/O in inner loop** — nearly always illegal for this pass.
- **Non-rectangular triangles** (`j < i`) — often unsupported.

## Neighbors

- DependenceAnalysis / LAA (Day 36).
- LoopVectorize — may benefit after better locality.
- TTI (Day 40) — profitability costs.

## Check yourself

1. **Q:** Pass name and pass kind?  
   **A:** `loop-interchange`, loop-nest pass.

2. **Q:** What analysis decides legality of the swap?  
   **A:** DependenceInfo / dependence direction vectors (plus structural checks).

3. **Q:** Name two hard restrictions.  
   **A:** Perfect/tight nesting; no general calls; simple rectangular IVs; supported dependence directions only.

4. **Q:** Debug flag?  
   **A:** `-debug-only=loop-interchange`.

5. **Q:** Why might a legal interchange still not happen?  
   **A:** Profitability heuristics (cache/TTI) say it is not worthwhile.

## Dependence permutation intuition

Suppose a dependence direction vector for (outer, inner) is `(=, <)`. After swapping loops, the levels permute to `(<, =)`. If the dependence still respects the temporal order of the original (source before sink in the dependence sense), interchange may be legal. If the swap would require the sink to execute before the source across the carried dependence, it is illegal. Classical texts use interchange matrices; LLVM encodes the check in LoopInterchange.cpp against DependenceInfo results.

## What “tightly nested” forbids

Any non-trivial statement between outer header and inner loop, sibling loops in the outer body, or complex control around the inner header trips `NotTightlyNested` or related remarks. Even a debug `printf` kills interchange—not because of dependence math, but because of structural rules.

## Reductions

Inner reductions create PHI patterns that many interchange implementations reject (`UnsupportedInnerReduction`). Parallelizing reductions is a different transform family (sometimes LV handles reduction vectorization without interchange). Do not expect interchange to move a reduction loop casually.

## Forced interchange for learning

Search LoopInterchange.cpp for options that ignore profitability. Lit tests use them to lock legality behavior. In production pipelines profitability remains on—so a legal column-major walk might still not swap if the heuristic misjudges. Always check remarks.

## Cache heuristic sketch

Profitability looks at strides of inner-loop memory accesses. Large constant strides relative to cache lines score poorly; unit strides score well. After a successful swap, re-print the access SCEVs to see the new inner AddRec step.

### Additional study points (1)

Suppose a dependence direction vector for (outer, inner) is `(=, <)`. After swapping loops, the levels permute to `(<, =)`. If the dependence still respects the temporal order of the original (source before sink in the dependence sense), interchange may be legal. If the swap would require the sink to execute before the source across the carried dependence, it is illegal. Classical texts use interchange matrices; LLVM encodes the check in LoopInterchange.cpp against DependenceInfo results.
Any non-trivial statement between outer header and inner loop, sibling loops in the outer body, or complex control around the inner header trips `NotTightlyNested` or related remarks. Even a debug `printf` kills interchange—not because of dependence math, but because of structural rules.
Inner reductions create PHI patterns that many interchange implementations reject (`UnsupportedInnerReduction`). Parallelizing reductions is a different transform family (sometimes LV handles reduction vectorization without interchange). Do not expect interchange to move a reduction loop casually.
Search LoopInterchange.cpp for options that ignore profitability. Lit tests use them to lock legality behavior. In production pipelines profitability remains on—so a legal column-major walk might still not swap if the heuristic misjudges. Always check remarks.
Profitability looks at strides of inner-loop memory accesses. Large constant strides relative to cache lines score poorly; unit strides score well. After a successful swap, re-print the access SCEVs to see the new inner AddRec step.

```bash
# revisit with debug (session 1)
opt -passes='loop-simplify,lcssa' -S in.ll -o /tmp/in.simp.ll
```

### Additional study points (2)

Suppose a dependence direction vector for (outer, inner) is `(=, <)`. After swapping loops, the levels permute to `(<, =)`. If the dependence still respects the temporal order of the original (source before sink in the dependence sense), interchange may be legal. If the swap would require the sink to execute before the source across the carried dependence, it is illegal. Classical texts use interchange matrices; LLVM encodes the check in LoopInterchange.cpp against DependenceInfo results.
Any non-trivial statement between outer header and inner loop, sibling loops in the outer body, or complex control around the inner header trips `NotTightlyNested` or related remarks. Even a debug `printf` kills interchange—not because of dependence math, but because of structural rules.
Inner reductions create PHI patterns that many interchange implementations reject (`UnsupportedInnerReduction`). Parallelizing reductions is a different transform family (sometimes LV handles reduction vectorization without interchange). Do not expect interchange to move a reduction loop casually.
Search LoopInterchange.cpp for options that ignore profitability. Lit tests use them to lock legality behavior. In production pipelines profitability remains on—so a legal column-major walk might still not swap if the heuristic misjudges. Always check remarks.
Profitability looks at strides of inner-loop memory accesses. Large constant strides relative to cache lines score poorly; unit strides score well. After a successful swap, re-print the access SCEVs to see the new inner AddRec step.

```bash
# revisit with debug (session 2)
opt -passes='loop-simplify,lcssa' -S in.ll -o /tmp/in.simp.ll
```

### Additional study points (3)

Suppose a dependence direction vector for (outer, inner) is `(=, <)`. After swapping loops, the levels permute to `(<, =)`. If the dependence still respects the temporal order of the original (source before sink in the dependence sense), interchange may be legal. If the swap would require the sink to execute before the source across the carried dependence, it is illegal. Classical texts use interchange matrices; LLVM encodes the check in LoopInterchange.cpp against DependenceInfo results.

```bash
# revisit with debug (session 3)
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

## Debug and remarks together

Combine `-debug-only=<DEBUG_TYPE>` with `-pass-remarks` / `-pass-remarks-missed` for the same pass. Debug logs show internal decisions; remarks show the user-facing summary. When they disagree with your mental model, trust the code path that emits the remark—jump to that string in the .cpp file.

## How this day connects forward and back

Loop and AA topics form a dependency chain: simplify/LCSSA → rotate/unswitch → SCEV/indvars → idiom/interchange → vectorize. Alias analyses (BasicAA, TBAA) and MemorySSA sit underneath almost every memory transform. TLI/TTI decide availability and profitability at the edges of the mid-end. Re-read neighboring day notes when a failure looks like “wrong pass” but is actually a missing analysis fact.

## Extended hands-on checklist (Day 30)

1. Build IR with the disable-llvm-passes recipe.
2. Run the minimal pass pipeline named in this day’s main note.
3. Diff against a pipeline that adds one neighbor pass.
4. Re-run with the day’s `-debug-only` flag.
5. Skim one upstream lit test RUN line and reproduce it.
6. Write one counterexample IR that must not transform.
7. Cross-check a fact against the linked github source file.
