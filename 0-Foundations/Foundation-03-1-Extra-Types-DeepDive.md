# Foundation 3 extra: Where types live in the C++ API

Companion to [Foundation 3](Foundation-03-0-Types.md). Examples: [type examples](Foundation-03-2-Extra-Types-Examples.md).

## Diagram

```text
  Type
    IntegerType          i32          getIntegerBitWidth() == 32
    PointerType          ptr          one opaque pointer type per address space
    ArrayType            [4 x i32]
    StructType           {i32, i8}    may be literal or named
    FixedVectorType      <4 x i32>
    FunctionType         i32 (i32, i32)
```

The header is [llvm/include/llvm/IR/Type.h](https://github.com/llvm/llvm-project/blob/main/llvm/include/llvm/IR/Type.h), with the aggregates in [DerivedTypes.h](https://github.com/llvm/llvm-project/blob/main/llvm/include/llvm/IR/DerivedTypes.h). `Type::isIntegerTy(32)` and `dyn_cast<IntegerType>` are the checks passes write. You do not need them to read IR. You need them when a later day's code does `if (I->getType()->isIntegerTy(1))`.

## Address spaces

`ptr` is address space 0, the default. A pointer in another space is `ptr addrspace(1)` (GPUs, some microcontroller ABIs). Unless a lesson names an address space, it means the default. An integer cast does not change address space; `addrspacecast` does.

## `void` is not zero bits you can store

`ret void` is the instruction for "return no value". There is no value of type `void` to pass to `add`. A function `define void @f()` still has a function type; the return type of that function type is `void`.

## How to see a type in the text

`opt -S` prints the type on every instruction that needs it. There is no separate "types section" except for named structs, which are printed at the top of the module:

```llvm
%struct.Point = type { i32, i32 }
```

That line introduces a name. It does not allocate a global.

## Check yourself

1. Which C++ class in the headers above represents `i32`?
2. Does `ptr addrspace(1)` match `ptr` in the type checker?

**Answers.** (1) `IntegerType`. (2) No. Address spaces are part of the pointer type.
