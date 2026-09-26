# Glossary of Core LLVM Optimization Passes

This glossary provides a detailed reference for the most common optimization passes you will see in LLVM's `-O2`/`-O3` pipelines, including comprehensive examples of how they transform IR.

---

## 1. SROA (Scalar Replacement of Aggregates)
**Full Name:** `SROAPass`
**What it does:** Breaks down aggregate types (like structs or arrays) allocated on the stack (via `alloca`) into individual scalar variables if accessed directly. This is crucial for allowing other passes to put those variables into registers.

**Example:**
*Before SROA (C-level conceptual):*
```c
struct Point { int x; int y; };
void foo() {
    struct Point p;  // alloca'd on the stack
    p.x = 10;
    p.y = 20;
    return p.x + p.y;
}
```
*After SROA (C-level conceptual):*
```c
void foo() {
    int p_x = 10;    // Now just a scalar integer, can live in a register
    int p_y = 20;    // Now just a scalar integer
    return p_x + p_y;
}
```

---

## 2. Mem2Reg (Promote Memory to Register)
**Full Name:** `PromotePass`
**What it does:** Converts `alloca` instructions (stack memory) into SSA registers (`%var`) by inserting Phi nodes. This pass requires a Dominator Tree to place Phi nodes at join points in the control flow graph. It is usually run immediately after SROA to lift the broken-down scalars into registers.

**Example:**
*Before Mem2Reg (Unoptimized IR):*
```llvm
entry:
  %x = alloca i32, align 4
  store i32 5, ptr %x, align 4
  %val = load i32, ptr %x, align 4
  ret i32 %val
```
*After Mem2Reg:*
```llvm
entry:
  ; The alloca, load, and store are completely removed
  ret i32 5
```

---

## 3. EarlyCSE (Early Common Subexpression Elimination)
**Full Name:** `EarlyCSEPass`
**What it does:** A fast, hash-based pass that searches for redundant computations (expressions calculated multiple times with the exact same inputs) and eliminates all but the first one, replacing later uses with the first computed value.

**Example:**
*Before EarlyCSE:*
```llvm
  %a = add i32 %x, %y
  %b = add i32 %x, %y  ; Exact same computation!
  %c = mul i32 %a, %b
```
*After EarlyCSE:*
```llvm
  %a = add i32 %x, %y
  ; %b is deleted
  %c = mul i32 %a, %a  ; Reuses %a
```

---

## 4. InstCombine (Instruction Combining)
**Full Name:** `InstCombinePass`
**What it does:** A massive peephole optimizer that pattern-matches combinations of instructions and replaces them with simpler, more efficient instructions (e.g., algebraic simplifications, bitwise tricks, canonicalizing IR).

**Example:**
*Before InstCombine:*
```llvm
  %a = add i32 %x, 0      ; x + 0 is just x
  %b = mul i32 %a, 2      ; a * 2 is a shift left by 1
```
*After InstCombine:*
```llvm
  %b = shl i32 %x, 1
```

---

## 5. SimplifyCFG (Simplify Control Flow Graph)
**Full Name:** `SimplifyCFGPass`
**What it does:** Cleans up the flow of execution. It merges basic blocks that have unconditional branches to each other, removes unreachable blocks, and converts simple conditional branches into `select` instructions to avoid branch prediction penalties.

**Example:**
*Before SimplifyCFG:*
```llvm
entry:
  br label %next
next:
  %x = add i32 %a, %b
  ret i32 %x
```
*After SimplifyCFG:*
```llvm
entry:
  %x = add i32 %a, %b
  ret i32 %x
```

---

## 6. Jump Threading
**Full Name:** `JumpThreadingPass`
**What it does:** If a block branches to another block based on a condition, and that condition was already proven true or false on a specific incoming path, Jump Threading redirects the incoming edge directly to the correct destination, bypassing the redundant check.

**Example:**
*Before Jump Threading (C-level conceptual):*
```c
  if (x == 5) {
      y = 10;
  }
  // Jump Threading realizes that if the first block ran, x IS 5.
  if (x == 5) {
      z = 20;
  }
```
*After Jump Threading:*
The compiler threads the execution path so that if the first block runs, it jumps directly to `z = 20` without evaluating `x == 5` a second time.

---

## 7. LICM (Loop Invariant Code Motion)
**Full Name:** `LICMPass`
**What it does:** Identifies computations inside a loop that produce the exact same result on every iteration. It "hoists" these computations out into the loop's preheader so they are executed only once.

**Example:**
*Before LICM:*
```c
for (int i=0; i<100; i++) {
    int max = x + y; // x and y do not change in this loop!
    arr[i] = max;
}
```
*After LICM:*
```c
int max = x + y;     // Hoisted out of the loop
for (int i=0; i<100; i++) {
    arr[i] = max;
}
```

---

## 8. Loop Unrolling
**Full Name:** `LoopUnrollPass`
**What it does:** Duplicates the body of a loop multiple times to reduce the overhead of the branch/counter instructions and to expose instructions across iterations to passes like InstCombine and SLP Vectorization.

**Example:**
*Before Loop Unrolling:*
```c
for(int i=0; i<4; i++) {
    a[i] = 0;
}
```
*After Loop Unrolling (Fully Unrolled):*
```c
// The loop structure is completely removed
a[0] = 0;
a[1] = 0;
a[2] = 0;
a[3] = 0;
```

---

