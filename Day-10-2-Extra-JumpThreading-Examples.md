# Day 10 Extra: Jump Threading Examples

```bash
opt -passes=jump-threading -S ex.ll -o jt.ll
opt -passes=jump-threading,simplifycfg,instcombine -S ex.ll -o pipe.ll
```

---

## Diagram

```text
        entry: %c = ...
          |
          br i1 %c
         /        \
      then         else
      %c is true   %c is false
        |            |
        br i1 %c     br i1 %c
       /    \          |
     T(dead) F        F is the only
                      possible target

   after jump threading, `else` branches
   straight to F. The dead edge is gone.
```

Threading copies a destination onto a predecessor when that predecessor already knows how the branch will go. It does not merge blocks; that is SimplifyCFG. Each threaded edge has a cost, and `-jump-threading-threshold` stops the pass from cloning a large block.

## Example 1: Phi condition

```llvm
define i32 @phi(i1 %c, i32 %x) {
entry:
  br i1 %c, label %left, label %right
left:
  br label %join
right:
  br label %join
join:
  %p = phi i32 [ 1, %left ], [ %x, %right ]
  %cmp = icmp eq i32 %p, 1
  br i1 %cmp, label %yes, label %no
yes:
  ret i32 10
no:
  ret i32 %p
}
```

**Expected:** `left` no longer flows through the compare. It goes to the `ret i32 10` block, possibly via a cloned copy of `join` that constant-folded the phi to 1. `right` still compares `%x` to 1.

**Why:** on the `left` incoming edge the phi is the constant 1, so the compare is true.

**Counterexample:** change the phi's `left` value from `1` to `%x`. Neither edge has a constant compare. `jump-threading` should not rewire anything. `diff` against the input should be empty (or only name changes, which this pass does not do if it made no edit).

---

## Example 2: The same test, written as two branches

```llvm
define i32 @corr(i32 %x) {
entry:
  %c1 = icmp eq i32 %x, 1
  br i1 %c1, label %is1, label %not1
is1:
  %c2 = icmp eq i32 %x, 1
  br i1 %c2, label %yes, label %no
not1:
  br label %no
yes:
  ret i32 10
no:
  %r = phi i32 [ %x, %not1 ], [ %x, %is1 ]
  ret i32 %r
}
```

**Expected:** the branch in `is1` disappears. On every path that reaches `is1`, `%x` is 1, so `c2` is true. This one is so redundant that SimplifyCFG or InstCombine may get it even without jump threading. Try each pass alone:

```bash
opt -passes=instcombine -S corr.ll
opt -passes=simplifycfg -S corr.ll
opt -passes=jump-threading -S corr.ll
```

**What you are looking for:** at least one of them removes the second compare. Jump threading's version rewires the edge. InstCombine's version may replace `%c2` with `true` because `%c1` dominates and implies it, but only if InstCombine asks LVI or CVP-like logic — classic InstCombine does not always do that. Note which pass actually fired on your LLVM version. That note is more useful than a textbook claim.

**Counterexample:**

```llvm
define i32 @notcorr(i32 %x, i32 %y) {
entry:
  %c1 = icmp eq i32 %x, 1
  br i1 %c1, label %is1, label %no
is1:
  %c2 = icmp eq i32 %y, 1
  br i1 %c2, label %yes, label %no
yes:
  ret i32 10
no:
  ret i32 0
}
```

`%y` is unrelated. The second branch must remain under `jump-threading`.

---

## Example 3: Switch

```llvm
define i32 @sw(i1 %c, i32 %x) {
entry:
  br i1 %c, label %L, label %R
L:
  br label %J
R:
  br label %J
J:
  %p = phi i32 [ 2, %L ], [ %x, %R ]
  switch i32 %p, label %def [
    i32 2, label %two
    i32 3, label %three
  ]
two:
  ret i32 2
three:
  ret i32 3
def:
  ret i32 %p
}
```

