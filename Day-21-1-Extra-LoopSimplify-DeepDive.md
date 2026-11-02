# Day 21 Extra: LoopSimplify — Deep Dive

Companion to [Day-21-0-LoopSimplify.md](Day-21-0-LoopSimplify.md). Examples: [Day-21-2-Extra-LoopSimplify-Examples.md](Day-21-2-Extra-LoopSimplify-Examples.md).

## Diagram

```text
   not simplified                     after loop-simplify

        +------+                         preheader
        |      v                         |
     entry -> header <--+                v
        |       |       |             header <--+
        +-> body+-------+                |      |
                |                     body+-----+
                v  (exit also            |
             outside                     v
             reached)                 loopexit
                                      (only the loop
                                       reaches it)
                                      |
                                      v
                                    real exit
```

A simplified loop has one preheader (the only outside predecessor of the header), one latch (the only backedge), and dedicated exits (blocks the loop exits to that have no outside predecessors). LICM hoists into the preheader. Indvars and the vectorizer assume this picture, so they do not each grow their own edge-splitting code.

## 1. Files and entry points

| Piece | Path |
|-------|------|
| Pass + `simplifyLoop` | [LoopSimplify.cpp](https://github.com/llvm/llvm-project/blob/main/llvm/lib/Transforms/Utils/LoopSimplify.cpp) |
| API / guarantees comment | [LoopSimplify.h](https://github.com/llvm/llvm-project/blob/main/llvm/include/llvm/Transforms/Utils/LoopSimplify.h) |
| `formDedicatedExitBlocks` | Declared in [LoopUtils.h](https://github.com/llvm/llvm-project/blob/main/llvm/include/llvm/Transforms/Utils/LoopUtils.h); logic historically factored from LoopSimplify |

```cpp
bool simplifyLoop(Loop *L, DominatorTree *DT, LoopInfo *LI,
                  ScalarEvolution *SE, AssumptionCache *AC,
                  MemorySSAUpdater *MSSAU, bool PreserveLCSSA);
```

Nullable analyses: pass what you have; SE/AC/MSSAU may be null depending on caller.

## 2. `LoopSimplifyPass::run`

New PM pass:

1. Obtain LoopInfo, DominatorTree, and any optional analyses from the function analysis manager.
2. For each top-level loop, call `simplifyLoop` (children handled recursively inside).
3. Mark preserved analyses carefully—CFG changes; LI/DT should be updated in place when non-null pointers were provided.

Read the pass’s preservation return for MemorySSA / SCEV invalidation behavior on your tree.

## 3. Preheader insertion

When the header has multiple outside predecessors—or a single outside pred that also does other work with multiple successors—the pass inserts a new block:

```
outside preds → [new preheader] → header
```

The preheader ends with an unconditional branch to the header. LICM and others treat “hoist to preheader” as “execute once before any iteration.”

Edge cases: landing pads / exception edges need care; read skip conditions near preheader creation.

## 4. `formDedicatedExitBlocks`

Contract: for every exit block `E` of loop `L`, every predecessor of `E` must be inside `L`. If some predecessor is outside, split the **in-loop** predecessors through a new dedicated exit (often named with a `.loopexit` style suffix via `SplitBlockPredecessors`).

Why: an exit shared with outside control means the exit is not dominated solely by loop exits—store sinking and exit-phi placement become wrong or overly conservative.

`PreserveLCSSA` tells the splitter to maintain LCSSA while rewriting edges. `MemorySSAUpdater` updates memory SSA if present.

Failure modes: `IndirectBrInst` predecessors cannot always be rewritten—helpers warn and skip.

## 5. Single latch / unique backedge

If the header has multiple in-loop predecessors (multiple continues), LoopSimplify introduces a new latch block that those predecessors target; the latch then branches to the header. Result: one backedge in the CFG sense that LoopInfo expects.

This is distinct from LoopRotate’s “exiting latch” preference—but rotation and unrolling assume at most one latch after simplify.

## 6. Separating nested loops

LoopSimplify contains logic to detect when blocks of a loop should form an outer loop around an inner loop (shared headers / unusual nesting). It reassigns blocks between `Loop` objects via LoopInfo, splits exit edges from the inner loop with `formDedicatedExitBlocks`, and may call `formLCSSA` when preserving LCSSA.

When reading, search for functions that create a “new outer” loop and reparent blocks. This is the hardest part of the file—pair with a lit test that mentions nested separation.

## 7. LCSSA interaction

LCSSA requires that uses outside a loop of in-loop definitions go through exit PHIs.

- If `PreserveLCSSA` is true, simplifyLoop must leave the nest in LCSSA form (forming as needed after CFG edits).
- If false, later `lcssa` pass must run before consumers that require it.

Many pipelines use `loop-simplify,lcssa` as a pair.

## 8. Analysis updates

Every inserted block needs:

- DominatorTree insertions / recomputation edges
- LoopInfo ownership (`changeLoopFor` / add block to loop)
- SCEV forget when SE non-null (canonical induction may change)
- MemorySSA via `MemorySSAUpdater` when MSSA is active

Hand-written IR that “looks” canonical but has stale LoopInfo will confuse printers and passes—always produce form via this pass.

## 9. Irreducible loops

LoopInfo’s natural-loop definition may still discover headers, but irreducible regions (multiple entry cycles) break the usual preheader/backedge story. LoopSimplify may no-op or only partially rewrite. Vectorizers generally bail on irreducible CFG anyway.

## 10. What LoopSimplify does **not** do

- Rotation to bottom-tested form (LoopRotate).
- Unswitching / unrolling / idiom recognition.
- Strength reduction of inductions (IndVars).
- Guaranteeing a single exit edge (dedicated exits ≠ unique exit).
- Fixing irreducible control into reducible form in full generality.

## 11. Debugging

```bash
opt -passes=loop-simplify -debug-only=loop-simplify -S in.ll -o out.ll 2>ls.log
opt -passes='print<loops>' -disable-output out.ll
opt -passes='print<domtree>' -disable-output out.ll
```

Look for messages about creating preheaders, dedicated exits, and nested separation warnings.

Verify:

```bash
opt -passes='loop-simplify,verify' -S in.ll -o /dev/null
```

## 12. Consumers’ expectations (why bugs hurt)

| Pass | Relies on |
|------|-----------|
| LICM | Preheader for hoist; dedicated exits for sink |
| LoopUnroll | Single latch; clean exit set |
| IndVarSimplify | Canonical shape + usually LCSSA |
| LoopVectorize | Simplify + LCSSA (+ rotate often) |
| SimpleLoopUnswitch | Dedicated exits helpers |

When a consumer asserts `L->getLoopPreheader()`, missing LoopSimplify is the first suspect.

## 13. Relationship to SimplifyCFG

Early SimplifyCFG keeps `NeedCanonicalLoop == true` to avoid undoing this work. Late SimplifyCFG may create multi-latch or non-dedicated exits again → pipelines re-run `loop-simplify`.

If you write a custom pipeline, place `loop-simplify` after aggressive CFG and before loop passes.

## 14. Study checklist

1. Force missing preheader (two outside brs to header); observe new block.
2. Force non-dedicated exit (outside br to exit); observe `.loopexit` split.
3. Two continues → one latch.
4. Run with and without pairing `lcssa`; inspect exit PHIs.
5. Read one nest-separation lit test end-to-end.

## 15. Takeaway

LoopSimplify is **CFG canonicalization for LoopInfo loops**: preheader, dedicated exits, single backedge, nest hygiene. Almost every serious loop transform begins by assuming someone already ran it.

## 16. SplitBlockPredecessors as the workhorse

Much of LoopSimplify’s CFG surgery bottoms out in utility splitters such as `SplitBlockPredecessors` (BasicBlockUtils). When a dedicated exit is formed, in-loop preds are split off into a new block that then jumps to the old exit. Understanding that one utility makes the loopexit naming and PHI rewriting obvious.

## 17. MemorySSA and MSSAU

When MemorySSA is in the pipeline, inserting blocks requires `MemorySSAUpdater` so memory phis and defining accesses stay consistent. LICM with MSSA is a primary consumer. If you run LoopSimplify without MSSAU while MSSA is cached as “preserved” incorrectly, you get subtle miscompiles—prefer the pass’s own `run` path over calling `simplifyLoop` with a null MSSAU while leaving stale MSSA around.

## 18. Asserts worth knowing

Debug builds often assert that after `simplifyLoop`, the loop has a preheader and dedicated exits when the CFG still looks like a natural loop. Hitting those asserts in a custom pass usually means you mutated CFG and forgot to call `simplifyLoop` (or `formDedicatedExitBlocks`) before invoking a consumer.

## 19. Historical note for readers of old blogs

Older docs mention `LoopSimplify` as a required pass ID in the legacy PM (`createLoopSimplifyPass`). New PM uses `LoopSimplifyPass` / pipeline name `loop-simplify`. Behavior of `simplifyLoop` is the stable core across both.

## 20. Pair with `verify<loops>` when available

Some LLVM versions expose loop-specific verification in the pass pipeline. After simplify, run whatever loop verifier your tree provides (often part of `opt -passes=verify` plus asserts in LoopInfo) before feeding LICM. Catching a broken LI early saves hours.
