# Day 29 Extra: IndVarSimplify — Examples

```bash
clang -S -emit-llvm -O1 -Xclang -disable-llvm-passes f.c -o f.ll
opt -passes='loop-simplify,lcssa,indvars' -S f.ll -o out.ll
```

---

## Diagram

```text
   %i = phi i32 [ 0, %pre ], [ %i.next, %latch ]
   %i.next = add nsw i32 %i, 1
   %cmp = icmp slt i32 %i, %n

   indvars rewrites users of %i onto a
   canonical induction variable, and
   widens a narrow IV when the arithmetic
   is legal:

   %i64 = phi i64 [ 0, %pre ], [ %next, %latch ]
   %next = add nuw nsw i64 %i64, 1
```

IndVarSimplify picks one induction variable as canonical and rewrites the others in terms of it. Strength reduction turns `i * scale` into a recurrence that only adds. Trip-count compares become compares against the canonical IV. Widening a 32-bit IV to 64-bit is done only when the extra bits cannot change the defined result; a wrapping IV is left narrow.

## Example 1: Exit value becomes a closed form

### Input

```c
int last(int n) {
  int i;
  for (i = 0; i < n; ++i) {
  }
  return i; /* == n when n >= 0 in this pattern */
}
```

### Commands

```bash
opt -passes='loop-simplify,lcssa,indvars' -S last.ll -o last.out.ll
```

### Expected result

The LCSSA of `i` at return is replaced by a value equivalent to `n` (possibly after casts), and the loop may simplify further.

### Why it fires

SCEV knows the exit value of `{0,+,1}` when exiting on `i < n` / related predicates.

### Counterexample

```c
int last_break(int n, int k) {
  int i;
  for (i = 0; i < n; ++i)
    if (i == k) break;
  return i;
}
```

Multiple exit reasons → exit value is not simply `n`; rewrite must not blindly substitute `n`.

---

## Example 2: LFTR-style exit test rewrite

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
opt -passes='loop-simplify,lcssa,indvars' -S sum.ll -o sum.out.ll
# Compare with LFTR disabled:
opt -passes='loop-simplify,lcssa,indvars' -disable-lftr -S sum.ll -o sum.nolftr.ll
```

### Expected result

With LFTR on, the exiting compare often becomes an EQ/NE against a canonical counter limit derived from the trip count, rather than a relational compare on a non-canonical IV—exact shape varies. Diff the two outputs to see LFTR’s contribution.

### Why it fires

`needsLFTR` finds a non-ideal exit test; SCEV provides a limit.

### Counterexample

```bash
opt -disable-lftr ...
```

LFTR does not run; exit icmp may remain relational on the original IV.

---

## Example 3: IV widening removes in-loop extends

### Input

```c
long sum64(int *a, int n) {
  long s = 0;
  for (int i = 0; i < n; ++i)
    s += a[i];
  return s;
}
```

On LP64, GEP indices are widened; clang may emit `sext i32 %i to i64` each iteration before indvars.

### Commands

Default indvars pipeline.

### Expected result

A widened IV (i64) feeding GEPs without a fresh `sext` from a narrow PHI each time (subject to legality).

### Why it fires

SCEV proves the narrow IV’s extends are congruent to a wide AddRec.

### Counterexample

Cases where narrow wraparound is required for correctness (IV that intentionally wraps in `i8`/`i16`) must **not** widen in a way that changes wrap behavior. Construct a loop that depends on `i8` overflow to terminate—and ensure indvars does not “fix” it into infinite `i32` arithmetic.

---

## Example 4: Redundant IV compare elimination

### Input

```c
int both(int n) {
  int c = 0;
  for (int i = 0, j = 0; i < n; ++i, ++j) {
    if (i == j) /* always true */
      c++;
  }
  return c;
}
```

### Commands

Default indvars (+ maybe instcombine afterward).

### Expected result

The `i == j` test folds; `c` becomes related to the trip count.

### Why it fires

SCEV shows `i` and `j` share the same AddRec.

### Counterexample

```c
int both2(int n, int step) {
  int c = 0;
  for (int i = 0, j = 0; i < n; ++i, j += step)
    if (i == j) c++;
  return c;
}
```

Different steps → compare not redundant; must remain.

---

## Example 5 (bonus): Poison-sensitive LFTR refusal (conceptual)

### Input sketch

An IV with `add` **without** `nsw`, used in a relational exit, where LFTR would need to assume non-wrapping to use a different counter.

### Expected result

LFTR may skip (`-debug-only=indvars` explains). Must not invent `nsw` compares that poison when the original did not.

### Counterexample / safe twin

Same loop with `add nsw` and SCEV flags that justify the rewrite → LFTR allowed.

---

## Table

| Example | Transform | Must not |
|---------|-----------|----------|
| `return i` after `i<n` | Exit rewrite → `n` | Multi-exit break |
| Sum loop | LFTR exit icmp | When `-disable-lftr` |
| `int i` used as i64 index | Widen | Intentional narrow wrap |
| `i` and `j` same step | Delete cmp | Different steps |
| Missing nsw | Conservative LFTR | Invent poison |
