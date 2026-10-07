# Foundation 7 extra: The Module object

Companion to [Foundation 7](Foundation-07-0-Modules-and-Globals.md). Examples: [module examples](Foundation-07-2-Extra-Module-Examples.md).

## Diagram

```text
  Module
    DataLayout
    Triple
    Function list          define and declare are both Functions
    GlobalVariable list    @counter, @msg
    named StructTypes
```

The class is [llvm/include/llvm/IR/Module.h](https://github.com/llvm/llvm-project/blob/main/llvm/include/llvm/IR/Module.h). A function is a [Function](https://github.com/llvm/llvm-project/blob/main/llvm/include/llvm/IR/Function.h). A global variable is a [GlobalVariable](https://github.com/llvm/llvm-project/blob/main/llvm/include/llvm/IR/GlobalVariable.h). Both are `GlobalValue`s: they have linkage, and they have `@` names.

`Function::isDeclaration()` is true for a `declare` line. Passes that walk function bodies skip declarations.

## Linkage, the short list

You will see these words in the printer. Day 49 uses them.

| Word | Meaning for a first reading |
| --- | --- |
| `external` | Visible to other modules. The default. |
| `internal` | This module only, like C `static`. |
| `private` | This module only, and the linker will not keep the name. |
| `linkonce_odr` | Emitted in many modules; the linker keeps one copy. C++ inline functions and templates. |
| `weak` | Similar merging, with different overriding rules. |

A pass may delete an `internal` function that nothing calls (Day 46). It must not delete an `external` function, because a call might arrive from outside the file.

## `unnamed_addr` and `local_unnamed_addr`

These are flags on a global, not types. They tell the optimizer that the address is insignificant. Merging two identical constant arrays into one is legal when both have `unnamed_addr`. It is not legal for a global whose address a program compares with `==`.

## Check yourself

1. How does the C++ API distinguish `declare i32 @f(i32)` from `define i32 @f(i32) { ... }`?
2. Why is an `external` function a root for dead-code elimination?

**Answers.** (1) `isDeclaration()` is true only for the `declare`. (2) Another module may call it. There is no caller to find inside this file, and that absence is not proof the function is unused.
