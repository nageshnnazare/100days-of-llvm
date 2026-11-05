# Day 26 Extra: Loop Rotation — Examples

Four worked cases plus counterexamples. Commands assume a modern `opt` new pass manager. Generate IR with:

```bash
clang -S -emit-llvm -O1 -Xclang -disable-llvm-passes file.c -o file.ll
```

See [Day-26-0-LoopRotate.md](Day-26-0-LoopRotate.md) for concepts.

---

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

## Example 1: Classic `while` becomes guarded do-while

### Input

```c
int add_range(int *p, int n) {
  int s = 0, i = 0;
  while (i < n) {
    s += p[i];
    ++i;
  }
  return s;
}
```

### Commands

```bash
clang -S -emit-llvm -O1 -Xclang -disable-llvm-passes add_range.c -o add_range.ll
opt -passes='loop-simplify,lcssa,loop-rotate' -S add_range.ll -o add_range.rot.ll
```

### Expected result

- A preheader/guard compares the initial induction (`0`) to `n` and can branch to the function exit without entering the loop.
- The latch performs `icmp` on the next index and either jumps to the body or exits.
- The old header is no longer the unique exiting block.

### Why it fires

The header is small, the exit condition is a cheap icmp on an IV, and the trip count may be zero—so rotation + guard is profitable and legal.

### Counterexample (must not / need not rotate)

```c
int once(int *p) {
  int s = 0, i = 0;
  do {
    s += p[i];
    ++i;
  } while (i < 1);
  return s;
}
```

Already bottom-tested with a trivial latch exit; further rotation is unnecessary. Profitability helpers should leave it alone or produce no useful change.

---

## Example 2: Enabling a later LICM-style hoist mental model

### Input

```c
extern int *G;
int sum_scaled(int *a, int n) {
  int s = 0;
  for (int i = 0; i < n; ++i)
    s += a[i] * (*G);
  return s;
}
```

### Commands

```bash
clang -S -emit-llvm -O1 -Xclang -disable-llvm-passes sum_scaled.c -o sum_scaled.ll
opt -passes='loop-simplify,lcssa,loop-rotate' -S sum_scaled.ll -o sum_scaled.rot.ll
# Then separately (illustration):
# opt -passes='loop-mssa(licm)' -S sum_scaled.rot.ll
```

### Expected result

After `loop-rotate`, the load of `*G` sits in a context where “body executes ⇒ loop entered.” Running LICM afterward can hoist `*G` to the guarded preheader without inventing a load on the `n <= 0` path (assuming aliasing/speculation rules allow).

### Why it fires

Same as Example 1: small for-loop header, affine IV, rotatable condition.

### Counterexample

```c
volatile int *G;
int sum_scaled_v(int *a, int n) {
  int s = 0;
  for (int i = 0; i < n; ++i)
    s += a[i] * (*G);
  return s;
}
```

Even if the loop rotates, **LICM must not** hoist the volatile load. Rotation enables structure; it does not override volatility. Use this to separate “CFG shape” from “hoist legality.”

---

## Example 3: Header too large — rotation refused

### Input

```c
void big(int n, int *out) {
  int i = 0;
  while (i < n) {
    // Many independent ops to inflate header/body cost in IR.
    int t = i;
    t ^= 1; t ^= 2; t ^= 3; t ^= 4; t ^= 5;
    t ^= 6; t ^= 7; t ^= 8; t ^= 9; t ^= 10;
    t ^= 11; t ^= 12; t ^= 13; t ^= 14; t ^= 15;
    t ^= 16; t ^= 17; t ^= 18; t ^= 19; t ^= 20;
    out[i] = t;
    ++i;
  }
}
```

### Commands

```bash
clang -S -emit-llvm -O1 -Xclang -disable-llvm-passes big.c -o big.ll
opt -passes='loop-simplify,lcssa,loop-rotate' -rotation-max-header-size=1 -S big.ll -o big.rot.ll
```

### Expected result

With a tiny threshold, the pass should **not** rotate (IR latch remains non-exiting / header still tests). Statistics / `-debug-only=loop-rotate` mention header size.

### Why it (does not) fire

Duplicating a large header would increase code size beyond the configured budget.

### Counterexample where it *does* fire

Same source with a large threshold:

```bash
opt -passes='loop-simplify,lcssa,loop-rotate' -rotation-max-header-size=1000 -S big.ll -o big.ok.ll
```

Now rotation may succeed—showing the knob is a heuristic, not a legality proof.

---

## Example 4: Non-duplicatable header side effects

### Input

```c
int maybe_call(int n) {
  int i = 0, s = 0;
  while (i < n) {
    s += i;
    ++i;
  }
  return s;
}
```

This *does* rotate. Contrast with a header that must call before the branch would be unnatural in C; construct IR instead:

```llvm
define void @header_call(i32 %n) {
entry:
  br label %header
header:
  %i = phi i32 [ 0, %entry ], [ %i2, %latch ]
  call void @side_effect()
  %cmp = icmp slt i32 %i, %n
  br i1 %cmp, label %body, label %exit
body:
  br label %latch
latch:
  %i2 = add i32 %i, 1
  br label %header
exit:
  ret void
}
declare void @side_effect()
```

### Commands

```bash
opt -passes='loop-simplify,lcssa,loop-rotate' -S header_call.ll -o header_call.rot.ll
```

### Expected result

If `@side_effect` is treated as unsafe to duplicate/speculate onto the guard/latch paths as required by the transform, rotation is skipped or the call is handled so it is not executed an extra time on zero-trip. Inspect IR: you must never observe **two** calls per iteration or a call when `n <= 0` if the original while would not call.

### Why legality matters

Rotation clones header instructions. Cloning a general call can change observable behavior. The pass’s safety checks exist precisely so “CFG sugar” cannot introduce extra side effects.

### Counterexample (safe, should rotate)

Remove the call; only `icmp` + add remain—Example 1 style. Rotation is expected.

---

## Example 5 (bonus): Disable via nonforced metadata

### Input IR fragment

```llvm
br label %header, !llvm.loop !0
!0 = distinct !{!0, !1}
!1 = !{!"llvm.loop.disable_nonforced"}
```

### Commands

```bash
opt -passes='loop-simplify,lcssa,loop-rotate' -S meta.ll -o meta.out.ll
```

### Expected result

Optional restructuring such as rotation should not apply when `disable_nonforced` is present (confirm against your LLVM version’s LoopUtils / metadata handling).

### Why

Frontends and `#pragma` machinery need a way to freeze loop shape for debugging or for ordered transform pipelines.

### Counterexample

Same loop **without** that metadata rotates normally.

---

## Quick checklist

| Case | Rotate? |
|------|---------|
| Small `while (i < n)` | Yes + guard |
| Already `do/while` latch-exit | Usually no |
| Huge header, tiny threshold | No |
| Duplicating would redo side effects unsafely | No |
| `llvm.loop.disable_nonforced` | No (optional transforms) |
