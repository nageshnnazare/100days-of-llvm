# Day 26: Loop Rotation (`loop-rotate`)

Loop rotation turns a top-tested (while-style) loop into a bottom-tested (do-while-style) loop with an optional zero-trip guard. That single CFG reshape unlocks LICM, simplifies unrolling, and is a prerequisite for many later loop transforms. Extra detail lives in [Day-26-1-Extra-LoopRotate-DeepDive.md](Day-26-1-Extra-LoopRotate-DeepDive.md) and [Day-26-2-Extra-LoopRotate-Examples.md](Day-26-2-Extra-LoopRotate-Examples.md).

## Why this exists

Clang and most frontends emit natural C `while` / `for` loops as:

1. Jump into the header.
2. Evaluate the condition in the header.
3. Either exit or fall into the body, then branch back to the header from the latch.

That shape is correct, but it is awkward for optimizers:

- Loop-invariant loads and calls sit *after* the header branch, so LICM cannot hoist them without proving the loop runs at least once (speculation).
- Full/partial unrolling prefers a latch that exits, so the unrolled remainder and epilog are clean.
- Many analyses (SCEV exit counts, vectorization prepasses) assume or benefit from a rotated form where the exiting edge is the latch.

Rotation copies the header’s condition (and a limited amount of header code) so the *latch* becomes the exiting block, and inserts a guard in the preheader when the trip count may be zero.

## Diagram

```text
   before rotate                         after loop-rotate

   header:                               preheader / guard
     br i1 %cond, %body, %exit              br i1 %cond, %body, %exit
   body:                                 body:          (the old header
     ... work ...                          ... work ...   test is now
     br label %header                      %cond2 = ...   at the latch)
   latch is the header                     br i1 %cond2, %body, %exit

   the first iteration's test is hoisted,
   so the body is a do-while
```

Rotation turns a while-loop into a do-while by peeling the condition into a guard and putting a copy of the test at the latch. The latch then branches on a value computed in the body, which is the shape IndVarSimplify and the vectorizer want. A loop that is already a do-while, or whose header is too profitable to duplicate, stays put.

## The idea

Conceptually:

```
       while (cond) { body; }
```

becomes:

```
       if (cond) {          // zero-trip guard (new preheader / guard)
         do { body; } while (cond);   // condition checked at bottom
       }
```

In CFG terms:

- The original header’s terminating branch is duplicated: one copy becomes the guard exiting from the preheader path; the other becomes the latch’s exit test.
- Phis in the header are rewritten so the preheader edge feeds initial values and the latch edge feeds updated values, matching do-while semantics.
- After rotation, the loop is still in LoopSimplify + LCSSA form (the pass preserves those invariants).

## Where it sits in the pipeline

