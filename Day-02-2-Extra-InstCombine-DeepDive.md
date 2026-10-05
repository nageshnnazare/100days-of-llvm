# Day 2 Extra Notes: A Deep Dive into InstCombine

This guide takes a closer look at the specific implementation files that make up the InstCombine pass, along with practical C examples that demonstrate these transformations.

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

## 1. The Engine: `InstCombineWorklist.h`
**Location:** `llvm/include/llvm/Transforms/InstCombine/InstCombineWorklist.h`

InstCombine is a worklist-driven algorithm. Instead of just doing a single pass over the instructions, it keeps a list of instructions that need to be processed. 

**Why a custom worklist?**
LLVM doesn't just use a standard `std::vector` or `std::queue`. The `InstCombineWorklist` is heavily optimized:
1.  **Deduplication:** It uses a `DenseMap` alongside a `SmallVector` to ensure that an instruction is never added to the worklist twice.
2.  **Cascading Updates:** When an instruction `I` is simplified, `I` and all instructions that *use* `I` are pushed back onto the worklist. This guarantees that one simplification can trigger another without needing to rescan the whole function.

---

## 2. Addition and Subtraction: `InstCombineAddSub.cpp`
**Location:** `llvm/lib/Transforms/InstCombine/InstCombineAddSub.cpp`

This file contains the logic for `add`, `sub`, `fadd`, and `fsub`. Inside `InstCombinerImpl::visitAdd`, you'll find hundreds of algebraic simplifications.

### Example: Folding `X + ~X`
One of the patterns it looks for is adding a number to its bitwise NOT. Mathematically, `X + ~X` always equals `-1` (in two's complement).
```cpp
// Pseudocode of LLVM pattern match:
if (match(I, m_Add(m_Value(X), m_Not(m_Specific(X)))))
  return replaceInstUsesWith(I, ConstantInt::getAllOnesValue(I.getType()));
```

### Practical C Example
```c
int test_add(int x) {
    int not_x = ~x;
    return x + not_x; 
}
```
*   **Unoptimized IR (`-O0`):** Contains `xor` (for the NOT) and `add`.
*   **InstCombine Output (`-O1`):** Completely replaced with `ret i32 -1`.

---

## 3. Multiplication, Division, and Remainder: `InstCombineMulDivRem.cpp`
**Location:** `llvm/lib/Transforms/InstCombine/InstCombineMulDivRem.cpp`

This file handles `mul`, `div` (sdiv/udiv), and `rem` (srem/urem). Multiplication and division are expensive operations, so InstCombine works hard to replace them with shifts or bitwise operations.

### Example: Power-of-Two Multiplication
Multiplying by a known power of two is notoriously converted to a left shift (`shl`).
```cpp
// Pseudocode of LLVM pattern match:
if (match(I, m_Mul(m_Value(X), m_Power2(C))))
  return BinaryOperator::CreateShl(X, ConstantInt::get(I.getType(), C.logBase2()));
```

### Practical C Example
```c
unsigned int test_mul(unsigned int x) {
    return x * 8;
}

unsigned int test_rem(unsigned int y) {
    return y % 4;
}
```
*   **Unoptimized IR (`-O0`):** Emits `mul` and `urem` instructions.
*   **InstCombine Output (`-O1`):** The `mul` becomes `shl %x, 3`. The `urem` becomes an `and %y, 3` (bitwise AND is significantly faster than modulo!).

---

## 4. Function Calls and Intrinsics: `InstCombineCalls.cpp`
**Location:** `llvm/lib/Transforms/InstCombine/InstCombineCalls.cpp`

InstCombine doesn't just simplify math; it also simplifies function calls. `InstCombineCalls.cpp` intercepts calls to known library functions (like `strlen`, `memcpy`, `sprintf`) and LLVM intrinsics.

### Example: `strlen` of a constant string
If InstCombine sees a call to `strlen` where the argument is a constant global string, it can evaluate the length at compile time.

### Practical C Example
```c
#include <string.h>

int test_strlen() {
    const char* str = "hello";
    return strlen(str);
}
```
*   **Unoptimized IR (`-O0`):** Emits a `call` to the external `@strlen` function.
*   **InstCombine Output (`-O1`):** The call is completely removed and replaced with `ret i32 5`.

---

## 5. Bitwise Operations: `InstCombineAndOrXor.cpp`
**Location:** `llvm/lib/Transforms/InstCombine/InstCombineAndOrXor.cpp`

### Practical C Example (De Morgan's Laws)
InstCombine knows De Morgan's laws: `~A & ~B` is the same as `~(A | B)`. The latter requires fewer instructions.
```c
int test_demorgan(int a, int b) {
    return (~a) & (~b);
}
```
*   **Unoptimized IR (`-O0`):** Two `xor`s (for NOT) and one `and`.
*   **InstCombine Output (`-O1`):** One `or` and one `xor` (`~(A | B)`). It reduces 3 instructions down to 2.
