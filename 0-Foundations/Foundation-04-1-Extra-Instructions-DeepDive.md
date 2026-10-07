# Foundation 4 extra: The opcode list is a `.def` file

Companion to [Foundation 4](Foundation-04-0-Instructions.md). Examples: [instruction examples](Foundation-04-2-Extra-Instructions-Examples.md).

## Diagram

```text
  llvm/include/llvm/IR/Instruction.def
        |
        |  macros: one line per opcode
        v
  enum Instruction::Add, ::Load, ::Br, ...
        |
        +-- class AddOperator / BinaryOperator
        +-- class LoadInst
        +-- class BranchInst
        +-- class PHINode
        +-- class GetElementPtrInst
```

When Day 3 says "an instruction is a class", the opcode in that class comes from [Instruction.def](https://github.com/llvm/llvm-project/blob/main/llvm/include/llvm/IR/Instruction.def). The behaviors are specified in LangRef, not in the `.def` file. The `.def` file is the list of names the C++ enum is allowed to have.

## How to look up an instruction you do not know

1. Search LangRef for the opcode (`shl`, `freeze`, `extractelement`).
2. If you need the C++ class, search `class LoadInst` under `llvm/include/llvm/IR/Instructions.h`.

A pass checks the opcode with `I->getOpcode() == Instruction::Add` or with a helper such as `I->isBinaryOp()`. PatternMatch (Day 2's deep dive) is sugar over those checks.

## Terminators are a property, not a comment

`Instruction::isTerminator()` is true for `ret`, `br`, `switch`, `indirectbr`, `invoke`, `callbr`, `resume`, `catchswitch`, `catchret`, `cleanupret`, and `unreachable`. Day 3 and the verifier use this. `invoke` is the terminator that can throw (Day 104). You will not see it until there is a `try` or a throwing call.

## Flags are not separate opcodes

`add`, `add nsw`, `add nuw`, and `add nuw nsw` are one opcode with bits on the instruction (`hasNoSignedWrap()`). The printer shows the bits as keywords. When you diff IR, a vanished `nsw` is a real change even though the opcode is still `add`.

## Check yourself

1. Where do you look up what `ashr` means, LangRef or `Instruction.def`?
2. Is `store` a terminator?

**Answers.** (1) LangRef. The `.def` file only assigns it an enum. (2) No. A block that stores still needs a `br` or a `ret` after the store.
