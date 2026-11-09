# Day 32 Extra: TBAA — Examples

## Diagram

```text
        omnipotent char
         /            \
       int           float

   store i32  via !tbaa int
   store float via !tbaa float

   the two tags meet only at omnipotent char,
   which does not make them alias each other.
   BasicAA, seeing only two pointers, says MayAlias.
   TBAA says NoAlias.
```

The examples below are that picture in IR. A `char` access is tagged at the root and aliases both children. That is the case the last example is there to show.

```bash
clang -S -emit-llvm -O1 -Xclang -disable-llvm-passes f.c -o f.ll
opt -passes='aa-eval' -aa-pipeline='basic-aa,tbaa' -print-all-alias-modref-info -disable-output f.ll
```

---

## Example 1: `int*` vs `float*` — TBAA wins where BasicAA cannot

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

BasicAA only: MayAlias. With TBAA: NoAlias between the two stores’ locations.

### Why it fires

Different scalar TBAA types under strict aliasing; neither is char.

### Counterexample

```bash
clang ... -fno-strict-aliasing mix.c -o mix.nos.ll
```

Without tags (or without tbaa in the pipeline), remains MayAlias.

---

## Example 2: `char*` aliases `int*`

### Input

```c
void touch(int *i, char *c) {
  *i = 1;
  *c = 2;
}
```

### Commands

`aa-eval` with `basic-aa,tbaa`.

### Expected result

TBAA **must not** prove NoAlias (omnipotent char). MayAlias remains.

### Why it must not fire as NoAlias

C allows character pointers to address any object; the TBAA root encodes that.

### Counterexample / twin

Replace `char*` with `float*` (Example 1) → NoAlias allowed.

---

## Example 3: Struct field path TBAA

### Input

```c
struct S { int a; float b; };
void fields(struct S *s) {
  s->a = 1;
  s->b = 2.0f;
}
```

### Commands

Inspect `!tbaa` on both stores; run `aa-eval`.

### Expected result

NoAlias between `a` and `b` accesses via struct-path offsets (typical under clang TBAA).

### Why it fires

Different field offsets / access types in the tag.

### Counterexample

```c
void via_char(struct S *s) {
  char *p = (char *)s;
  p[0] = 1;
  s->b = 2.0f;
}
```

Char involvement → cannot rely on NoAlias between `p[0]` and `s->b` through TBAA alone the same way.

---

## Example 4: Same type, two pointers — TBAA silent

### Input

```c
void two_ints(int *a, int *b) {
  *a = 1;
  *b = 2;
}
```

### Commands

`basic-aa,tbaa` pipeline.

### Expected result

Still **MayAlias** (TBAA types match; they may be the same object). Need `restrict` for NoAlias.

### Why TBAA does not help

TBAA answers type compatibility, not pointer inequality.

### Counterexample

```c
void two_restrict(int *restrict a, int *restrict b) {
  *a = 1; *b = 2;
}
```

BasicAA/noalias proves NoAlias; TBAA optional.

---

## Example 5 (bonus): `!tbaa.struct` on memcpy

### Input

```c
struct S { int a; float b; };
void copy(struct S *d, struct S *s) {
  *d = *s; /* may become memcpy with !tbaa.struct */
}
```

### Commands

```bash
clang -S -emit-llvm -O1 -Xclang -disable-llvm-passes copy.c -o copy.ll
grep -E 'tbaa\.struct|llvm\.memcpy' copy.ll
```

### Expected result

Either field-wise copies with `!tbaa`, or `llvm.memcpy` carrying `!tbaa.struct` metadata describing members.

### Why it matters

Lets AA treat the copy as typed field transfers rather than a char blob when metadata is present.

### Counterexample

`memcpy(d, s, n)` with runtime `n` and no tbaa.struct → conservative AA.

---

## Table

| Case | TBAA NoAlias? |
|------|----------------|
| int* vs float* | Yes |
| int* vs char* | No |
| struct fields a vs b | Yes (path) |
| int* vs int* | No (need restrict) |
| no metadata | No |
