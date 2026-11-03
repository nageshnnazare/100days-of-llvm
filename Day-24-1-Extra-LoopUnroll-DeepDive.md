# Day 24 Extra: Loop Unroll — Deep Dive

Companion to [Day-24-0-LoopUnroll.md](Day-24-0-LoopUnroll.md). Examples: [Day-24-2-Extra-LoopUnroll-Examples.md](Day-24-2-Extra-LoopUnroll-Examples.md).

## Diagram

```text
   for (i = 0; i < 4; i++)           i = 0
     a[i] = i;                       a[0] = 0
                                     a[1] = 1
   unroll count 4, no remainder:     a[2] = 2
                                     a[3] = 3

   for (i = 0; i < n; i++)           if n is not a multiple of 4:
     a[i] = i;                       a prologue or epilogue loop
                                     runs the leftover iterations,
                                     and the main loop steps by 4
```

Full unrolling copies the body and deletes the branch when the trip count is a small constant. Runtime unrolling peels off a remainder so the unrolled body can assume a multiple of the unroll factor. The pass estimates code size against a threshold; a large body with a huge trip count stays rolled.

## 1. Two layers

**LoopUnrollPass** answers: *should* we unroll, by how much, full/partial/runtime/peel?  
**UnrollLoop / UnrollRuntimeLoopRemainder** answer: *how* to rewrite CFG and SSA.

Reading only LoopUnroll.cpp misses why a loop stayed rolled. Reading only the pass misses clone/remainder mechanics.

## 2. `UnrollingPreferences` and cl::opt

`gatherUnrollingPreferences` in LoopUnrollPass.cpp:

1. Fill UP with historic defaults (thresholds, `DefaultUnrollRuntimeCount`, etc.).
2. Call `TTI.getUnrollingPreferences`.
3. Apply command-line overrides when the option was explicitly set (`getNumOccurrences()` pattern for `-unroll-runtime`, threshold, count, …).

Important knobs (names as in source):

- `unroll-threshold` / `unroll-threshold-default` / `unroll-threshold-aggressive`
- `unroll-count`
- `unroll-runtime`
- `pragma-unroll-threshold`
- `pragma-unroll-full-max-iterations`

Exact default integers drift; read the `cl::init` values in your tree rather than memorizing numbers from old blogs.

## 3. Pragma priority inside `computeUnrollCount`

The decision function documents an ordered search (debug strings like “Trying pragma unroll…”, “Trying loop peeling…”, “Trying runtime unroll…”). Conceptually:

1. User `-unroll-count` and explicit pragma counts.
2. `llvm.loop.unroll.full` / enable metadata → aggressive full or enable runtime.
3. Cost-limited partial for constant trips.
4. Peeling preferences (`PeelingPreferences` / `PP.PeelCount`).
5. Runtime unroll if `UP.Runtime` and not `llvm.loop.unroll.runtime.disable`.

Pragma enable tends to force runtime allowance and relax expensive trip-count checks so the compiler tries harder to honor the hint.

## 4. `UnrollLoopOptions`

