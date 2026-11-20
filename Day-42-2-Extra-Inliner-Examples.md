# Day 42 Extra: Inliner Examples

Companion to [Day-42-0-Inliner.md](Day-42-0-Inliner.md). Deep dive: [Day-42-1-Extra-Inliner-DeepDive.md](Day-42-1-Extra-Inliner-DeepDive.md).

```bash
clang -S -emit-llvm -O1 -Xclang -disable-llvm-passes f.c -o f.ll
opt -passes='cgscc(inline)' f.ll -S -o f.out.ll
opt -passes='cgscc(inline)' -pass-remarks=inline -pass-remarks-missed=inline f.ll -S
```

---

## Diagram

```text
   define i32 @caller() {
     %r = call i32 @callee(i32 4)
     ret i32 %r
   }
   define i32 @callee(i32 %x) {
     %y = shl i32 %x, 1
     ret i32 %y
   }

   after inline, @callee's body is
   copied and the argument is substituted:

   define i32 @caller() {
     %y = shl i32 4, 1
     ret i32 %y
   }
   ; InstCombine then folds shl 4, 1 -> 8
```

Inlining replaces a call with a copy of the callee, substituting arguments for parameters. The point is not the copy itself but what later passes can do with the constants that are now local. The cost model refuses when the callee is large, has `noinline`, or is in a different module that this compilation cannot see. ThinLTO's job is to make the profitable cross-module cases visible.

## Example 1: Constant arguments make a “large” callee cheap

**Input:**

```c
static int classify(int x) {
  if (x < 0) return -1;
  if (x == 0) return 0;
  if (x < 10) return 1;
  if (x < 100) return 2;
  return 3;
}
int only_zero(void) { return classify(0); }
```

**Commands:**

```bash
clang -S -emit-llvm -O1 -Xclang -disable-llvm-passes c0.c -o c0.ll
opt -passes='cgscc(inline)' c0.ll -S -o c0.out.ll
```

**Expected result:** `classify` is inlined into `only_zero`. Follow-on simplification collapses to `ret i32 0` (possibly after instcombine in a fuller pipeline).

**Why:** Cost model assigns a large bonus when `%x` is constant; most branches are dead.

**Counterexample:** Call `classify(y)` with unknown `y` in a hot function and lower `-inline-threshold` until remarks say cost too high—same callee, different decision.

---

## Example 2: `alwaysinline` vs `inlinehint` vs `noinline`

**Input:**

```c
__attribute__((alwaysinline)) static int a(int x) { return x + 1; }
__attribute__((noinline)) static int n(int x) { return x + 1; }
__attribute__((gnu_inline)) static int h(int x) { return x + 1; } /* may map to inlinehint depending on dialect */

int use(int x) { return a(x) + n(x) + h(x); }
```

Inspect IR attributes after Clang emit; adjust if your Clang encodes hints differently:

```llvm
; Goal illustration
define internal i32 @a(i32 %x) alwaysinline { ... }
define internal i32 @n(i32 %x) noinline { ... }
define internal i32 @h(i32 %x) inlinehint { ... }
```

**Commands:**

```bash
opt -passes='cgscc(inline)' -inline-threshold=0 attrs.ll -S -o attrs.out.ll
```

**Expected result:** With threshold 0, `@a` still inlines; `@n` never; `@h` may or may not (hint alone may be insufficient at threshold 0).

**Why:** Attributes override or bias the numeric threshold; `alwaysinline` is mandatory-if-legal, `noinline` is absolute veto, `inlinehint` is soft.

**Counterexample:** Put `noinline` on a function that also has `alwaysinline`—verifier/Clang normally prevent the combo; if IR is hand-written illegally, behavior is “don’t do that.” Prefer testing attributes one at a time.

---

## Example 3: Recursion not inlined by default

**Input:**

```c
static int fact(int n) {
  if (n <= 1) return 1;
  return n * fact(n - 1);
}
int go(int n) { return fact(n); }
```

**Commands:**

```bash
clang -S -emit-llvm -O1 -Xclang -disable-llvm-passes fact.c -o fact.ll
opt -passes='cgscc(inline)' -pass-remarks-missed=inline fact.ll -S -o fact.out.ll
```

**Expected result:** `go` may inline `fact` **once** (the top call), but the recursive call inside `fact` is not expanded unboundedly; remarks often cite recursion. The recursive call remains.

**Why:** Default policy avoids recursive inline explosion.

**Counterexample:** `__attribute__((alwaysinline))` on a tiny recursive function can force attempts; still may be limited—inspect remarks rather than assuming infinite unroll-by-inline.

---

## Example 4: Dynamic alloca penalty

**Input:**

```c
static int sum_dyn(int n) {
  int buf[n]; /* VLA → dynamic alloca in IR */
  int s = 0;
  for (int i = 0; i < n; i++) { buf[i] = i; s += buf[i]; }
  return s;
}
int caller(int n) { return sum_dyn(n); }
```

**Commands:**

```bash
clang -S -emit-llvm -O1 -Xclang -disable-llvm-passes vla.c -o vla.ll
opt -passes='cgscc(inline)' -pass-remarks-missed=inline -inline-threshold=50 vla.ll -S
```

**Expected result:** Inlining often **missed** due to dynamic alloca cost/penalty even when the loop looks small.

**Why:** Inlining VLAs pulls stack-adjustment complexity into every caller; cost model charges heavily.

**Counterexample:** Change to `int buf[8]` fixed size—static alloca, much more likely to inline at the same threshold.

---

## Example 5: Remarks show the decision

**Input:** any of the above.

**Commands:**

```bash
clang -O2 -Rpass=inline -Rpass-missed=inline -c file.c 2>remarks.txt
# or
opt -passes='cgscc(inline)' -print-inline-cost file.ll -S 2>cost.txt
```

**Expected result:** Lines naming caller/callee and “cost vs threshold” or attribute reason.

**Why:** Heuristics are opaque without remarks; this is the supported observability channel.

**Counterexample:** Release builds without remarks flags—IR changes but you cannot see *why*. Always keep a remarks run when tuning thresholds.

---

## Example 6: `optnone` blocks everything

**Input:** IR from `clang -O0` (has `optnone`) vs `-O1 -Xclang -disable-llvm-passes`.

**Commands:**

```bash
clang -O0 -S -emit-llvm tiny.c -o o0.ll
opt -passes='cgscc(inline)' o0.ll -S -o o0.out.ll   # often no change
clang -O1 -Xclang -disable-llvm-passes -S -emit-llvm tiny.c -o o1.ll
opt -passes='cgscc(inline)' o1.ll -S -o o1.out.ll   # inlines
```

**Expected result:** O0 functions keep calls; O1-disable-passes IR inlines.

**Why:** `optnone` means “leave me alone,” and inlining respects that.

**Counterexample:** Manually deleting `optnone`/`noinline` from O0 IR (fragile) also allows `opt`—prefer the disable-passes emission path.
