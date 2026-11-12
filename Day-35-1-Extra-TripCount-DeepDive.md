# Day 35 Extra: Trip Counts — Deep Dive

All in [ScalarEvolution.cpp](https://github.com/llvm/llvm-project/blob/main/llvm/lib/Analysis/ScalarEvolution.cpp) / [ScalarEvolution.h](https://github.com/llvm/llvm-project/blob/main/llvm/include/llvm/Analysis/ScalarEvolution.h). Parent: [Day-35-0-SCEV-TripCount.md](Day-35-0-SCEV-TripCount.md).

## Diagram

```text
   for (i = 0; i < n; ++i)        trip count = n
                                  backedge taken count = n-1
                                  (the latch runs n-1 times
                                   if n >= 1; 0 times if n == 0)

   SCEV:
     backedge-taken count = umax(n, 1) - 1
     or, with a guard:      n - 1

   a data-dependent exit
     if (a[i] == 0) break;
   has no computable trip count
```

The trip count is how many times the header executes. The backedge-taken count is how many times the latch branches back, which is one less when the loop runs at least once. ScalarEvolution computes the backedge count from the exit predicate. A loop with a break that depends on loaded memory has an unknown count, and the vectorizer must then version or give up.

## API map

| Method | Returns |
|--------|---------|
| `getBackedgeTakenCount(L)` | Exact BTC if known; else max/sentinel strategies via overloads |
| `getBackedgeTakenCount(L, Exact/ConstantMaximum/SymbolicMaximum/…)` | Flavor selection |
| `getPredicatedBackedgeTakenCount` | BTC + predicate set |
| `getExitCount(L, ExitingBlock)` | How many times control can pass this exit’s test |
| `getSmallConstantTripCount` | Small *trip* count when constant-ish |
| `getSmallConstantMaxTripCount` | Small maximum |
| `getSymbolicMaxBackedgeTakenCount` | Symbolic upper bound |

Read the header comments for each— they carefully distinguish trip vs backedge.

## `computeExitLimit`

Given a loop and an exiting block:

1. Inspect the terminator’s condition.
2. Handle logical combinations of conditions with caching (`ExitLimitCacheTy`) to avoid exponential blow-up on huge boolean trees.
3. For a single `ICmpInst`, call `computeExitLimitFromICmp`.
4. Produce an `ExitLimit` structure: exact SCEV, max SCEV, predicates, etc.

This is the workhorse behind public getters.

## `howManyLessThans` / `howManyGreaterThans`

Signature themes (see header): take LHS/RHS SCEVs, the loop, whether the predicate is signed, and whether equality is included (`lt` vs `le`).

For AddRec `{0,+,1}` vs invariant `n` with `slt`:

- Number of times true is `n` when `n>0` (trip), with BTC `n-1`.

The functions deal with:

- Positive vs negative steps
- Wrap flags
- Non-affine RHS (give up)

When studying, pick one signed-less-than path and follow it line-by-line with a concrete example on paper.

## Predicates

Predicated answers might require:

- No signed overflow of the IV
- `n >= 0`
- Non-zero step

Clients such as LoopVectorize can emit runtime assumes/checks matching those predicates; IndVarSimplify LFTR may refuse if predicates cannot be discharged.

## Multiple exits in practice

```c
for (i = 0; i < n; ++i) {
  if (a[i] == 0) break;
}
```

Exit A: `i < n` fails. Exit B: early break. BTC is min of (limit from `n`, limit from break)—break is usually data-dependent → `CouldNotCompute` or max-only.

## Zero-trip and guards

`for (i=0; i<n; ++i)` lowered without a guard still has a header that tests `i<n` first—BTC formulas often use:

- `umax(n, 1) - 1` style constructions, or
- predicated “valid if n > 0”

After loop-rotate, the scalar count for the do-while body is easier to interpret as “iterations of the body.”

## Small constant helpers

`getSmallConstantTripCount` exists so unrollers can cheaply ask “is this 8?” without parsing huge symbolic SCEVs. It returns 0 when not a small constant (check header for the sentinel convention in your version—read carefully before relying on “0 means unknown”).

## Poison

If LFTR or unroll assumes BTC `n-1` but `n` is poison, bad things happen. SCEV’s predicated forms and freeze/assume interactions in clients exist for this reason. When teaching, connect to Day 28’s freeze discussion: counts feeding branches need definedness.

## Debugging

```bash
opt -debug-only=scalar-evolution -passes='print<scalar-evolution>' -disable-output f.ll
```

Look for exit-limit computation messages. Then modify the loop (add `nsw`, change `i8`, add a second exit) and watch the printed BTC change.

## Reading path

1. Header comments for BTC vs trip.
2. `computeExitLimitFromICmp`.
3. `howManyLessThans` for `{0,+,1}` vs `n`.
4. `getPredicatedBackedgeTakenCount` call sites in LoopVectorize (optional).
5. Examples file.

## Off-by-one drill

Say out loud for `for (i=0;i<n;++i)` with `n=5`: header runs 5 times; backedge taken 4 times; last IV value before exit depends on pre/post-inc form. Then write the SCEV BTC and trip count. Repeat for `i<=n` and for countdown loops. Most LFTR bugs are off-by-one bugs.

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

## Source navigation checklist (Day 35 DeepDive)

1. Open the primary .cpp linked from the main note.
2. Jump to DEBUG_TYPE / cl::opt block.
3. Find the public `run` method or analysis entry.
4. Find the core legality helper and the core transform helper.
5. Grep remark strings / LLVM_DEBUG strings you saw experimentally.
6. Note which analyses are required vs preserved.
7. Skim one Failure path and one Success path to completion.

## Off-by-one drill

Say out loud for `for (i=0;i<n;++i)` with `n=5`: header runs 5 times; backedge taken 4 times; last IV value before exit depends on pre/post-inc form. Then write the SCEV BTC and trip count. Repeat for `i<=n` and for countdown loops. Most LFTR bugs are off-by-one bugs.

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

## Source navigation checklist (Day 35 DeepDive)

1. Open the primary .cpp linked from the main note.
2. Jump to DEBUG_TYPE / cl::opt block.
3. Find the public `run` method or analysis entry.
4. Find the core legality helper and the core transform helper.
5. Grep remark strings / LLVM_DEBUG strings you saw experimentally.
6. Note which analyses are required vs preserved.
7. Skim one Failure path and one Success path to completion.
