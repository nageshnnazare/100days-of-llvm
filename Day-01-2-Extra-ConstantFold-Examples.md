# Day 1 Extra Notes: Constant Folding Examples

This file provides additional practical C examples to demonstrate how Constant Folding and early expression evaluation work in LLVM.

## Example 1: Floating Point Folding
LLVM handles floating-point constants just as easily as integers, provided the math doesn't rely on runtime environment specifics (like dynamic rounding modes).

```c
double test_fp() {
    double pi = 3.14159;
    double radius = 5.0;
    // The area calculation is fully constant.
    return pi * radius * radius;
}
```

*   **How to view raw IR:** `clang -O0 -S -emit-llvm fp.c`
    (You will see `fmul` instructions in the output).
*   **How to view optimized IR:** `clang -O1 -S -emit-llvm fp.c`
    (The math collapses into a single `ret double 7.853975e+01`).

## Example 2: Control Flow Elimination via Constants
Constant folding is often the prerequisite for Dead Code Elimination. If a branch condition is folded to a constant, the branch can be removed entirely.

```c
int test_branch() {
    int debug_mode = 0;
    
    if (debug_mode == 1) {
        return 404; // This is dead code!
    }
    
    return 200;
}
```

*   **Behind the scenes:** 
    1. The `IRBuilder` folds `0 == 1` into the constant `false` (`i1 0`).
    2. The `SimplifyCFG` pass sees the branch condition is unconditionally `false`.
    3. It removes the true block and the branch instruction, leaving just the `ret 200`.

## Example 3: Pointer Casting and GEP
LLVM uses `getelementptr` (GEP) to calculate memory addresses. GEPs on constant structures can often be folded if the indices are constant.

```c
struct Point {
    int x;
    int y;
};

int test_struct() {
    struct Point p;
    p.x = 10;
    p.y = 20;
    return p.x + p.y;
}
```
*   **Optimization:** The `SROA` (Scalar Replacement of Aggregates) and Constant Folding passes work together here. They map the memory locations to registers, replace the loads with the constants `10` and `20`, and then constant-fold `10 + 20` to `30`.
