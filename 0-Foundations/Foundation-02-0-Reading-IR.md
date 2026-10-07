# Foundation 2: How to read a `.ll` file

LLVM IR's text format is a programming language with a small grammar. Every later day quotes it. This note is the grammar you need in order to read those quotes.

Deep dive: [syntax rules](Foundation-02-1-Extra-IR-Syntax-DeepDive.md). Examples: [syntax examples](Foundation-02-2-Extra-IR-Syntax-Examples.md). The specification is [LangRef's identifier and instruction sections](https://llvm.org/docs/LangRef.html#identifiers).

## Diagram

```text
  define i32 @add(i32 %a, i32 %b) {
     |   |    |    |  |
     |   |    |    |  +-- parameter name (local, so %)
     |   |    |    +----- parameter type
     |   |    +---------- function name (global, so @)
     |   +--------------- return type
     +------------------- "this is a function body"
 
  entry:                         block label
    %s = add i32 %a, %b          an instruction that defines %s
    |    |   |   |   |
    |    |   |   |   +---------- second operand
    |    |   |   +-------------- first operand
    |    |   +------------------ type of the operands
    |    +---------------------- opcode
    +--------------------------- defined name

    ret i32 %s                   terminator: the block ends here
  }
```

Read a function from the signature inward, then each instruction left to right: defined name, opcode, types, operands.

## Names: `@` and `%`

`@` names something that lives in the module: a function, a global variable, an alias.

`%` names something that lives in one function: an argument, an instruction's result, a basic block.

```llvm
@g = global i32 0          ; module scope

define i32 @add(i32 %a, i32 %b) {
  %s = add i32 %a, %b      ; function scope
  ret i32 %s
}
```

`%s` in `@add` and `%s` in another function are different values. `@add` is visible to the whole module.

## A name is the instruction

`%s = add i32 %a, %b` does two things: it adds, and it gives the result a name. There is no separate "variable" that the add stores into. When a later instruction says `%s`, it means "the result of that add".

That is why the left-hand side appears once. Writing a second `%s = ...` in the same function is an error. Foundation 5 is this rule, called SSA, with the `phi` exception.

## Types sit next to the values

Integer addition needs the width, because `i32` and `i64` are different operations:

```llvm
%s = add i32 %a, %b
%w = add i64 %a64, %b64
```

The type after `add` is the type of both operands and of the result. You do not look it up in a side table. Foundation 3 is the full set of types. For reading, recognize `i32`, `i64`, `i1`, `ptr`, `float`, and `double` first.

## Blocks and terminators

A label (`entry:`) starts a basic block. The block runs from that label to a terminator. Terminators are the instructions that leave the block: `ret`, `br`, `switch`, `unreachable`.

```llvm
define i32 @abs(i32 %x) {
entry:
  %neg = icmp slt i32 %x, 0
  br i1 %neg, label %minus, label %plus
minus:
  %y = sub i32 0, %x
  ret i32 %y
plus:
  ret i32 %x
}
```

`br i1 %neg, label %minus, label %plus` means: if `%neg` is true, go to `minus`, otherwise go to `plus`. Nothing can be written after a terminator in that block. An instruction between `br` and the next label is a parse or verifier error.

`label %minus` is not a pointer. It is the name of a block.

## Comments and noise you can skip on a first read

- A comment starts at `;` and runs to the end of the line.
- Attributes (`noundef`, `nounwind`, `#0`) decorate a function or a parameter. Foundation 8 is the field guide. On a first reading, skip them and read the types and the opcodes.
- `align 4` on a load or an alloca is an alignment in bytes. Skip it until a pass starts caring.
- `!dbg !12` and other `!` metadata are debug info and hints. Day 98. Skip them while learning the instructions.

## Unnamed numbers

If Clang does not print a name, values are numbered: `%0`, `%1`, `%2`. The numbers include unnamed basic blocks, not only instructions. This function:

```llvm
define i32 @add(i32 %0, i32 %1) {
  %3 = add i32 %0, %1
  ret i32 %3
}
```

has no `%2` on an instruction. `%2` is the unnamed entry block. The parameters took `%0` and `%1`. Naming the block `entry:` usually makes the add come out as `%2`, because the block no longer consumes a number. When a number seems skipped, look for an unnamed block before you look for a deleted instruction.

## Hands-on

Take `add.ll` from Foundation 1. For the `add` instruction, name out loud: the defined value, the opcode, the type, and both operands. Then find the terminator.

## Pitfalls

- `%a` is not "the address of a". It is the value. An address has type `ptr` and is produced by `alloca`, a global `@name`, or `getelementptr`.
- `i32` on `ret i32 %s` is the type of the returned value, not a cast.
- A block name in `label %plus` must match a label in this function. It is not a global.

## Check yourself

1. In `%s = add i32 %a, %b`, what is the type of `%s`?
2. Why is `@add` spelled with `@` and `%a` with `%`?
3. What is wrong with a block that contains `add` after `ret`?

**Answers.** (1) `i32`. (2) `@add` is a function in the module; `%a` is an argument of that function. (3) `ret` is a terminator, so the block has already ended. The `add` is unreachable text, and the verifier rejects it.
