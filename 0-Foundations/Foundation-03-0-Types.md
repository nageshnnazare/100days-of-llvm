# Foundation 3: Types

Every operand in LLVM IR has a type, written in the instruction. There is no separate symbol table to consult. This note is the set of types you will see in almost every file. Layout in memory (padding, endianness) is Day 102. Poison, which is a property of a value and not a type, is Day 101.

Deep dive: [how types are represented](Foundation-03-1-Extra-Types-DeepDive.md). Examples: [type examples](Foundation-03-2-Extra-Types-Examples.md). Specification: [LangRef types](https://llvm.org/docs/LangRef.html#type-system).

## Diagram

```text
  first-class types you can compute with
  +-------------------+------------------+------------------+
  | integers          | floating point   | pointer          |
  | i1  i8  i32  i64  | half float double| ptr              |
  +-------------------+------------------+------------------+
  | aggregates                         | vectors            |
  | {i32, i8}   [4 x i32]              | <4 x i32>          |
  +------------------------------------+--------------------+

  not values in the same way
  +-------------------+------------------+
  | void              | label            |
  | the type of a     | the type of a    |
  | function that     | basic-block      |
  | returns nothing   | name in br / phi |
  +-------------------+------------------+
```

## Integers

An integer type is `i` plus a bit width: `i1`, `i8`, `i16`, `i32`, `i64`, and other widths the target may not support natively (`i24` exists in IR; Day 62 is how code generation legalizes it).

`i1` is the type of a condition. It is not an `i8` that happens to be 0 or 1. `br` takes an `i1`. A C `_Bool` or a comparison becomes `i1`. A C `int` is `i32` on the usual ABIs.

There is no `bool` keyword and no `int` keyword. Width is always explicit.

Signedness is not part of the type. `i32` is a bag of 32 bits. The instruction chooses the interpretation: `sdiv` is signed, `udiv` is unsigned, `icmp slt` is signed less-than, `icmp ult` is unsigned less-than.

## Floating point

`half`, `float`, `double`, and a few more (`fp128`, `bfloat`) are the scalar floating types. `float` is IEEE binary32. Arithmetic is `fadd`, `fmul`, not `add`, `mul`.

## Pointers

A pointer is `ptr`. It points at an address. The type does not say what it points at.

```llvm
%p = alloca i32
%v = load i32, ptr %p
```

The `alloca` result is `ptr`. The `load` says "read an `i32` from that address". The loaded type is on the `load`, not on the pointer.

Older IR wrote `i32*`. LLVM 15 and later use opaque pointers. These notes always use `ptr`. If a blog shows `i32*`, translate the star away and put the element type on the `load`, `store`, or `getelementptr`.

## Aggregates

An array type is `[N x Element]`: `[4 x i32]` is four `i32`s in a row, in the type system.

A struct type is `{i32, i8}` or a named form `%struct.S = type { i32, i8 }`. Named struct types exist so a struct can refer to itself (a linked-list node).

An array or struct in a register is rare. In real Clang output they usually live in memory via `alloca`, and SROA (Day 6) splits them.

## Vectors

`<4 x i32>` is four `i32` lanes in one value. A vector is a value the vectorizer (Days 51–58) and the backend want in a vector register. An array `[4 x i32]` is a memory aggregate. They are not interchangeable. `add <4 x i32> %a, %b` adds four lanes. `add [4 x i32]` is not an instruction.

## `void`, functions, and labels

```llvm
define void @nop() {
  ret void
}

define i32 @id(i32 %x) {
  ret i32 %x
}
```

`void` is a return type. You cannot have `%t = add void ...`.

A function type is written in the `define` or `declare` line: arguments and a return type. Function types are not passed around as ordinary values; a function pointer has type `ptr`.

`label` appears in branch and phi operands (`label %entry`). You cannot `add` a label.

## Hands-on

In a Clang-generated `.ll` file, highlight every type token (`i32`, `ptr`, `i1`). For each `load`, say which token is the loaded type and which is the address type. They are different.

## Pitfalls

- Treating `i1` and `i8` as the same. Zero-extending a condition to a C `int` is a `zext i1 ... to i32`.
- Assuming `ptr` remembers the element type. After opaque pointers, `getelementptr` takes the element type as an explicit argument (Foundation 4, Day 102).
- Assuming a struct's IR type includes padding. `{i8, i32}` may occupy more than 5 bytes in memory. The type lists fields; DataLayout assigns offsets.

## Check yourself

1. What is the type of a Clang `int` comparison used as an `if` condition, once it reaches `br`?
2. Why is signedness not written on `i32`?
3. Which of `[4 x i32]` and `<4 x i32>` can be the type of an `add`?

**Answers.** (1) `i1`. (2) The same bits are added either way; signed versus unsigned is the opcode (`sdiv` versus `udiv`, `slt` versus `ult`). (3) `<4 x i32>`.