## 9. Loop Deletion
**Full Name:** `LoopDeletionPass`
**What it does:** Completely removes loops if it can prove that they are finite and do not have any side effects (like writing to memory outside the loop or doing I/O operations).

**Example:**
*Before Loop Deletion:*
```c
for (int i=0; i<1000; i++) {
    int x = i * 2;
    // x is never used outside, no memory is written
}
```
*After Loop Deletion:*
The entire loop is deleted by the compiler, leaving nothing.

---

## 10. Inliner
**Full Name:** `InlinerPass`
**What it does:** Replaces a function call with the actual body of the called function. This eliminates function call overhead (saving registers, pushing stack frames) and exposes the function's internal logic to the caller's optimization passes.

**Example:**
*Before Inlining:*
```c
int add(int a, int b) { return a + b; }

int main() {
    return add(5, 5);
}
```
*After Inlining:*
```c
int main() {
    return 5 + 5; // Will be immediately constant folded to 10
}
```

---

## 11. GVN (Global Value Numbering)
**Full Name:** `GVNPass`
**What it does:** Assigns a unique "value number" to expressions. If two instructions compute the same value number, the second is deleted. It handles redundant memory loads heavily using Memory Dependence Analysis, which EarlyCSE cannot do.

**Example:**
*Before GVN:*
```llvm
  %v1 = load i32, ptr %p
  ; ... some complex code that doesn't write to %p ...
  %v2 = load i32, ptr %p  ; Redundant load!
  %sum = add i32 %v1, %v2
```
*After GVN:*
```llvm
  %v1 = load i32, ptr %p
  ; ... some complex code ...
  %sum = add i32 %v1, %v1 ; Second load is eliminated
```

---

## 12. SCCP (Sparse Conditional Constant Propagation)
**Full Name:** `SCCPPass`
**What it does:** Propagates constants through the program. If it proves a branch condition is always false based on constant propagation, it ignores the dead block completely, never propagating values through it.

**Example:**
*Before SCCP:*
```c
int x = 5;
if (x > 10) {
    // SCCP proves 5 > 10 is false.
    complex_math(); 
}
return x;
```
*After SCCP:*
```c
return 5; // The if-statement and complex math are completely deleted
```

---

## 13. Correlated Value Propagation (CVP)
**Full Name:** `CorrelatedValuePropagationPass`
**What it does:** Uses Lazy Value Info (LVI) to track ranges of variables based on previous branch conditions. It can eliminate switch cases, bounds checks, or redundant conditional branches.

**Example:**
*Before CVP:*
```c
if (x > 100) {
    if (x > 0) { // This must be true if x > 100!
        do_something();
    }
}
```
*After CVP:*
```c
if (x > 100) {
    do_something(); // The redundant check is eliminated
}
```

---

## 14. ADCE (Aggressive Dead Code Elimination)
**Full Name:** `ADCEPass`
**What it does:** Removes instructions that do not contribute to the final observable output of the function. Works backwards from returns/stores, marking "live" instructions. Deletes everything unmarked.

**Example:**
*Before ADCE:*
```llvm
  %dead = mul i32 %x, 500  ; Result is never used!
  %live = add i32 %x, 1
  ret i32 %live
```
*After ADCE:*
```llvm
  %live = add i32 %x, 1
  ret i32 %live
```

---

## 15. GlobalOpt (Global Optimizer)
**Full Name:** `GlobalOptPass`
**What it does:** Optimizes global variables. If a global is only ever written to once, it makes it a constant. If a global is never used, it deletes it. It can also localize globals to functions if they aren't accessed externally.

**Example:**
*Before GlobalOpt:*
```c
int G = 0; // Global variable
void init() { G = 5; }
int read() { return G; }
```
*After GlobalOpt:*
The compiler may realize `G` is only ever 5 after initialization, turning it into a constant and just returning `5` from `read()`.

---

## 16. Loop Vectorizer
**Full Name:** `LoopVectorizePass`
**What it does:** Analyzes loops for data dependencies. If mathematically safe, it converts scalar instructions into wide vector instructions (SIMD like AVX or NEON) to process multiple loop iterations at the exact same time.

**Example:**
*Before Loop Vectorizer:*
```c
for(int i=0; i<4; i++) {
    a[i] = b[i] + c[i];
}
```
*After Loop Vectorizer (Conceptual):*
```llvm
  ; Loads 4 elements of b and c at once into a 128-bit vector register
  %vec_b = load <4 x i32>, ptr %b
  %vec_c = load <4 x i32>, ptr %c
  ; Does a single vector addition!
  %vec_sum = add <4 x i32> %vec_b, %vec_c
  ; Stores 4 elements to a
  store <4 x i32> %vec_sum, ptr %a
```

---

## 17. SLP Vectorizer (Superword-Level Parallelism)
**Full Name:** `SLPVectorizerPass`
**What it does:** Looks for straight-line scalar code that performs identical operations on adjacent memory locations, and bundles them into a single vector operation.

**Example:**
*Before SLP Vectorizer:*
```c
a.x = b.x + c.x;
a.y = b.y + c.y;
```
*After SLP Vectorizer (Conceptual):*
The compiler bundles `x` and `y` into a vector `[x, y]` and performs a single vector addition:
```llvm
  %vec_b = load <2 x i32> ...
  %vec_c = load <2 x i32> ...
  %vec_a = add <2 x i32> %vec_b, %vec_c
```
