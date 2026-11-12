# Day 36 Extra: Dependence / LAA — Examples

```bash
clang -S -emit-llvm -O1 -Xclang -disable-llvm-passes f.c -o f.ll
opt -passes='print<access-info>' -disable-output f.ll
opt -passes='print<da>' -disable-output f.ll
```

---

## Diagram

```text
   for i:
     A[i+1] = A[i] + 1

   dependence from the load A[i]
   to the store A[i+1]:

     direction:  forward  (<)
     distance:   1 iteration

   that is a loop-carried flow dependence.
   Interchange and vectorization must
   respect it.

   A[i] = B[i] + 1
   has no carried dependence on A if B
   does not alias A.
```

Dependence analysis asks whether two memory accesses in a loop nest touch the same location, and if so in which direction and at what distance. A distance of zero is loop-independent. A positive distance is carried by that loop. The vectorizer needs "no carried dependence, or a dependence it can honor with a vector width." The answer is conservative: if the subscripts are not affine, the result is "do not know."

## Example 1: Loop-carried dependence (LAA)

### Input

```c
void carried(int *a, int n) {
  for (int i = 1; i < n; ++i)
    a[i] = a[i - 1] + 1;
}
```

### Commands

`print<access-info>`.

### Expected result

Reports unsafe vectorization due to forward dependence / distance 1 on `a` (wording varies). Runtime checks cannot generally fix true carried flow dependences of distance 1 for arbitrary VF.

### Why it fires

`a[i]` depends on `a[i-1]` from the previous iteration.

### Counterexample

```c
void shift(int *a, int n) {
  for (int i = 0; i < n - 4; ++i)
    a[i] = a[i + 4] + 1; /* distance 4 backward may allow VF≤4 in some cases */
}
```

Larger distances may be vectorizable with smaller VF—LAA encodes distance limits; still a dependence, but not always a hard “never.”

---

## Example 2: Independent with restrict (LAA happy)

### Input

```c
void add(int *restrict a, int *restrict b, int n) {
  for (int i = 0; i < n; ++i)
    a[i] = b[i] + 1;
}
```

### Commands

`print<access-info>`; optionally `opt -passes=loop-vectorize -S`.

### Expected result

Access info allows vectorization; may note no memory runtime checks needed when noalias is solid—or still emit checks depending on version/AA.

### Why it fires

No loop-carried dependence; objects disjoint.

### Counterexample

Drop `restrict` → MayAlias between `a` and `b` → LAA may require runtime checks or refuse; vectorize becomes conditional.

---

## Example 3: Classical DA for interchange nest

### Input

```c
#define N 32
void nest(int A[N][N]) {
  for (int i = 0; i < N; ++i)
    for (int j = 1; j < N; ++j)
      A[i][j] = A[i][j - 1] + 1;
}
```

### Commands

```bash
opt -passes='print<da>' -disable-output nest.ll
opt -passes='loop-simplify,lcssa,loop-interchange' -pass-remarks-missed=loop-interchange -S nest.ll
```

### Expected result

DA shows an inner-level directed dependence. Interchange with the outer loop should be **illegal** / missed (dependence remark)—swapping would break `j` order.

### Why it must not interchange

Direction vector on the `j` level is essential; permutation breaks semantics.

### Counterexample

```c
void nest2(int A[N][N]) {
  for (int i = 0; i < N; ++i)
    for (int j = 0; j < N; ++j)
      A[i][j] = 1; /* no dependence on neighbors */
}
```

DA independence → interchange legality OK (profitability separate).

---

## Example 4: GCD-style independence intuition

### Input

```c
void gcdish(int *a, int n) {
  for (int i = 0; i < n; ++i) {
    a[2 * i] = 1;
    a[2 * i + 1] = 2;
  }
}
```

Even vs odd indices—writes do not conflict with each other across those two stores’ patterns (same iteration). Cross-iteration: `2*i` sequences are independent of each other for different `i` in the sense of distinct locations.

### Commands

`print<da>` / `print<access-info>`.

### Expected result

No harmful conflict between the even/odd stores; vectorization may still see them as strided stores (interleave groups)—LAA discusses strides more than GCD by name.

### Why

Distinct locations; GCD-type reasoning on `2*i` vs `2*i+1` shows never equal.

### Counterexample

```c
a[2*i] = a[2*i - 2] + 1; /* carried on even indices */
```

Dependence along the even subsequence.

---

## Example 5 (bonus): MayAlias forces LAA runtime checks

### Input

```c
void maybe_alias(int *a, int *b, int n) {
  for (int i = 0; i < n; ++i)
    a[i] = b[i] + 1;
}
```

### Commands

`print<access-info>`.

### Expected result

LAA reports that **runtime pointer checking** is needed (or similar), not a static independence proof.

### Why

BasicAA MayAlias on parameters.

### Counterexample

`restrict` on both (Example 2) → checks may disappear.

---

## Table

| Example | Tool | Result |
|---------|------|--------|
| `a[i]=a[i-1]` | LAA | Not freely vectorizable |
| restrict add | LAA | OK |
| `A[i][j]=A[i][j-1]` | DA + interchange | No swap |
| Independent fill nest | DA | Independent |
| maybe alias add | LAA | Runtime checks |
