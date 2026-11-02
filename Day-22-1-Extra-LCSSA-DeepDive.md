# Day 22 Extra: LCSSA — Deep Dive

Companion to [Day-22-0-LCSSA.md](Day-22-0-LCSSA.md). Examples: [Day-22-2-Extra-LCSSA-Examples.md](Day-22-2-Extra-LCSSA-Examples.md). Source of truth: [LCSSA.cpp](https://github.com/llvm/llvm-project/blob/main/llvm/lib/Transforms/Utils/LCSSA.cpp).

## Diagram

```text
   header
     |
    body:  %i = phi ...
     |
     +----> exit:  use %i

   %i is defined inside the loop and used
   outside. That use is not in LCSSA form.

   after lcssa:

   header
     |
    body:  %i = phi ...
     |
     +----> loopexit:  %i.lcssa = phi i32 [ %i, %body ]
                          |
                          v
                        exit: use %i.lcssa
```

LCSSA adds a phi at each loop exit for values that live out. The phi has one incoming value, so it does not change the program. It gives later passes a single place to update when the loop body is rewritten: they edit the live-out phi instead of hunting for uses outside the loop.

## 1. Why “closed” loops simplify every later pass

A loop transform that duplicates the body (unroll), changes the latch (rotate), or removes the backedge (deletion) must keep SSA valid for values that escape. Without LCSSA, escape edges are unrestricted: GEPs, adds, and loads inside the loop can be named directly from `return`, from outer PHIs, or from code in a sibling branch after a multi-exit loop.

With LCSSA, the **export surface** is finite and structural: look at PHIs at the start of exit blocks. `deleteDeadLoop`, `UnrollLoop`, and LICM sink paths all lean on that.

## 2. Dominance filter before scanning instructions

Naively, every instruction in the loop might need a use scan. LCSSA instead computes `BlocksDominatingExits`: start from exit blocks, walk immediate dominators upward while staying inside the loop, stop at the header.

Intuition: if block `B` does not dominate any exit, no value defined in `B` can be used outside the loop along a path that left through those exits—dominance would be violated. Skipping those blocks is a compile-time win and matches SSA rules.

Subloop blocks are skipped when processing the outer loop: inners must already be in LCSSA (`formLCSSARecursively` order; expensive checks assert it).

## 3. `formLCSSAForInstructionsImpl` in detail

For each instruction `I` on the worklist:

1. **Resolve exit list** for `LI.getLoopFor(I)` (cached in `LoopExitBlocksTy`).
2. **Classify each use:**
   - Lifetime markers: collect; if any marker is outside the loop, drop *all* markers for that alloca-related set rather than PHI-rewrite.
   - Unreachable users: set operand to poison.
   - PHI users: treat use block as `getIncomingBlock(U)`.
   - If use block ≠ def block and not inside the loop → `UsesToRewrite`.
3. **Dominating exit insertion:** for each exit dominated by `I` (invoke: dominate from normal dest), create `%name.lcssa` PHI at the front of the exit, one incoming per predecessor (`PredIteratorCache`), seed `SSAUpdater`.
4. **Rewrite:** single PHI → direct replace; else `SSAUpdate.RewriteUse`. Uses already in an exit that begins with the new PHI get special casing because SSAUpdater assumes PHIs-at-end differently.
5. **Dbg values** outside the loop are retargeted when a value for their block is known.
6. **PostProcessPHIs:** LoopSimplify sometimes fails (e.g. `indirectbr`); an exit of `L1` can be the header of disjoint `L2`. PHIs inserted there may themselves escape `L2`—push them back on the worklist.
7. **Dead PHI cleanup:** unused inserted PHIs erased (or returned to caller via `PHIsToRemove`).

## 4. SCEV coupling

If `SE` is non-null and `I` already has a SCEV, the pass calls `SE->getSCEV(PN)` on each new LCSSA PHI. Passes that depend on BECounts tied to those values then invalidate when the PHI is removed—silent cache staleness is a historical footgun.

## 5. Pass wrappers

### New PM — `LCSSAPass::run`

Gets `LoopAnalysis` + `DominatorTreeAnalysis`, optional cached `ScalarEvolutionAnalysis`, runs `formLCSSAOnAllLoops`. On change, preserves CFG set, SCEV, MemorySSA.

### Legacy — `LCSSAWrapperPass`

Same construction; `verifyAnalysis` optionally asserts every top-level loop is recursively LCSSA when `-verify-loop-lcssa` is set. Comments warn this can be extremely expensive on loop-heavy TUs.

### `LCSSAVerificationPass`

Empty analysis used as a **marker** so the legacy `LPPassManager` knows to call `isRecursivelyLCSSAForm` after a loop pass that claims to preserve LCSSA. It does not transform IR.

## 6. Interaction with LoopSimplify

LCSSA does not require a perfect simplified loop to *run*, but dedicated exits and a single preheader make the PHI placement obvious and keep later passes happy. Pipelines almost always run `loop-simplify` before `lcssa`.

Multi-exit loops still get one PHI per dominated exit; outside uses may need SSAUpdater to pick the right exit PHI along different paths.

## 7. Trivial PHIs and InstCombine

An LCSSA PHI with a single predecessor is semantically a copy. Leaving it in place is the point of the form. Once a consumer has finished (or never needed the PHI), InstCombine folds `%x.lcssa` into `%x` when legal—often after the loop is gone or the use pattern simplified.

Do not teach “LCSSA permanently clutters IR”; teach “LCSSA is an invariant between loop passes.”

## 8. Invoke / EH edge cases

`invoke` results are not available on the unwind edge. LCSSA adjusts the dominance root to the normal destination before inserting exit PHIs. Catchswitch / token values that are live across loops on Windows EH paths are skipped when token-like.

## 9. How other passes call the util

Many transforms call `formLCSSA` / `formLCSSARecursively` / `formLCSSAForInstructions` directly after editing a loop, rather than scheduling the full `lcssa` pass again. When reading LICM, LoopUnroll, or LoopDeletion, grep those names.

## 10. Suggested reading path in the file

1. File header diagram.
2. `computeBlocksDominatingExits`.
3. `formLCSSAForInstructionsImpl` use classification + PHI insert loop.
4. `formLCSSARecursivelyImpl` ordering.
5. `LCSSAPass::run` / wrapper `verifyAnalysis`.

## 11. Mental model checklist

- Outside use? → must be exit PHI (for that loop).
- Nested loop? → close inside-out.
- Dominates exit? → PHI may be needed there.
- Trivial PHI? → OK until cleanup.
- Verify? → `isLCSSAForm`, not a mythical `verifyLCSSA()` free function.

## 12. Tie-in to Day 23–25

- LICM sinking clones/hoists values to exits and wires LCSSA PHIs.
- Unroll may fully unroll and then repair outer LCSSA (`NeedToFixLCSSA` in LoopUnroll.cpp).
- Deletion assumes LCSSA so exit PHI incoming values are the only live-outs to rewrite (often to poison or invariant).

## 13. Worked walk of one external use

Suppose `%t = add i32 %a, %b` in latch `L`, used by `ret i32 %t` in `exit`, and `L` dominates `exit` with a single predecessor edge latch→exit.

1. Worklist contains `%t`.
2. Use scan: `ret`’s parent is outside → `UsesToRewrite`.
3. For `exit`, create `%t.lcssa = phi i32 [ %t, %latch ]`.
4. Single AddedPHI → set `ret` operand to `%t.lcssa`.
5. `NumLCSSA` increments.

If a second exit `exit2` is also dominated, a second PHI appears; a use in a join block after both exits needs `SSAUpdater` to insert additional PHIs along the path—or the join already PHIs the two `.lcssa` values after simplify.

## 14. Why DominatorTree is mandatory

Without DT:

- You cannot cheaply know which exits are dominated by a def.
- You cannot skip unreachable users safely.
- `isLCSSAForm` itself takes a `DominatorTree` to validate.

LCSSA preserves CFG (`AU.setPreservesCFG()` / new-PM CFG set) but *consumes* DT as analysis.

## 15. Nested disjoint-header hazard

The `PostProcessPHIs` path exists because LoopSimplify is incomplete for some CFGs (`indirectbr`). Inserting a PHI into another loop’s header can break *that* loop’s LCSSA. The worklist revisit is not academic—leave it in your mental model when reading weird EH or irreducible tests.

## 16. What “trivial PHI” means here

Trivial = one meaningful incoming (or all incomings the same value after cleanup). Still required for the form while the loop exists. After the loop is deleted or the value is rewritten to a closed form outside, InstCombine/SimplifyCFG remove the copy. Teaching IR that still shows `.lcssa` immediately after `-passes=lcssa` is the right snapshot; full `-O2` IR may have cleaned them.

## 17. Comparison to “closed SSA” in papers

Literature sometimes defines loop-closed SSA slightly differently (e.g., requiring specific placement relative to landing pads). LLVM’s operational definition is whatever `isLCSSAForm` checks and whatever `formLCSSA*` constructs. When writing a new loop pass, match LLVM’s predicate, not a paper’s ASCII diagram.

## 18. Practical debugging recipe

1. `opt -passes='loop-simplify,lcssa' -S` and inspect exits.
2. Run your suspect pass alone; on assert, reduce with `llvm-reduce` / manual IR.
3. Enable `-verify-loop-lcssa` only on small TUs—it is expensive.
4. Call `L->isLCSSAForm(DT)` from a local debug build around the mutation site.
5. Check whether you updated uses in PHI *incoming* blocks correctly.

## 19. API cheat sheet

| API | Role |
|-----|------|
| `formLCSSA` | One loop (subs already closed) |
| `formLCSSARecursively` | Nest, inner first |
| `formLCSSAForInstructions` | Sparse worklist |
| `isLCSSAForm` | Validate one loop |
| `isRecursivelyLCSSAForm` | Validate nest |
| `LCSSAPass` | Function-wide new PM |
| `-verify-loop-lcssa` | Expensive verifyAnalysis |

## 20. Exercises

1. Build Example 1 IR; prove by inspection every outside use is a PHI in an exit.
2. Add `break`; redraw exit PHIs.
3. Grep llvm-project for `formLCSSAForInstructions` call sites—list three passes.
4. Intentionally break LCSSA in a unit test IR and watch which pass asserts first.
5. Run `-stats` (if enabled) and find `NumLCSSA` after closing a multi-exit loop.

## 21. One-sentence invariant

**No value defined in loop `L` is used outside `L` except through an LCSSA PHI in an exit of `L` (recursively for nests).** That sentence is the acceptance test for `isRecursivelyLCSSAForm`.

