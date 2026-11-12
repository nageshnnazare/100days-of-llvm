# Day 36 Extra: Dependence Analysis & LAA — Deep Dive

Sources: [DependenceAnalysis.cpp](https://github.com/llvm/llvm-project/blob/main/llvm/lib/Analysis/DependenceAnalysis.cpp), [LoopAccessAnalysis.cpp](https://github.com/llvm/llvm-project/blob/main/llvm/lib/Analysis/LoopAccessAnalysis.cpp). Parent: [Day-36-0-DependenceAnalysis.md](Day-36-0-DependenceAnalysis.md).

## Diagram

```text
   for i:
     A[i+1] = A[i] + 1

   dependence from the load A[i]
   to the store A[i+1]:

     direction:  forward  (<)
     distance:   1 iteration

   that is a loop-carried flow dependence.
   Interchange and vectorization must
   respect it.

   A[i] = B[i] + 1
   has no carried dependence on A if B
   does not alias A.
```

Dependence analysis asks whether two memory accesses in a loop nest touch the same location, and if so in which direction and at what distance. A distance of zero is loop-independent. A positive distance is carried by that loop. The vectorizer needs "no carried dependence, or a dependence it can honor with a vector width." The answer is conservative: if the subscripts are not affine, the result is "do not know."

## DependenceAnalysis structure

`DependenceInfo` is created per function with SE, AA, DT, LI. Query methods take two instructions and return a `std::unique_ptr<Dependence>` describing the relation (or independence).

Statistics counter applications of GCD, Banerjee, etc. A cl::opt selects which MIV tests run (`DependenceTestType`: Default, GCDMIV, BanerjeeMIV, …). Comments note Banerjee correctness issues—**Default may exclude BanerjeeMIV**. Always verify before teaching Banerjee as “always on.”

### Strong SIV

`strongSIVtest` handles subscripts where each side is an affine function of the **same** single loop IV with matching coefficients of magnitude enabling a closed-form distance/direction. This is the common `a[i]` vs `a[i+k]` case.

### Weak-crossing / GCD / Banerjee

- Weak-crossing: opposing slopes.
- GCD test: for integer linear Diophantine constraints on indices—if gcd of coefficients does not divide the constant term → independent.
- Banerjee: inequality-based bounds tests on MIV subscripts.

Read strong SIV fully; skim GCD; treat Banerjee as optional historical depth.

### `DVEntry`

Each nest level gets a direction bitfield (`LT|EQ|GT|ALL`) plus optional distance and scalar flags. Methods on `Dependence` query `getDirection(Level)`, `getDistance(Level)`.

Interchange reverses/permutes levels and checks remaining directions still legal.

## LoopAccessAnalysis structure

`LoopAccessInfo` aggregates:

- Can vectorize memory?
- Need runtime pointer checks?
- Dependency distance / store-to-load forwarding limits
- Symbolic strides

Construction walks the loop, queries AA and SCEV for each access pair, and may invent `RuntimePointerChecking` structures that LoopVectorize lowers into IR checks before the vector loop.

Printer: `print<access-info>` / `LoopAccessInfoPrinterPass` in PassRegistry.def.

### Why LV prefers LAA

Vectorization needs:

1. “Safe with these runtime checks” answers, not only static independence.
2. Tight integration with interleaved accesses, strided loads, and gap analysis.
3. Per-innermost-loop focus rather than full classical nest matrices.

Classical DA does not emit those checks. Interchange does not need them—it needs static legality of a reshape.

## Shared foundation: SCEV + AA

Both analyses fail closed when SCEV cannot see AddRecs (opaque indices) or AA says MayAlias without further structure. Improving TBAA/BasicAA often improves LAA without changing DA tests.

## Teaching demo flow

1. `print<access-info>` on a carried dependence loop → unsafe / distance 1.
2. Same on `restrict` parallel loop → safe.
3. `print<da>` on a 2D nest used for interchange experiments.
4. Open LoopVectorize and grep `getInfo(*L)` / `LoopAccessInfo`—zero in on LAA.
5. Open LoopInterchange and grep `Dependence`—zero in on DA.

## Pitfalls when reading papers vs LLVM

Textbook Banerjee/GCD may not match LLVM’s gated Default suite. Always reconcile with the .cpp options. Papers about “dependence analysis for vectorization” in LLVM-land usually mean LAA-style reasoning even if they use classical vocabulary.

## Reading path

1. `strongSIVtest`
2. `DVEntry` / Dependence printing
3. LoopAccessInfo constructor / runtime check builder
4. One LV dependency on LAA
5. Examples

## Runtime checks vs static DA

LAA’s runtime checks typically compare base pointers and region sizes so vector code runs only when objects do not overlap. Classical DA trying to prove static independence cannot invent those checks. When `print<access-info>` says checks are needed, LoopVectorize may still vectorize behind a guard—successful vectorization ≠ static NoAlias.

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

## Source navigation checklist (Day 36 DeepDive)

1. Open the primary .cpp linked from the main note.
2. Jump to DEBUG_TYPE / cl::opt block.
3. Find the public `run` method or analysis entry.
4. Find the core legality helper and the core transform helper.
5. Grep remark strings / LLVM_DEBUG strings you saw experimentally.
6. Note which analyses are required vs preserved.
7. Skim one Failure path and one Success path to completion.

## Runtime checks vs static DA

LAA’s runtime checks typically compare base pointers and region sizes so vector code runs only when objects do not overlap. Classical DA trying to prove static independence cannot invent those checks. When `print<access-info>` says checks are needed, LoopVectorize may still vectorize behind a guard—successful vectorization ≠ static NoAlias.

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

## Source navigation checklist (Day 36 DeepDive)

1. Open the primary .cpp linked from the main note.
2. Jump to DEBUG_TYPE / cl::opt block.
3. Find the public `run` method or analysis entry.
4. Find the core legality helper and the core transform helper.
5. Grep remark strings / LLVM_DEBUG strings you saw experimentally.
6. Note which analyses are required vs preserved.
7. Skim one Failure path and one Success path to completion.

## Runtime checks vs static DA

LAA’s runtime checks typically compare base pointers and region sizes so vector code runs only when objects do not overlap. Classical DA trying to prove static independence cannot invent those checks. When `print<access-info>` says checks are needed, LoopVectorize may still vectorize behind a guard—successful vectorization ≠ static NoAlias.

## IR hygiene for experiments

Always generate IR with `clang -S -emit-llvm -O1 -Xclang -disable-llvm-passes` so functions are not marked `optnone`. Opaque pointers (`ptr`) are mandatory in modern IR examples; do not revive typed pointer syntax in new notes.

When comparing before/after, prefer `diff -u` on files produced by the same opt pipeline with one pass removed. That isolates the transform better than comparing against clang -O3, which runs dozens of passes.

## Verifier and analysis invalidation

After a transform, `opt -passes='verify'` (or the verifier embedded in pipelines) should still succeed. Loop passes must preserve LCSSA and simplified form when they advertise those preservations. Analysis managers invalidate SCEV, AA, and MemorySSA according to the preserved-analyses set returned by the pass—reading `PreservedAnalyses` in the pass’s `run` method teaches what clients may keep.
