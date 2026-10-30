# Day 15 Extra: Tail Call Elimination Examples

```bash
clang -S -emit-llvm -O1 -Xclang -disable-llvm-passes FILE.c -o FILE.ll
opt -passes=tailcallelim -S FILE.ll -o FILE.tce.ll
opt -passes=tailcallelim -debug-only=tailcallelim -S FILE.ll
```

---

## Diagram

```text
   before                              after tailcallelim

   define i32 @fact(i32 %n)            define i32 @fact(i32 %n, i32 %acc)
     ...                                  loop:
     %r = call @fact(%n-1)                  %r = ... compute in %acc ...
     %m = mul %r, %n                        br label %loop
     ret %m                               ; the recursive call is gone
                                            ; the mul moved into the accumulator
```

A call in tail position is one whose result is returned with nothing to do after it. The recursive factorial is not in tail position until the multiply is turned into an extra argument. After that, the call is a branch back to the top, and no stack frame accumulates.

## Example 1: Classic accumulator TRE

### Goal
Turn self-recursive tail calls into a loop with PHIs.

### Input (C)

```c
int fact(int n, int acc) {
  if (n <= 1)
    return acc;
  return fact(n - 1, acc * n);
}
```

### Input (IR sketch)

```llvm
define i32 @fact(i32 %n, i32 %acc) {
entry:
  %cmp = icmp sle i32 %n, 1
  br i1 %cmp, label %ret, label %rec
ret:
  ret i32 %acc
rec:
  %n1 = add i32 %n, -1
  %acc1 = mul i32 %acc, %n
  %r = call i32 @fact(i32 %n1, i32 %acc1)
  ret i32 %r
}
```

### Commands

```bash
opt -passes=tailcallelim -S fact.ll -o fact.tce.ll
```

### Expected
Recursive call gone; PHIs for `%n`/`%acc` in a loop back to entry (or dedicated header); single function activation stack-wise.

### Why it fires
Tail position, same callee, static frame, forwarded return.

### Counterexample
Do work after the call (`ret i32 %r` becomes `add` then ret) → not tail → no TRE.

---

## Example 2: Invented accumulator for `n + f(n-1)`

### Goal
TRE with associative combine on the return path.

### Input (C)

```c
int sum(int n) {
  if (n <= 0)
    return 0;
  return n + sum(n - 1);
}
```

### Commands

```bash
clang -S -emit-llvm -O1 -Xclang -disable-llvm-passes sum.c -o sum.ll
opt -passes=tailcallelim -S sum.ll -o sum.tce.ll
```

### Expected
If `canTransformAccumulatorRecursion` accepts the `add`, IR gains an accumulator PHI (identity 0) updated by `add` each iteration; recursion removed.

### Why it fires
Unary/associative accumulator recurrence pattern on the returned call.

### Counterexample
`return n - sum(n-1)` (non-associative / unsuitable as implemented) → may fail accumulator TRE.

---

## Example 3: Dynamic alloca blocks TRE

### Goal
Show `canTRE` failing.

### Input (C)

```c
int rec(int n, int m) {
  int flex[m];           /* VLA → dynamic alloca in IR */
  flex[0] = n;
  if (n <= 0)
    return flex[0];
  return rec(n - 1, m);
}
```

### Commands

```bash
clang -S -emit-llvm -O1 -Xclang -disable-llvm-passes vla_rec.c -o vla_rec.ll
# confirm alloca is dynamic in IR, then:
opt -passes=tailcallelim -S vla_rec.ll -o vla_rec.tce.ll
```

### Expected
TRE does not loopify (dynamic alloca). Marking `tail` may also be restricted depending on escapes of `flex`.

### Why it fires / blocked
`canTRE` requires all allocas static.

### Counterexample
Fixed-size `int flex[8]` static alloca — TRE may proceed if no escape blocks the candidate.

---

## Example 4: Mark `tail` on a non-recursive call

### Goal
See `tail` attribute without TRE.

### Input (C)

```c
int g(int);
int f(int x) {
  return g(x + 1);
}
```

### Input (IR)

```llvm
declare i32 @g(i32)
define i32 @f(i32 %x) {
entry:
  %a = add i32 %x, 1
  %r = call i32 @g(i32 %a)
  ret i32 %r
}
```

### Commands

```bash
opt -passes=tailcallelim -S mark.ll -o mark.tce.ll
```

### Expected
`call i32 @g` becomes `tail call i32 @g` when escape analysis allows (no bad alloca captures). No loop created (`@g` ≠ `@f`).

### Why it fires
`markTails` success; TRE candidate requires same function.

### Counterexample
Pass address of local to `@g` → escape → may refuse `tail`.

---

## Example 5 (bonus): `musttail` vs `tail`

### Goal
Distinguish IR features.

### Input (IR)

```llvm
declare i32 @h(i32)
define i32 @wrap(i32 %x) {
entry:
  %r = musttail call i32 @h(i32 %x)
  ret i32 %r
}
```

### Commands

```bash
opt -passes=tailcallelim -S must.ll -o must.tce.ll
opt -passes=verify -disable-output must.ll
```

### Expected
`musttail` already mandates a tail call with matching prototypes; `tailcallelim` is not what introduces `musttail`. Verifier enforces `musttail` rules. Do not expect TRE (different callee).

### Why it matters
Sibling-call codegen + verifier contract vs optional mid-end `tail` mark vs TRE loopify.

### Counterexample
Broken `musttail` with intervening instructions → verifier fails; TRE will not “fix” that.

---

## Hands-on checklist

1. Acc-passing factorial → loop.
2. `n + sum(n-1)` accumulator.
3. VLA blocks TRE.
4. Non-recursive `tail` mark.
5. Read `-debug-only=tailcallelim` remarks on a success and a failure.
