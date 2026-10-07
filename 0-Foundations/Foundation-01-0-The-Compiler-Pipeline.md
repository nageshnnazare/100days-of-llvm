# Foundation 1: What LLVM is

Read this before Day 0. Day 0 explains how passes are scheduled. This note explains what those passes are scheduled *on*, and which program is responsible for each stage.

LLVM is a set of libraries and tools for compiling and analyzing programs. It is not a programming language. Clang is a C, C++, and Objective-C frontend that produces LLVM IR. `opt` transforms that IR. `llc` turns IR into assembly or an object file. A linker (`ld.lld`, or the system linker) produces the executable.

Deep dive: [monorepo map](Foundation-01-1-Extra-Monorepo-DeepDive.md). Examples: [pipeline examples](Foundation-01-2-Extra-Pipeline-Examples.md).

## Diagram

```text
  hello.c                         you write this
     |
     |  clang  (parse, type-check, AST, CodeGen)
     v
  hello.ll                        LLVM IR, the text this course reads
     |
     |  opt    (InstCombine, SROA, LICM, ...)
     v
  hello.opt.ll                    still IR, usually smaller
     |
     |  llc    (instruction selection, registers)
     v
  hello.s  or  hello.o            assembly, or a relocatable object
     |
     |  linker
     v
  hello                           a program the OS can run
```

Most days in this repo sit on the middle arrow: a pass that reads IR and writes IR. Days 61–80 sit on `llc`. Day 92 is the Clang arrow. Day 116 is the linker when the object files are still bitcode.

## The idea

A compiler is a pipeline of representations. Each representation is easier for one kind of work:

| Representation | Good at | Bad at |
| --- | --- | --- |
| C source | Humans, types, names | Rewriting a loop without breaking syntax |
| Clang AST | The language's rules | Target instructions |
| LLVM IR | Target-independent optimization | C++ templates, source layout |
| Machine IR | Registers and opcodes | "What did the C look like?" |
| Object file | Linking, relocations | Algebraic simplification |

LLVM IR is the contract between frontends and the optimizer. Clang, Rust's `rustc`, Swift, and Julia all emit it. A pass written against IR does not know which frontend produced the file.

## Three programs, three jobs

```bash
clang -S -emit-llvm -O1 -Xclang -disable-llvm-passes hello.c -o hello.ll
opt -passes=instcombine -S hello.ll -o hello.opt.ll
llc hello.opt.ll -o hello.s
```

- `clang -S -emit-llvm` stops after IR. `-S` means "write text", not "write a binary".
- `opt -S` reads IR and prints IR. `-passes=` names the transforms.
- `llc` reads IR and prints assembly. Add `-filetype=obj` for an object file.

`clang -O2 hello.c -o hello` runs all of those stages inside one process. You cannot see the IR unless you ask.

## What this course will not start from

Day 0's pass manager, Day 3's C++ class hierarchy (`Module`, `Function`, `Instruction`), and Day 5's phi-placement algorithm all assume you can read a function like this:

```llvm
define i32 @add(i32 %a, i32 %b) {
  %s = add i32 %a, %b
  ret i32 %s
}
```

Foundations 2 through 8 are that reading skill. After them, Day 0 is the machinery that runs a pass, and Day 3 is the same IR as a C++ object graph.

## Where the source lives

The project is one git repository, [llvm/llvm-project](https://github.com/llvm/llvm-project). The pieces you will open:

| Directory | What it is |
| --- | --- |
| `llvm/` | The IR, the optimizers, the code generator, `opt`, `llc` |
| `clang/` | The C/C++ frontend |
| `lld/` | The linker |
| `mlir/` | A sibling IR, Day 99 |
| `compiler-rt/` | Runtime helpers the compiler calls (`memcpy` on some targets, sanitizer runtimes) |

Inside `llvm/`, optimizers are under `llvm/lib/Transforms/`, the IR classes under `llvm/include/llvm/IR/`, and the language definition in [llvm/docs/LangRef.rst](https://github.com/llvm/llvm-project/blob/main/llvm/docs/LangRef.rst).

## Hands-on

Save this as `add.c`:

```c
int add(int a, int b) { return a + b; }
```

Run the three commands above. Open `add.ll` and find `add` and `ret`. Open `add.s` and find a machine add (`add` on x86, `add` on AArch64; the register names differ). You should be able to point at the IR line and the assembly line that implement the same C statement.

## Pitfalls

- `clang -S add.c -o add.s` writes assembly, not IR. IR requires `-emit-llvm`.
- `clang -O0` marks functions `optnone`. Later days use the command in [Setup and toolchain](Setup-and-Toolchain.md) so `opt` is allowed to change the function.
- IR is not assembly. `%a` is not a machine register. Register allocation is `llc`'s job (Day 71).

## Check yourself

1. Which program turns a `.ll` file into a `.s` file?
2. Why can the same `opt` pass run on IR that came from C and IR that came from Rust?
3. Where in the pipeline does Day 5 (mem2reg) sit: before IR exists, on IR, or after assembly?

**Answers.** (1) `llc`. (2) Both are LLVM IR; the pass never sees the original language. (3) On IR, after Clang has emitted allocas and before code generation.
