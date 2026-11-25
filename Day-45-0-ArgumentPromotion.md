# Day 45: Argument Promotion

Extra notes: [DeepDive](Day-45-1-Extra-ArgumentPromotion-DeepDive.md) · [Examples](Day-45-2-Extra-ArgumentPromotion-Examples.md)

## Why

C and C++ pass large objects **by pointer** even when the callee only reads a few fields. That forces memory traffic and hides values from SSA-based opts. **Argument promotion** turns “pointer to values the callee only loads” into **scalar arguments passed in registers**, moving the loads to the callers.

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

## The idea

If an `internal` function has a pointer argument `%p` such that:

- `%p` is **not captured** (does not escape the callee),
- the callee only performs **simple loads** of specific offsets/types through `%p` (no stores through `%p`),
- aliasing / ordering allow those loads to be performed **in the caller before the call** without changing semantics,

then replace `%p` with the loaded scalar values as new formals, and at each call site insert the loads and pass the scalars.

This is **not** `byval` (which means “pass a hidden copy”). Promotion *eliminates* the need for the callee to touch that memory.

## Pipeline position and `opt -passes=` name

```bash
opt -passes=argpromotion input.ll -S -o out.ll
opt -passes=argpromotion -debug-only=argpromotion input.ll -S
```

Code: [`llvm/lib/Transforms/IPO/ArgumentPromotion.cpp`](https://github.com/llvm/llvm-project/blob/main/llvm/lib/Transforms/IPO/ArgumentPromotion.cpp).

Often O3-leaning in PassBuilder historically (check your `PassBuilderPipelines.cpp`—argpromotion has been gated by level).

```bash
clang -S -emit-llvm -O1 -Xclang -disable-llvm-passes file.c -o file.ll
```

## Core concepts

### Conditions (checklist)

1. **Internal linkage** (ABI private).
2. **No store** through the pointer argument (and usually no `atomicrmw` / volatile weirdness).
3. **No capture** — `%p` not stored, not returned, not passed to capturing callees.
4. Loads are of **promotable simple types** (integers/floats/pointers of known size—not opaque VLAs).
5. Callers can safely load — AA proves nothing else may modify that memory between caller’s load point and the callee’s old load (for non-escaping local allocas this is easy; for globals harder).
6. All call sites are visible and updatable (same as DAE).

### Alias analysis role

Promotion must ensure that a load hoisted to the caller still sees the value the callee would have loaded. If another argument might alias and the caller stores through it before the call, hoisting can be wrong unless AA proves no alias / no intervening write.

### `byval` vs promotion

| Mechanism | Meaning |
| --- | --- |
| `byval` | ABI: pass a copy of the pointed-to object |
| ArgPromotion | Optimization: pass loaded fields as SSA values; callee never sees the pointer |

## Algorithm

1. For each internal function, examine pointer args.
2. Collect load set (offsets / types).
3. Prove no store/capture; query AA at call sites.
4. Build new function type with scalars instead of the pointer.
5. In callee: replace loads of `%p` with the new formals.
6. In callers: insert loads, pass scalars; drop pointer arg.

## Code map

| Item | Link |
| --- | --- |
| Pass | [`ArgumentPromotion.cpp`](https://github.com/llvm/llvm-project/blob/main/llvm/lib/Transforms/IPO/ArgumentPromotion.cpp) |
| AA | [`llvm/include/llvm/Analysis/AliasAnalysis.h`](https://github.com/llvm/llvm-project/blob/main/llvm/include/llvm/Analysis/AliasAnalysis.h) |
| Related | DeadArgElim (Day 44), Attributor `AANoCapture` (Day 48) |

## Worked example

```c
typedef struct { int a; int b; } S;
static int sum(const S *p) { return p->a + p->b; }
int caller(void) {
  S s = {1, 2};
  return sum(&s);
}
```

After argpromotion (conceptual IR):

```llvm
define internal i32 @sum(i32 %a, i32 %b) {
  %t = add i32 %a, %b
  ret i32 %t
}
define i32 @caller() {
  ; loads of s.a / s.b (or constants 1,2 already in regs)
  %r = call i32 @sum(i32 1, i32 2)
  ret i32 %r
}
```

## Reading order

1. Pass file comment + eligibility in `ArgumentPromotion.cpp`.
2. Load collection / isSafeToPromote style helpers.
3. Call-site mutation.
4. [DeepDive](Day-45-1-Extra-ArgumentPromotion-DeepDive.md).

## Hands-on

1. Struct pointer, loads only — promote.
2. Add a store through the pointer — refuse.
3. Pass pointer to external function — capture → refuse.
4. Two pointer args that may alias with a store — study AA behavior.
5. `-debug-only=argpromotion`.

## Pitfalls

- Confusing with `byval`.
- Expecting promotion on exported functions.
- Padding / alignment: loads must match ABI-safe types.
- Incomplete call-site set (address taken).

## Neighbors

- **Day 44** — remove args; **Day 45** — replace pointer args with scalars.
- **Day 48** — capture / alias attributes feed similar reasoning.
- **Day 47 GlobalOpt** — sometimes localizes globals making promotion easier.

## Opaque-pointer IR reminder

All illustrations use `ptr`:

```llvm
define internal i32 @sum(ptr %p) {
  %pa = getelementptr inbounds nuw i8, ptr %p, i64 0
  %a = load i32, ptr %pa, align 4
  %pb = getelementptr inbounds nuw i8, ptr %p, i64 4
  %b = load i32, ptr %pb, align 4
  %t = add i32 %a, %b
  ret i32 %t
}
```

Promotion turns this into two `i32` formals. Do not write typed-pointer `i32*` in study notes.

## Check yourself

**Q1.** Pass name?  
**A1.** `argpromotion`.

**Q2.** Must the callee store through the pointer?  
**A2.** No—stores generally **disqualify** promotion.

**Q3.** Where do loads go after promotion?  
**A3.** Inserted at call sites in callers; callee receives scalars.

**Q4.** Linkage requirement?  
**A4.** Internal (ABI private), same family as DAE.

**Q5.** Debug flag?  
**A5.** `-debug-only=argpromotion`.

## Promotion rewrite in opaque-pointer IR

Before:

```llvm
define internal i32 @sum(ptr %p) {
  %pa = getelementptr inbounds i8, ptr %p, i64 0
  %a = load i32, ptr %pa, align 4
  %pb = getelementptr inbounds i8, ptr %p, i64 4
  %b = load i32, ptr %pb, align 4
  %t = add i32 %a, %b
  ret i32 %t
}
define i32 @caller() {
  %s = alloca { i32, i32 }, align 4
  store { i32, i32 } { i32 1, i32 2 }, ptr %s
  %r = call i32 @sum(ptr %s)
  ret i32 %r
}
```

After a successful promotion the callee no longer receives `ptr`. Callers load (or already know) the two `i32`s and pass them by value. Subsequent `mem2reg`/`instcombine` often fold the caller’s loads of the constant aggregate into `call i32 @sum(i32 1, i32 2)`.

## Why a store through an aliasing argument is fatal

```llvm
define internal i32 @f(ptr %p, ptr %q) {
  store i32 7, ptr %q, align 4
  %v = load i32, ptr %p, align 4
  ret i32 %v
}
```

If the call site passes the same pointer for `%p` and `%q`, the callee must observe `7`. Hoisting the load of `%p` to the caller would read the pre-store value. AA must prove no-alias, or the pass refuses `%p`.

## `byval` is not this transform

`ptr byval(%struct.S)` means the ABI already made a hidden copy. Argument promotion’s goal is different: the callee stops touching that memory and receives scalars. A function can have `byval` and still be a poor promotion candidate, and a function can be promoted without ever having been `byval`.

## Partial field promotion

If the callee loads only `p->a` and never mentions `p->b`, the new prototype gains one `i32`, not the whole struct. Any leftover use of the pointer (a GEP that is stored, or a call that captures `%p`) keeps the pointer formal and usually aborts promotion of that argument. All direct call sites must be rewritten; an address-taken internal function is treated like DAE treats it: conservatively skipped when the pass cannot see every use.

## Alignment and simple types

Promotion only moves loads of simple scalar types (integers, floats, pointers) at constant offsets. A load of an unsized or opaque aggregate, a volatile load, or an atomic load is not a candidate. The loads inserted in the caller must use an alignment that is legal for that type; the pass copies alignment from the original load rather than inventing a wider one. Debug with `-debug-only=argpromotion` and look for the first failed safety check: store, capture, or alias.

Promotion inserts the loads in the caller, before the call, at the original alignment.
The callee’s GEPs of that pointer disappear along with the pointer formal.
A store through the pointer disqualifies it even if the store is on a path that looks unused.
Capture includes storing the pointer, returning it, or passing it to a callee that is not nocapture.
AA is required when another argument may alias the promoted pointer and is written in the callee.
byval means the ABI already copied the object; it is not the same rewrite.
Only internal linkage is eligible, for the same ABI reason as deadargelim.
If only one field is loaded, only that field becomes a scalar argument.
Address-taken functions are skipped when every use cannot be updated.
The pass is often enabled at O3 in PassBuilder; at O2 you can still run -passes=argpromotion by hand.
-debug-only=argpromotion prints the safety check that failed.
Implementation: llvm/lib/Transforms/IPO/ArgumentPromotion.cpp.
After promotion, instcombine folds constant loads in the caller into constant arguments.
