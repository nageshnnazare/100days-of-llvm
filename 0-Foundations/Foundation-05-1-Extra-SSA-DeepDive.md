# Foundation 5 extra: Definitions, uses, and the dominance rule

Companion to [Foundation 5](Foundation-05-0-SSA.md). Examples: [SSA examples](Foundation-05-2-Extra-SSA-Examples.md).

## Diagram

```text
  %b = add i32 %a, 1
         |
         +------------------+
                            v
  %c = mul i32 %b, 2       use of %b

  one Value in memory
    def:  the add instruction (it *is* %b)
    users: the mul, and any other instruction
           whose operand pointer is that add
```

In the C++ API a value is a `Value`. An instruction is a `Value` that also has operands. Each operand is a `Use` pointing back at the defining value. Day 3 walks those classes. The picture above is the whole idea: the defining instruction and the list of uses are the same object, looked at from two sides.

The header for the use list is [llvm/include/llvm/IR/Value.h](https://github.com/llvm/llvm-project/blob/main/llvm/include/llvm/IR/Value.h) (`users()`, `use_begin()`). `PHINode` is in [Instructions.h](https://github.com/llvm/llvm-project/blob/main/llvm/include/llvm/IR/Instructions.h).

## Dominance, stated for readers

A use of `%b` must be in a block that `%b`'s definition dominates, unless the use is a phi incoming value from a predecessor where `%b` is defined. Day 4 makes "dominates" precise. The practical test: every path from the function entry to the use passes through the definition.

```llvm
define i32 @bad(i1 %c) {
entry:
  br i1 %c, label %t, label %f
t:
  %b = add i32 1, 1
  br label %f
f:
  ret i32 %b
}
```

`%b` does not dominate the return, because the path `entry -> f` never executes the add. The verifier diagnoses this. The repair is a phi in a join that both paths reach, or a computation that really is on every path.

## `replaceAllUsesWith`

When a pass proves `%b` is always `1`, it replaces every use of `%b` with the constant `1`, then deletes the add if nothing uses it. You will see this phrase in every transform day's source. It is "change each Use's pointer". It is correct only because there is one definition. There is no hidden second `%b` to miss.

## Check yourself

1. If an instruction has an empty use list and no side effect, what is a later pass allowed to do with it?
2. Why is `replaceAllUsesWith` harder to imagine in a language where a variable can be assigned twice?

**Answers.** (1) Delete it. That is DCE, Day 7. A `store` has an empty SSA use list and is not dead, because the use is memory. (2) Because you would have to know which assignment each read saw. SSA makes that pointer explicit.
