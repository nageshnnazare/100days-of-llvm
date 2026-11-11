# Day 34 Extra: SCEV — Examples

```bash
clang -S -emit-llvm -O1 -Xclang -disable-llvm-passes f.c -o f.ll
opt -passes='print<scalar-evolution>' -disable-output f.ll
```

Parent: [Day-34-0-SCEV.md](Day-34-0-SCEV.md).

---

## Diagram

```text
   %i = phi i32 [ 0, %pre ], [ %i.next, %latch ]
   %i.next = add nsw i32 %i, 1

   SCEV expression for %i:

        {0,+,1}<header>

   start 0, step 1, in this loop

   %p = getelementptr i32, ptr %base, i32 %i
   SCEV of the pointer:
        {%base,+,4}<header>
```

SCEV is a normal form for integer recurrences. An induction variable becomes an add recurrence `{start,+,step}`. A GEP becomes a recurrence in bytes. Two IVs that look different in the IR often become the same SCEV, which is how trip counts, dependence distances, and legality checks compare them. A value with a possible overflow that SCEV cannot treat as `nsw` stays a more pessimistic expression.

## Example 1: Basic integer AddRec

### Input

```c
int sum(int n) {
  int s = 0;
  for (int i = 0; i < n; ++i)
    s += i;
  return s;
}
```

### Commands

`print<scalar-evolution>` as above.

### Expected result

`i` classified with SCEV `{0,+,1}<%for.cond>` (names vary). `s` may be an AddRec of a more complex form or Unknown depending on rewriting.

### Why it fires

PHI + add-by-1 latch pattern matches AddRec construction.

### Counterexample

```c
int sum_mul(int n, int k) {
  int s = 0;
  for (int i = 1; i < n; i *= k) /* multiplicative */
    s += i;
  return s;
}
```

Multiplicative update is not a constant-stride AddRec of the usual affine kind SCEV prefers—expect Unknown or non-affine complexity; not `{0,+,1}`.

---

## Example 2: Pointer AddRec

### Input

```c
void fill(int *p, int n) {
  for (int i = 0; i < n; ++i)
    *p++ = i;
}
```

### Commands

Same printer.

### Expected result

The pointer used for stores evolves as `{%p,+,4}<loop>` (for 4-byte `int`), while `i` is `{0,+,1}`.

### Why it fires

GEP/pointer PHI with constant step.

### Counterexample

```c
void fill_indexed(int *p, int *idx, int n) {
  for (int i = 0; i < n; ++i)
    p[idx[i]] = i;
}
```

Index from a load is Unknown → pointer SCEV not a simple constant-stride AddRec on `p` alone.

---

## Example 3: Loop-invariant SCEV

### Input

```c
int f(int *a, int n, int k) {
  int s = 0;
  for (int i = 0; i < n; ++i)
    s += a[i] * k;
  return s;
}
```

### Commands

Printer + mentally check `k`.

### Expected result

`k` is loop-invariant (Unknown but invariant). `i` is AddRec. Address `a[i]` combines base `a` with AddRec offset.

### Why it matters

LICM uses invariance; SCEV’s disposition answers it for many expressions beyond raw IR Motion.

### Counterexample

```c
s += a[i] * i; /* i not invariant */
```

---

## Example 4: Casts — sext of an IV

### Input

```c
long gather(int *a, int n) {
  long s = 0;
  for (int i = 0; i < n; ++i)
    s += a[i];
  return s;
}
```

### Commands

Printer; look for `sext` expressions wrapping AddRecs.

### Expected result

SCEV shows something like `(sext i32 {0,+,1} to i64)` until indvars widens.

### Why it fires

`getSCEV` builds cast nodes explicitly.

### Counterexample

After `indvars`, the printed form may be a wide AddRec without the cast—showing pass interaction, not a SCEV bug.

---

## Example 5 (bonus): Not floats

### Input

```c
float fsum(float *a, int n) {
  float s = 0;
  for (int i = 0; i < n; ++i)
    s += a[i];
  return s;
}
```

### Expected result

`i` still has an integer AddRec; `s` is **not** a float SCEV AddRec—SCEV leaves fp values outside its algebra (Unknown / not modeled as SCEV float).

### Why

SCEV is an integer/pointer evolution analysis.

### Twin

Integer `s` in Example 1 is in scope (as integer SCEV, possibly complex).

---

## Table

| Pattern | SCEV shape |
|---------|------------|
| `i++` from 0 | `{0,+,1}` |
| `p++` on `int*` | `{p,+,4}` typically |
| `i *= k` | Not simple affine AddRec |
| Loop-invariant `k` | Invariant Unknown/const |
| `sext i32 IV` | SignExtend of AddRec |
| `float` sum | No float AddRec |
