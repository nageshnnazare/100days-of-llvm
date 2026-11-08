# Day 30 Extra: Loop Interchange — Examples

```bash
clang -S -emit-llvm -O1 -Xclang -disable-llvm-passes f.c -o f.ll
opt -passes='loop-simplify,lcssa,loop-interchange' -S f.ll -o out.ll
```

Many nests will **not** interchange in a given LLVM revision; treat “expected” as “when the pass accepts,” and use remarks when it does not. Overview: [Day-30-0-LoopInterchange.md](Day-30-0-LoopInterchange.md).

---

## Diagram

```text
   for i:                         for j:
     for j:            --->         for i:
       A[j][i] = ...                  A[j][i] = ...

   row-major A[j][i] is contiguous
   in j, not in i. The inner loop
   should walk j.

   legality: every dependence
   distance must stay legal after
   the interchange (DependenceAnalysis)
```

Loop interchange swaps two perfectly nested loops so the inner one walks memory with unit stride. The swap is legal only when dependence analysis says the direction vectors still hold. A loop that carries a dependence on the outer index, or that is not tightly nested, is left alone. The profitability check asks which order is actually contiguous.

## Example 1: Column-stride inner loop (locality win)

### Input

```c
#define N 128
void col_major_touch(double A[N][N]) {
  for (int i = 0; i < N; ++i)
    for (int j = 0; j < N; ++j)
      A[j][i] += 1.0;
}
```

### Commands

```bash
opt -passes='loop-simplify,lcssa,loop-interchange' \
    -pass-remarks=loop-interchange -pass-remarks-missed=loop-interchange \
    -S col.ll -o col.out.ll
```

### Expected result

If accepted: loops swap so the inner IV walks the contiguous dimension of `A`. Remark `Interchanged`. If missed, remarks explain which check failed—still educational.

### Why it fires (when it does)

Tight rectangular nest, simple stores, dependence vectors allow swap, profitability sees huge stride improvement.

### Counterexample

```c
void already_good(double A[N][N]) {
  for (int i = 0; i < N; ++i)
    for (int j = 0; j < N; ++j)
      A[i][j] += 1.0; /* already contiguous inner */
}
```

Interchange is legal but **unprofitable** (or no-op benefit)—pass should skip.

---

## Example 2: Dependence prevents interchange

### Input

```c
#define N 64
void dep(int A[N]) {
  for (int i = 0; i < N; ++i)
    for (int j = 1; j < N; ++j)
      A[j] = A[j - 1] + i;
}
```

### Commands

Default interchange pipeline + missed remarks.

### Expected result

No interchange; dependence between `A[j]` and `A[j-1]` along the inner dimension is directional. Swapping loops would break the carried dependence order.

### Why it must not fire

Direction vector legality fails after permutation.

### Counterexample (independent)

```c
void indep(int A[N][N]) {
  for (int i = 0; i < N; ++i)
    for (int j = 0; j < N; ++j)
      A[i][j] = i + j;
}
```

No harmful cross-iteration dependence on a swapped order—legality OK (profitability TBD).

---

## Example 3: Call instruction blocks

### Input

```c
void g(int *);
void with_call(int A[64][64]) {
  for (int i = 0; i < 64; ++i)
    for (int j = 0; j < 64; ++j) {
      A[j][i]++;
      g(&A[j][i]);
    }
}
```

### Commands

Default pipeline.

### Expected result

Missed remark involving `CallInst` / unsupported; no interchange.

### Why it must not fire

Opaque call kills dependence certainty and violates the pass’s structural rules.

### Counterexample

Remove `g(...)`; Example 1 style may proceed.

---

## Example 4: Not tightly nested

### Input

```c
int sink;
void not_tight(int A[32][32]) {
  for (int i = 0; i < 32; ++i) {
    sink = i; /* intervening code */
    for (int j = 0; j < 32; ++j)
      A[j][i]++;
  }
}
```

### Commands

Default pipeline.

### Expected result

`NotTightlyNested` (or similar) missed remark; no interchange.

### Why it must not fire

Perfect nesting required; outer body is not just the inner loop.

### Counterexample

Move `sink = i` outside both loops or into a form the pass accepts as trivial—tight nest of only the store loop.

---

## Example 5 (bonus): Triangular bounds

### Input

```c
void triangle(int A[32][32]) {
  for (int i = 0; i < 32; ++i)
    for (int j = 0; j < i; ++j)
      A[j][i]++;
}
```

### Expected result

Typically unsupported (non-rectangular bounds); no interchange.

### Why it must not fire

Inner bound depends on outer IV in a triangular way the pass does not handle.

### Positive twin

Rectangular `j < 32` with legal/profitable access pattern (Example 1).

---

## Summary

| Case | Interchange? |
|------|----------------|
| `A[j][i]++` rectangular | Maybe yes |
| Already `A[i][j]++` | Usually no (unprofitable) |
| Inner carried dep `A[j]=A[j-1]` | No |
| Call in nest | No |
| Statement between loops | No |
| Triangular `j < i` | No |
