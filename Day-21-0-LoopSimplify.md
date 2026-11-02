# Day 21: LoopSimplify

LoopSimplify canonicalizes natural loops into a form almost every later loop pass assumes: a **preheader**, **dedicated exits**, and a **single latch/backedge**. Pass name: `loop-simplify`. Implementation: [LoopSimplify.cpp](https://github.com/llvm/llvm-project/blob/main/llvm/lib/Transforms/Utils/LoopSimplify.cpp), entry `simplifyLoop`. Deep dive: [Day-21-1-Extra-LoopSimplify-DeepDive.md](Day-21-1-Extra-LoopSimplify-DeepDive.md). Examples: [Day-21-2-Extra-LoopSimplify-Examples.md](Day-21-2-Extra-LoopSimplify-Examples.md).

## 1. Why this exists

Natural loops from Clang can have:

- Multiple branches into the header from outside (no unique preheader).
- Exit blocks that also have non-loop predecessors (not dedicated).
- Multiple backedges into the header (multiple latches).
- Nested structure that is “one LoopInfo loop” but should be split into outer/inner.

LICM, LoopUnroll, IndVarSimplify, LoopVectorize, and many others are written against the canonical shape. Without LoopSimplify, each would reimplement edge splitting. Centralizing that CFG surgery keeps LoopInfo and DominatorTree updates consistent.

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

## 2. The idea

For each loop (innermost-first / recursive nest processing via `simplifyLoop`):

1. Ensure a unique **preheader**: single outside predecessor that unconditionally branches to the header; all external entries funnel through it.
2. Ensure **dedicated exits** via `formDedicatedExitBlocks`: every exit block’s predecessors are only inside the loop (split edges otherwise).
3. Ensure a **single backedge**: if several latches exist, merge them through an intermediate block so the header has one loop backedge.
4. Separate awkwardly nested loops when analysis finds “nested” control that LoopInfo should treat as distinct loops.
5. Optionally preserve/form **LCSSA** when asked (`PreserveLCSSA`).

Inserting blocks is the tool: split predecessors, create `.preheader` / `.loopexit` style names (exact suffixes depend on helpers), update DT and LI.

## 3. Where it sits in the -O2/-O3 pipeline

| Item | Value |
|------|--------|
| Pass name | `loop-simplify` |
| Function | `llvm::simplifyLoop` |
| File | [LoopSimplify.cpp](https://github.com/llvm/llvm-project/blob/main/llvm/lib/Transforms/Utils/LoopSimplify.cpp) |
| Header | [LoopSimplify.h](https://github.com/llvm/llvm-project/blob/main/llvm/include/llvm/Transforms/Utils/LoopSimplify.h) |
| Dedicated exits util | `formDedicatedExitBlocks` (declared with Loop utils; implemented alongside LoopSimplify) |
| Debug | `-debug-only=loop-simplify` (confirm `DEBUG_TYPE` in file) |

Runs early in loop pipelines and again whenever CFG opts might have broken canonical form. SimplifyCFG’s `NeedCanonicalLoop` exists partly so this form survives mid-pipeline.

```bash
clang -S -emit-llvm -O1 -Xclang -disable-llvm-passes file.c -o file.ll
opt -passes=loop-simplify -S file.ll -o file.ls.ll
opt -passes='loop-simplify,lcssa' -S file.ll -o file.lcssa.ll
```

## 4. Core concepts

### Preheader

A block outside the loop with a single successor (the header) such that every edge from outside into the header goes through this block. Ideal landing pad for LICM hoists.

### Dedicated exit

An exit block dominated by the header for its loop-incoming edges, with **no** predecessors from outside the loop. Lets LICM sink stores to exits safely and simplifies exit-value reasoning.

### Single latch / backedge

Exactly one in-loop edge to the header (the latch). Multiple `continue`-like paths are merged into one latch block that then branches to the header.

### Nested loop separation

When a loop contains control that is itself a loop sharing the same header awkwardly, LoopSimplify can rewrite CFG so LoopInfo sees a proper outer/inner nest.

### LCSSA

Loop-Closed SSA: values defined in the loop and used outside feed PHIs in exits. LoopSimplify can preserve or form LCSSA when `PreserveLCSSA` is true; dedicated LCSSA pass also exists.

### Irreducible CFG

Loops that are not natural (irreducible control) may not fully canonicalize. LoopSimplify does what it can; do not expect miracles on irreducible graphs.

## 5. The algorithm

`simplifyLoop(L, DT, LI, SE, AC, MSSAU, PreserveLCSSA)` roughly:

1. Recursively simplify child loops first (nest bottom-up).
2. Insert/repair preheader if missing.
3. Call `formDedicatedExitBlocks(L, DT, LI, MSSAU, PreserveLCSSA)`.
4. If multiple backedges, create a unique latch by splitting/merging predecessors of the header that come from inside the loop.
5. Separate nested loops when needed; fix LCSSA if requested.
6. Update DominatorTree, LoopInfo, and optionally ScalarEvolution / MemorySSA via updater.

The pass `LoopSimplifyPass::run` walks loops in the function and calls `simplifyLoop` with analyses from the managers.

## 6. Where to read the code

1. [LoopSimplify.h](https://github.com/llvm/llvm-project/blob/main/llvm/include/llvm/Transforms/Utils/LoopSimplify.h) — contract comment (preheader, exits, single backedge).
2. [LoopSimplify.cpp](https://github.com/llvm/llvm-project/blob/main/llvm/lib/Transforms/Utils/LoopSimplify.cpp) — `simplifyLoop`, nest separation, latch merging.
3. `formDedicatedExitBlocks` declaration in [LoopUtils.h](https://github.com/llvm/llvm-project/blob/main/llvm/include/llvm/Transforms/Utils/LoopUtils.h).
4. Consumers: LICM, LoopUnroll, IndVars, LoopVectorize (their “require LoopSimplify” comments).
5. Tests under `llvm/test/Transforms/LoopSimplify/`.

## 7. Worked before/after

```c
int sum(int *a, int n) {
  int s = 0;
  for (int i = 0; i < n; ++i)
    s += a[i];
  return s;
}
```

```bash
clang -S -emit-llvm -O1 -Xclang -disable-llvm-passes sum.c -o sum.ll
opt -passes=loop-simplify -S sum.ll -o sum.ls.ll
opt -passes='print<loops>' -disable-output sum.ls.ll
```

**Before (typical Clang shape):** header reached from entry and from latch; exit may share structure with other edges; multiple paths can look non-canonical depending on lowering.

**After:** recognizable `%for.cond.preheader` (or similar), unique latch backedge, dedicated `%for.end` / loopexit block whose only preds are loop-exiting blocks.

PHI in the header has exactly two incoming edges of interest conceptually: one from preheader (initial values), one from latch (updated values)—the clean LoopSimplify + rotate story later builds on this.

## 8. Reading order for today

1. Header comment in LoopSimplify.h (canonical guarantees).
2. `formDedicatedExitBlocks` contract in LoopUtils.h.
3. `simplifyLoop` control flow in LoopSimplify.cpp.
4. Print loops before/after on a `for` and on a loop with two `continue`s.
5. [Day-21-1-Extra-LoopSimplify-DeepDive.md](Day-21-1-Extra-LoopSimplify-DeepDive.md).
6. [Day-21-2-Extra-LoopSimplify-Examples.md](Day-21-2-Extra-LoopSimplify-Examples.md).
7. Revisit SimplifyCFG `NeedCanonicalLoop` (Day 18).

## 9. Hands-on commands

```bash
clang -S -emit-llvm -O1 -Xclang -disable-llvm-passes ls.c -o ls.ll
opt -passes='print<loops>' -disable-output ls.ll
opt -passes=loop-simplify -S ls.ll -o ls.out.ll
opt -passes='print<loops>' -disable-output ls.out.ll
opt -passes=loop-simplify -debug-only=loop-simplify -S ls.ll -o /dev/null 2>ls.log
```

Construct IR with two external preds into a header to force preheader insertion; two exits sharing a block with an outside pred to force dedicated exits.

## 10. Pitfalls and invariants

- **Irreducible loops:** partial simplification only.
- **Indirectbr** exits can block dedicated-exit formation (helpers bail when they cannot split).
- **Analyses must update:** DT/LI out of sync is a verifier nightmare—always run through the pass, do not hand-edit half of it.
- **LCSSA optional:** check `PreserveLCSSA`; do not assume exit PHIs always appear from LoopSimplify alone.
- **SimplifyCFG can destroy form** if `NeedCanonicalLoop` is false—re-run LoopSimplify.
- **`optnone`:** strip via the usual `-O1 -Xclang -disable-llvm-passes` recipe.
- **Nested separation** changes which blocks belong to which `Loop *`—print loops after.

### What “dedicated exit” is not

Dedicated exits do **not** mean the loop has only one exit edge. A loop may exit from the header and from a break in the body to two different dedicated exit blocks. The invariant is per exit block: no mixture of outside and inside predecessors.

### Preheader vs guard

LoopRotate may insert a zero-trip **guard** before the loop. That guard is related to but distinct from LoopSimplify’s preheader. After both passes, you often see: guard → preheader → header, or a merged structure depending on thresholds. When reading IR, label blocks by role (guard / preheader / header / latch / loopexit) before chasing PHIs.

### Minimal canonical checklist

After `loop-simplify`, for each loop `L` you study:

1. `L->getLoopPreheader()` non-null.
2. `L->getLoopLatch()` unique (single latch).
3. `L->hasDedicatedExits()` true (API may be named similarly—use Loop methods in the debugger or print).
4. Children recursively satisfy the same.

If any fail, either the pass skipped (indirectbr / irreducible) or another pass broke form.

## 11. Neighbors

- **LoopInfo / Dominators** — analyses this pass maintains.
- **LCSSA** — often paired; required by many loop opts.
- **LICM** — needs preheader + dedicated exits.
- **LoopRotate (Day 26)** — assumes simplifiable form.
- **IndVarSimplify / LoopUnroll / LoopVectorize** — major consumers.
- **SimplifyCFG (Day 18)** — tension via `NeedCanonicalLoop`.
- **SimpleLoopUnswitch** — also calls dedicated-exit helpers.

## 12. Check yourself

**Q1.** Pass name and C++ entry function?  
**A1.** `loop-simplify` / `simplifyLoop`.

**Q2.** Name the three canonical guarantees.  
**A2.** Preheader; dedicated exits; single backedge/latch.

**Q3.** What helper forms dedicated exits?  
**A3.** `formDedicatedExitBlocks`.

**Q4.** Why do vectorizers care?  
**A4.** They assume canonical loop shape for legality and skeleton construction (with LCSSA).

**Q5.** What CFG class may not fully simplify?  
**A5.** Irreducible control flow.

**Q6.** Which Day 18 option tries to preserve this form mid-pipeline?  
**A6.** `NeedCanonicalLoop` in SimplifyCFGOptions.

Tip: nest/latch implementation notes in [Day-21-1-Extra-LoopSimplify-DeepDive.md](Day-21-1-Extra-LoopSimplify-DeepDive.md); CFG recipes in [Day-21-2-Extra-LoopSimplify-Examples.md](Day-21-2-Extra-LoopSimplify-Examples.md).

### Pipeline fragment to memorize

```text
loop-simplify → lcssa → loop-rotate → licm → …
```

Many `-O2` loop pass managers are elaborations of that spine. If LICM says “no preheader,” walk backward: did SimplifyCFG run with `NeedCanonicalLoop` false, and did you forget to re-run `loop-simplify`?

### Block naming conventions

While not guaranteed by the verifier, helpers often create names containing `preheader`, `loopexit`, or `latch`. Use them as reading aids; rename-free IR from hand tests may use `%ph` / `%lex`—judge by CFG role, not string matching.

### `hasForm` mental API

Even if you never call C++ from the debugger, remember the predicates Loop passes use: unique preheader, unique latch, dedicated exits, optionally LCSSA. LoopSimplify’s job is to make those predicates true for natural loops so callers can `assert` and transform without reimplementing edge splits.

### Diff-reading tip

A LoopSimplify diff that only adds empty blocks with `br label` is still doing real work—those blocks are the preheader or loopexit plumbing LICM needs. Judge success by `print<loops>` and PHI incoming counts, not by “no interesting instructions moved.”

Keep LoopSimplify output verified (`opt -passes='loop-simplify,verify'`) before chaining Day 26 rotation experiments on the same bitcode.
