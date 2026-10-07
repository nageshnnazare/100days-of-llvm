# Foundation 2 extra: Syntax the printer and the parser agree on

Companion to [Foundation 2](Foundation-02-0-Reading-IR.md). Examples: [syntax examples](Foundation-02-2-Extra-IR-Syntax-Examples.md).

## Diagram

```text
  .ll text
     |
     |  LLParser  (llvm/lib/AsmParser/LLParser.cpp)
     v
  Module in memory          the C++ objects Day 3 names
     |
     |  AsmWriter (llvm/lib/IR/AsmWriter.cpp)
     v
  .ll text again

  opt -S  is this round trip, plus passes in the middle
```

The text format is a serialization. Passes do not edit the text. They edit the objects, and the writer prints them. Two files can mean the same module even when the names differ (`%s` versus `%3`), as long as the instructions and their operands match.

## What the parser is strict about

- Every instruction except a few (notably `store`, `br`, `ret` of void) defines a value, so it needs a result type and, if you name it, a `%` name.
- A use must be dominated by its definition, except operands of a `phi`, which are defined on the predecessor path. The parser accepts the text; the verifier checks dominance (Day 4).
- The type of an operand must be the type the opcode requires. `add i32 %a, %b` is rejected when `%a` is `i64`.

The verifier lives in [llvm/lib/IR/Verifier.cpp](https://github.com/llvm/llvm-project/blob/main/llvm/lib/IR/Verifier.cpp). `opt` runs it before passes. A lesson that shows IR has already passed that check, unless the lesson is about the error.

## Constants are written in place

```llvm
%c = add i32 %a, 1
```

`1` is a constant operand. It has no `%` name. The type is the `i32` on the instruction, not a suffix on the `1`. Boolean constants used by `br` are `true` and `false`, and they have type `i1`.

A bigger constant can be a global:

```llvm
@msg = private unnamed_addr constant [6 x i8] c"hello\00"
```

`c"hello\00"` is a six-byte array: the five letters and a trailing NUL. The type `[6 x i8]` must match the length.

## Labels in the text versus names in the API

In the `.ll` file a block is `entry:`. In an operand it is `label %entry`. Both refer to the same basic block. When you read C++ (Day 3), that block is a `BasicBlock*` and the branch is a `BranchInst` whose successor is that pointer. The text `%entry` is the printed form of the pointer.

## Commas, not spaces, separate operands

```llvm
%c = icmp slt i32 %a, %b
```

`slt` is part of the opcode (a predicate), not an operand. The operands are `%a` and `%b`. Foundation 4 lists the `icmp` predicates.

## Check yourself

1. Does InstCombine rewrite the `.ll` characters, or the in-memory instruction?
2. Why can `%s` in your file become `%1` after `opt -S` and still be the same program?

**Answers.** (1) The in-memory instruction. The writer prints whatever names are left. (2) Names are not semantics. The opcode, the types, and which definition each use points at are the program.
