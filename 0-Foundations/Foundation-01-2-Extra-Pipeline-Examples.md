# Foundation 1 extra: Pipeline examples

Companion to [Foundation 1](Foundation-01-0-The-Compiler-Pipeline.md). Deep dive: [monorepo map](Foundation-01-1-Extra-Monorepo-DeepDive.md).

## Diagram

```text
  add.c  --clang -emit-llvm-->  add.ll  --opt-->  add.opt.ll  --llc-->  add.s
           still has the          maybe a
           add instruction        folded constant
```

## Example 1: Stop after IR

```c
int add(int a, int b) { return a + b; }
```

```bash
clang -S -emit-llvm -O1 -Xclang -disable-llvm-passes add.c -o add.ll
```

**What you should see.** A `define` of `@add` with an `add` instruction and a `ret`. Arguments may carry attributes such as `noundef` (Foundation 8). The body should still be the add, not a constant.

**Why.** `-emit-llvm` selects the IR writer. `-disable-llvm-passes` keeps Clang from optimizing before you look.

## Example 2: One pass

```bash
opt -passes=instcombine -S add.ll -o add.opt.ll
```

**What you should see.** For this function, InstCombine often changes nothing: an add of two arguments is already simple. The point is that the output is still IR.

**Counterexample.** `clang -S add.c -o add.s` never produces a `define` line. That file is assembly. `opt` will not accept it.

## Example 3: IR to assembly

```bash
llc add.ll -o add.s
```

**What you should see.** A function label, a machine add, and a return. Register names are physical (`%edi`, `w0`, …) and depend on your machine. There is no `%s = add i32`.

**Why.** This is the boundary the rest of the early days stay above. If a file contains `$` registers or `%eax`, you have left IR.

## Example 4: The whole chain in one command, then the same chain split

```bash
clang -O2 -S add.c -o add.O2.s
```

**What you should see.** Assembly, with no IR left to read. The same source, split into the three tools, lets you see the IR that `-O2` would have optimized. Use the split form whenever a later day asks "what did the pass do?"

## Example 5: A file the tools reject

```text
this is not ir
```

```bash
opt -S not-ir.ll
```

**What you should see.** A parse error from `opt`, not a miscompilation. The verifier and the parser run before any pass. Foundation 2 shows a function that parses and one that does not.
