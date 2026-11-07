# Day 28 Extra: Simple Loop Unswitch — Examples

```bash
clang -S -emit-llvm -O1 -Xclang -disable-llvm-passes f.c -o f.ll
opt -passes='loop-simplify,lcssa,simple-loop-unswitch' -S f.ll -o out.ll
```

Overview: [Day-28-0-SimpleLoopUnswitch.md](Day-28-0-SimpleLoopUnswitch.md).

---

## Diagram

```text
   for (...) {                         if (cond) {
     if (cond)          unswitch         for (...) { A }
       A;              -------->       } else {
     else                               for (...) { B }
       B;                              }
   }

   `cond` does not change in the loop,
   so the branch is invariant
```

SimpleLoopUnswitch hoists an invariant branch out of the loop and duplicates the loop into the two arms. Each copy then has straight-line control where the branch used to be, so LICM and the vectorizer see a simpler body. The pass will not unswitch a branch that is trivial to fold another way, and it refuses when duplicating the body would blow the size threshold. Non-trivial unswitching freezes a possibly poison condition rather than branching on poison.

## Example 1: Non-trivial unswitch of an invariant `if`

### Input

```c
void accum(int *a, int n, int flag) {
  for (int i = 0; i < n; ++i) {
    if (flag)
      a[i] += 2;
    else
      a[i] += 3;
  }
}
```

### Commands

```bash
opt -passes='loop-simplify,lcssa,simple-loop-unswitch' -S accum.ll -o accum.out.ll
```

### Expected result

A preheader-level branch on `flag` (often after `freeze`) selecting between two loops; each loop body adds a constant without an inner flag branch.

### Why it fires

`flag` is loop-invariant; both sides stay in-loop; body is small → under `unswitch-threshold`.

### Counterexample

```c
void accum_var(int *a, int n, int *flags) {
  for (int i = 0; i < n; ++i) {
    if (flags[i])  /* varies per iteration */
      a[i] += 2;
    else
      a[i] += 3;
  }
}
```

Condition is **not** invariant → must not unswitch.

---

## Example 2: Trivial unswitch (invariant early exit)

### Input

```c
int find(int *a, int n, int enabled) {
  for (int i = 0; i < n; ++i) {
    if (!enabled)
      return -1; /* exit loop + function */
    if (a[i] == 0)
      return i;
  }
  return -2;
}
```

### Commands

Same pipeline as Example 1.

### Expected result

The `enabled` test can be hoisted: if `!enabled`, never enter the scan loop. This is the trivial flavor (exit edge), not necessarily a full dual clone of the search loop.

### Why it fires

Invariant condition gates an exiting path; trivial unswitch rewrites CFG cheaply.

### Counterexample

```c
int find2(int *a, int n) {
  for (int i = 0; i < n; ++i) {
    if (a[i] == 0)   /* depends on i */
      return i;
  }
  return -1;
}
```

Varying condition → no unswitch of that `if`.

---

## Example 3: Threshold refuses a huge body

### Input

```c
void bulky(int *a, int n, int flag) {
  for (int i = 0; i < n; ++i) {
    if (flag) {
      /* many statements → large IR */
      a[i] += 1; a[i] ^= 1; a[i] ^= 2; a[i] ^= 3; a[i] ^= 4;
      a[i] ^= 5; a[i] ^= 6; a[i] ^= 7; a[i] ^= 8; a[i] ^= 9;
      a[i] ^= 10; a[i] ^= 11; a[i] ^= 12; a[i] ^= 13; a[i] ^= 14;
    } else {
      a[i] -= 1; a[i] ^= 1; a[i] ^= 2; a[i] ^= 3; a[i] ^= 4;
      a[i] ^= 5; a[i] ^= 6; a[i] ^= 7; a[i] ^= 8; a[i] ^= 9;
      a[i] ^= 10; a[i] ^= 11; a[i] ^= 12; a[i] ^= 13; a[i] ^= 14;
    }
  }
}
```

### Commands

```bash
opt -passes='loop-simplify,lcssa,simple-loop-unswitch' -unswitch-threshold=1 -S bulky.ll -o bulky.out.ll
```

### Expected result

With threshold `1`, non-trivial unswitch should **not** duplicate; inner branch remains.

### Why (not)

Cost model exceeds `-unswitch-threshold`.

### Counterexample (does fire)

```bash
opt -passes='loop-simplify,lcssa,simple-loop-unswitch' -unswitch-threshold=100000 -S bulky.ll -o bulky.ok.ll
```

High threshold allows duplication—showing the limit is profitability, not a hard semantic ban.

---

## Example 4: Freeze on a possibly poisonous condition

### Input IR

```llvm
define void @f(ptr %a, i32 %n, i1 %cond) {
entry:
  br label %header
header:
  %i = phi i32 [ 0, %entry ], [ %i2, %latch ]
  %cmp = icmp slt i32 %i, %n
  br i1 %cmp, label %body, label %exit
body:
  br i1 %cond, label %then, label %else
then:
  store i32 1, ptr %a
  br label %latch
else:
  store i32 2, ptr %a
  br label %latch
latch:
  %i2 = add i32 %i, 1
  br label %header
exit:
  ret void
}
```

Here `%cond` is an arbitrary `i1` argument (can be poison in LLVM’s model).

### Commands

```bash
opt -passes='loop-simplify,lcssa,simple-loop-unswitch' -S freeze_cond.ll -o freeze_cond.out.ll
```

### Expected result

Look for `freeze i1 %cond` feeding the selector branch when `freeze-loop-unswitch-cond` is true.

### Why it fires

Invariant inner branch + freeze policy for poison safety.

### Counterexample

```bash
opt -passes='loop-simplify,lcssa,simple-loop-unswitch' -freeze-loop-unswitch-cond=false -S freeze_cond.ll
```

Freeze may be omitted (experimental/testing only)—do not use in production teaching without discussing poison risk.

---

## Example 5 (bonus): Switch unswitch (when supported)

### Input

```c
void sw(int *a, int n, int mode) {
  for (int i = 0; i < n; ++i) {
    switch (mode) {
    case 0: a[i] += 1; break;
    case 1: a[i] += 2; break;
    default: a[i] += 3; break;
    }
  }
}
```

### Commands

Same default pipeline.

### Expected result

Possibly multiple specialized loops or a tree of outer branches on `mode`. Exact shape depends on version; size threshold still applies.

### Why

Invariant switch condition.

### Counterexample

`switch (a[i])` — varies with iteration → no unswitch.

---

## Checklist

| Case | Unswitch? |
|------|-----------|
| Invariant if, small body | Yes (non-trivial) |
| Varying if | No |
| Invariant exit guard | Yes (trivial) |
| Huge body, tiny threshold | No |
| Invariant if + freeze on | Yes + `freeze` |
| Switch on `a[i]` | No |
