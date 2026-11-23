# Day 44 Extra: Dead Argument Elimination Examples

Companion to [Day-44-0-DeadArgElim.md](Day-44-0-DeadArgElim.md). Deep dive: [Day-44-1-Extra-DeadArgElim-DeepDive.md](Day-44-1-Extra-DeadArgElim-DeepDive.md).

```bash
opt -passes=deadargelim in.ll -S -o out.ll
opt -passes=deadargelim -debug-only=deadargelim in.ll -S
```

---

## Diagram

```text
   define i32 @f(i32 %a, i32 %b) {
     ret i32 %a          ; %b has no use
   }
   call i32 @f(i32 %x, i32 %y)

   after deadargelim, if @f is internal:

   define i32 @f(i32 %a) {
     ret i32 %a
   }
   call i32 @f(i32 %x)
```

DeadArgElim removes an argument that the function never reads, but only when every call site can be updated. An external function keeps its ABI: the argument stays even if the body ignores it. A return value that no caller reads can be removed the same way, for internal linkage.

## Example 1: Plain unused argument

**Input:**

```llvm
define internal i32 @f(i32 %dead, i32 %x) {
  %y = shl i32 %x, 1
  ret i32 %y
}
define i32 @entry(i32 %v) {
  %r = call i32 @f(i32 0, i32 %v)
  ret i32 %r
}
```

**Commands:** `opt -passes=deadargelim e1.ll -S`

**Expected result:** `@f(i32 %x)` only; call becomes `call i32 @f(i32 %v)`.

**Why:** `%dead` has no uses; linkage internal.

**Counterexample:** `define i32 @f(...)` external — signature unchanged.

---

## Example 2: Dead constant argument

**Input:**

```llvm
define internal i32 @addk(i32 %k, i32 %x) {
  %y = add i32 %x, %k
  ret i32 %y
}
define i32 @a(i32 %x) { %r = call i32 @addk(i32 4, i32 %x)  ret i32 %r }
define i32 @b(i32 %x) { %r = call i32 @addk(i32 4, i32 %x)  ret i32 %r }
```

**Commands:** `opt -passes=deadargelim e2.ll -S`

**Expected result:** `%k` removed; body uses `add i32 %x, 4`; both callers pass only `%x`.

**Why:** Same constant at all sites.

**Counterexample:** Change `@b` to pass `5` — `%k` must remain.

---

## Example 3: Dead return → void

**Input:**

```llvm
define internal i32 @work(i32 %x) {
  call void @side(i32 %x)
  ret i32 %x
}
declare void @side(i32)
define void @caller(i32 %x) {
  call i32 @work(i32 %x)
  ret void
}
```

**Commands:** `opt -passes=deadargelim e3.ll -S`

**Expected result:** `@work` becomes `void`; `ret void`; call is `call void @work(i32 %x)`.

**Why:** Return value unused at all sites.

**Counterexample:** `%v = call i32 @work(...)` used in an `add` — return stays.

---

## Example 4: Varargs blocks DAE

**Input:**

```llvm
define internal void @log(i32 %tag, ...) {
  ret void
}
define void @c() {
  call void (i32, ...) @log(i32 1, i32 2)
  ret void
}
```

**Commands:** `opt -passes=deadargelim e4.ll -S`

**Expected result:** No signature change (pass skips varargs).

**Why:** Reshaping varargs ABI is unsafe/complex.

**Counterexample:** Remove `...` and unused `%tag` — then DAE can remove `%tag`.

---

## Example 5: musttail blocks

**Input:**

```llvm
define internal i32 @fwd(i32 %dead, i32 %x) {
  %r = musttail call i32 @impl(i32 %dead, i32 %x)
  ret i32 %r
}
define i32 @impl(i32 %a, i32 %b) {
  ret i32 %b
}
```

**Commands:** `opt -passes=deadargelim e5.ll -S`

**Expected result:** Often **no** removal of `%dead` from `@fwd` because musttail requires matching prototypes with the callee convention.

**Why:** musttail correctness constraints.

**Counterexample:** Replace `musttail call` with ordinary `call` and use `%x` only in `@fwd` without forwarding dead — DAE can remove.

---

## Example 6: IPSCCP then DAE

**Input (C):**

```c
static int f(int mode, int x) {
  if (mode == 0) return x;
  return x + 1;
}
int g(int x) { return f(0, x); }
```

**Commands:**

```bash
clang -S -emit-llvm -O1 -Xclang -disable-llvm-passes e6.c -o e6.ll
opt -passes='ipsccp,deadargelim' e6.ll -S -o e6.out.ll
```

**Expected result:** `mode` folded/removed; `f` simplified; call sites updated.

**Why:** Classic IPO handoff.

**Counterexample:** `opt -passes=deadargelim` alone may still remove `mode` if IPSCCP isn’t needed when uses disappear another way—but with a live `icmp` on `mode`, DAE alone might keep it unless the dead-constant rule applies (`mode` always 0).
