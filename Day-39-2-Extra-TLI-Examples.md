# Day 39 Extra: TLI — Examples

```bash
clang -S -emit-llvm -O1 -Xclang -disable-llvm-passes f.c -o f.ll
opt -mtriple=x86_64-unknown-linux-gnu -passes=instcombine -S f.ll -o out.ll
```

Parent: [Day-39-0-TargetLibraryInfo.md](Day-39-0-TargetLibraryInfo.md).

---

## Diagram

```text
   source:  memcpy(d, s, n);

   IR:      call void @memcpy(ptr %d, ptr %s, i64 %n)
            ; recognized because TargetLibraryInfo
            ; says this libcall is available and
            ; has the standard signature

   if -fno-builtin-memcpy, TLI marks memcpy
   unavailable, and the call stays an opaque
   call: no memcpy->load/store fold, no DSE
   across it from the libcall summary
```

TargetLibraryInfo knows which C library functions this module is allowed to treat as the standard ones. `memcpy` has a known set of read and write locations; an unknown function does not. Compiling with `-fno-builtin` clears those entries, and every pass that special-cases the name has to stop. The pass does not emit the library call; it answers questions about one that is already in the IR.

## Example 1: `strlen` of a constant

### Input

```c
unsigned long strlen(const char *);
unsigned long f(void) { return strlen("abcd"); }
```

### Commands

InstCombine with a Linux triple as above.

### Expected result

Constant return `4` (width per ABI). Call deleted.

### Why it fires

TLI recognizes `strlen`; LibCallSimplifier folds constant string length.

### Counterexample

```c
unsigned long strlen(const char *);
unsigned long f(const char *s) { return strlen(s); }
```

Non-constant argument → call remains (unless other facts exist).

---

## Example 2: `printf` → `puts`

### Input

```c
int printf(const char *, ...);
void f(void) { printf("hello\n"); }
```

### Commands

Same InstCombine + triple.

### Expected result

Call to `puts` with `"hello"` (newline handled by puts), when `puts` is available in TLI.

### Why it fires

Format is a simple string ending in `\n` with no `%` conversions; TLI has puts.

### Counterexample

```c
void g(int x) { printf("hello %d\n", x); }
```

Must not become puts—format has conversions.

---

## Example 3: Wrong prototype → not a LibFunc

### Input IR

```llvm
declare i32 @strlen(ptr)   ; wrong return type on LP64
define i32 @f(ptr %s) {
  %n = call i32 @strlen(ptr %s)
  ret i32 %n
}
```

### Commands

```bash
opt -mtriple=x86_64-unknown-linux-gnu -passes=instcombine -S wrong.ll
```

### Expected result

**No** strlen-based folds that assume the real libc strlen; recognition fails the signature check.

### Why it must not fire

TLI requires ABI-correct prototypes.

### Counterexample / fix

```llvm
declare i64 @strlen(ptr)
```

on LP64 → recognition succeeds.

---

## Example 4: `-fno-builtin` / disabled TLI

### Input

```c
void *memset(void *, int, unsigned long);
void clear(void *p, unsigned long n) { memset(p, 0, n); }
```

### Commands

```bash
clang -S -emit-llvm -O1 -fno-builtin -Xclang -disable-llvm-passes clear.c -o clear.ll
opt -mtriple=x86_64-unknown-linux-gnu -passes='loop-idiom,instcombine' -S clear.ll
```

Also compare without `-fno-builtin`.

### Expected result

With builtins disabled, many memset promotions / folds are blocked. With builtins enabled, InstCombine may rewrite to `llvm.memset`.

### Why

`disableAllFunctions` / no-builtin attributes make TLI report unavailable libfuncs.

### Counterexample

Default clang without `-fno-builtin` → memset recognized.

---

## Example 5 (bonus): Vector lib mapping needs veclib

### Input

```c
double sin(double);
void v(double *restrict a, double *restrict b, int n) {
  for (int i = 0; i < n; ++i)
    a[i] = sin(b[i]);
}
```

### Commands

```bash
# Conceptual — clang drives -fveclib into metadata / TLI setup:
clang -O3 -fveclib=SVML -S -emit-llvm v.c -o v.svml.ll
# vs
clang -O3 -S -emit-llvm v.c -o v.none.ll
```

### Expected result

With a vector library configured, LV may call vector `sin` variants (`VecDesc` mappings). Without veclib, scalar `sin` calls remain inside a vectorized body or block vectorization of that call.

### Why it fires only with veclib

`isFunctionVectorizable("sin", VF)` is false unless VecDesc entries were installed.

### Counterexample

Same loop without `sin` (only arithmetic) vectorizes via ordinary LV without TLI vector maps.

---

## Table

| Case | Needs TLI? | Result |
|------|------------|--------|
| strlen("ab") | Yes | Constant |
| printf("x\n") | Yes + puts | puts |
| printf("%d") | Yes | stays printf |
| wrong strlen type | Fail getLibFunc | no fold |
| -fno-builtin memset | Disabled | fewer folds |
| sin loop + SVML | VecDesc | vector sin |
| sin loop, no veclib | No map | scalar sin |
