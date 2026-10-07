# Foundation 7: Modules, globals, and the header

A `.ll` file is one module. The module has a header (the triple and the data layout), then named types, global variables, and functions. Functions are the part Foundations 2–6 were about. This note is everything in the file above the first `define`.

Day 49 is linkage as an optimization (internalize, and what it unlocks). Day 102 is how to decode a data-layout string. Here you only need to know what each line is for.

Deep dive: [the Module class](Foundation-07-1-Extra-Module-DeepDive.md). Examples: [module examples](Foundation-07-2-Extra-Module-Examples.md).

## Diagram

```text
  ; header
  target datalayout = "e-m:e-p:64:64-..."
  target triple = "x86_64-unknown-linux-gnu"

  ; named types
  %struct.S = type { i32, i32 }

  ; globals and declarations
  @counter = global i32 0
  @msg = private constant [6 x i8] c"hello\00"
  declare i32 @puts(ptr)

  ; functions
  define i32 @main() { ... }
```

Order in the file is a printing choice. The module is a set of globals plus a list of functions. A function may refer to a global that is printed later.

## `target triple` and `target datalayout`

The triple names the architecture, vendor, operating system, and environment: `x86_64-apple-darwin`, `aarch64-unknown-linux-gnu`. It chooses the backend when you run `llc`.

The datalayout string tells IR passes the size of a pointer, the alignment of types, and endianness, without loading the whole backend. `e` near the start means little-endian. `p:64:64` means a pointer is 64 bits. You do not need to parse the rest until Day 102. If the string is missing, passes assume a default that may not match your target, and address arithmetic can be wrong.

## `define` and `declare`

`define i32 @f(...) { ... }` provides a body.

`declare i32 @f(i32)` promises a signature and no body. Calls are allowed. The linker must find a definition in some other object file. Inlining across that boundary requires LTO (Days 81–83) or the body being present.

A mismatch between a `declare` and the real definition (different argument types) is undefined at link time. The IR verifier only checks this file.

## Globals

```llvm
@counter = global i32 0
@limit = constant i32 4
@msg = private constant [6 x i8] c"hello\00"
```

`@counter` is a pointer to a mutable `i32`, from the point of view of `load` and `store`: the name `@counter` has type `ptr` in opaque-pointer IR. The type of the memory is written on the global (`i32`).

`constant` means stores to that memory are not allowed. Passes may fold a load of `@limit` to the constant `4`.

`private` means the linker cannot name this symbol from another module. A private constant string can be merged or deleted if this module stops using it. `external` (the default when you write nothing but `global`) means other modules may refer to it.

Initializers are constants: `0`, `c"hello\00"`, `null`, or a compound `{ i32 1, i32 2 }`.

## Strings

```llvm
@msg = private unnamed_addr constant [6 x i8] c"hello\00"
```

The array length includes the NUL. `unnamed_addr` means the address itself is unimportant; only the bytes matter, so the linker may merge identical constants. A pointer comparison against `@msg` is not something your program should depend on when this flag is present.

A call `call i32 @puts(ptr @msg)` passes the address of the first byte.

## Hands-on

Write a module with one `constant` global and a function that returns it via `load`. Run `opt -passes=instcombine -S`. The load often disappears and the function returns the constant directly. Change `constant` to a plain `global` and try again. The load should stay, because some other module might store to an external global.

## Pitfalls

- Thinking `@counter` has type `i32`. In expressions it is a `ptr`. The `i32` is what a load of it produces.
- Defining the same `@name` twice in one module. The parser rejects it.
- Removing `target datalayout` to "simplify" a test. Size-sensitive passes then use the wrong pointer width.

## Check yourself

1. What is the type of the name `@counter` in a `load`'s address operand?
2. What does `declare` promise that `define` does not need to?
3. Why can a pass fold a load of a `constant` global but not a load of an external `global`?

**Answers.** (1) `ptr`. (2) A body. `declare` is a signature for a function defined elsewhere. (3) A `constant` global cannot be stored to. An external global can be stored by another module, so the load's result is not known.
