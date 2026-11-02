# Day 22: Loop-Closed SSA (`lcssa`)

LCSSA places PHI nodes at loop exits for every value defined inside a loop and used outside it. Outside code then reads those exit PHIs, not the in-loop defs. That “closes” the SSA live range of the loop so later rewrites only have to update exit PHIs. Extra detail: [Day-22-1-Extra-LCSSA-DeepDive.md](Day-22-1-Extra-LCSSA-DeepDive.md). Worked IR: [Day-22-2-Extra-LCSSA-Examples.md](Day-22-2-Extra-LCSSA-Examples.md).

## 1. Why this exists

Without LCSSA, a single SSA value defined in the loop body can have uses scattered through exit blocks and outer code:

```llvm
; %x defined in the loop, used both in the latch and after exit
%x = add i32 %i, 1
; ... later, outside the loop ...
%y = mul i32 %x, 2
```

Any transform that clones, rotates, unrolls, or deletes the loop must find *every* external use and rewrite it. That is brittle. LCSSA inserts a PHI in each relevant exit block (`%x.lcssa = phi i32 [ %x, %latch ]`) and rewires outside uses through that PHI. Loop passes then treat the loop as a sealed region whose only exports are exit PHIs.

The PHIs are often trivial (one incoming edge). InstCombine / simplifycfg removes them later when they stop being useful as an invariant.

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

## 2. The idea

**Loop-closed SSA** means: if a value is defined in loop `L` and used outside `L`, every outside use is of an LCSSA PHI sitting in an exit block of `L` (or of a PHI chain produced while rewriting across nested exits)—never of the original in-loop instruction.

Consequences:

1. Uses outside the loop are localized at exits.
2. Rewriting the loop body updates the PHI incoming values; outside IR stays pointed at the PHIs.
3. Nested loops are processed inner-first (`formLCSSARecursively`) so an outer loop sees already-closed inners.

DominatorTree matters: an LCSSA PHI is created in an exit block only when the definition dominates that exit. Exit blocks are obtained from `Loop::getExitBlocks`.

## 3. Where it sits in the -O2/-O3 pipeline

