# Day 31 Extra: BasicAA — Examples

```bash
clang -S -emit-llvm -O1 -Xclang -disable-llvm-passes f.c -o f.ll
opt -passes='aa-eval' -aa-pipeline='basic-aa' -print-all-alias-modref-info -disable-output f.ll
```

Parent: [Day-31-0-BasicAA.md](Day-31-0-BasicAA.md).

---

## Diagram

```text
   %p = alloca i32
   %q = alloca i32
   store 1, ptr %p
   %v = load ptr %q          BasicAA: different allocas
                             never alias -> load is not
                             clobbered by the store

   %g = getelementptr i32, ptr %base, i64 %i
   %h = getelementptr i32, ptr %base, i64 %j
   if %i and %j are not known equal,
   BasicAA says MayAlias
```

BasicAA answers "must", "no", or "may" from the IR shape: distinct allocas do not alias, a GEP into an object does not alias a different object, and two GEPs of the same base alias unless the offsets are provably different. It does not look at types. TBAA is a different analysis that adds type-based facts on top.

## Example 1: Distinct allocas

### Input

```c
int f(void) {
  int a = 0, b = 1;
  return a + b;
}
```

### Commands

`aa-eval` with `basic-aa` as above on the generated IR (look at stores/loads to `a` and `b`).

### Expected result

Locations of `a` and `b` are **NoAlias**.

### Why it fires

Distinct alloca underlying objects.

### Counterexample

```c
int g(int *p) {
  int a = 0;
  *p = 1;
  return a;
}
```

`&a` vs `p` is NoAlias **only if** `a` does not escape and `p` cannot be `&a`. If you take `escape(&a)` first, capture tracking may force MayAlias with unknown pointers.

---

## Example 2: `restrict` / noalias parameters

### Input

```c
void add(int *restrict a, int *restrict b) {
  *a += *b;
}
```

### Commands

Inspect IR for `noalias` on arguments; run `aa-eval`.

### Expected result

Load from `b` NoAlias with store to `a` (as locations).

### Why it fires

`noalias` argument attribute / restrict contract.

### Counterexample

```c
void add2(int *a, int *b) {
  *a += *b;
}
```

Without restrict → **MayAlias**; LICM/DSE must be conservative (store may clobber `*b` if `a==b`).

---

## Example 3: GEP constant offset disjointness

### Input IR

```llvm
define i32 @h(ptr %p) {
  %p1 = getelementptr inbounds i32, ptr %p, i64 1
  store i32 1, ptr %p
  %v = load i32, ptr %p1
  ret i32 %v
}
```

### Commands

```bash
opt -passes='aa-eval' -aa-pipeline='basic-aa' -print-all-alias-modref-info -disable-output h.ll
```

### Expected result

Store to `%p` (size 4) **NoAlias** load from `%p1` (offset +4).

### Why it fires

DecomposedGEP constant offsets + sizes prove disjoint ranges.

### Counterexample

```llvm
  store i64 1, ptr %p          ; 8-byte store
  %v = load i32, ptr %p1       ; overlaps bytes 4–7
```

MayAlias or PartialAlias—must not claim NoAlias.

---

## Example 4: TBAA-needed case (BasicAA alone fails)

### Input

```c
void mix(int *i, float *f) {
  *i = 1;
  *f = 2.0f;
}
```

### Commands

```bash
opt -passes='aa-eval' -aa-pipeline='basic-aa' -print-all-alias-modref-info -disable-output mix.ll
opt -passes='aa-eval' -aa-pipeline='basic-aa,tbaa' -print-all-alias-modref-info -disable-output mix.ll
```

### Expected result

`basic-aa` alone: **MayAlias** between the int and float locations (same as two unknown pointers if they might be the same address via casting at the call site). With TBAA metadata from clang under strict aliasing, `basic-aa,tbaa` can prove **NoAlias**.

### Why BasicAA must not claim NoAlias alone

Pointers could be the same bit pattern; only the language type rules (encoded as `!tbaa`) justify disjointness.

### Counterexample to TBAA

Compile with `-fno-strict-aliasing` — missing/ignored TBAA → back to MayAlias.

---

## Example 5 (bonus): PartialAlias

### Input IR

```llvm
define void @partial(ptr %p) {
  store i32 0, ptr %p
  %q = getelementptr i8, ptr %p, i64 2
  store i16 1, ptr %q
  ret void
}
```

### Expected result

AA may report **PartialAlias** (or MayAlias depending on path)—ranges overlap but bases/offsets differ. Must not be treated as NoAlias by DSE.

### Why

Access sizes overlap in the middle of the object.

### Counterexample

`getelementptr i8, ptr %p, i64 4` with i32 store at `%p` → NoAlias with i16 store at +4.

---

## Table

| Case | basic-aa |
|------|----------|
| Two allocas | NoAlias |
| Two plain ptr args | MayAlias |
| restrict args | NoAlias |
| `p` vs `p+1` i32 | NoAlias |
| Overlapping widths | Partial/May |
| int* vs float* w/o TBAA | MayAlias |
