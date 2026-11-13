# Day 34 Extra: SCEV — Deep Dive

Sources: [ScalarEvolution.cpp](https://github.com/llvm/llvm-project/blob/main/llvm/lib/Analysis/ScalarEvolution.cpp), [ScalarEvolutionExpander.cpp](https://github.com/llvm/llvm-project/blob/main/llvm/lib/Analysis/ScalarEvolutionExpander.cpp), [ScalarEvolutionExpressions.h](https://github.com/llvm/llvm-project/blob/main/llvm/include/llvm/Analysis/ScalarEvolutionExpressions.h). Parent: [Day-34-0-SCEV.md](Day-34-0-SCEV.md).

## Diagram

```text
   %i = phi i32 [ 0, %pre ], [ %i.next, %latch ]
   %i.next = add nsw i32 %i, 1

   SCEV expression for %i:

        {0,+,1}<header>

   start 0, step 1, in this loop

   %p = getelementptr i32, ptr %base, i32 %i
   SCEV of the pointer:
        { %base,+,4 }<header>
```

SCEV is a normal form for integer recurrences. An induction variable becomes an add recurrence `{start,+,step}`. A GEP becomes a recurrence in bytes. Two IVs that look different in the IR often become the same SCEV, which is how trip counts, dependence distances, and legality checks compare them. A value with a possible overflow that SCEV cannot treat as `nsw` stays a more pessimistic expression.

## Expression DAG design

Every `SCEV` subclass is immutable and uniqued. Comparing expressions is pointer equality after canonicalization. Constructors go through `ScalarEvolution` factory methods (`getAddExpr`, `getMulExpr`, `getAddRecExpr`, …) that fold constants and reassociate when legal.

`scUnknown` wraps a `Value*` that SCEV will not analyze further (arbitrary loads, calls, args). Casts are explicit nodes so that `zext({0,+,1})` is distinct from `{0,+,1}` at a wider type until folding proves equality.

## AddRec details

`SCEVAddRecExpr` stores operands: start, then one or more step components for affine/nested forms, plus the `Loop *`. Affineness matters: `{0,+,1}` is affine; more complex addrecs appear with nested loops.

No-wrap flags on AddRecs enable stronger trip counts and LFTR. Without flags, SCEV assumes modular arithmetic for integers—counts may become `CouldNotCompute`.

## Disposition and classification

Methods such as `getLoopDisposition` classify how an expression behaves relative to a loop. Passes call:

- `isLoopInvariant(S, L)`
- `hasComputableLoopEvolution(S, L)` — true for nice AddRecs in L

Invariant address of a GEP with invariant indices → LICM-friendly. Computable evolution → indvars/vectorize interest.

## `getSCEV` pipeline (mental model)

1. Look up Value in cache.
2. Switch on instruction opcode.
3. For PHI: try create AddRec from loop backedge pattern.
4. For add/mul/…).
5. For GEP: combine pointer base SCEV with scaled indices (pointer SCEVs).
6. Default: `scUnknown`.

Loads are usually Unknown unless a loop-invariant load is recognized specially in some paths—do not assume load folding.

## Expander

`SCEVExpander` maintains an IRBuilder insertion point and expands:

- Constants → `ConstantInt`
- Unknown → reuse the Value
- Add/Mul → emit ops (with flags when safe)
- AddRec → often emit PHI + add in the loop, or closed form at exits
- Casts → emit trunc/zext/sext

IndVarSimplify and LoopUnroll use expander for trip-count materialization. Expansion can create IR that itself should not be re-analyzed carelessly (expander has modes to avoid rewriting the same IV twice).

## Poison and flags

When folding `add nsw`, SCEV may keep NSW on the expression. Consumers that expand must not introduce NSW if the SCEV lacked it. Trip-count math (Day 35) carefully uses flags when converting `icmp` predicates into ranges.

`scSequentialUMinExpr` exists to model sequential (short-circuit) style umin that interacts with poison differently from ordinary umin—advanced reading.

## Invalidation

`ScalarEvolutionAnalysis` result is invalidated by the pass manager when required analyses say so, or when a pass explicitly forgets values (`SE->forgetValue`, `forgetLoop`). Loop passes that rewrite IVs must notify SCEV.

## Printer

```bash
opt -passes='print<scalar-evolution>' -disable-output f.ll
```

Prints per-loop classifications and Class/SCEV for instructions. Prefer this over `opt -analyze -scalar-evolution`.

## What not to expect

- Float AddRecs
- Accurate modeling of integer division beyond limited `scUDivExpr` cases
- Full AA (SCEV is not alias analysis; LAA uses SCEV *plus* AA)

## Reading path

1. `enum SCEVTypes`
2. `getAddRecExpr` / PHI handling
3. One trip-count function signature in the header (warm-up for Day 35)
4. `SCEVExpander::expandCodeFor`
5. Examples file

## Pointer SCEV and DataLayout

Pointer AddRec steps are in bytes according to DataLayout. An `i32` GEP step is not abstractly “+1 element” inside SCEV—it is `+4` (or whatever DL says). Wrong DataLayout in `opt` experiments produces wrong steps and broken idiom/vectorize decisions. Copy the `target datalayout` line from clang’s IR into hand-written tests.

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

## Source navigation checklist (Day 34 DeepDive)

1. Open the primary .cpp linked from the main note.
2. Jump to DEBUG_TYPE / cl::opt block.
3. Find the public `run` method or analysis entry.
4. Find the core legality helper and the core transform helper.
5. Grep remark strings / LLVM_DEBUG strings you saw experimentally.
6. Note which analyses are required vs preserved.
7. Skim one Failure path and one Success path to completion.

## Pointer SCEV and DataLayout

Pointer AddRec steps are in bytes according to DataLayout. An `i32` GEP step is not abstractly “+1 element” inside SCEV—it is `+4` (or whatever DL says). Wrong DataLayout in `opt` experiments produces wrong steps and broken idiom/vectorize decisions. Copy the `target datalayout` line from clang’s IR into hand-written tests.

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

## Source navigation checklist (Day 34 DeepDive)

1. Open the primary .cpp linked from the main note.
2. Jump to DEBUG_TYPE / cl::opt block.
3. Find the public `run` method or analysis entry.
4. Find the core legality helper and the core transform helper.
5. Grep remark strings / LLVM_DEBUG strings you saw experimentally.
6. Note which analyses are required vs preserved.
7. Skim one Failure path and one Success path to completion.

## Pointer SCEV and DataLayout

Pointer AddRec steps are in bytes according to DataLayout. An `i32` GEP step is not abstractly “+1 element” inside SCEV—it is `+4` (or whatever DL says). Wrong DataLayout in `opt` experiments produces wrong steps and broken idiom/vectorize decisions. Copy the `target datalayout` line from clang’s IR into hand-written tests.

## IR hygiene for experiments

Always generate IR with `clang -S -emit-llvm -O1 -Xclang -disable-llvm-passes` so functions are not marked `optnone`. Opaque pointers (`ptr`) are mandatory in modern IR examples; do not revive typed pointer syntax in new notes.

When comparing before/after, prefer `diff -u` on files produced by the same opt pipeline with one pass removed. That isolates the transform better than comparing against clang -O3, which runs dozens of passes.
