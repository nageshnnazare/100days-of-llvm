# Day 1: Constant Folding

## Diagram

```text
   IRBuilder::CreateAdd(%a, %b)
              |
              v
       +--------------+
       |ConstantFolder|
       +--------------+
          /            \
   both Constant?      one is not
        /                  \
       v                    v
  ConstantInt           an `add` Instruction
  (folded now)          (left for a later pass)

   2 + 3  ->  5          %x + 2  stays `add`
```

Folding happens while the IR is being built, before any pass runs. If both operands are constants, the builder never emits the instruction. If either operand is a runtime value, the `add` stays, and InstCombine or SCCP may fold it later.

## 1. What is Constant Folding?
Constant folding is a foundational compiler optimization where expressions consisting entirely of constants are evaluated at compile time rather than at runtime. This reduces the number of instructions executed and shrinks the size of the final executable.

For example, `int x = 2 * 3;` becomes `int x = 6;` during compilation. 
In LLVM, Constant Folding is so fundamental that it often happens "on the fly" as the intermediate representation (IR) is being built, even before dedicated optimization passes like InstCombine run.

## 2. Where to Read the Code
The core of LLVM’s constant folding logic lives in two main locations. I recommend starting with the IR-level constant folding:

### The IR Level (Target Independent)
This logic operates purely on LLVM's Intermediate Representation without knowing about the target architecture (like x86 or ARM).
* **Header file:** `llvm-project/llvm/include/llvm/IR/ConstantFold.h`
* **Source file:** `llvm-project/llvm/lib/IR/ConstantFold.cpp`

### The Analysis Level (Target Dependent)
This provides more advanced constant folding that relies on the Data Layout (e.g., how many bytes a pointer takes up on the target CPU).
* **Header file:** `llvm-project/llvm/include/llvm/Analysis/ConstantFolding.h`
* **Source file:** `llvm-project/llvm/lib/Analysis/ConstantFolding.cpp`

> [!TIP]
> **Want a deep dive into these files?**
> Read the companion guide: [Day 1 Extra Notes: A Deep Dive into Constant Folding](Day-01-1-Extra-ConstantFold-DeepDive.md) where we break down the most important C++ functions inside these files (like `ConstantFoldBinaryInstruction` and how `DataLayout` works).

## 3. Understanding the Infrastructure

To understand how constant folding works in LLVM, you need to understand a few core concepts of the LLVM IR infrastructure:

### A. The `llvm::Constant` Class
In LLVM, everything is a `Value`. A `Constant` is a subclass of `Value`. If an instruction’s result is a constant, it can be represented by an instance of `llvm::Constant` (or one of its subclasses like `ConstantInt` for integers, `ConstantFP` for floating-point).

### B. `IRBuilder` and the "Folder"
When frontends (like Clang) or optimization passes construct new instructions, they typically use an `IRBuilder`. 
The `IRBuilder` uses a "Folder" class (like `ConstantFolder`) by default. When you ask the `IRBuilder` to create an addition instruction (`CreateAdd(LHS, RHS)`), the folder intercepts it:
1. It checks if both `LHS` and `RHS` are constants.
2. If they are, instead of creating an `add` instruction, it directly computes the math and returns a `ConstantInt` with the result.
3. If they are not both constants, it falls back to creating the actual `add` instruction.

### C. Constant Expressions (`ConstantExpr`)
LLVM has a special kind of constant called a `ConstantExpr`. This represents a computation that evaluates to a constant. For example, if you cast a constant pointer to an integer, LLVM might represent that as a `ConstantExpr` until it knows enough about the target to turn it into a raw number.

## 4. What to Read Today
1. **Open `llvm-project/llvm/lib/IR/ConstantFold.cpp`**: Look for the function `ConstantFoldBinaryInstruction`. This is a great place to see the switch statement that handles adding, subtracting, multiplying, etc., when both operands are constants.
2. **Open `llvm-project/llvm/include/llvm/IR/ConstantFold.h`**: Look at the API declarations. Notice how functions like `ConstantFoldInstruction` take an `Instruction` and return a `Constant*` (which is null if it couldn't be folded).

## 5. Experiment / Hands-on
Try writing a C program like this:
```c
int main() {
    int x = 10 + 20;
    return x;
}
```
Compile it to LLVM IR without optimizations to see the raw instructions:
`clang -O0 -S -emit-llvm main.c`

Then compile it with basic optimizations to see the constant folding in action:
`clang -O1 -S -emit-llvm main.c`
You should see that the addition instruction completely disappears, replaced by the constant `30`.
