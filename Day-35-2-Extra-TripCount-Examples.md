# Day 35 Extra: Trip Count — Examples

```bash
clang -S -emit-llvm -O1 -Xclang -disable-llvm-passes f.c -o f.ll
opt -passes='print<scalar-evolution>' -disable-output f.ll
```

Parent: [Day-35-0-SCEV-TripCount.md](Day-35-0-SCEV-TripCount.md).

---

## Diagram

```text
   for (i = 0; i < n; ++i)        trip count = n
                                  backedge taken count = n-1
                                  (the latch runs n-1 times
                                   if n >= 1; 0 times if n == 0)

   SCEV:
     backedge-taken count = umax(n, 1) - 1
     or, with a guard:      n - 1

   a data-dependent exit
     if (a[i] == 0) break;
   has no computable trip count
```

The trip count is how many times the header executes. The backedge-taken count is how many times the latch branches back, which is one less when the loop runs at least once. ScalarEvolution computes the backedge count from the exit predicate. A loop with a break that depends on loaded memory has an unknown count, and the vectorizer must then version or give up.

## Example 1: Symbolic `for (i=0; i<n; ++i)`

### Input

```c
void f(int n) {
  for (int i = 0; i < n; ++i) { }
}
```

### Commands

`print<scalar-evolution>` and/or debug.

### Expected result

IV `{0,+,1}<L>`. Backedge-taken count symbolic in `n` (classically `n-1` when `n>0`), with possible `umax`/predicate formulation when `n` may be zero. Trip count conceptually `n` when entered.

### Why it fires

`howManyLessThans` on AddRec vs `n`.

### Counterexample / complication

Caller may pass `n==0`: zero-trip; BTC formulas must not imply the body ran. Guarded/rotated form clarifies.

---

## Example 2: Constant trip count

### Input

```c
void g(int *p) {
  for (int i = 0; i < 64; ++i)
    p[i] = 0;
}
```

### Commands

Same printer; also:

```bash
opt -passes='loop-unroll' -S g.ll   # may fully unroll when BTC constant
```

### Expected result

BTC `63`, trip count `64` → `getSmallConstantTripCount` nonzero small constant. Unroll may fully expand.

### Why it fires

Constant limit + affine IV + no wrap issues.

### Counterexample

```c
void g2(int *p, int n) {
  for (int i = 0; i < 64; ++i)
    if (i == n) break;
  ...
}
```

Second exit destroys simple constant BTC for the whole loop.

---

## Example 3: Multiple exits → minimum / unknown

### Input

```c
int find(int *a, int n, int k) {
  for (int i = 0; i < n; ++i)
    if (a[i] == k) return i;
  return -1;
}
```

### Commands

Printer on the loop.

### Expected result

Per-exit counts: one related to `n`, one data-dependent. Overall exact BTC often **CouldNotCompute**; max may still be `n`-related.

### Why

`getExitCount` for the early return is not a closed SCEV of `i` alone.

### Counterexample

Remove the early return → single exit → BTC `n-1` (with zero-trip caveats).

---

## Example 4: Wrapping `i8` IV

### Input

```c
void wrap(void) {
  unsigned char i = 0;
  do {
    ++i;
  } while (i != 0); /* 256 iterations if it starts at 0 and increments */
}
```

Or an open-coded loop that relies on `i8` wrap. Exact IR depends on clang.

### Commands

`print<scalar-evolution>` on the IV type `i8`.

### Expected result

Either a known constant 255 BTC for a full wrap cycle when SCEV understands it, or failure when flags/structure obscure wrap—**do not assume** the same formulas as `i32` `i<n`. Teaching point: width and wrap change counts.

### Why careful

8-bit modular arithmetic is easy to get wrong in mental models; SCEV’s answer depends on add flags and icmp form.

### Counterexample

`i32` loop `i < 256` with `nsw` — BTC `255` cleanly without modular IV wrap of the counter through its whole domain mid-loop in the same way.

---

## Example 5 (bonus): Predicated count need

### Input

```c
void h(int n) {
  for (int i = 0; i < n; ++i) { }
}
```

with IV updates missing `nsw` in raw IR.

### Expected result

`getPredicatedBackedgeTakenCount` may return a count that is only valid under no-wrap predicates; `getBackedgeTakenCount` exact may be unavailable.

### Why it matters

Vectorizers may add runtime overflow checks to unlock the predicated count.

### Counterexample

Canonical IR with `add nsw` and simplified icmp → exact BTC without extra predicates.

---

## Quick reference

| Loop | BTC (typical) | Trip (entered) |
|------|---------------|----------------|
| `i=0; i<n; ++i`, `n>0` | `n-1` | `n` |
| `i=0; i<64; ++i` | `63` | `64` |
| Early break | Often unknown exact | ≤ `n` |
| Zero `n` | 0 backedges | 0 trips |
| Wrapping i8 tricks | Special-case | Special-case |
