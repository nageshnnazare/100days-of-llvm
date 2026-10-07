# Foundation 6 extra: The lowering, statement by statement

Companion to [Foundation 6](Foundation-06-0-C-to-IR.md). Examples: [C-to-IR examples](Foundation-06-2-Extra-C-to-IR-Examples.md). Day 92 is the C++ walk through CodeGen. This note is the correspondence you can check by eye.

## Diagram

```text
  Clang AST                         IR it emits

  FunctionDecl                      define, plus allocas for params
    ParmVarDecl x          ------>  store %x, ptr %x.addr
    CompoundStmt                      (so the parameter has an address)
      DeclStmt int y       ------>  %y = alloca i32
      IfStmt               ------>  icmp + br + two blocks + a join
      ReturnStmt           ------>  load + ret
```

Clang gives parameters addresses too, when it emits unoptimized IR. The incoming `%x` is stored to an alloca, and later reads load it. That is why a one-parameter function can start with several allocas. Mem2reg deletes the ones that never have their address taken.

## Expressions versus statements

An expression produces a value. `a + b` is an `add` (or a load, an add, and stores, if `a` and `b` are memory). A statement transfers control. `if`, `while`, `return`, and `for` create blocks.

`x = a + b` is both: the add produces a value, the store performs the assignment.

## Short-circuit `&&`

```c
if (p && *p) use(*p);
```

The IR must not load `*p` when `p` is null. You get a branch after the null check, and the load lives in the true block only. A single `and` instruction would be wrong if it forced the load to exist on both paths. When both sides are `i1` values already computed, a later pass may turn the branches into `and`.

## Integer promotions, only the visible part

C promotes a `char` operand of `+` to `int`. In the IR you see `sext` or `zext` from `i8` to `i32` before the `add`, then a `trunc` if the result is stored back to a `char`. Those conversions are the promotion, written out. Foundation 8's `signext` on a parameter is a related ABI rule: the caller widened the value before the call.

## What is intentionally missing here

- The ABI for structs passed in registers (Day 92).
- Exception pads (Day 104).
- `invoke` instead of `call` when the callee can throw in C++.

For C functions that do not throw and take scalar arguments, the picture in the diagram is the whole lowering.

## Check yourself

1. Why does unoptimized IR store a parameter to an alloca immediately?
2. Why is `p && *p` lowered to a branch rather than a load on every path?

**Answers.** (1) So the parameter has an address, the same representation as a local. (2) Because the load must not execute when `p` is null.