| Item | Value |
|------|--------|
| Pass name | `lcssa` |
| Implementation | [llvm/lib/Transforms/Utils/LCSSA.cpp](https://github.com/llvm/llvm-project/blob/main/llvm/lib/Transforms/Utils/LCSSA.cpp) |
| Header / API | [llvm/include/llvm/Transforms/Utils/LCSSA.h](https://github.com/llvm/llvm-project/blob/main/llvm/include/llvm/Transforms/Utils/LCSSA.h), declarations also via [LoopUtils.h](https://github.com/llvm/llvm-project/blob/main/llvm/include/llvm/Transforms/Utils/LoopUtils.h) |
| New PM | `LCSSAPass` |
| Legacy | `LCSSAWrapperPass`, `createLCSSAPass()` |
| Debug | `-debug-only=lcssa` (`DEBUG_TYPE "lcssa"`) |
| Verify flag | `-verify-loop-lcssa` (expensive; off by default) |

Typical prep before other loop opts:

```bash
clang -S -emit-llvm -O1 -Xclang -disable-llvm-passes file.c -o file.ll
opt -passes='loop-simplify,lcssa' -S file.ll -o file.lcssa.ll
```

Many loop pipelines assume **LoopSimplify + LCSSA** already hold (LICM, unroll, deletion, unswitch, rotate consumers).

## 4. Core concepts

### Exit PHIs and outside uses

For instruction `%I` defined in `L` with a use in block `U` outside `L`:

1. Collect exit blocks of `L`.
2. For each exit dominated by `%I`, insert a PHI named like `%I.lcssa`.
3. Fill every predecessor edge into that exit with `%I` when the pred is in `L` (and arrange further rewriting if a pred is outside).
4. Rewrite the outside use to the appropriate PHI (`SSAUpdater` when multiple exits).

PHI uses are attributed to the **incoming predecessor** block (LangRef / LCSSA convention), so “is this use outside the loop?” looks at that predecessor, not only the PHI’s parent.

### `formLCSSA` / `formLCSSAForInstructions`

- `llvm::formLCSSA(Loop &L, const DominatorTree &DT, const LoopInfo *LI, ScalarEvolution *SE)` — close one loop (expects subloops already closed under expensive checks).
- `llvm::formLCSSAForInstructions(...)` — close a worklist of instructions that may have external uses.
- `llvm::formLCSSARecursively` — depth-first: subloops first, then `L`.

Optional `ScalarEvolution *`: if a SCEV already exists for the original instruction, the pass also asks SCEV for the new PHI so BECount / cached expressions invalidate correctly when the PHI dies later.

### Tokens and lifetime markers

Token-like values cannot sit in PHIs; they are skipped. Lifetime start/end that cross the loop boundary are dropped rather than rewritten through PHIs (markers must refer to the alloca directly).

### Verification (no lone `verifyLCSSA` API)

Current tree verifies with:

- `Loop::isLCSSAForm(const DominatorTree &DT, ...)`
- `Loop::isRecursivelyLCSSAForm(const DominatorTree &DT, const LoopInfo &LI, ...)`
- Legacy `LCSSAVerificationPass` (signals LPPassManager to check preservation)
- `cl::opt` `-verify-loop-lcssa` driving expensive full-function checks in `LCSSAWrapperPass::verifyAnalysis`

When notes say “verify LCSSA,” they mean these predicates / the verification pass—not a separate free function named `verifyLCSSA`.

## 5. The algorithm

High-level path in LCSSA.cpp:

1. For each top-level loop, call `formLCSSARecursively`.
2. For a loop `L`, get exit blocks; if none, nothing to do.
3. Compute blocks in `L` that dominate at least one exit (`computeBlocksDominatingExits` walks idoms upward from exits until the header).
4. Scan those blocks (skipping blocks belonging to subloops) for instructions with possible external uses; push them on a worklist.
5. `formLCSSAForInstructionsImpl`: for each instruction, classify uses; insert exit PHIs; rewrite uses; sync dbg values; post-process PHIs accidentally inserted into other loops’ headers; erase unused trivial PHIs.

Statistic: `NumLCSSA` — “Number of live out of a loop variables.”

## 6. Where to read the code

1. File-top comment in [LCSSA.cpp](https://github.com/llvm/llvm-project/blob/main/llvm/lib/Transforms/Utils/LCSSA.cpp) — the classic before/after diagram.
2. `formLCSSAForInstructions` / `formLCSSA` / `formLCSSARecursively`.
3. `Loop::isLCSSAForm` / `isRecursivelyLCSSAForm` in [LoopInfo](https://github.com/llvm/llvm-project/blob/main/llvm/include/llvm/Analysis/LoopInfo.h) (implementations under Analysis).
4. Tests: `llvm/test/Transforms/LCSSA/`.

## 7. Worked before/after

C:

```c
int last(int n) {
  int x = 0;
  for (int i = 0; i < n; ++i)
    x = i;
  return x;
}
```

Shape before LCSSA (simplified, after mem2reg / simplify):

```llvm
loop:
  %i = phi i32 [ 0, %ph ], [ %i.next, %loop ]
  %x = phi i32 [ 0, %ph ], [ %i, %loop ]
  %i.next = add i32 %i, 1
  %cmp = icmp slt i32 %i.next, %n
  br i1 %cmp, label %loop, label %exit

exit:
  ret i32 %x          ; use of in-loop %x directly
```

After `lcssa`:

```llvm
exit:
  %x.lcssa = phi i32 [ %x, %loop ]
  ret i32 %x.lcssa
```

Unroll / delete / rotate now only retarget `%x.lcssa`’s incoming.

## 8. Reading order for today

1. This note’s before/after until exit PHIs feel inevitable.
2. Top-of-file comment + `formLCSSAForInstructionsImpl` in LCSSA.cpp.
3. Day 22 DeepDive (dominance filter, SSAUpdater, SCEV cache).
4. Day 22 Examples.
5. Then Day 23 (LICM), which expects LCSSA when sinking/promoting across exits.

## 9. Hands-on commands

```bash
cat > lcssa.c <<'EOF'
int last(int n) {
  int x = 0;
  for (int i = 0; i < n; ++i)
    x = i;
  return x;
}
EOF
clang -S -emit-llvm -O1 -Xclang -disable-llvm-passes lcssa.c -o lcssa.ll
opt -passes='loop-simplify,lcssa' -S lcssa.ll -o lcssa.out.ll
grep lcssa lcssa.out.ll
# Optional: opt -debug-only=lcssa -passes='loop-simplify,lcssa' -S lcssa.ll
```

Compare `ret` operands before and after. Add a second exit (`if (i == k) break;`) and watch multiple exit PHIs / SSAUpdater behavior.

## Multi-exit shape (what to expect in IR)

After LoopSimplify, a loop with `break` often looks like:

```llvm
loop:
  %i = phi i32 [ 0, %ph ], [ %i.next, %latch ]
  %cmp.break = icmp eq i32 %i, %k
  br i1 %cmp.break, label %exit.break, label %latch

latch:
  %i.next = add i32 %i, 1
  %cmp = icmp slt i32 %i.next, %n
  br i1 %cmp, label %loop, label %exit.fall

exit.break:
  %i.lcssa.b = phi i32 [ %i, %loop ]
  br label %join

exit.fall:
  %i.lcssa.f = phi i32 [ %i.next, %latch ]
  br label %join

join:
  %i.out = phi i32 [ %i.lcssa.b, %exit.break ], [ %i.lcssa.f, %exit.fall ]
  ret i32 %i.out
```

The `.lcssa` PHIs are the loop’s sealed exports; `%i.out` is ordinary SSA after the region. Unroll rewriting `%i` / `%i.next` only retargets the exit PHI incomings.

## Relationship to “single-entry single-exit”

LCSSA does **not** require a single exit. It requires that live-outs be PHIs *at* exits. SESE is a stronger CFG property some textbooks assume; LLVM loops frequently have several dedicated exits, each with its own LCSSA PHIs.

## When passes re-form LCSSA

Transform utilities often call `formLCSSA` / `formLCSSARecursively` after editing a nest rather than relying on a later full-function `lcssa` pass. When debugging “LCSSA broken” assertions, grep the failing pass for those calls and for `PreserveLCSSA` flags (notably LoopUnroll).

## 10. Pitfalls and invariants

- Hand-writing loop opts that leave external uses of in-loop values — later passes assert `isLCSSAForm`.
- Forgetting LoopSimplify: dedicated exits / preheaders make LCSSA and friends much simpler.
- Expecting LCSSA PHIs to survive forever — InstCombine deletes trivial ones; that is intended.
- Treating PHI parent block as the use site — for LCSSA classification, predecessor matters.
- Assuming a function named `verifyLCSSA` — use `isLCSSAForm` / `-verify-loop-lcssa`.
- Scanning every loop block for external uses mentally — dominance filter skips blocks that cannot reach exits under SSA rules.

## 11. Neighbors

- **LoopSimplify** — dedicated preheader and exits; almost always run first.
- **LICM** (Day 23) — sinks/hoists; exit PHIs are the LCSSA contract for sunk values.
- **LoopUnroll / LoopDeletion / IndVarSimplify** — rewrite or replace exit PHIs.
- **SSAUpdater** — shared machinery inside LCSSA rewriting.
- **LoopInfo** — `isLCSSAForm` / `isRecursivelyLCSSAForm` live on `Loop`.

## 12. Check yourself

1. **Q:** What does LCSSA guarantee about outside uses of in-loop defs?  
   **A:** They go through PHI(s) in loop exit block(s), not directly to the in-loop instruction.

2. **Q:** Why do loop passes want this?  
   **A:** So rewriting the loop only updates exit PHIs instead of hunting every external use.

3. **Q:** Name the two primary form APIs.  
   **A:** `llvm::formLCSSA` and `llvm::formLCSSAForInstructions` (plus `formLCSSARecursively`).

4. **Q:** Who removes trivial LCSSA PHIs later?  
   **A:** Typically InstCombine (and related cleanup)—they are redundant SSA.

5. **Q:** How do you verify LCSSA in-tree?  
   **A:** `Loop::isLCSSAForm` / `isRecursivelyLCSSAForm`, optional `-verify-loop-lcssa`, and `LCSSAVerificationPass` in the legacy loop PM.

6. **Q:** Does LCSSA require a unique exit?  
   **A:** No—one PHI placement per dominated exit is normal.

> Tip: Deep dive in `Day-22-1-Extra-LCSSA-DeepDive.md`. Examples in `Day-22-2-Extra-LCSSA-Examples.md`.