**Expected:** from `L`, control goes to `two` without executing the switch.

**Why:** the phi is 2 on that edge, and 2 is a switch case.

**Counterexample:** if the `L` incoming value is `%x` as well, there is nothing to thread. A switch on a non-constant phi stays a switch. (SimplifyCFG may later turn a small switch into branches. Still not threading.)

---

## Example 4: Threshold refusal

```llvm
define i32 @big(i1 %c, i32 %x) {
entry:
  br i1 %c, label %L, label %R
L:
  br label %J
R:
  br label %J
J:
  %p = phi i32 [ 1, %L ], [ %x, %R ]
  %a1 = add i32 %p, 1
  %a2 = add i32 %a1, 1
  %a3 = add i32 %a2, 1
  %a4 = add i32 %a3, 1
  %a5 = add i32 %a4, 1
  %a6 = add i32 %a5, 1
  %a7 = add i32 %a6, 1
  %a8 = add i32 %a7, 1
  %cmp = icmp eq i32 %a8, 9
  br i1 %cmp, label %yes, label %no
yes:
  ret i32 %a8
no:
  ret i32 0
}
```

```bash
opt -passes=jump-threading -jump-threading-threshold=1 -S big.ll -o tight.ll
opt -passes=jump-threading -jump-threading-threshold=100 -S big.ll -o loose.ll
```

**Expected:** `tight.ll` still has `L` flowing into `J`, because cloning eight adds exceeds a threshold of 1. `loose.ll` threads `L` and folds the known `%p = 1` chain on that copy, so the true side returns a constant.

**Why:** duplication budget.

**Counterexample:** delete `%a1` through `%a7` and branch on `%p == 1` directly, with threshold 1. The block is small. Threading should happen even at threshold 1. If it does not, the cost model is counting something you did not expect (the phis, the compare). Look at `-debug-only=jump-threading`.

---

## Example 5: Must not clone a `noduplicate` call

```llvm
declare void @barrier() nounwind
define void @nd(i1 %c, i1 %k) {
entry:
  br i1 %c, label %L, label %R
L:
  br label %J
R:
  br label %J
J:
  call void @barrier() noduplicate
  br i1 %k, label %yes, label %no
yes:
  ret void
no:
  ret void
}
```

The call sits in `J` and the branch condition `%k` is **not** known on either incoming edge. Threading should not fire anyway. Change the example so the branch is on a phi like Example 1, and keep `noduplicate` on the call. The pass must refuse to clone `J`. The call stays in one block. Cloning a `noduplicate` call is a miscompile for intrinsics and barriers that the frontend promised appear once (OpenMP barriers, some setjmp uses).

**Counterexample that should thread:** remove the call entirely. Then it reduces to Example 1 and should thread.

---

## Example 6: Correlation created by inlining

```c
static int positive(int x) {
  if (x > 0)
    return x;
  return 0;
}
int caller(int x) {
  if (x > 0)
    return positive(x);
  return -1;
}
```

```bash
clang -S -emit-llvm -O1 -Xclang -disable-llvm-passes corr.c -o corr.ll
opt -passes=inline,jump-threading,simplifycfg -S corr.ll
```

**Expected:** after inlining, the inner `x > 0` is redundant on the path that already checked `x > 0`. Jump threading or CVP removes the second test. The function returns `%x` on the positive path and `-1` on the other, with one compare.

**Counterexample:** compile without `inline` in the pipeline. `jump-threading` on the un-inlined IR does not see across the call. The second compare is in another function. This is the Day 42 interaction: threading cleans up what inlining exposes.

---

## How to draw the result

For Example 1, draw four boxes before running `opt`, then draw the edges `opt` produced. The edge that should have moved is `left -> join -> yes` becoming `left -> yes`. If you cannot point at that edge in the output, print the CFG:

```bash
opt -passes=dot-cfg -disable-output jt.ll
```

and look at the arrow out of `left`.
