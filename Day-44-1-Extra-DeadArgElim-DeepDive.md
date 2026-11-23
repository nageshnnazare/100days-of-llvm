# Day 44 Extra: Dead Argument Elimination Deep Dive

Companion to [Day-44-0-DeadArgElim.md](Day-44-0-DeadArgElim.md). Examples: [Day-44-2-Extra-DeadArgElim-Examples.md](Day-44-2-Extra-DeadArgElim-Examples.md).

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

## 1. Why a dedicated IPO pass?

InstCombine can delete *instructions* that use an argument, and ADCE can delete unused code, but **neither changes `FunctionType`**. As long as the formal exists, every call must still pass an operand. DAE is the pass that admits “this ABI is compiler-private” and shrinks it.

That is why linkage is the gate: `internal` / `private` mean the linker never merges this symbol with another TU’s idea of the type.

## 2. Control flow of the pass

[`DeadArgumentElimination.cpp`](https://github.com/llvm/llvm-project/blob/main/llvm/lib/Transforms/IPO/DeadArgumentElimination.cpp) roughly:

1. **Collect** candidates.
2. **Analyze** each argument and each return “lane” (structs can have multiple elements).
3. **Mark** dead / live / maybe-dead-constant.
4. **Rewrite** functions that improved.
5. Iterate if needed when changes expose more deadness (depends on implementation details—read the run loop).

Struct returns and aggregate arguments need element-wise liveness: callers might use only `{i32, i32}`’s first field.

## 3. Unused vs dead-constant

### Unused

```llvm
define internal void @f(i32 %dead, i32 %x) {
  call void @use(i32 %x)
  ret void
}
```

`%dead` has no uses → remove.

### Dead constant

```llvm
define internal i32 @g(i32 %c, i32 %x) {
  ; %c is used, but only in ways equivalent to the constant all callers pass
  %t = add i32 %x, %c
  ret i32 %t
}
; all sites: call i32 @g(i32 5, i32 %…)
```

If every site passes `5`, DAE can substitute `5` into the body and drop `%c`. If any site passes a non-constant or a different constant, `%c` is live as a parameter.

IPSCCP may already have replaced `%c`’s uses with `5`, making the argument unused—then DAE’s simpler unused rule fires. The dedicated dead-constant logic catches cases where uses remain but are still ABI-redundant.

## 4. Dead returns

```llvm
define internal i32 @compute(i32 %x) {
  %y = mul i32 %x, %x
  ret i32 %y
}
define void @caller(i32 %x) {
  call i32 @compute(i32 %x)  ; result unused
  ret void
}
```

DAE can turn `@compute` into `void` and erase the returned value. If *any* caller uses the result, the return stays.

## 5. Call-site rewriting details

Call sites may look like:

```llvm
%v = call i32 @old(i32 %a, i32 %b)
```

or with bundles, operand bundles, `fastcc`, parameter attributes (`zeroext`, `nonnull` on later args). When removing arg 0, attributes on arg 1 must become attributes on arg 0 of the new type. Bugs here show up as verifier failures or miscompiles on platforms where zeroext matters.

`invoke` must be updated the same way as `call`. Blockaddress and `musttail` callees need extra checks—often the pass **bails** on the whole function.

## 6. Things that force a bail-out

Search the file for early `return` / skip conditions. Expect at least:

- `isVarArg()`
- `hasAddressTaken()` without full call-site knowledge (address taken makes signature changes dangerous unless all uses are accounted for)
- `musttail` present
- `naked`
- incompatible calling conventions edge cases
- non-local linkage

Address-taken internal functions are especially thorny: `@f` might be stored in a table of a specific type. Changing the type without rewriting the table is wrong → DAE conservatively skips.

## 7. Ordering with other IPO

Recommended mental pipeline:

```
ipsccp / instcombine
    → deadargelim          ; drop dead formals
    → argpromotion         ; ptr → scalars (Day 45)
    → inline               ; smaller ABIs inline easier
    → globaldce
```

PassBuilder’s real order differs by level; locally you can experiment with explicit pass lists.

## 8. Debugging

```bash
opt -passes=deadargelim -debug-only=deadargelim in.ll -S -o out.ll 2>dae.log
```

Log typically states which arguments are dead and which functions are transformed. Pair with `llvm-diff` or plain `diff` on IR.

## 9. Verifier-driven development

After any hand-reduced test:

```bash
opt -passes='deadargelim,verify' t.ll -S -o /dev/null
```

If DAE produces ill-typed calls, verify fails loudly—good.

## 10. Interaction with `available_externally`

`available_externally` provides a body for optimization **without emitting** the function (the real definition is elsewhere). DAE may optimize the local body / calls carefully but must not emit a conflicting ABI. Read the pass’s linkage checks rather than assuming full internal freedom.

## 11. Source reading checklist

1. Linkage eligibility.
2. Argument liveness survey including constant case.
3. Return liveness.
4. Function replacement utility calls (Create / splice body).
5. Attribute remapping.

## 12. Takeaway

DAE is **ABI rewriting for private functions**. If you remember only one sentence, remember that—and that varargs/musttail/naked/external are the usual “no.”
