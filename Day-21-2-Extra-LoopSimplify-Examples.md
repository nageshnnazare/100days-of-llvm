# Day 21 Extra: LoopSimplify — Examples

Overview: [Day-21-0-LoopSimplify.md](Day-21-0-LoopSimplify.md). Deep dive: [Day-21-1-Extra-LoopSimplify-DeepDive.md](Day-21-1-Extra-LoopSimplify-DeepDive.md).

```bash
clang -S -emit-llvm -O1 -Xclang -disable-llvm-passes file.c -o file.ll
opt -passes=loop-simplify -S file.ll -o file.out.ll
opt -passes='print<loops>' -disable-output file.out.ll
opt -passes=loop-simplify -debug-only=loop-simplify -S file.ll
```

---

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

## Example 1: Insert a preheader

### Goal

Header with two outside predecessors gains a unique preheader.

### Input

```llvm
define void @two_entry(i1 %c, i32 %n) {
entry:
  br i1 %c, label %a, label %b
a:
  br label %header
b:
  br label %header
header:
  %i = phi i32 [ 0, %a ], [ 0, %b ], [ %i.next, %latch ]
  %cmp = icmp slt i32 %i, %n
  br i1 %cmp, label %latch, label %exit
latch:
  %i.next = add i32 %i, 1
  br label %header
exit:
  ret void
}
```

### Commands

```bash
opt -passes=loop-simplify -S two_entry.ll -o two_entry.out.ll
opt -passes='print<loops>' -disable-output two_entry.out.ll
```

### Expected

A new preheader block; `%a` and `%b` branch to it; preheader branches to `header`. Header PHI rewritten with a single outside incoming from the preheader (plus latch).

### Why it fires

Canonical form requires a unique preheader.

### Counterexample

Already a unique unconditional outside pred → preheader may already exist; pass is a no-op for that part.

---

## Example 2: Dedicated exit via edge split

### Goal

Exit block shared with an outside predecessor becomes dedicated.

### Input

```llvm
define i32 @shared_exit(i1 %c, i32 %n) {
entry:
  br label %header
header:
  %i = phi i32 [ 0, %entry ], [ %i.next, %latch ]
  %cmp = icmp slt i32 %i, %n
  br i1 %cmp, label %latch, label %exit
latch:
  %i.next = add i32 %i, 1
  br i1 %c, label %header, label %exit
exit:
  %p = phi i32 [ %i, %header ], [ %i.next, %latch ], [ 7, %side ]
  ret i32 %p
side:
  ; modeled as another pred into exit — attach via an alternate entry for demo
  br label %exit
}
```

For a cleaner minimal case, use a block `%exit` with preds from the loop and from `%entry` directly:

```llvm
define void @shared_exit2(i1 %early, i32 %n) {
entry:
  br i1 %early, label %exit, label %header
header:
  %i = phi i32 [ 0, %entry ], [ %i.next, %header ]
  %i.next = add i32 %i, 1
  %cmp = icmp slt i32 %i, %n
  br i1 %cmp, label %header, label %exit
exit:
  ret void
}
```

### Commands

```bash
opt -passes=loop-simplify -S shared_exit2.ll -o shared_exit2.out.ll
```

### Expected

Loop-exiting edge goes to a new dedicated loopexit block that then jumps to `%exit`. `%exit` may still have the early outside pred; the loop’s exit is dedicated.

### Why it fires

`formDedicatedExitBlocks` splits in-loop preds away from mixed exits.

### Counterexample

Exit whose only preds are already in-loop → no split.

---

## Example 3: Multiple continues → single latch

### Goal

Two backedges collapse through one latch.

### Input

```llvm
define void @two_continues(i32 %n, i32 %k) {
entry:
  br label %header
header:
  %i = phi i32 [ 0, %entry ], [ %i1, %c1 ], [ %i2, %c2 ]
  %cmp = icmp slt i32 %i, %n
  br i1 %cmp, label %body, label %exit
body:
  %hit = icmp eq i32 %i, %k
  br i1 %hit, label %c1, label %c2
c1:
  %i1 = add i32 %i, 1
  br label %header
c2:
  %i2 = add i32 %i, 2
  br label %header
exit:
  ret void
}
```

### Commands

```bash
opt -passes=loop-simplify -S two_continues.ll -o two_continues.out.ll
```

### Expected

`c1` and `c2` branch to a new latch; latch branches once to `header`. Header has one in-loop predecessor.

### Why it fires

Single-backedge canonicalization.

### Counterexample

Already a unique latch block that both paths jump to before the header → already canonical.

---

## Example 4: Pair with LCSSA

### Goal

See exit PHIs for externally used in-loop values.

### Input

```llvm
define i32 @need_lcssa(i32 %n) {
entry:
  br label %header
header:
  %i = phi i32 [ 0, %entry ], [ %i.next, %header ]
  %i.next = add i32 %i, 1
  %cmp = icmp slt i32 %i, %n
  br i1 %cmp, label %header, label %exit
exit:
  ret i32 %i
}
```

### Commands

```bash
opt -passes=loop-simplify -S need_lcssa.ll -o need_lcssa.ls.ll
opt -passes='loop-simplify,lcssa' -S need_lcssa.ll -o need_lcssa.out.ll
```

### Expected

With `lcssa`, `%i` used in `exit` feeds through a PHI in the exit (or dedicated loopexit) such that the loop-closed form holds. LoopSimplify alone may or may not insert it depending on `PreserveLCSSA` for that pass invocation—compare both outputs.

### Why it fires

LCSSA requirement for external uses of in-loop defs.

### Counterexample

Use `%i` only inside the loop; no exit PHI needed.

---

## Example 5: Clang `for` before/after print

### Goal

End-to-end on real frontend IR.

### Input

```c
int sum(int *a, int n) {
  int s = 0;
  for (int i = 0; i < n; ++i)
    s += a[i];
  return s;
}
```

### Commands

```bash
clang -S -emit-llvm -O1 -Xclang -disable-llvm-passes sum.c -o sum.ll
opt -passes='print<loops>' -disable-output sum.ll 2>before.txt
opt -passes=loop-simplify -S sum.ll -o sum.ls.ll
opt -passes='print<loops>' -disable-output sum.ls.ll 2>after.txt
diff -u before.txt after.txt || true
```

### Expected

After: explicit preheader naming; cleaner latch/exit structure in the loop dump. Depth and block list become what LICM expects.

### Why it fires

Frontend loops are natural but not always canonical.

### Counterexample

Run `simplifycfg` with options that destroy canonical loops, then print again—shape regresses until `loop-simplify` re-runs.

---

## Verification

```bash
opt -passes='loop-simplify,verify' -disable-output file.out.ll
```

Confirm with opaque pointers only (`ptr` on GEPs/loads in Clang output).
