# Day 6 Extra: SROA Examples

```bash
clang -S -emit-llvm -O1 -Xclang -disable-llvm-passes ex.c -o ex.ll
opt -passes=mem2reg -S ex.ll -o ex.m2r.ll
opt -passes=sroa -S ex.ll -o ex.sroa.ll
```

Keep the mem2reg output. The educational bit is the difference.

---

## Diagram

```text
   alloca { i32, i32 }          SROA slices
   +--------+--------+          +--------+  +--------+
   | field0 | field1 |   --->   | alloca |  | alloca |
   | 4 bytes| 4 bytes|          | i32    |  | i32    |
   +--------+--------+          +--------+  +--------+
        ^        ^                   |           |
   gep 0,0   gep 0,1            mem2reg      mem2reg
                                     |           |
                                     v           v
                                   SSA %a0     SSA %a1
```

SROA does not promote a struct in one step. It partitions the alloca by the offsets that loads, stores, and GEPs actually touch, rewrites those uses onto the new slices, then calls the same promoter as mem2reg. A slice whose address escapes stays memory.

## Example 1: Two fields

```c
typedef struct { int x; int y; } Point;
int sum(int a, int b) {
  Point p;
  p.x = a;
  p.y = b;
  return p.x + p.y;
}
```

**After mem2reg:** `alloca` remains, GEPs remain.

**After SROA:** no `alloca`. The return is `add i32 %a, %b`.

**Why:** slices `[0, 4)` and `[4, 8)`, no overlap, no escape.

**Counterexample:**

```c
extern void mutate(Point *p);
int blocked(int a, int b) {
  Point p;
  p.x = a;
  p.y = b;
  mutate(&p);
  return p.x + p.y;
}
```

SROA must leave the alloca. `mutate` can swap the fields. Check that `ex.sroa.ll` still has a `call` and an `alloca`.

---

## Example 2: A small fixed array

```c
int sum4(int a) {
  int v[4];
  v[0] = a;
  v[1] = a + 1;
  v[2] = a + 2;
  v[3] = a + 3;
  return v[0] + v[1] + v[2] + v[3];
}
```

**Expected:** four scalar adds, no alloca. SROA treats each constant index as its own slice.

**Counterexample:**

```c
int sum_i(int a, int i) {
  int v[4];
  v[0] = a; v[1] = a; v[2] = a; v[3] = a;
  return v[i];
}
```

The load's index is not constant. The alloca escapes the analysis. You may still see the four stores. You will also see a load that SROA did not delete.

---

## Example 3: memcpy of the whole object

```c
typedef struct { int x; int y; } Point;
int copy(Point *src) {
  Point p;
  p = *src;
  return p.x;
}
```

Clang lowers `p = *src` to a `memcpy` or a pair of loads, depending on version and flags. Look at `ex.ll`.

- If you see a `llvm.memcpy` of 8 bytes into the alloca, SROA can split it and keep the load of `x`. The `y` half may be deleted as a dead store once it is a separate slice with no load.
- If `src` might alias `p`, that cannot happen: `p` is a fresh alloca, so it does not alias `src`. SROA relies on that IR rule.

**Counterexample:** a variable-length copy

```c
void partial(char *dst, char *src, int n) {
  for (int i = 0; i < n; ++i)
    dst[i] = src[i];
}
```

There is no constant-size alloca partition here at all. This example belongs to LoopIdiom (Day 27), which may turn it into `llvm.memcpy`. Do not expect SROA to do that.

---

## Example 4: Integer that covers two fields

```llvm
define i32 @wide(i64 %bits) {
entry:
  %p = alloca { i32, i32 }, align 8
  store i64 %bits, ptr %p, align 8
  %xptr = getelementptr inbounds { i32, i32 }, ptr %p, i32 0, i32 0
  %x = load i32, ptr %xptr, align 8
  ret i32 %x
}
```

```bash
opt -passes=sroa -S wide.ll -o wide.sroa.ll
```

**Expected:** the `i64` is shifted or truncated down to the low 32 bits on a little-endian target (`trunc i64 %bits to i32`), and the alloca is gone. On a big-endian data layout the high half is the first field; pass `-data-layout` accordingly if you care. Default `opt` on x86 and Apple Silicon is little-endian.

**Why:** the store's slice covers both partitions, so SROA splits the integer.

**Counterexample:** mark the store `volatile`. SROA must not split it. The alloca remains.

---

## Example 5: Phi of field addresses

```llvm
define i32 @pick(i1 %c, i32 %a, i32 %b) {
entry:
  %p = alloca { i32, i32 }, align 4
  %px = getelementptr inbounds { i32, i32 }, ptr %p, i32 0, i32 0
  %py = getelementptr inbounds { i32, i32 }, ptr %p, i32 0, i32 1
  store i32 %a, ptr %px
  store i32 %b, ptr %py
  br i1 %c, label %L, label %R
L:
  br label %J
R:
  br label %J
J:
  %sel = phi ptr [ %px, %L ], [ %py, %R ]
  %v = load i32, ptr %sel
  ret i32 %v
}
```

**Expected:** a phi of the *values* `%a` and `%b`, no alloca. SROA speculated the pointer phi.

**Counterexample:** change one incoming pointer to a function argument `ptr %other`. The phi can now point outside the alloca. Speculation is illegal. SROA leaves the memory alone (or leaves at least the escaping portion).

---

## Example 6: After inlining, SROA wakes up

```c
typedef struct { int x; int y; } Point;
static int getx(Point *p) { return p->x; }
int caller(int a) {
  Point p;
  p.x = a;
  p.y = 3;
  return getx(&p);
}
```

```bash
opt -passes=sroa -S call.c.ll          # likely still an alloca, because getx captures it
opt -passes=inline,sroa -S call.c.ll   # getx is internal; inliner then SROA
```

Use the Clang command from the top to make `call.c.ll`. `getx` is `static`, so it is `internal` and the inliner is allowed to inline it even without a cost model saying "always".

**Expected after `inline,sroa`:** `ret i32 %a`.

**Counterexample:** remove `static`. The inliner may refuse because the function is externally visible and the call is not profitable enough, or it may inline a copy while keeping the out-of-line body. If the alloca remains, check whether the call is still there. This example is the pipeline story: SROA's power depends on inlining (Day 42), not on a cleverer partitioner.

---

## A diff habit

When `both.sroa.ll` is not what you expected, classify the alloca's uses into: constant-offset load/store, variable GEP, call, phi. The first category is SROA's. Any of the others, unless the phi is speculatable, explains a refusal. Write the category down before you read more of `SROA.cpp`. It is faster than `-debug-only`.
