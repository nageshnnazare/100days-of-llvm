# Day 25 Extra: Loop Deletion — Deep Dive

Companion to [Day-25-0-LoopDeletion.md](Day-25-0-LoopDeletion.md). Examples: [Day-25-2-Extra-LoopDeletion-Examples.md](Day-25-2-Extra-LoopDeletion-Examples.md). Source: [LoopDeletion.cpp](https://github.com/llvm/llvm-project/blob/main/llvm/lib/Transforms/Scalar/LoopDeletion.cpp).

## Diagram

```text
   header:
     %c = icmp slt i32 %i, %n
     br i1 %c, label %body, label %exit
   body:
     ; no side effects, no volatile, no escape
     br label %header
   exit:
     ret i32 0

   mustprogress is set, trip count may be infinite
   only if the loop never exits -- but a side-effect-free
   loop that the language says must make progress
   can be deleted:

   exit:
     ret i32 0
```

LoopDeletion removes a loop that does nothing observable. A finite loop whose body has no side effects disappears and control goes to the exit. An infinite loop is deleted only when the function is `mustprogress` (C++ forward-progress). Without that attribute the infinite loop stays, because deleting it would let later code run when the source never did.

## 1. Pass structure

`LoopDeletionPass::run`:

1. Log the loop (`DEBUG_TYPE "loop-delete"`).
2. `Result = deleteLoopIfDead(...)`.
3. If not deleted, merge with `breakBackedgeIfNotTaken(...)`.
4. On `Deleted`, `Updater.markLoopAsDeleted`.
5. Preserve standard loop analyses + MemorySSA if present.

`LoopDeletionResult` is `Unmodified | Modified | Deleted` with a small `merge` helper (`Modified` can occur when `makeLoopInvariant` changed IR but deletion still failed).

## 2. LCSSA as the live-out contract

Comment in `isLoopDead`: because of LCSSA, values used outside must pass through exit PHIs. Therefore checking that every exit PHI’s incoming from the loop is invariant (and identical across exiting blocks) is enough to know the loop does not feed variant data to the outside.

Multiple different incoming values from different exiting blocks → cannot statically know which value the deleted loop “would have” produced → refuse.

## 3. Side-effect scan

```cpp
I.mayHaveSideEffects() && !I.isDroppable()
```

Any such instruction kills deletion. That covers:

- Stores (escaping / observable memory)
- Volatile loads/stores
- Calls that may write, throw, not return, etc., as modeled by attributes

A call that is `readnone`/`readonly`+`willreturn`+`nounwind` (as reflected in IR attributes so `mayHaveSideEffects` is false) can appear in a “dead” loop. Do not memorize attribute spellings from old notes—check the IR’s `mayHaveSideEffects` behavior.

## 4. mustprogress mechanics in code order

After the side-effect scan succeeds:

```text
if (function.mustProgress())
  return true;  // SEF => deletable even if infinite
```

Else walk the loop nest:

- If `hasMustProgress(Current)` on a subloop, that subloop is OK (progress required there).
- Else `SE.getConstantMaxBackedgeTakenCount(Current)`; if `SCEVCouldNotCompute`, return false.
- Also reject irreducible CFG in the nest when relying on this path (`containsIrreducibleCFG`)—irreducible cycles may loop forever without a SCEV trip count.

`hasMustProgress` consults loop metadata such as `llvm.loop.mustprogress`. Function-level `mustprogress` is the broad hammer Clang uses heavily for C++.

### Language intuition

- **Without** mustprogress: a thread may spin in an empty loop forever; removing it changes termination → illegal as an “optimization.”
- **With** mustprogress: the abstract machine requires forward progress; an infinite SEF loop is UB / invalid; deleting it is allowed.

## 5. Never-executed recognition

`isLoopNeverExecuted` is deliberately local/cheap:

- Preheader must not be the entry block.
- Every pred of the preheader must be `br i1 Const, ...` where the **taken** successor is not the preheader.

No full constant propagation across the function—only obvious constant branches. That still catches `if (0) { for (...) }` patterns after SimplifyCFG-ish constant conditions land on predecessors.

On success, exit PHIs become poison (no defined incoming), then `deleteDeadLoop`.

## 6. Symbolic execution of the first iteration

`canProveExitOnFirstIteration` (gated by `-loop-deletion-enable-symbolic-execution`, default true):

1. RPOT over the loop; bail on irreducible CFG.
2. Track which blocks/edges are live on iteration 0.
3. Fold PHIs to the sole live incoming; simplify binops/icmps/selects via `SimplifyQuery`.
4. For br/switch, if the condition simplifies to a constant, mark only that successor live.
5. If the latch→header edge is never live, the backedge is untaken on the first iteration → safe to break.

This deletes one-trip-never-backedge cases SCEV might still phrase awkwardly.

## 7. `breakBackedgeIfNotTaken`

Uses SCEV:

- If max BTC is zero, or BTC is zero, or BTC known non-zero fails and first-iter proof succeeds → `breakLoopBackedge`.

Leaves invariant control/side effects in place but removes the cycle—often a prelude to further simplify/deletion.

## 8. Unique exit limitation

Deletion of invariant loops wants a single exit block so the preheader can branch straight there. Multiple exits would require preserving the exit-choice logic as loop-invariant control—out of scope for this pass’s simple model (`!ExitBlock && !LI.hasNoExitBlocks` path).

EH pad exits are refused (`isEHPad`).

## 9. Interaction with other passes

- **Indvars** may replace exit PHIs with closed forms → easier invariant checks.
- **LICM** may hoist everything until the loop is empty SEF.
- **ADCE** after deletion cleans poisoned/unused values.
- Running deletion **before** LCSSA is wrong for this implementation (asserts `isLCSSAForm`).

## 10. Remarks

Optimization remarks:

- `NeverExecutes` — “Loop deleted because it never executes”
- `Invariant` — “Loop deleted because it is invariant”

Useful when `-pass-remarks=loop-delete` (or remark flags matching `DEBUG_TYPE`) is enabled.

## 11. Statistics

- `NumDeleted` — loops removed via `deleteDeadLoop`
- `NumBackedgesBroken` — latch broken path

## 12. Study checklist

- Recite the mustprogress table without flipping infinite cases.
- Point to `isLoopDead`’s side-effect loop and SCEV/`mustProgress` tail.
- Explain poison on never-executed exit PHIs + `SE.forgetLoop`.
- Know debug type spelling: `loop-delete`.
- Separate “delete whole loop” from “break backedge only.”

## 13. `makeLoopInvariant` side path

Before giving up, `isLoopDead` tries `L->makeLoopInvariant` on PHI incomings. That can hoist trivial ops to the preheader (`Changed = true`) even when the loop is not deleted—hence `LoopDeletionResult::Modified`. When reading tests that expect a hoist but not deletion, this is why.

## 14. Irreducible CFG

Both the mustprogress-fallback path and first-iteration symbolic exec refuse irreducible nests. Irreducible cycles break RPOT “predecessors already visited” assumptions and can spin without a SCEV trip count. If deletion mysteriously fails on goto-heavy IR, check `containsIrreducibleCFG`.

## 15. Subloops and recursive progress

The worklist over subloops ensures an outer SEF loop is not deleted while an inner loop might still be an infinite non-mustprogress SEF loop. Parent deletion would remove the child’s infinite spin incorrectly. Each level needs mustprogress or known max BTC.

## 16. Poison vs undef historically

Never-executed deletion fills exit PHI incomings with `PoisonValue`. Older LLVM used undef more liberally; modern notes should say poison. Downstream users of those PHIs are already on dead paths if the loop never ran—but SCEV still must forget the loop first.

## 17. Option: symbolic execution toggle

`-loop-deletion-enable-symbolic-execution` (default true) gates `canProveExitOnFirstIteration`. Disabling it is a useful bisection tool when a backedge break miscompiles or you want only SCEV-zero BTC breaks.

## 18. How this differs from ADCE

ADCE removes dead *instructions* but will not necessarily dismantle a loop that still has a backedge and “live” empty control. LoopDeletion understands loop structure, SCEV, and mustprogress. Both may run; deletion is the loop-aware hammer.

## 19. Remark-driven learning

Build with remarks enabled and delete Example 1–2; match remark ids `NeverExecutes` vs `Invariant` to the code paths. That locks the mental model faster than reading only the CFG diff.

## 20. Exercises

1. Empty loop with/without `mustprogress` in hand-written IR—compare.
2. `if (0) for(...)` never-executed path.
3. Loop calling an external function—show refusal.
4. Force BTC zero with a constant-false latch; observe backedge break.
5. Read `deleteDeadLoop` in LoopUtils once so CFG edits are not magic.

## 21. One-line mustprogress rule

**Infinite side-effect-free loops are deletable only when progress is required (`mustprogress` on the function or loop); otherwise they must stay.**
