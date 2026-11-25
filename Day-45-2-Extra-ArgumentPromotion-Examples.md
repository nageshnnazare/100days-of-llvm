# Day 45 Extra: Argument Promotion Examples

Companion to [Day-45-0-ArgumentPromotion.md](Day-45-0-ArgumentPromotion.md). Deep dive: [Day-45-1-Extra-ArgumentPromotion-DeepDive.md](Day-45-1-Extra-ArgumentPromotion-DeepDive.md).

```bash
clang -S -emit-llvm -O1 -Xclang -disable-llvm-passes f.c -o f.ll
opt -passes=argpromotion f.ll -S -o f.out.ll
opt -passes=argpromotion -debug-only=argpromotion f.ll -S
```

---

## Diagram

```text
   define internal void @f(ptr %p) {
     %v = load i32, ptr %p
     ...
   }
   ; the only uses of %p are loads of the i32

   after argument promotion:

   define internal void @f(i32 %p.val) {
     ; the load is in the caller now
     ...
   }
   %v = load i32, ptr %obj
   call void @f(i32 %v)
```

Argument promotion rewrites a pointer argument into the scalar value it points at, when the function only loads through it and the pointed-to memory does not change between the call and the loads. The load moves to the caller. The function has to be internal so every call site can be rewritten. If the pointer is stored or passed onward, it stays a pointer.

## Example 1: Promote two struct fields

**Input:**

```c
typedef struct { int a; int b; } AB;
static int sum(AB *p) { return p->a + p->b; }
int ex(void) {
  AB s = {3, 4};
  return sum(&s);
}
```

**Commands:** emit IR, then `opt -passes=argpromotion`.

**Expected result:** `@sum(i32, i32)`; caller passes `3` and `4` (or loads from `%s`).

**Why:** Internal, loads only, alloca non-escaping.

**Counterexample:** `sum` stores `p->a = 0` — no promotion.

---

## Example 2: Capture via escape

**Input:**

```c
static void sink(int *q);
static int read_x(int *p) {
  sink(p);          /* may capture */
  return *p;
}
int ex(int v) {
  int x = v;
  return read_x(&x);
}
```

**Commands:** `opt -passes=argpromotion -debug-only=argpromotion`

**Expected result:** No promotion of `%p` (capture through `@sink`).

**Why:** Callee may retain the pointer; caller-side load alone is insufficient / unsafe as a replacement for callee semantics.

**Counterexample:** Remove `sink(p)` and only `return *p` — promotion can proceed.

---

## Example 3: Aliasing store through other arg

**Input (IR):**

```llvm
define internal i32 @f(ptr %p, ptr %q) {
  store i32 7, ptr %q, align 4
  %v = load i32, ptr %p, align 4
  ret i32 %v
}
define i32 @ex() {
  %a = alloca i32, align 4
  store i32 1, ptr %a
  %r = call i32 @f(ptr %a, ptr %a)
  ret i32 %r
}
```

**Commands:** `opt -passes=argpromotion e3.ll -S`

**Expected result:** Do **not** promote `%p` to a caller load of `1` (would disagree with callee loading `7`).

**Why:** `%p` and `%q` alias; store changes the value before load.

**Counterexample:** Use two distinct non-aliasing allocas and never store through a may-alias arg — promotion of read-only `%p` becomes plausible.

---

## Example 4: External linkage — refuse

**Input:**

```c
int sum(const int *p) { return p[0] + p[1]; }
```

**Commands:** `opt -passes=argpromotion`

**Expected result:** No change to signature.

**Why:** Other TUs may call `sum(int *)`.

**Counterexample:** `static int sum(...)` — eligible.

---

## Example 5: byval is different

**Input (IR sketch):**

```llvm
define internal i32 @g(ptr byval(%struct.S) align 8 %p) {
  %a = load i32, ptr %p
  ret i32 %a
}
```

**Commands:** inspect before/after `argpromotion`.

**Expected result:** Pass may still do something in some cases, but **do not equate** “has byval” with “already promoted.” `byval` is an ABI copy; promotion’s goal is scalar SSA formals without relying on the callee loading from a hidden copy—or cleaning patterns differently.

**Why:** Teaching distinction.

**Counterexample:** Hand-written scalar formals `i32 %a` — already the end state of promotion; pass is a no-op.

---

## Example 6: Only one field loaded

**Input:**

```c
typedef struct { int a; int b; int c; } T;
static int geta(T *p) { return p->a; }
int ex(T *q) { return geta(q); } /* careful: if ex is exported and passes external ptr */
```

Better test: keep `geta` static and call from an internal-only path with a local `T`.

**Commands:** `opt -passes=argpromotion`

**Expected result:** Promote only `a` (`i32`), not entire struct—when safe.

**Why:** Load set is a single offset.

**Counterexample:** Also pass `p` to an external opaque function—capture blocks.
