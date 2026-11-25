# Day 45 Extra: Argument Promotion Deep Dive

Companion to [Day-45-0-ArgumentPromotion.md](Day-45-0-ArgumentPromotion.md). Examples: [Day-45-2-Extra-ArgumentPromotion-Examples.md](Day-45-2-Extra-ArgumentPromotion-Examples.md).

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

## 1. Transform shape

Argument promotion is a **search + prove + rewrite** IPO pass:

```
pointer formal %p
   │
   ├─ collect loads(GEP offsets, types)
   ├─ prove: no store via %p, no capture of %p
   ├─ prove at each call site: loads hoistable
   └─ rewrite prototype: %p → %v0, %v1, …
```

Implementation: [`ArgumentPromotion.cpp`](https://github.com/llvm/llvm-project/blob/main/llvm/lib/Transforms/IPO/ArgumentPromotion.cpp).

## 2. Why “loads move to the caller”

The callee’s `load` is delayed until after the call executes prologue. Semantically, for a non-capturing read-only pointer to stable memory, loading earlier is fine:

```
caller: store 1 into s.a; store 2 into s.b
caller: load a,b          ← inserted by promotion
caller: call sum(a,b)
callee: use a,b           ← was load inside callee
```

If something could write the same memory after the call begins but before the old load (e.g. via aliasing other args or globals in a concurrent tale), promotion would be wrong. LLVM’s pass is **single-threaded / well-synchronized IR** under LLVM’s usual memory model assumptions; it still uses AA for *same-thread* intervening stores through may-alias pointers.

## 3. Capture

A pointer is captured if its value may be observed after the call returns, or passed to code that might store it. Examples of capture:

- `store ptr %p, ptr @g`
- `ret ptr %p`
- `call void @external(ptr %p)` when `@external` is not known `nocapture`

The Attributor’s `AANoCapture` / function-attrs `nocapture` are related analyses; ArgPromotion has its own checks but benefits when attributes already say `nocapture readonly`.

## 4. Stores kill promotion

```llvm
define internal void @set(ptr %p, i32 %v) {
  store i32 %v, ptr %p
  ret void
}
```

Cannot promote `%p` to an `i32` by value *as a load promotion*—the callee writes memory the caller may still have as an alloca. (Different transforms exist for other patterns; this pass’s classic form is load promotion.)

## 5. AA between arguments

Classic hazard:

```llvm
define internal i32 @f(ptr %p, ptr %q) {
  store i32 0, ptr %q
  %v = load i32, ptr %p
  ret i32 %v
}
```

If `%p` and `%q` may alias, promoting `%p`’s load to the caller (before `store` through `%q`) reads the **old** value, but the callee would read **0**. Promotion must refuse or only promote when AA proves `%p`/`%q` no-alias and there is no other may-write.

## 6. Partial struct promotion

Only some fields may be loaded. The pass can promote **just those offsets**, leaving other fields behind—or refuse if residual pointer uses remain. Residual GEPs that still need `%p` after removing some loads typically block full elimination of `%p`; either promote all needed elements or bail.

## 7. Call-site plumbing

For each direct call:

1. Compute pointer actual (`%s` alloca, GEP, etc.).
2. Emit `load`s with correct alignment / TBAA metadata when preserved.
3. Call new function with scalars.
4. Remove old pointer operand.

Must handle `invoke`, attributes (`nonnull`, `align`, `dereferenceable` on the pointer may become properties of the promotion legality rather than moving verbatim onto scalars).

## 8. Pipeline placement

Historically `ArgumentPromotionPass` is more aggressive at **O3** in PassBuilder. At O2 you may still run it manually:

```bash
opt -passes=argpromotion -S in.ll
```

After promotion, instcombine and SROA clean redunant allocas in callers.

## 9. Debugging

```bash
opt -passes=argpromotion -debug-only=argpromotion in.ll -S 2>ap.log
```

Look for messages about unsafe stores, capture, or AA failure.

## 10. Comparison table

| Pass | Changes formals how? |
| --- | --- |
| deadargelim | Deletes unused / dead-constant formals |
| argpromotion | Replaces pointer formal with scalar formals |
| IPSCCP | Does not change types (values only) |
| inliner | Removes the call entirely |

## 11. Source reading order

1. Safety checks for a candidate argument.
2. Offset/type collection from loads.
3. Per-call-site AA / hoistability.
4. Function cloning / argument remapping.
5. Tests under `llvm/test/Transforms/ArgumentPromotion/` (in a full checkout).

## 12. Practical advice

Prefer promoting pointers to **stack allocas** first in experiments—AA and non-escape are obvious. Global pointers are harder and more often refused. Use `-debug-only=argpromotion` before assuming the pass “does nothing”—it may be silently skipping for capture reasons.
