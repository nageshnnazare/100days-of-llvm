# Day 16 Extra: Branch Weights Examples

```bash
clang -S -emit-llvm -O1 -Xclang -disable-llvm-passes FILE.c -o FILE.ll
opt -passes=lower-expect -S FILE.ll -o FILE.exp.ll
opt -passes='print<branch-prob>' -disable-output FILE.exp.ll
```

---

## Diagram

```text
   source                         IR

   if (__builtin_expect(c, 0))   br i1 %c, label %cold, label %hot
     cold();                      !prof !{!"branch_weights", i32 1, i32 2000}
   else
     hot();                       lower-expect turns llvm.expect
                                  into these weights

   MachineBlockPlacement later
   lays the heavy edge out as fall-through
```

`__builtin_expect` is not a branch hint the CPU sees directly. Clang emits `llvm.expect`, the `lower-expect` pass turns that into `!prof` branch weights, and the weights survive until block placement. A weight is a ratio, not a cycle count.

## Example 1: `__builtin_expect` unlikely true path

### Goal
Attach `branch_weights` favoring the false successor.

### Input (C)

```c
int errpath(int *p) {
  if (__builtin_expect(p == 0, 0))
    return -1;
  return *p;
}
```

### Commands

```bash
clang -S -emit-llvm -O1 -Xclang -disable-llvm-passes errpath.c -o errpath.ll
opt -passes=lower-expect -S errpath.ll -o errpath.exp.ll
grep -A2 'branch_weights' errpath.exp.ll
```

### Expected
`br` carries `!prof !{!"branch_weights", i32 W_true, i32 W_false}` with `W_true << W_false` (null path unlikely). Expect intrinsic gone.

### Why it fires
`lower-expect` maps expected value `0` on an equality-to-null style condition to unlikely-taken true edge (exact icmp shape depends on Clang IR).

### Counterexample
Omit the builtin → no expect intrinsic; BPI may still treat null as unlikely via **heuristics**, but metadata from expect will be missing.

---

## Example 2: `__builtin_expect_with_probability`

### Goal
Encode an explicit probability.

### Input (C)

```c
int rarely(int x) {
  if (__builtin_expect_with_probability(x == 1, 1, 0.05))
    return 42;
  return 0;
}
```

### Commands

```bash
clang -S -emit-llvm -O1 -Xclang -disable-llvm-passes rarely.c -o rarely.ll
# look for llvm.expect.with.probability
opt -passes=lower-expect -S rarely.ll -o rarely.exp.ll
```

### Expected
Weights roughly reflect ~5% for the expected equality path and the remainder on the other successor (scaled integers, not exact floats).

### Why it fires
`getBranchWeight` for `expect_with_probability` uses the double argument.

### Counterexample
Probability `0.0` for expected value → that case treated as unlikely (weight swap logic in PHI/branch handling).

---

## Example 3: Hand-written `branch_weights` on switch

### Goal
Read switch operand layout.

### Input (IR)

```llvm
define i32 @sw(i32 %x) {
entry:
  switch i32 %x, label %def [
    i32 0, label %a
    i32 1, label %b
  ], !prof !0
def:
  ret i32 -1
a:
  ret i32 10
b:
  ret i32 11
}

!0 = !{!"branch_weights", i32 1, i32 100, i32 5}
; default=1, case0=100, case1=5
```

### Commands

```bash
opt -passes='print<branch-prob>' -disable-output sw.ll
```

### Expected
BPI reports case 0 hottest among the three edges; default coldest.

### Why it fires
Metadata present → BPI uses weights directly.

### Counterexample
Wrong number of weight operands → metadata ignored / fallback heuristics.

---

## Example 4: `[[likely]]` / `[[unlikely]]` (C++)

### Goal
See FE metadata without writing builtins by hand.

### Input (C++)

```cpp
int classify(int x) {
  if (x)
    [[likely]] return 1;
  else
    [[unlikely]] return 0;
}
```

### Commands

```bash
clang++ -S -emit-llvm -O1 -Xclang -disable-llvm-passes likely.cpp -o likely.ll
opt -passes=lower-expect -S likely.ll -o likely.exp.ll
grep branch_weights -n likely.ll likely.exp.ll
```

### Expected
Weights favoring the likely label’s successor after lowering/FE emission (Clang may emit expect or weights depending on version—inspect the actual `.ll`).

### Why it fires
C++ attributes lower into the expect / prof channel.

### Counterexample
Same code without attributes → heuristic-only BPI.

---

## Example 5 (bonus): assume vs expect

### Goal
Keep the semantic distinction sharp.

### Input (IR)

```llvm
declare void @llvm.assume(i1)
declare i64 @llvm.expect.i64(i64, i64)

define i32 @assume_vs_expect(i32 %x) {
entry:
  %c = icmp ne i32 %x, 0
  ; Fact for optimizers:
  call void @llvm.assume(i1 %c)
  ; Hint only (after lower-expect on a real expect use):
  br i1 %c, label %a, label %b
a:
  ret i32 1
b:
  ret i32 0
}
```

### Commands

```bash
opt -passes=instcombine -S assume_vs_expect.ll -o assume.ic.ll
# Separately study expect.c through lower-expect
```

### Expected
`assume` can delete the false path or fold uses of `%x` under `%c` in ways **branch_weights never may**. Expect only changes probabilities.

### Why it matters
Teaching trap: treating expect like assume miscompiles mental models (and can mislead when reading passes).

### Counterexample
Replacing assume with expect metadata and hoping InstCombine deletes the false branch for “proof” reasons—must not happen from weights alone.

---

## Hands-on checklist

1. Builtin expect → weights.
2. Probability builtin scaling.
3. Hand-written switch weights + BPI print.
4. C++ likely/unlikely IR.
5. State aloud: assume = fact, expect = hint; this day ≠ instrumentation PGO.
