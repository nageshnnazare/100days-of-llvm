# Day 24 Extra: Loop Unroll — Examples

```bash
clang -S -emit-llvm -O1 -Xclang -disable-llvm-passes file.c -o file.ll
opt -passes='loop-simplify,lcssa,loop-unroll' -S file.ll -o file.out.ll
```

See [Day-24-0-LoopUnroll.md](Day-24-0-LoopUnroll.md).

---

## Diagram

```text
   for (i = 0; i < 4; i++)           i = 0
     a[i] = i;                       a[0] = 0
                                     a[1] = 1
   unroll count 4, no remainder:     a[2] = 2
                                     a[3] = 3

   for (i = 0; i < n; i++)           if n is not a multiple of 4:
     a[i] = i;                       a prologue or epilogue loop
                                     runs the leftover iterations,
                                     and the main loop steps by 4
```

Full unrolling copies the body and deletes the branch when the trip count is a small constant. Runtime unrolling peels off a remainder so the unrolled body can assume a multiple of the unroll factor. The pass estimates code size against a threshold; a large body with a huge trip count stays rolled.

## Example 1: Full unroll of a tiny constant trip count

### Input

```c
void add4(int *restrict a, int *restrict b) {
  for (int i = 0; i < 4; ++i)
    a[i] = b[i] + 1;
}
```

### Commands

```bash
opt -passes='loop-simplify,lcssa,loop-unroll' -S add4.ll -o add4.unroll.ll
# Force if thresholds disagree on your target:
opt -passes='loop-simplify,lcssa,loop-unroll' -unroll-count=4 -S add4.ll -o add4.force.ll
```

### Expected result

Loop disappears (fully unrolled) or becomes straight-line loads/stores for `i=0..3`. No backedge to a 4-iteration header.

### Why it fires

SCEV constant trip count 4; cost under threshold (or forced count).

### Counterexample

Raise trip count to 10 000 without pragma — cost model refuses full unroll.

---

## Example 2: Runtime unroll with remainder

### Input

```c
void addn(int *restrict a, int *restrict b, int n) {
  for (int i = 0; i < n; ++i)
    a[i] = b[i] + 1;
}
```

### Commands

```bash
opt -passes='loop-simplify,lcssa,loop-unroll' -unroll-runtime -unroll-count=4 -S addn.ll -o addn.rt.ll
```

### Expected result

CFG with an unrolled main loop stepping by 4 and a remainder/epilogue loop for `n % 4` (shape depends on epilogue vs prologue). Two loops is normal.

### Why it fires

Unknown trip count + runtime enabled + explicit count.

### Counterexample

Metadata `llvm.loop.unroll.runtime.disable` or omitting `-unroll-runtime` when TTI leaves `UP.Runtime` false → stays singly rolled.

---

## Example 3: Pragma full unroll

### Input

```c
void add8(int *restrict a, int *restrict b) {
#pragma clang loop unroll(full)
  for (int i = 0; i < 8; ++i)
    a[i] = b[i] + 1;
}
```

### Commands

```bash
clang -S -emit-llvm -O1 -Xclang -disable-llvm-passes add8.c -o add8.ll
grep unroll add8.ll   # expect llvm.loop.unroll.full (or related)
opt -passes='loop-simplify,lcssa,loop-unroll' -S add8.ll -o add8.out.ll
```

### Expected result

Metadata present; pass tries hard to fully unroll (subject to `pragma-unroll-full-max-iterations` style guards).

### Why it fires

`UnrollPragmaInfo` / `hasUnrollFullPragma` path boosts full unroll priority.

### Counterexample

Trip count not SCEV-constant and huge — full pragma cannot invent a finite unroll of an unbounded loop; may fall back to runtime/other strategies or fail.

---

## Example 4: Threshold blocks partial unroll

### Input

Large loop body (many statements) with moderate constant trip count.

```c
void fat(int *a, int *b) {
  for (int i = 0; i < 16; ++i) {
    a[i] = b[i] + 1;
    a[i] += b[i] * 2;
    a[i] ^= b[i] + i;
    /* imagine many more ops */
  }
}
```

### Commands

```bash
opt -passes='loop-simplify,lcssa,loop-unroll' -unroll-threshold=1 -S fat.ll -o fat.no.ll
opt -passes='loop-simplify,lcssa,loop-unroll' -unroll-threshold=100000 -S fat.ll -o fat.yes.ll
```

### Expected result

Tiny threshold → little or no unroll. Huge threshold → partial/full more likely.

### Why it fires / refuses

`UnrollCostEstimator` vs `UP.Threshold`.

### Counterexample

`-unroll-count=16` can still force a count even when threshold would discourage heuristic selection (force/pragma paths)—compare both.

---

## Example 5: Unroll-and-jam is a different pass

### Input

Nested loops suitable for jam (dependence-friendly).

### Commands

```bash
opt -passes='loop-simplify,lcssa,loop-unroll' -S nest.ll -o only.unroll.ll
opt -passes='loop-simplify,lcssa,loop-unroll-and-jam' -S nest.ll -o jam.ll
```

### Expected result

Different IR shapes. Unroll-and-jam unrolls an outer loop and fuses inner bodies; ordinary unroll does not jam.

### Why it matters

Metadata `llvm.loop.unroll_and_jam` may cause `loop-unroll` to skip so jam can run.

### Counterexample

Expecting `loop-unroll` alone to produce jammed outer/inner code — it will not.

---

## Example 6 (bonus): Debug trace

```bash
opt -debug-only=loop-unroll -passes='loop-simplify,lcssa,loop-unroll' \
  -unroll-runtime -unroll-count=4 -S addn.ll -o /dev/null
```

Look for trip-count dumps, “Trying runtime unroll”, remainder construction, or precondition failures (“Can't unroll; loop preheader-insertion failed”).

---

## Table

| Goal | Lever |
|------|--------|
| Full small TC | default or `-unroll-count` |
| Runtime + tail | `-unroll-runtime` (+ count) |
| Honor source hint | `#pragma` → `llvm.loop.unroll.*` |
| Stop size blow-up | lower `-unroll-threshold` |
| Jam nests | `loop-unroll-and-jam` |
