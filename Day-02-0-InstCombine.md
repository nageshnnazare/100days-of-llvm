# Day 2: Instruction Combining (InstCombine)

## 1. What is Instruction Combining?
Instruction Combining (commonly called **InstCombine**) is a fundamental peephole optimization pass in LLVM. Its goal is to simplify instructions and transform sequences of instructions into simpler, more efficient forms. 

While *Constant Folding* (Day 1) focuses on evaluating expressions with known constants at compile time, InstCombine focuses on **algebraic identities**, **redundancy elimination**, and **canonicalization**.

For example, InstCombine knows that:
* `x + 0` can be simplified to just `x`.
* `x * 2` can be transformed into a faster shift operation `x << 1`.
* `(x & 255) & 255` is redundant and can be reduced to `x & 255`.

## 2. Where to Read the Code
The InstCombine pass is massive, so its implementation is split across several files based on the types of instructions being optimized.

### The Core Infrastructure
* **Header file:** `llvm-project/llvm/include/llvm/Transforms/InstCombine/InstCombineWorklist.h` and `InstCombine.h`
* **Source file:** `llvm-project/llvm/lib/Transforms/InstCombine/InstructionCombining.cpp`

### Instruction-Specific Folders
To keep things organized, LLVM splits the simplification logic into separate files:
* `llvm-project/llvm/lib/Transforms/InstCombine/InstCombineAddSub.cpp` (for `add`, `sub`, `fadd`, `fsub`)
* `llvm-project/llvm/lib/Transforms/InstCombine/InstCombineMulDivRem.cpp` (for `mul`, `div`, `rem`)
* `llvm-project/llvm/lib/Transforms/InstCombine/InstCombineCalls.cpp` (for simplifying function calls and intrinsics)

## 3. Understanding the Infrastructure

To understand how InstCombine works, you need to be familiar with its core mechanics:

### A. The `InstCombineWorklist`
InstCombine operates using a worklist algorithm. It doesn't just scan the function once. 
1. Initially, all instructions in the function are added to the worklist.
2. It pops an instruction off the worklist and tries to combine/simplify it.
3. If the instruction is successfully simplified, **the instruction itself and all of its users** are added *back* onto the worklist.
This allows cascading optimizations, where one simplification unlocks another.

### B. LLVM Pattern Matching (`PatternMatch.h`)
Writing C++ code to check if an instruction is an `add`, and if its second operand is a `ConstantInt` of `0`, can be very verbose. LLVM provides a powerful pattern-matching DSL (Domain Specific Language) in C++ to make this elegant.
Using `llvm::PatternMatch`, you can write code like:
```cpp
using namespace llvm::PatternMatch;
Value *X;
if (match(I, m_Add(m_Value(X), m_Zero()))) {
  // It's an addition of X and 0! We can replace I with X.
}
```
This is used extensively throughout all of the InstCombine source files.

### C. Canonicalization
InstCombine also acts as a "canonicalizer". If there are multiple ways to write the same IR, InstCombine will always transform it into one standard (canonical) form. For example, it ensures constants are always on the right-hand side of commutative operators (like `add`). This makes life much easier for subsequent optimization passes, as they only need to look for one specific pattern instead of every possible variation.

## 4. What to Read Today
1. **Open `llvm-project/llvm/include/llvm/IR/PatternMatch.h`**: Skim through this file to see how matchers like `m_Add`, `m_Zero`, and `m_One` are defined. It's a great example of C++ template metaprogramming.
2. **Open `llvm-project/llvm/lib/Transforms/InstCombine/InstCombineAddSub.cpp`**: Look for the `InstCombinerImpl::visitAdd` function. See if you can spot some of the basic algebraic simplifications using `match()`.

## 5. Experiment / Hands-on
Try writing a C program with some obvious algebraic redundancies:
```c
int test(int x) {
    int a = x + 0;
    int b = a * 1;
    int c = b - b; // Should be 0
    return c;
}
```

First, compile it to LLVM IR without optimizations to see the raw instructions:
`clang -O0 -S -emit-llvm main.c -o main.ll`

Then, run *only* the InstCombine pass using the `opt` tool:
`opt -passes=instcombine main.ll -S -o main.opt.ll`

Compare `main.ll` and `main.opt.ll`. You should see that InstCombine completely collapses the math into simply returning `0`.
