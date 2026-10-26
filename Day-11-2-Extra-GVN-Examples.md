# Day 11 Extra: GVN Examples

Four complete examples plus counterexamples. Opaque pointers throughout. Prepare IR with:

```bash
clang -S -emit-llvm -O1 -Xclang -disable-llvm-passes FILE.c -o FILE.ll
opt -passes=gvn -S FILE.ll -o FILE.gvn.ll
# optional:
opt -passes=gvn -debug-only=gvn -S FILE.ll
```

---

## Diagram

```text
   %a = add %x, 1
   ...
   %b = add %x, 1          GVN: %b and %a are the same
                           value number -> replace %b with %a

   store 7, ptr %p
   %v = load ptr %p        MemoryDependence says the store
                           is the clobber -> replace %v with 7
```

GVN gives two instructions the same value number when their operands have the same numbers and the opcode matches. Loads are included only when a memory dependence query says the loaded address was written by a known store and not clobbered in between. A load is not "the same" as another load just because the address expression matches.

## Example 1: Scalar CSE of identical adds

### Goal
Collapse two congruent `add` instructions to one leader.

### Input (C)

```c
int sum2(int a, int b) {
  int x = a + b;
  int y = a + b;
  return x ^ y; /* becomes 0 after CSE + simplify, or x^x */
}
```

### Input (IR sketch you can paste into a `.ll`)

```llvm
define i32 @sum2(i32 %a, i32 %b) {
entry:
  %x = add i32 %a, %b
  %y = add i32 %a, %b
  %r = xor i32 %x, %y
  ret i32 %r
}
```

### Commands

```bash
opt -passes=gvn -S sum2.ll -o sum2.gvn.ll
```

### Expected
`%y` eliminated; `%r` becomes `xor i32 %x, %x` (InstCombine later yields `0`).

### Why it fires
Same opcode and operand value numbers; leader `%x` dominates `%y`.

### Counterexample
Different operands (`a+b` vs `a+c`) → distinct numbers → no CSE.

---

## Example 2: Load CSE with no clobber

### Goal
Eliminate a redundant load of the same pointer.

### Input (C)

```c
int twoload(int *p) {
  int a = *p;
  int b = *p;
  return a + b;
}
```

### IR

```llvm
define i32 @twoload(ptr %p) {
entry:
  %a = load i32, ptr %p, align 4
  %b = load i32, ptr %p, align 4
  %r = add i32 %a, %b
  ret i32 %r
}
```

### Commands

```bash
opt -passes=gvn -S twoload.ll -o twoload.gvn.ll
```

### Expected

```llvm
define i32 @twoload(ptr %p) {
entry:
  %a = load i32, ptr %p, align 4
  %r = add i32 %a, %a
  ret i32 %r
}
```

### Why it fires
Memory analysis finds no clobber between loads; congruent load expression.

### Counterexample (must NOT fire)

```llvm
define i32 @twoload_store(ptr %p, ptr %q) {
entry:
  %a = load i32, ptr %p, align 4
  store i32 1, ptr %q, align 4
  %b = load i32, ptr %p, align 4
  %r = add i32 %a, %b
  ret i32 %r
}
```

If `%q` may alias `%p`, both loads remain.

---

## Example 3: Calls do not CSE unless safe

### Goal
Show that ordinary calls stay unique; readnone can CSE.

### Input (IR)

```llvm
declare i32 @pure_add(i32, i32) readnone

define i32 @call_cse(i32 %x) {
entry:
  %a = call i32 @pure_add(i32 %x, i32 1)
  %b = call i32 @pure_add(i32 %x, i32 1)
  ret i32 %a
}

define void @call_nocse(i32 %x) {
entry:
  call void @impure(i32 %x)
  call void @impure(i32 %x)
  ret void
}
```

Fix the accidental space: use `@impure` consistently:

```llvm
declare void @impure(i32)
```

### Commands

```bash
opt -passes=gvn -S calls.ll -o calls.gvn.ll
```

### Expected
In `@call_cse`, the second `pure_add` may be removed / RAUW’d to `%a`. In `@call_nocse`, both `impure` calls remain.

### Why it fires / does not
`lookupOrAddCall` shares numbers only with safe memory effects and equal args.

### Counterexample
Same `readnone` callee but different arguments → no CSE.

---

## Example 4: Load PRE shape (conceptual)

### Goal
Understand when load PRE helps: value available on one predecessor, missing on another.

### Input (IR)

```llvm
define i32 @load_pre(i1 %c, ptr %p) {
entry:
  br i1 %c, label %left, label %merge

left:
  %v = load i32, ptr %p, align 4
  br label %merge

merge:
  %phi = phi i32 [ %v, %left ], [ 0, %entry ]
  %l2 = load i32, ptr %p, align 4
  %r = add i32 %phi, %l2
  ret i32 %r
}
```

### Commands

```bash
opt -passes=gvn -enable-load-pre -S load_pre.ll -o load_pre.gvn.ll
opt -passes=gvn -enable-load-pre=false -S load_pre.ll -o load_pre.off.ll
```

### Expected
With PRE enabled, GVN may insert a load on the `%entry → %merge` edge (often after splitting) and reuse a PHI, removing `%l2` when safe. With PRE disabled, `%l2` more often remains.

Exact IR depends on critical edges and safety checks — compare the two outputs rather than memorizing one golden file.

### Why it fires
Partial availability + `-enable-load-pre` + legal edge insertion.

### Counterexample
Volatile load at `%l2`:

```llvm
  %l2 = load volatile i32, ptr %p, align 4
```

PRE/CSE must not remove it.

---

## Quick checklist

| Case | Fires? |
| --- | --- |
| Dominating congruent `add` | Yes |
| Two loads, no clobber | Yes |
| Load, may-alias store, load | No |
| Two `impure` calls | No |
| Two `readnone` equal calls | Yes |
| Volatile second load | No |
