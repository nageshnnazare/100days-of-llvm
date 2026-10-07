# Foundation 1 extra: The monorepo, one directory at a time

Companion to [Foundation 1](Foundation-01-0-The-Compiler-Pipeline.md). Examples: [pipeline examples](Foundation-01-2-Extra-Pipeline-Examples.md).

## Diagram

```text
  llvm-project/
    clang/          C, C++, Objective-C  ->  IR
    llvm/
      include/llvm/IR/     C++ classes: Module, Function, Type
      lib/IR/              implementations
      lib/Transforms/      opt passes (Scalar, IPO, Vectorize, ...)
      lib/CodeGen/         llc's machine passes
      lib/Target/X86/      one backend
      lib/Target/AArch64/
      tools/opt/           the opt executable
      tools/llc/
      docs/LangRef.rst     the IR specification
    lld/            linker
    mlir/           another IR, lowered toward LLVM IR
    compiler-rt/    runtime library linked into your program
```

When a lesson says "open `llvm/lib/Transforms/Scalar/SROA.cpp`", the path is inside this tree on [GitHub](https://github.com/llvm/llvm-project/blob/main/llvm/lib/Transforms/Scalar/SROA.cpp). The notes link to `main`. They do not pin line numbers, because lines move.

## What is a library versus a tool

`opt` is a thin `main()`. The pass manager, the IR, and InstCombine are libraries in `llvm/lib`. Clang links the same libraries. That is why a transform you can run with `opt -passes=instcombine` also runs inside `clang -O2`.

You can read a pass without building LLVM. You need a local build when you want to change it (Day 93, Day 120).

## The documents that define behavior

| Document | Use it for |
| --- | --- |
| [LangRef](https://llvm.org/docs/LangRef.html) | What an instruction means, including poison and atomics |
| [Programmer's Manual](https://llvm.org/docs/ProgrammersManual.html) | How the C++ classes are meant to be used |
| [Getting Started](https://llvm.org/docs/GettingStarted.html) | Building from source |
| [Code Generator](https://llvm.org/docs/CodeGenerator.html) | The `llc` pipeline, after you can read IR |

When a lesson and your memory disagree, LangRef wins for IR behavior. The `.cpp` file wins for what the pass actually does today.

## How a frontend fits

Clang's last step that this course cares about is CodeGen: AST to IR, in `clang/lib/CodeGen/`. Day 92 walks that. Foundation 6 only shows the IR Clang emits, so you can recognize it.

Other frontends are not in this repository. Their IR still has to follow LangRef, or `opt` will miscompile them.

## Targets are optional at the IR stage

A `.ll` file may name a triple:

```llvm
target triple = "x86_64-unknown-linux-gnu"
```

IR passes are mostly target-independent. Some of them ask the target questions through TargetTransformInfo (Day 40): "how wide is a vector register?" Code generation cannot proceed without a triple. `llc` defaults to the host if the file does not name one.

## Check yourself

1. Is InstCombine part of Clang, or part of LLVM that Clang calls?
2. Which file is the specification of `add`, the C++ source or LangRef?

**Answers.** (1) It lives in `llvm/lib/Transforms/InstCombine/` and Clang calls it. (2) LangRef. The C++ is an implementation of that specification.