| Item | Value |
|------|--------|
| New PM pass name | `loop-rotate` |
| Implementation | [llvm/lib/Transforms/Scalar/LoopRotation.cpp](https://github.com/llvm/llvm-project/blob/main/llvm/lib/Transforms/Scalar/LoopRotation.cpp) |
| Shared util | [llvm/lib/Transforms/Utils/LoopRotationUtils.cpp](https://github.com/llvm/llvm-project/blob/main/llvm/lib/Transforms/Utils/LoopRotationUtils.cpp), header [LoopRotationUtils.h](https://github.com/llvm/llvm-project/blob/main/llvm/include/llvm/Transforms/Utils/LoopRotationUtils.h) |
| Public entry | `llvm::LoopRotation(...)` (internal helper class is `LoopRotate` in the utils file) |
| Debug | `-debug-only=loop-rotate` |
| Size knobs | `-rotation-max-header-size` (default historically 16; may differ by revision) |

Typical pipeline position: after loop simplify / LCSSA setup, early in the loop pass manager, *before* LICM and unrolling so those passes see a rotatable form.

Produce transformable IR (avoid `optnone` from plain `-O0`):

```bash
clang -S -emit-llvm -O1 -Xclang -disable-llvm-passes file.c -o file.ll
opt -passes='loop-simplify,lcssa,loop-rotate' -S file.ll -o rotated.ll
```

## Core concepts

### While vs do-while in IR

Before (simplified):

```llvm
preheader:
  br label %header

header:                         ; exiting block
  %i = phi i32 [ 0, %preheader ], [ %i.next, %latch ]
  %cmp = icmp slt i32 %i, %n
  br i1 %cmp, label %body, label %exit

body:
  ; ... work using %i ...
  br label %latch

latch:
  %i.next = add nsw i32 %i, 1
  br label %header
```

After rotation (shape):

```llvm
preheader:
  %guard = icmp slt i32 0, %n
  br i1 %guard, label %body, label %exit   ; zero-trip guard

body:                           ; new header (non-exiting)
  %i = phi i32 [ 0, %preheader ], [ %i.next, %latch ]
  ; ... work ...
  br label %latch

latch:                          ; exiting latch
  %i.next = add nsw i32 %i, 1
  %cmp = icmp slt i32 %i.next, %n
  br i1 %cmp, label %body, label %exit
```

Exact block names and which header instructions are duplicated depend on size thresholds and profitability.

### Header duplication budget

Duplicating an enormous header would explode code size. The pass counts “header size” (instruction cost / metrics) against `-rotation-max-header-size`. If the header is too large, rotation is skipped (`NumNotRotatedDueToHeaderSize` statistic).

### Already bottom-tested loops

If the latch is already the exiting block and further rotation does not help (see `profitableToRotateLoopExitingLatch` in LoopRotationUtils), the pass declines. Rotating again would only shuffle blocks without enabling new hoist/unroll opportunities.

### Side effects that cannot be duplicated

Header instructions with non-duplicatable side effects (certain calls, volatile ops, etc., as judged by the duplication safety helpers used during cloning) block rotation. The transform must not execute extra I/O or traps on the zero-trip path incorrectly, and must not invent extra side effects on the latch path.

### Metadata

Current upstream does **not** define a dedicated `llvm.loop.rotate.disable` string in the loop metadata helpers the way unroll/vectorize do. Optional high-level restructuring (including rotation) is suppressed with `llvm.loop.disable_nonforced`. Always verify against [TransformMetadata](https://llvm.org/docs/TransformMetadata.html) for your checkout. Forced vectorize metadata can still bias the pass toward rotating even when header duplication is otherwise discouraged (see `LoopRotatePass::run` comments about vectorization).

## The algorithm

High-level steps inside `LoopRotation` / `LoopRotate::processLoop`:

1. **Eligibility.** Loop must be in a simplifiable form (unique preheader, latch, etc., as expected by LoopSimplify). Reject if already profitable as-is, header too large, or unsafe to duplicate.
2. **Fold / hoist cheap header ops into the preheader** when legal (reduces what must be cloned). Statistics track hoisted vs duplicated instructions.
3. **Clone the header’s exit condition** (and necessary supporting instructions) so the latch can branch like the old header.
4. **Insert a zero-trip guard** when the trip count may be zero: branch from the old preheader around the loop. Profile metadata may attach cold weights for the never-entered edge (`ZeroTripCountWeights` in the utils file).
5. **Rewrite phis and LCSSA** so values defined in the loop still reach exits cleanly; update DominatorTree / MemorySSA via `MemorySSAUpdater` when MSSA is present.
6. **Preserve** LoopSimplify and LCSSA so subsequent loop passes keep their preconditions.

## Where to read the code

1. [LoopRotation.cpp](https://github.com/llvm/llvm-project/blob/main/llvm/lib/Transforms/Scalar/LoopRotation.cpp) — `LoopRotatePass::run`, threshold selection, MemorySSA preservation.
2. [LoopRotationUtils.cpp](https://github.com/llvm/llvm-project/blob/main/llvm/lib/Transforms/Utils/LoopRotationUtils.cpp) — real transform: `LoopRotate` class, `LoopRotation` entry point, profitability for exiting latches, header metrics.
3. [LoopRotationUtils.h](https://github.com/llvm/llvm-project/blob/main/llvm/include/llvm/Transforms/Utils/LoopRotationUtils.h) — public API surface.
4. Tests under `llvm/test/Transforms/LoopRotate/` (in-tree) for expected CFG shapes.

## Worked before / after

C input:

```c
int sum(int *a, int n) {
  int s = 0;
  int i = 0;
  while (i < n) {
    s += a[i];
    ++i;
  }
  return s;
}
```

Build IR without running mid-level opts:

```bash
clang -S -emit-llvm -O1 -Xclang -disable-llvm-passes sum.c -o sum.ll
opt -passes='loop-simplify,lcssa,loop-rotate' -S sum.ll -o sum.rot.ll
```

**Before:** condition `icmp` lives in the loop header; latch is an unconditional branch back to the header.

**After:** a guard compares the initial induction against `n`; the loop body becomes the header; the latch holds the next-`i` compare and the exiting branch. LICM can then hoist invariants from the rotated header more aggressively because execution of the body implies the loop was entered.

## Reading order

1. Skim this note’s CFG diagrams until the while→do-while picture is automatic.
2. Read `LoopRotatePass::run` in LoopRotation.cpp (short).
3. Read `LoopRotate::processLoop` and the duplication helpers in LoopRotationUtils.cpp.
4. Run the hands-on below with `-debug-only=loop-rotate`.
5. Continue with Day 26 extras, then Day 27 (Loop Idiom) which often follows cleaned loops.

## Hands-on

```bash
cat > rotate.c <<'EOF'
void sink(int);
void demo(int n) {
  int i = 0;
  while (i < n) {
    sink(i);
    ++i;
  }
}
EOF
clang -S -emit-llvm -O1 -Xclang -disable-llvm-passes rotate.c -o rotate.ll
opt -passes='loop-simplify,lcssa,loop-rotate' -S rotate.ll -o rotate.out.ll
# Optional: opt -debug-only=loop-rotate -passes='loop-simplify,lcssa,loop-rotate' -S rotate.ll
```

Inspect whether the latch exits. Then bump a huge header (many unrelated statements) and watch rotation fail under a small `-rotation-max-header-size=1`.

## Pitfalls

- **`optnone` / `-O0` IR:** `opt` will skip the function; always use `-O1 -Xclang -disable-llvm-passes` (or strip `optnone`).
- **Missing LoopSimplify/LCSSA:** run those first or use a pipeline that includes them; rotation assumes a sane loop form.
- **Assuming every loop rotates:** large headers, unsafe duplication, and already-exiting latches all skip.
- **Confusing public API names:** the pass calls `LoopRotation`; there is no separate exported `RotateLoop` symbol—the logic lives in the `LoopRotate` class inside LoopRotationUtils.cpp.
- **Metadata myths:** do not invent `llvm.loop.rotate.disable` without confirming it in your tree; use `llvm.loop.disable_nonforced` for broad disable.

## Neighbors

- **LoopSimplify / LCSSA** — prerequisites and preserved forms.
- **LICM** — primary consumer of the rotated shape (speculative hoist from header).
- **LoopUnroll / runtime unroll** — prefer bottom-tested loops.
- **IndVarSimplify / LoopVectorize** — later; vectorize metadata can force rotation interest.
- **SimpleLoopUnswitch** (Day 28) — also reshapes loop CFG; different legality.

## Check yourself

1. **Q:** Why does rotation help LICM?  
   **A:** After rotation, the header executes only if the loop was entered, so invariants in the (new) header are not speculative with respect to the zero-trip case—the guard already handled that.

2. **Q:** What does `-rotation-max-header-size` limit?  
   **A:** How large a loop header may be and still be duplicated/rotated; oversized headers are left alone to control code size.

3. **Q:** Name the util entry point and the pass name.  
   **A:** `llvm::LoopRotation` in LoopRotationUtils; opt pass `loop-rotate`.

4. **Q:** When must a zero-trip guard appear?  
   **A:** When the loop might execute zero times; without the guard, a bottom-tested body would run once incorrectly.

5. **Q:** Does rotation destroy LCSSA?  
   **A:** No—the pass is written to preserve LoopSimplify/LCSSA (and can update MemorySSA).
