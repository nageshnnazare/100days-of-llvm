# Day 2 Extra Notes: InstCombine Examples

This file provides a collection of practical C examples to demonstrate the power of LLVM's Instruction Combining (InstCombine) pass. InstCombine is responsible for peephole optimizations, algebraic simplifications, and redundancy elimination.

To test these examples, compile them to LLVM IR and run the InstCombine pass:
1. **Generate raw IR:** `clang -O0 -S -emit-llvm example.c -o unoptimized.ll`
2. **Run InstCombine:** `opt -passes=instcombine unoptimized.ll -S -o optimized.ll`

---

## Diagram

```text
   worklist = every instruction
            |
            v
      +-----------+
      | pop inst I|
      +-----------+
            |
     PatternMatch?
       /         \
     yes          no
      |            |
      v            v
  replace I      next
  with simpler
  value
      |
      +---- push users of I back on the worklist
                 (one fold exposes the next)
```

InstCombine is a worklist, not a single walk. Replacing `add X, 0` with `X` can make a user foldable, so that user goes back on the list. The patterns are canonical: the pass prefers one shape (`icmp slt` rather than a mirrored predicate) so later patterns only have to match one spelling.

## Example 1: Algebraic Simplification
InstCombine excels at collapsing algebraic identities, which often arise after other optimizations (like function inlining or constant folding) have run.

```c
int test_algebra(int x) {
    int a = x + 0;       // Identity: Add 0
    int b = a * 1;       // Identity: Multiply 1
    int c = b - x;       // x - x = 0
    return c;
}
```
**Expected Optimization:** 
InstCombine will realize that `c` evaluates to `0` and replace the entire mathematical sequence with `ret i32 0`.

---

## Example 2: Bitwise Redundancy Elimination
Bitwise operations frequently contain redundancies, especially when dealing with masks or boolean logic.

```c
int test_bitwise(int x) {
    // Masking an 8-bit value twice is redundant.
    int masked_once = x & 0xFF;
    int masked_twice = masked_once & 0xFF;
    
    // x ^ x is always 0.
    int self_xor = x ^ x;
    
    return masked_twice + self_xor;
}
```
**Expected Optimization:**
*   `masked_twice` is simplified back to just `x & 0xFF`.
*   `self_xor` is replaced with the constant `0`.
*   The final addition of `0` is eliminated, leaving just `ret i32 %masked_once`.

---

## Example 3: Strength Reduction (Multiplication to Shifts)
Multiplication and division on CPUs are typically much slower than bitwise shifts. InstCombine performs "strength reduction" to replace expensive operations with cheaper ones.

```c
unsigned int test_strength_reduction(unsigned int x) {
    unsigned int multiplied = x * 16;  // Power of 2 multiplication
    unsigned int divided = multiplied / 4; // Power of 2 division
    return divided;
}
```
**Expected Optimization:**
*   `x * 16` is transformed into `x << 4` (left shift).
*   Dividing by 4 is transformed into a logical right shift (`lshr` by 2).
*   InstCombine might even merge these two shifts into a single `x << 2`!

---

## Example 4: De Morgan's Laws
InstCombine implements standard boolean algebra rules, such as De Morgan's laws: `~(A & B) == ~A | ~B`.

```c
int test_demorgan(int a, int b) {
    // Return (~a) | (~b)
    int not_a = ~a;
    int not_b = ~b;
    return not_a | not_b;
}
```
**Expected Optimization:**
Instead of computing two `NOT` operations (represented as `xor` in LLVM) and one `OR`, InstCombine will transform this into `~(a & b)`—which is one `AND` and one `NOT`, saving an instruction.

---

## Example 5: Library Call Simplification
InstCombine doesn't just optimize math; it knows the semantics of standard C library functions.

```c
#include <string.h>

int test_libcalls() {
    const char *str1 = "Hello";
    const char *str2 = "World";
    
    // InstCombine knows the length of these static strings.
    return strlen(str1) + strlen(str2); 
}
```
**Expected Optimization:**
The calls to the external `strlen` function are entirely removed. InstCombine evaluates `strlen("Hello")` to `5` and `strlen("World")` to `5`, then folds the addition to simply return `10`.
