# Day 29 Extra: IndVarSimplify — Deep Dive

File: [IndVarSimplify.cpp](https://github.com/llvm/llvm-project/blob/main/llvm/lib/Transforms/Scalar/IndVarSimplify.cpp). Parent: [Day-29-0-IndVarSimplify.md](Day-29-0-IndVarSimplify.md).

## Diagram

```text
   %i = phi i32 [ 0, %pre ], [ %i.next, %latch ]
   %i.next = add nsw i32 %i, 1
   %cmp = icmp slt i32 %i, %n

   indvars rewrites users of %i onto a
   canonical induction variable, and
   widens a narrow IV when the arithmetic
   is legal:

   %i64 = phi i64 [ 0, %pre ], [ %next, %latch ]
   %next = add nuw nsw i64 %i64, 1
```

IndVarSimplify picks one induction variable as canonical and rewrites the others in terms of it. Strength reduction turns `i * scale` into a recurrence that only adds. Trip-count compares become compares against the canonical IV. Widening a 32-bit IV to 64-bit is done only when the extra bits cannot change the defined result; a wrapping IV is left narrow.

## DEBUG and options

`#define DEBUG_TYPE "indvars"`.

Notable options:

- `disable-lftr` — turn off LFTR entirely.
- `indvars-predicate-loops` — internal predication-related behavior in this pass (default true); **not** `loop-predication`.

Statistic `NumLFTR` counts successful LFTR rewrites.

## Why SCEV is mandatory

Every profitability and legality decision asks SCEV:

- Is this PHI an AddRec for loop L?
- Are two IVs congruent (same AddRec)?
- What is the exit value of this AddRec at the latched exit?
- What is the backedge-taken count for LFTR’s limit?

Without a computable SCEV, indvars leaves the instruction alone. That is why opaque or wrapping-hard IVs survive.

## Widening walkthrough

Widening replaces a narrow IV with a wider one when:

- All in-loop uses are extends to the wide type (or can be rewritten).
- Wide arithmetic does not introduce poison beyond what the original allowed.
- Data layout / legal integer widths support the wide type.

Implementation carefully recreates the PHI and updates GEP indices. Afterward, `sext i32 %i to i64` in the loop body becomes dead.

## Congruent IV elimination

If two PHIs have the same SCEV AddRec, one can replace the other. Redundant `icmp`s implied by the same AddRec relations are folded. This shrinks the set of live IVs before LFTR.

## `rewriteLoopExitValues`

Calls into utility logic (same family as `llvm::rewriteLoopExitValues`) with a `SCEVExpander` configured with `setDebugType(DEBUG_TYPE)`.

For each LCSSA phi of a loop-defined value:

1. Ask SCEV for the value at the exit (`getSCEVAtScope` / exit-value helpers).
2. If expandable cheaply and safely, expand in the exit block and replace the phi.
3. Respect expansion cost / TTI so huge expressions are not pasted blindly.

This often turns “return the final `i`” into “return `n`” when the exit condition was `i == n`.

## LFTR deep dive

### `needsLFTR`

Returns true when the exiting branch’s condition is not already the ideal simple counter compare—e.g., comparing a non-counter IV, or an icmp that is not EQ/NE against a clean limit. The pass skips exiting blocks that are already LFTR-friendly.

### Performing LFTR

High-level:

1. Pick a **counter** IV suitable for the exit test (helpers select among candidates).
2. Compute the limit from the trip count / exit count SCEV.
3. Expand the limit with SCEVExpander if needed.
4. Replace the exiting `icmp` with one on the counter vs limit (pre-inc or post-inc form carefully chosen).
5. Ensure no new poison: switching to an IV that wraps differently, or adding `nsw` incorrectly, is forbidden.

Comments in the LFTR section discuss:

- Pre-inc vs post-inc checks.
- Avoiding LFTR when it would make a dynamically dead IV suddenly drive the exit in a poison-visible way.
- Single-backedge assumptions for some cases.

### Interaction with strength reduction

Once the exit only mentions the canonical counter, pointer IVs used solely for the old compare can be deleted or rewritten as GEPs off a base—setting up LSR.

## Predication option vs LoopPredication pass

Search `LoopPredication` / `indvars-predicate-loops` in IndVarSimplify.cpp to see the local meaning (guarding some rewrites). Separately, [LoopPredication.cpp](https://github.com/llvm/llvm-project/blob/main/llvm/lib/Transforms/Scalar/LoopPredication.cpp) (pass name `loop-predication`) hoists checks using assumes/guards. When filing bugs or reading Phabricator history, check which binary you mean.

## Safe ordering relative to other loop passes

Typical: rotate → (unswitch) → **indvars** → idiom / vectorize / unroll. Indvars likes rotated, simplified loops with analyzable exits. Running indvars too early on messy CFG yields fewer SCEV hits.

## Debugging session recipe

```bash
opt -debug-only=indvars -passes='loop-simplify,lcssa,indvars' -S in.ll 2> indvars.log
```

Correlate log lines with LFTR / widen / exit rewrite. Then `print<scalar-evolution>` before and after to see AddRec simplification in the IR users.

## Common student misconceptions

1. “Indvars always deletes all sexts” — only when widen proves safe.
2. “LFTR always uses `n-1`” — form depends on pre/post-inc and predicate.
3. “Exit rewrite is GVN” — it is SCEV closed form, not value numbering.
4. “`-indvars-predicate-loops` enables loop-predication” — false.

## Exit value expansion costs

`rewriteLoopExitValues` consults TTI/cost heuristics so a huge SCEV expression is not pasted into every exit. Sometimes leaving an LCSSA phi is better than expanding a forest of adds/muls/udivs. When an exit value “should” be `n` but remains a phi, check whether expansion was deemed too expensive or unsafe.

## Single backedge assumption

Comments note LFTR relies on having a single backedge in some paths. LoopSimplify normally provides that. If you run indvars without simplify, LFTR may skip. Always include `loop-simplify,lcssa` in experimental pipelines.

## SCEVExpander debug type

The pass sets the expander’s debug type to `indvars` so `-debug-only=indvars` shows expansion chatter too. That is handy when distinguishing “SCEV had no answer” from “expander refused.”

## Ordering versus idiom and unswitch

Unswitch first can remove invariant branches so SCEV sees a simpler loop for indvars. Idiom after indvars can see cleaner AddRecs for memset pointers. Exact O3 order evolves; the dependency idea stays: **SCEV-friendly IR in, simpler IVs out**.

### Additional study points (1)

`rewriteLoopExitValues` consults TTI/cost heuristics so a huge SCEV expression is not pasted into every exit. Sometimes leaving an LCSSA phi is better than expanding a forest of adds/muls/udivs. When an exit value “should” be `n` but remains a phi, check whether expansion was deemed too expensive or unsafe.
Comments note LFTR relies on having a single backedge in some paths. LoopSimplify normally provides that. If you run indvars without simplify, LFTR may skip. Always include `loop-simplify,lcssa` in experimental pipelines.
The pass sets the expander’s debug type to `indvars` so `-debug-only=indvars` shows expansion chatter too. That is handy when distinguishing “SCEV had no answer” from “expander refused.”
Unswitch first can remove invariant branches so SCEV sees a simpler loop for indvars. Idiom after indvars can see cleaner AddRecs for memset pointers. Exact O3 order evolves; the dependency idea stays: **SCEV-friendly IR in, simpler IVs out**.

```bash
# revisit with debug (session 1)
opt -passes='loop-simplify,lcssa' -S in.ll -o /tmp/in.simp.ll
```

### Additional study points (2)

`rewriteLoopExitValues` consults TTI/cost heuristics so a huge SCEV expression is not pasted into every exit. Sometimes leaving an LCSSA phi is better than expanding a forest of adds/muls/udivs. When an exit value “should” be `n` but remains a phi, check whether expansion was deemed too expensive or unsafe.
Comments note LFTR relies on having a single backedge in some paths. LoopSimplify normally provides that. If you run indvars without simplify, LFTR may skip. Always include `loop-simplify,lcssa` in experimental pipelines.
The pass sets the expander’s debug type to `indvars` so `-debug-only=indvars` shows expansion chatter too. That is handy when distinguishing “SCEV had no answer” from “expander refused.”
Unswitch first can remove invariant branches so SCEV sees a simpler loop for indvars. Idiom after indvars can see cleaner AddRecs for memset pointers. Exact O3 order evolves; the dependency idea stays: **SCEV-friendly IR in, simpler IVs out**.

```bash
# revisit with debug (session 2)
opt -passes='loop-simplify,lcssa' -S in.ll -o /tmp/in.simp.ll
```

## LFTR and poison checklist

Before trusting an LFTR rewrite in a patch:

1. Does the new compare’s IV already execute on all paths to the latch?
2. Are nsw/nuw flags on the IV update justified?
3. Could poison of the old condition differ from poison of the new one?
4. Is the limit expansion itself poison-free under the loop predicates?

If any answer is cloudy, SCEV/indvars should leave the test alone. That conservatism is intentional.

## IR hygiene for experiments

Always generate IR with `clang -S -emit-llvm -O1 -Xclang -disable-llvm-passes` so functions are not marked `optnone`. Opaque pointers (`ptr`) are mandatory in modern IR examples; do not revive typed pointer syntax in new notes.

When comparing before/after, prefer `diff -u` on files produced by the same opt pipeline with one pass removed. That isolates the transform better than comparing against clang -O3, which runs dozens of passes.

## Verifier and analysis invalidation

After a transform, `opt -passes='verify'` (or the verifier embedded in pipelines) should still succeed. Loop passes must preserve LCSSA and simplified form when they advertise those preservations. Analysis managers invalidate SCEV, AA, and MemorySSA according to the preserved-analyses set returned by the pass—reading `PreservedAnalyses` in the pass’s `run` method teaches what clients may keep.
