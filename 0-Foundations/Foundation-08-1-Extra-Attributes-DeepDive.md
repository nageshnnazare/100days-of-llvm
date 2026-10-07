# Foundation 8 extra: Attribute classes

Companion to [Foundation 8](Foundation-08-0-Attributes.md). Examples: [attribute examples](Foundation-08-2-Extra-Attributes-Examples.md).

## Diagram

```text
  Function
    AttributeList
      function index     nounwind, optnone, memory(...)
      return index       noundef, signext
      parameter index 0  nocapture, readonly, noundef
      parameter index 1  ...

  CallInst
    its own attribute list for this site
    (can be stricter or different from the callee's list)
```

The class is [llvm/include/llvm/IR/Attributes.h](https://github.com/llvm/llvm-project/blob/main/llvm/include/llvm/IR/Attributes.h). The enum of known names is generated from [llvm/include/llvm/IR/Attributes.td](https://github.com/llvm/llvm-project/blob/main/llvm/include/llvm/IR/Attributes.td). A pass asks `F->hasFnAttribute(Attribute::NoUnwind)` or `F->doesNotAccessMemory()` rather than scanning the printed text.

## Call-site versus callee

```llvm
declare void @g() nounwind

define void @f() {
  call void @g()
  ret void
}
```

The declaration says every call of `@g` is `nounwind`. A call site may also carry attributes the declaration does not, when this particular call has a stronger fact. Inlining copies the body according to the call, so the attributes on that call are the ones the inliner must respect.

## `optnone` implies the function is skipped

The pass manager checks this attribute before running a function pass. The pass's code does not run. Adding `-passes=instcombine` does not override it. Stripping the attribute, or not emitting it, is the fix. The setup note's `-disable-O0-optnone` exists so Clang does not add it.

`optnone` is paired with `noinline` so the function is also a poor inlining candidate. Otherwise a caller could inline it and the body would be optimized in the caller, defeating the attribute.

## String attributes

Some attributes carry an argument: `dereferenceable(8)`, `align 4`, `"probe-stack"`. In the API these are not simple enum checks; they have a value. The printed form puts the value in parentheses or after the name.

## Check yourself

1. Can a call site have `nounwind` when the declaration does not?
2. Why does the printer sometimes show `#0` instead of a list of words?

**Answers.** (1) Yes. The site describes that call. (2) The words are factored into `attributes #0 = { ... }` so each function can name the group.
