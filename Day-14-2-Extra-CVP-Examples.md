# Day 14 Extra: CVP Examples

```bash
clang -S -emit-llvm -O1 -Xclang -disable-llvm-passes FILE.c -o FILE.ll
opt -passes=correlated-propagation -S FILE.ll -o FILE.cvp.ll
opt -passes=correlated-propagation -debug-only=correlated-value-propagation -S FILE.ll
```

---

## Diagram

```text
   %c = icmp ult i32 %x, 10
   br i1 %c, label %then, label %else

   then:                         else:
   LVI facts:                    LVI facts:
   %x in [0, 9]                  %x in [10, unsigned max]
   %d = icmp ult i32 %x, 10      %e = icmp ult i32 %x, 5
   --> true                      --> false
```

CVP uses LazyValueInfo, which intersects conditions along the path to the block. Inside `%then` the compiler already knows `%x` is below 10, so a second identical compare is the constant `true`. The information is a range or a predicate, not only a single constant, which is why this catches cases SCCP misses.

## Example 1: Fold icmp under a dominating range

### Goal
After `x <= 5` is known, fold `x < 10` to true.

### Input (C)

```c
int f(unsigned x) {
  if (x > 5)
    return 0;
  if (x < 10)
    return 1;
  return 2;
}
```

### Input (IR sketch)

```llvm
define i32 @f(i32 %x) {
entry:
  %cmp = icmp ugt i32 %x, 5
  br i1 %cmp, label %ret0, label %bb
ret0:
  ret i32 0
bb:
  %cmp2 = icmp ult i32 %x, 10
  br i1 %cmp2, label %ret1, label %ret2
ret1:
  ret i32 1
ret2:
  ret i32 2
}
```

### Commands

```bash
opt -passes=correlated-propagation -S f.ll -o f.cvp.ll
```

### Expected
In `%bb`, LVI knows `%x ∈ [0,5]`; `%cmp2` folds to true; `%ret2` becomes dead for later CFG passes.

### Why it fires
`processCmp` / `constantFoldCmp` with ranges at the icmp use.

### Counterexample
Remove the first guard → `%cmp2` not constant.

---

## Example 2: Delete impossible switch cases

### Goal
Remove switch cases outside the known range.

### Input (IR)

```llvm
define i32 @sw(i32 %x) {
entry:
  %c = icmp ult i32 %x, 3
  br i1 %c, label %dispatch, label %out
out:
  ret i32 -1
dispatch:
  switch i32 %x, label %def [
    i32 0, label %a
    i32 1, label %b
    i32 2, label %c
    i32 9, label %deadcase
  ]
a:
  ret i32 10
b:
  ret i32 11
c:
  ret i32 12
def:
  ret i32 99
deadcase:
  ret i32 77
}
```

### Commands

```bash
opt -passes=correlated-propagation -S sw.ll -o sw.cvp.ll
```

### Expected
Case `9` removed (`NumDeadCases`); range at switch is `{0,1,2}`.

### Why it fires
`processSwitch` + `getConstantRangeAtUse` on the condition.

### Counterexample
Switch on unconstrained `%x` → case 9 remains.

---

## Example 3: urem with nonzero denominator range

### Goal
Simplify or keep safe `urem` when denominator cannot be zero.

### Input (C)

```c
unsigned rem(unsigned x, unsigned d) {
  if (d == 0)
    return 0;
  return x % d;
}
```

### Input (IR sketch after branch)

```llvm
; in block where %d != 0:
;   %r = urem i32 %x, %d
```

### Commands

```bash
clang -S -emit-llvm -O1 -Xclang -disable-llvm-passes rem.c -o rem.ll
opt -passes=correlated-propagation -S rem.ll -o rem.cvp.ll
```

### Expected
CVP can rewrite/narrow based on ranges; at minimum, LVI proves `%d` nonzero on that edge so transforms that require nonzero denom are legal. Compare IR before/after for `processUDivOrURem` effects (narrowing when high bits unused).

### Why it fires
Dominating `d != 0` edge fact → safe unsigned rem/div rewrites.

### Counterexample
`urem` without a nonzero guard → CVP must not introduce UB by assuming nonzero.

---

## Example 4: and/or of icmps / select correlation

### Goal
Simplify a select or boolean chain when one icmp is implied.

### Input (IR)

```llvm
define i32 @sel(i32 %x) {
entry:
  %c1 = icmp ult i32 %x, 4
  br i1 %c1, label %then, label %else
then:
  %c2 = icmp ult i32 %x, 8
  %s = select i1 %c2, i32 1, i32 2
  ret i32 %s
else:
  ret i32 0
}
```

### Commands

```bash
opt -passes=correlated-propagation -S sel.ll -o sel.cvp.ll
```

### Expected
In `%then`, `%x < 4` ⇒ `%x < 8` true ⇒ select collapses to `1`.

### Why it fires
`processSelect` / cmp folding with correlated ranges.

### Counterexample
Outer guard is `%x < 10` and inner is `%x < 4` — inner not constant true.

---

## Example 5 (bonus): nsw from ranges (poison care)

### Goal
See CVP add `nsw` when ranges prove no signed overflow—and understand the poison contract.

### Input (IR)

```llvm
define i32 @add_nsw(i32 %x) {
entry:
  %c = icmp ult i32 %x, 100
  br i1 %c, label %bb, label %out
out:
  ret i32 0
bb:
  %a = add i32 %x, 1
  ret i32 %a
}
```

### Commands

```bash
opt -passes=correlated-propagation -S add_nsw.ll -o add_nsw.cvp.ll
```

### Expected
Possible `add nuw`/`nsw` on `%a` when LVI ranges prove no wrap (`processBinOp`). If flags appear, overflow would be poison—only legal because `%x` is in a safe range on that edge.

### Why it fires
`makeGuaranteedNoWrapRegion` contains the operand range.

### Counterexample
`add` in a block where `%x` can be `INT_MAX` → no nsw.

---

## Hands-on checklist

1. Range-fold icmp.
2. Switch case deletion.
3. Div/rem with nonzero denom.
4. Select collapse.
5. Inspect nowrap flags; argue poison safety aloud.