From [UnrollLoop.h](https://github.com/llvm/llvm-project/blob/main/llvm/include/llvm/Transforms/Utils/UnrollLoop.h):

- `Count` — copies of the body
- `Force` — bypass some profitability (still not magic legality)
- `Runtime` — expect remainder path interplay
- `AllowExpensiveTripCount`
- `UnrollRemainder` — whether to unroll the remainder loop too
- `ForgetAllSCEV` — SCEV invalidation aggressiveness
- `SCEVExpansionBudget`
- `RuntimeUnrollMultiExit`
- `Heart` — optional convergence/heart instruction (special targets)

`LoopUnrollResult`: `Unmodified`, `PartiallyUnrolled`, `FullyUnrolled`.

## 5. Preconditions inside `UnrollLoop`

Early exits:

- No preheader / no latch
- `!isSafeToClone()` (e.g. `indirectbr`)
- Header address-taken (loop-rotate often helps)

Exiting-block analysis builds `ExitInfo` with SCEV small constant trip counts / multiples / breakout trip (`TripCount % Count`) to know which unrolled copy exits.

`CompletelyUnroll` when `Count == MaxTripCount`. Full unroll disables the runtime flag. Exit PHIs imply `NeedToFixLCSSA` so outer LCSSA can be rebuilt after the loop disappears.

## 6. Runtime remainder

`UnrollRuntimeLoopRemainder` (LoopUnrollRuntime.cpp) invents control that branches to:

- An unrolled main loop with trip count divisible by `Count`, and
- A remainder loop for the leftover iterations,

or a prologue form depending on options (`UseEpilogRemainder`). Only then does `UnrollLoop` clone the main body `Count` times.

Multi-exit runtime unroll is gated (`RuntimeUnrollMultiExit`) because remainder construction is harder with several exits.

## 7. Peeling vs unrolling

`peelLoop` from LoopPeel.h runs when heuristics set `PP.PeelCount` and unroll count stays 1. Peeling helps:

- Misaligned early iterations before a vectorizable main loop,
- Provable dead first-iteration cold paths,
- Canonicalizing conditions for LICM/unswitch later.

Cannot combine explicit peel count with conflicting unroll strategies in one step—the pass `reportFatalUsageError`s on bad test flag combinations.

## 8. LCSSA and simplify after unroll

`PreserveLCSSA` is threaded through. After clones, `simplifyLoopAfterUnroll` cleans IVs and branches. Full unroll that orphaned exit PHIs triggers LCSSA repair on the outer loop (`formLCSSA` / simplify with preserve flag)—see comments near `NeedToFixLCSSA` in LoopUnroll.cpp.

## 9. Unroll-and-jam boundary

If the loop or parent carries `llvm.loop.unroll_and_jam`, ordinary unroll may deliberately skip so the jam pass can apply its dependency-checked transform (`UnrollAndJamLoop` also declared in UnrollLoop.h but driven elsewhere).

## 10. Convergence / GPU notes

`UnrollCostEstimator` tracks convergence kinds and whether runtime unroll with remainder is safe when threads must agree on trip count. Skip on first CPU-centric read; know the field exists before debugging SPIR-V/GPU unroll bugs.

## 11. Debug practice

```bash
opt -debug-only=loop-unroll -passes='loop-simplify,lcssa,loop-unroll' -S f.ll -o /dev/null
```

Watch messages for trip counts, chosen strategy, threshold rejection, pragma paths, and “Can't unroll; …” preconditions.

## 12. Study checklist

- State the five: full, partial, runtime+remainder, peel, unroll-and-jam.
- Point to `getUnrollingPreferences` and one pragma MD kind.
- Explain why LoopSimplify+LCSSA are assumed.
- Find `UnrollLoop`’s preheader/latch checks in LoopUnroll.cpp.
- Know remainder IR is expected for runtime success.

## 13. Exit folding after clones

When trip count is known, exit branches in early unrolled copies can become `br label %next_copy` (never exit), and only the last copy keeps a real exit—or for partial unroll, exits recur every `BreakoutTrip` pattern. Reading `ExitInfos` population (TripCount, TripMultiple, BreakoutTrip) makes those folds less mysterious.

## 14. `PreserveOnlyFirst` / MaxOrZero

When completely unrolling a loop that SCEV says is “max or zero” trip, special casing preserves only the first iteration’s side-effect path carefully—zero-trip vs max-trip ambiguity. Skim references to `PreserveOnlyFirst` when debugging full-unroll miscompiles on weird trip counts.

## 15. Follow-up metadata

`LLVMLoopUnrollFollowupAll`, `FollowupUnrolled`, `FollowupRemainder` in UnrollLoop.h mark loops created for later passes (vectorize, further unroll). After runtime unroll, the remainder loop may carry different metadata than the main unrolled loop so heuristics do not thrash.

## 16. Interaction with LCSSA repair

Full unroll removes the loop; outer code may still have PHIs that assumed LCSSA of the vanished loop. `NeedToFixLCSSA` triggers recomputation for the outer loop. If you write a custom unroll, either preserve exit PHIs correctly or call `formLCSSA` on the parent.

## 17. Testing knobs vs production

`-unroll-count` is explicitly documented for testing / overriding pragmas. Production Clang relies more on TTI + PGO + pragmas. When reproducing a bug, record whether the count came from cl::opt, pragma, or heuristic—the priority order differs.

## 18. Peeling preferences struct

Parallel to `UnrollingPreferences`, peeling has its own preferences (count, peel-last, etc.) filled before `computeUnrollCount` chooses peel vs unroll. Reading only UP misses why a loop peeled three times and stopped.

## 19. Multi-exit runtime limits

Runtime remainder construction historically assumed simple exit structure. `RuntimeUnrollMultiExit` opts into harder cases; default may refuse multi-exit runtime unroll. If a loop has several breaks, do not expect `-unroll-runtime` to always succeed.

## 20. Exercises

1. Fully unroll `add4`; show latch gone.
2. Runtime-unroll `addn` with count 4; mark main vs remainder in the CFG.
3. Add `#pragma clang loop unroll(full)` and find metadata in `.ll`.
4. Set `-unroll-threshold=1` and show refusal.
5. Grep `Trying runtime unroll` in `-debug-only=loop-unroll` output.

## 21. Result enum reminder

After a successful transform, interpret `LoopUnrollResult`:

- `FullyUnrolled` — loop gone as a Cycle.
- `PartiallyUnrolled` — still a loop (and maybe a remainder loop sibling).
- `Unmodified` — policy or legality refused; IR unchanged.
