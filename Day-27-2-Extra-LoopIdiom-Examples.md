# Day 27 Extra: Loop Idiom — Examples

Base command pattern:

```bash
clang -S -emit-llvm -O1 -Xclang -disable-llvm-passes file.c -o file.ll
opt -passes='loop-simplify,lcssa,loop-idiom' -S file.ll -o out.ll
```

Use `-mtriple=x86_64-unknown-linux-gnu` on `opt` if libcall recognition matters. Theory: [Day-27-0-LoopIdiom.md](Day-27-0-LoopIdiom.md).

---

## Diagram

```text
   for (i = 0; i < n; ++i)
     a[i] = 0;

   loop-idiom recognizes a store of a
   loop-invariant byte across a unit stride

              |
              v
   memset(a, 0, n * sizeof(a[0]))

   the loop is gone; the library call remains
```

Loop idiom recognition looks for a loop that is really a memory primitive: a zeroing store becomes `llvm.memset`, a copy becomes `llvm.memcpy`, and a popcount-style shift loop can become a bit-count intrinsic. The replacement is profitable because the library call lowers to a tuned sequence. A loop that writes a non-invariant value, or that has a carried dependence, is not a memset.

## Example 1: Zeroing loop → `llvm.memset`

### Input

```c
void clear_bytes(char *p, unsigned n) {
  for (unsigned i = 0; i < n; ++i)
    p[i] = 0;
}
```

### Commands

```bash
clang -S -emit-llvm -O1 -Xclang -disable-llvm-passes clear.c -o clear.ll
opt -passes='loop-simplify,lcssa,loop-idiom' -S clear.ll -o clear.out.ll
```

### Expected result

The loop disappears; a `call void @llvm.memset.p0.i64(...)` (or similar) appears with value `i8 0` and length derived from `n`.

### Why it fires

Affine pointer AddRec, constant zero store, contiguous i8 coverage, non-volatile, analyzable trip count.

### Counterexample

```c
void clear_volatile(volatile char *p, unsigned n) {
  for (unsigned i = 0; i < n; ++i)
    p[i] = 0;
}
```

Volatile stores must remain a loop (or volatile memset only if explicitly formed—LIR will not silently drop volatility).

---

## Example 2: Copy loop → `llvm.memcpy`

### Input

```c
void copy_ints(int *restrict d, int *restrict s, int n) {
  for (int i = 0; i < n; ++i)
    d[i] = s[i];
}
```

`restrict` (noalias at IR level when clang emits it) helps AA prove disjointness.

### Commands

```bash
clang -S -emit-llvm -O1 -Xclang -disable-llvm-passes copy.c -o copy.ll
opt -passes='loop-simplify,lcssa,loop-idiom' -S copy.ll -o copy.out.ll
```

### Expected result

A single `llvm.memcpy` of `n * sizeof(int)` bytes (sext/zext of `n` as needed).

### Why it fires

Matched load/store pair, SCEV AddRecs with equal stride, NoAlias between objects thanks to `restrict` / noalias.

### Counterexample

```c
void copy_maybe_overlap(int *d, int *s, int n) {
  for (int i = 0; i < n; ++i)
    d[i] = s[i];
}
```

Without restrict, BasicAA often returns MayAlias for two `int*` parameters → memcpy idiom **must not** fire as plain memcpy. (memmove might be applicable only if the pass proves that path; do not assume it.)

---

## Example 3: Popcount loop → `ctpop`

### Input

```c
int popcount(unsigned x) {
  int c = 0;
  while (x) {
    x &= x - 1;
    ++c;
  }
  return c;
}
```

### Commands

```bash
clang -S -emit-llvm -O1 -Xclang -disable-llvm-passes pop.c -o pop.ll
opt -passes='loop-simplify,lcssa,loop-idiom' -S pop.ll -o pop.out.ll
```

### Expected result

On successful recognition, something equivalent to `llvm.ctpop.i32` (possibly after slight IR canonicalization). If your revision’s matcher misses, `-debug-only=loop-idiom` shows where the pattern failed—still a useful lesson.

### Why it fires

Each iteration clears exactly one set bit; iteration count equals population count; finite for any `unsigned`.

### Counterexample

```c
int popcount_noisy(unsigned x) {
  int c = 0;
  while (x) {
    x &= x - 1;
    ++c;
    sink(); /* extra side effect */
  }
  return c;
}
void sink(void);
```

Extra calls break the pure idiom; must not replace with only `ctpop` (would drop calls).

---

## Example 4: Shift-until-zero style

### Input

```c
unsigned shift_until_zero(unsigned x) {
  unsigned n = 0;
  while (x) {
    x >>= 1;
    ++n;
  }
  return n;
}
```

### Commands

```bash
clang -S -emit-llvm -O1 -Xclang -disable-llvm-passes shf.c -o shf.ll
opt -debug-only=loop-idiom -passes='loop-simplify,lcssa,loop-idiom' -S shf.ll -o shf.out.ll
```

### Expected result

When recognized, the loop becomes a bit-length computation related to `ctlz`/`cttz` style math (exact intrinsic depends on matcher: shift-until-zero path). Debug may print `shift-until-zero idiom detected!`.

### Why it fires

Analyzable shift recurrence until zero; finite for logical right shift on unbounded iterations bounded by bit width.

### Counterexample

```c
unsigned shift_by_var(unsigned x, unsigned s) {
  unsigned n = 0;
  while (x) {
    x >>= s; /* non-constant / may never hit zero if s==0 */
    ++n;
  }
  return n;
}
```

If `s == 0` the loop may be infinite; SCEV/finiteness checks should reject. Must not rewrite blindly.

---

## Example 5 (bonus): Non-affine store — no memset

### Input

```c
void scatter_zero(int *p, int n) {
  for (int i = 0; i < n; ++i)
    p[i * i] = 0; /* when i*i fits */
}
```

### Commands

Same idiom pipeline as Example 1.

### Expected result

Loop remains. No memset covering a contiguous range starting at `p`.

### Why it must not fire

Pointer is not an affine AddRec with constant stride relative to `i` in the memset sense (`i*i` is quadratic). Contiguity fails.

### Positive twin

`p[i] = 0` (Example 1) fires.

---

## Summary table

| Example | Idiom | Fires? |
|---------|-------|--------|
| Zero bytes | memset | Yes |
| Volatile zero | — | No |
| restrict copy | memcpy | Yes |
| May-alias copy | memcpy | No |
| `x &= x-1` count | ctpop | Often yes |
| Noisy popcount | — | No |
| Affine shift-to-zero | bit scan | Often yes |
| Quadratic index | memset | No |
