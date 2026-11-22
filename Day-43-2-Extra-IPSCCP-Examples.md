# Day 43 Extra: IPSCCP Examples

Companion to [Day-43-0-IPSCCP.md](Day-43-0-IPSCCP.md). Deep dive: [Day-43-1-Extra-IPSCCP-DeepDive.md](Day-43-1-Extra-IPSCCP-DeepDive.md).

```bash
clang -S -emit-llvm -O1 -Xclang -disable-llvm-passes f.c -o f.ll
opt -passes=ipsccp f.ll -S -o f.out.ll
opt -passes=ipsccp -debug-only=sccp f.ll -S
```

---

## Diagram

```text
   define void @caller() {
     call void @callee(i32 7)
   }
   define void @callee(i32 %x) {
     ; SCCP inside @callee sees %x as overdefined,
     ; because a direct call could pass anything.
   }

   IPSCCP looks at the call edge:
     the only call passes 7
     so %x is Constant 7 in @callee
     and folds uses of %x
```

IPSCCP is SCCP plus the call graph. A parameter that is the same constant at every call site becomes that constant inside the function. A return value that is a constant flows back to the callers. This needs the whole SCC: a recursive function that sometimes passes a different value stays overdefined.

## Example 1: Constant argument folds callee

**Input:**

```c
static int pick(int mode) {
  if (mode == 1) return 10;
  if (mode == 2) return 20;
  return -1;
}
int once(void) { return pick(1); }
```

**Commands:**

```bash
clang -S -emit-llvm -O1 -Xclang -disable-llvm-passes e1.c -o e1.ll
opt -passes=ipsccp e1.ll -S -o e1.out.ll
```

**Expected result:** `pick`’s `mode` lattice is `1` (sole caller); returns `10`; `once` can become `ret i32 10` (possibly leaving a simplified `pick` or folding the call).

**Why:** IP argument meet + conditional SCCP inside `pick`.

**Counterexample:** Add `int twice(void){ return pick(2); }` — `mode` overdefined; neither caller necessarily folds through IPSCCP alone (specialization might clone).

---

## Example 2: Constant return without inlining

**Input:**

```c
static int answer(void) { return 42; }
int use(void) { return answer() + answer(); }
```

**Commands:**

```bash
opt -passes=ipsccp e2.ll -S -o e2.out.ll
```

**Expected result:** Both calls replaced by `42`; `use` becomes `84` after trivial folding (IPSCCP and/or later instsimplify).

**Why:** Return lattice is constant `42` for the internal function.

**Counterexample:** Make `answer` `noinline` **external** with a body in another TU only declared here—without a body, IPSCCP cannot prove the return.

---

## Example 3: Conflicting call sites → overdefined

**Input:**

```c
static int addk(int x, int k) { return x + k; }
int a(int x) { return addk(x, 1); }
int b(int x) { return addk(x, 2); }
```

**Commands:**

```bash
opt -passes=ipsccp e3.ll -S -o e3.out.ll
opt -passes=func-specialization e3.ll -S -o e3.spec.ll
```

**Expected result:** `ipsccp` does **not** turn `addk` into “always +1”. `func-specialization` *may* create specialized clones (profitability-dependent).

**Why:** Demonstrates overdefined vs specialization.

**Counterexample:** Only keep `a` — then `k` is constant `1` and IPSCCP folds.

---

## Example 4: Dead blocks after IP fold

**Input:**

```c
static int f(int flag) {
  if (flag) return 1;
  /* large cold code */
  return helper();
}
int g(void) { return f(1); }
int helper(void); /* external */
```

**Commands:**

```bash
opt -passes=ipsccp e4.ll -S -o e4.out.ll
```

**Expected result:** With `flag` constant true, the false branch calling `@helper` may become non-executable and be removed from `@f`’s body (internal).

**Why:** Sparse reachability + constant arg.

**Counterexample:** `f` also called as `f(0)` elsewhere—false branch stays live.

---

## Example 5: External ABI locked, local call folded

**Input (IR):**

```llvm
define i32 @ext_const() {
  ret i32 7
}
define i32 @caller() {
  %v = call i32 @ext_const()
  ret i32 %v
}
```

**Commands:**

```bash
opt -passes=ipsccp e5.ll -S -o e5.out.ll
```

**Expected result:** `@caller` may `ret i32 7` while `@ext_const` still exists with the same signature for other TUs / linker.

**Why:** Call-site replacement ≠ signature change.

**Counterexample:** `define weak i32 @ext_const()` — interposition may make folding unsafe; weaker passes must respect linkage (observe whether your LLVM folds or conservatively skips).

---

## Example 6: Do not confuse with DAE

**Input:**

```c
static int ignore(int dead, int x) { (void)dead; return x; }
int h(int x) { return ignore(123, x); }
```

**Commands:**

```bash
opt -passes=ipsccp e6.ll -S -o e6.ipsccp.ll
opt -passes='ipsccp,deadargelim' e6.ll -S -o e6.both.ll
```

**Expected result:** IPSCCP may constant-fold/zap uses of `dead` but leave the parameter in the type. `deadargelim` removes the formal and updates call sites.

**Why:** Separation of concerns between value propagation and ABI rewrite.

**Counterexample:** Running only `deadargelim` without proving anything may still remove a truly unused formal—but the “always same constant” case is the interesting IPSCCP→DAE handoff (Day 44).
