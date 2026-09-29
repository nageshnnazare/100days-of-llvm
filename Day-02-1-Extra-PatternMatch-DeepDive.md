# Day 2 Extra Notes: Deep Dive into LLVM PatternMatching

**File Location:** `llvm/include/llvm/IR/PatternMatch.h`

`PatternMatch.h` is one of the most brilliant pieces of C++ infrastructure in LLVM. It provides a Domain-Specific Language (DSL) embedded directly in C++ for matching subgraphs of LLVM Intermediate Representation (IR). 

It is heavily used in InstCombine, but you will also find it used in Instruction Selection (ISel), Vectorization, and various Analysis passes.

---

## 1. The Problem it Solves
Suppose you want to write an optimization that transforms `(X + 0)` into just `X`.
Without PatternMatch, verifying this structure requires verbose and error-prone casting:

```cpp
// The old, painful way:
if (auto *BO = dyn_cast<BinaryOperator>(I)) {
  if (BO->getOpcode() == Instruction::Add) {
    if (auto *CI = dyn_cast<ConstantInt>(BO->getOperand(1))) {
      if (CI->isZero()) {
        Value *X = BO->getOperand(0);
        // It's a match! We can optimize it.
      }
    }
  }
}
```

With `PatternMatch.h`, this becomes a single, highly readable statement:

```cpp
// The PatternMatch way:
using namespace llvm::PatternMatch;

Value *X;
if (match(I, m_Add(m_Value(X), m_Zero()))) {
    // It's a match! 'X' is automatically bound to the first operand.
}
```

---

## 2. How it Works Under the Hood
The entry point is the `llvm::PatternMatch::match(Value, Pattern)` function.

1.  **`match()`** takes two arguments: the `Value*` you are inspecting, and a `Pattern` (which is a tree of temporary C++ objects).
2.  **Matchers (like `m_Add`)** are essentially functions that return lightweight struct objects (e.g., `BinaryOp_match`).
3.  These structs overload the `match()` method. When you evaluate the top-level `match`, it recursively calls `.match()` down the tree of objects. If any step fails, the whole match fails immediately and returns `false`.

It relies heavily on C++ templates, meaning this elegant syntax compiles down to extremely fast, highly optimized machine code—usually exactly the same as the manual `dyn_cast` approach.

---

## 3. The Matcher Vocabulary
`PatternMatch.h` provides an extensive library of matchers. Here are the most important categories:

### A. Capture Matchers
These don't enforce a specific structure; they simply bind whatever they see to a variable so you can use it later.
*   `m_Value(V)`: Matches *any* `Value` and assigns it to `V`.
*   `m_Specific(V)`: Matches only if the node is exactly the specific pointer `V`.

### B. Constant Matchers
*   `m_Zero()`: Matches integer `0`, floating-point `+0.0`, or a null pointer.
*   `m_One()`: Matches integer `1`.
*   `m_AllOnes()`: Matches an integer where all bits are `1` (e.g., `-1`).
*   `m_SpecificInt(N)`: Matches a constant integer exactly equal to `N`.

### C. Instruction Matchers
Every standard LLVM instruction has a matcher.
*   `m_Add(LHS, RHS)`: Matches an `add` instruction.
*   `m_Mul(LHS, RHS)`: Matches a `mul` instruction.
*   `m_Select(Cond, TrueVal, FalseVal)`: Matches a `select` instruction.
*   `m_ICmp(Pred, LHS, RHS)`: Matches an integer comparison (`icmp`). It captures the predicate (like `==`, `>`, `<`) into `Pred`.

### D. Bitwise Matchers
*   `m_Not(X)`: LLVM doesn't have a `not` instruction; it represents NOT as `xor X, -1`. The `m_Not` matcher acts as syntactic sugar to detect this exact `xor` pattern.

---

## 4. The Magic of Commutativity (`m_c_`)
In math, `A + B` is the same as `B + A`. But in an Abstract Syntax Tree (AST) or IR graph, they are different trees. If you want to check if an addition involves a zero, it could be `X + 0` OR `0 + X`. 

Instead of writing a massive `if` statement to check both variations, `PatternMatch.h` provides **commutative matchers**, prefixed with `m_c_`.

```cpp
Value *X;
// This will match BOTH (X * 2) and (2 * X)
if (match(I, m_c_Mul(m_Value(X), m_SpecificInt(2)))) {
   // Optimization logic
}
```

When you chain these together, it becomes incredibly powerful. For example, to match `(A * 2) + (B * 2)` regardless of the order of operands in any of the three operations:
```cpp
if (match(I, m_c_Add(m_c_Mul(m_Value(A), m_SpecificInt(2)), 
                     m_c_Mul(m_Value(B), m_SpecificInt(2))))) {
    // ...
}
```
Writing the manual `dyn_cast` equivalent for that commutative tree would take dozens of lines of code!
