# Day 44: Dead Argument Elimination

Extra notes: [DeepDive](Day-44-1-Extra-DeadArgElim-DeepDive.md) · [Examples](Day-44-2-Extra-DeadArgElim-Examples.md)

## Why

After inlining, IPSCCP, and ordinary DCE, many **formals are never used**—or every caller passes the same constant the callee ignores. Those arguments still occupy ABI slots, register pressure at call sites, and mental noise in IR. **Dead argument elimination** rewrites the function type and all call sites for functions whose linkage allows it.

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

## The idea

For an eligible function (typically `internal` or sometimes `available_externally`):

1. Find arguments that are **unused**, or **dead constants** (every caller passes the same constant and the callee does not observe a different value).
2. Find **return values** that no caller uses → turn the function into returning `void` (internal).
3. Replace the function with a new `Function` type (or surgically update), bitcast/replace all call sites, delete old formals.

This is true ABI surgery—more than IPSCCP’s “zap uses.”

## Pipeline position and `opt -passes=` name

```bash
opt -passes=deadargelim input.ll -S -o out.ll
opt -passes=deadargelim -debug-only=deadargelim input.ll -S
```

Code: [`llvm/lib/Transforms/IPO/DeadArgumentElimination.cpp`](https://github.com/llvm/llvm-project/blob/main/llvm/lib/Transforms/IPO/DeadArgumentElimination.cpp).

```bash
clang -S -emit-llvm -O1 -Xclang -disable-llvm-passes file.c -o file.ll
```

## Core concepts

### Unused argument

No use of the `Argument` in the function body (after accounting for dbg intrinsics / lifetime noise the pass understands). Safe to remove for internal functions: drop from type; drop corresponding call operands.

### Dead constant argument

Every call site passes the **same constant**, and the function’s uses of that argument are consistent with replacing them by that constant (the incoming value is not “live” as a varying parameter). The pass can eliminate the formal and substitute the constant in the body.

### Dead return

If every call instruction’s value is unused (`call void`-like uses), an internal function can be rewritten to `void` return, and `ret` instructions adjusted.

### What blocks the pass

- **External / weak / linkonce** (interposable) — cannot change signature; other TUs / replaceable copies disagree.
- **Varargs** — hard to reshape.
- **`musttail`** chains — signature matching requirements.
- **`naked`** functions — no ordinary prologue; don’t touch.
- Indirect calls through bitcasted prototypes — must update all *known* call sites; unknown ones forbid changes.

## Algorithm

1. Survey functions; skip ineligible linkage / attributes.
2. Liveness survey of arguments and returned values w.r.t. all call sites in the module.
3. Compute a new prototype for each mutable function.
4. Create new function / mutate type; clone body mapping kept args.
5. Rewrite every direct call/invoke; erase old function if replaced.

## Code map

| Item | Link |
| --- | --- |
| Pass | [`DeadArgumentElimination.cpp`](https://github.com/llvm/llvm-project/blob/main/llvm/lib/Transforms/IPO/DeadArgumentElimination.cpp) |
| Related cleanup | IPSCCP (Day 43), ArgPromotion (Day 45), GlobalDCE (Day 46) |

## Worked example

```llvm
define internal i32 @add3(i32 %unused, i32 %x) {
  %y = add i32 %x, 3
  ret i32 %y
}
define i32 @go(i32 %v) {
  %r = call i32 @add3(i32 999, i32 %v)
  ret i32 %r
}
```

```bash
opt -passes=deadargelim dae.ll -S -o dae.out.ll
```

Expected shape:

```llvm
define internal i32 @add3(i32 %x) {
  %y = add i32 %x, 3
  ret i32 %y
}
define i32 @go(i32 %v) {
  %r = call i32 @add3(i32 %v)
  ret i32 %r
}
```

## Reading order

1. File header comment in `DeadArgumentElimination.cpp`.
2. Eligibility checks (linkage, varargs, musttail, naked).
3. Liveness of returns vs arguments.
4. Call-site rewriting.
5. [DeepDive](Day-44-1-Extra-DeadArgElim-DeepDive.md).

## Hands-on

1. Unused formal — remove.
2. Same constant at all sites — dead-constant case.
3. External function with unused arg — **no** change.
4. Dead return → `void`.
5. Combine `ipsccp,deadargelim`.

## Pitfalls

- Expecting DAE on exported functions.
- Forgetting to update *all* call sites (hand-written IR with bitcast calls).
- `musttail` pair must keep matching types.
- Debug info / bundle operands can look like “uses.”

## Neighbors

- **Day 43 IPSCCP** — creates dead/constant args.
- **Day 45 ArgPromotion** — replaces pointer args with scalars (different transform).
- **Day 49 Linkage** — why `internal` matters.

## Signature rewrite checklist

When reading a DAE diff, verify all of:

1. `Function` type string changed (param count / return type).
2. Every `call` / `invoke` operand list matches.
3. Attributes on kept arguments shifted to new indices correctly.
4. No dangling `ptrbitcast` of the old function type left behind.
5. Personality / `musttail` call still verifier-clean (`opt -passes=verify`).

A pass that drops an argument but leaves a call with the old arity will fail the IR verifier immediately—use that as your first sanity check in hands-on work.

## Check yourself

**Q1.** Pass name?  
**A1.** `deadargelim`.

**Q2.** Can it change `external` signatures?  
**A2.** No.

**Q3.** What is the dead-constant case?  
**A3.** All callers pass the same constant and the parameter is not needed as a varying value.

**Q4.** Name three blockers.  
**A4.** varargs, musttail, naked (also bad linkage).

**Q5.** Debug flag?  
**A5.** `-debug-only=deadargelim`.

## Dead-constant IR before and after

```llvm
define internal i32 @addk(i32 %k, i32 %x) {
  %y = add i32 %x, %k
  ret i32 %y
}
define i32 @a(i32 %x) {
  %r = call i32 @addk(i32 4, i32 %x)
  ret i32 %r
}
define i32 @b(i32 %x) {
  %r = call i32 @addk(i32 4, i32 %x)
  ret i32 %r
}
```

After `deadargelim`, both callers pass only `%x`, and the body contains `add i32 %x, 4`. If `@b` passed `5`, the constant case fails and `%k` stays.

## Dead return rewrite

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

No caller uses the returned `i32`, so DAE can change `@work` to `void` and turn the call into `call void @work(i32 %x)`. One additional user of the return anywhere in the module blocks the rewrite.

## Why address-taken functions are skipped

```llvm
@tbl = internal constant [1 x ptr] [ptr @f]
define internal i32 @f(i32 %unused, i32 %x) {
  ret i32 %x
}
```

An indirect call through `@tbl` is not a typed direct call the pass can rewrite safely if the stored type must stay stable. DAE usually refuses the whole function rather than leaving a mismatched function pointer. Direct-call-only internal functions are the happy path.

## Attribute index shift

Removing argument 0 must slide `zeroext` / `signext` / `noundef` from old index 1 onto new index 0. A correct DAE diff is not only fewer operands; the attribute groups on the new `FunctionType` must match the kept parameters. `opt -passes='deadargelim,verify'` catches most mistakes.

## Varargs, naked, musttail — one sentence each

A varargs function’s trailing arguments are not a normal formal the pass can drop from a fixed prototype. A `naked` function has no compiler-generated prologue, so reshaping the ABI is outside DAE’s model. A `musttail` call must match the callee’s prototype and calling convention; deleting a forwarded argument would break that match, so DAE bails out of the function instead of producing illegal IR.

## available_externally is not a free ABI rewrite

`available_externally` gives this module a body for optimization and promises the real definition is elsewhere. DAE may clean the local body in limited ways, but it must not publish a different signature. If you need a signature change, the symbol has to be `internal` or `private` in this module, which is what LTO’s internalize step arranges for functions that are not part of the public API.

DAE’s output must pass the verifier: every call’s operand count matches the new type.
invoke is rewritten the same way as call, including unwind edges.
Parameter attributes move with the kept arguments; they are not left on deleted indices.
Debug info uses of an argument are not enough to keep it live for codegen, but the pass must not crash on them.
A function whose address is stored in a table of a fixed type is not rewritten.
musttail, naked, and varargs are whole-function bailouts, not per-argument bailouts.
The dead-constant rule needs every call site to pass the same constant, not merely a constant.
