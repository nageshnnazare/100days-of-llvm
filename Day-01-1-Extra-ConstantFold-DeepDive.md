# Day 1 Extra Notes: A Deep Dive into Constant Folding

This document takes a closer look at the actual C++ code that performs Constant Folding in LLVM. As mentioned in the main Day 1 tutorial, Constant Folding is split into two layers: **Target-Independent** (in the `IR` directory) and **Target-Dependent** (in the `Analysis` directory).

---

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

## 1. The Target-Independent Layer (`lib/IR/ConstantFold.cpp`)

This file is responsible for folding constants based purely on the rules of mathematics and the LLVM IR type system. It does not know what CPU you are compiling for. If you ask it to fold `1 + 1`, it knows the answer is `2`. 

If you open `llvm/lib/IR/ConstantFold.cpp`, you will notice a series of massive `switch` statements. Here are the core functions to look for:

### `ConstantFoldBinaryInstruction(unsigned Opcode, Constant *V1, Constant *V2)`
This is the workhorse for standard math. It takes an opcode (like `Instruction::Add` or `Instruction::Mul`) and two `Constant` operands.
* **Integer Math:** It delegates to `ConstantInt::get()` after performing the operation using LLVM's `APInt` (Arbitrary Precision Integer) class. 
* **Floating Point Math:** It uses `APFloat`. LLVM is very careful with floating point math. It won't fold a floating point operation if it might generate a different result or exception at compile-time compared to run-time (unless flags like `fast-math` are used).
* **Special Cases:** You'll see optimizations like: if `V2` is 0 and Opcode is `Add`, it just returns `V1`. 

### `ConstantFoldCastInstruction(unsigned Opcode, Constant *V, Type *DestTy)`
This handles casting (converting types). For example, if you have a `ConstantInt` of `i8 255` and you zero-extend (`Instruction::ZExt`) it to `i32`, this function calculates the new constant (`i32 255`).
It handles all LLVM cast types: `Trunc`, `ZExt`, `SExt`, `FPTrunc`, `FPExt`, `BitCast`, etc.

### `ConstantFoldCompareInstruction(unsigned short pred, Constant *C1, Constant *C2)`
Handles operations like `icmp eq` (integer compare equal). If `C1` and `C2` are identical constants, it instantly returns a `ConstantInt` representing `true` (`1`). 

**Summary of IR level:** It is highly recursive. If you build a `ConstantExpr` that contains a binary operation, it calls these functions. If the result can be resolved, it returns a literal `ConstantInt`/`ConstantFP`. If it can't (for instance, trying to add two pointer addresses whose layout isn't known), it returns the `ConstantExpr` representing the unresolved math.

---

## 2. The Target-Dependent Layer (`lib/Analysis/ConstantFolding.cpp`)

Sometimes, math requires knowing how the target CPU lays out memory. The target-independent layer in `IR/` doesn't know how big a pointer is (is it 32-bit? 64-bit?). 

This is where `llvm/lib/Analysis/ConstantFolding.cpp` comes in. It requires a `DataLayout` object, which tells LLVM about the target architecture's pointer sizes, endianness, and struct alignments.

### `ConstantFoldInstOperands(Instruction *I, ArrayRef<Constant *> Ops, const DataLayout &DL)`
This is the most important function in the file. Pass pipelines like InstCombine call this function when they see an instruction whose operands are all constants.

* **Step 1:** It first tries the target-independent folder we discussed above! It literally calls `llvm::ConstantFoldInstOperandsImpl` which eventually reaches into `IR/ConstantFold.cpp`.
* **Step 2 (The Data Layout magic):** If the IR folder fails, this function steps in. 
  * It can fold `getelementptr` (GEP) instructions. A GEP calculates memory addresses. If you have a struct, and you want the offset of the 3rd field, `ConstantFoldInstOperands` uses the `DataLayout` to figure out exactly how many bytes that field is offset by, and returns a constant integer offset. The `IR` folder cannot do this because struct padding changes depending on the CPU!
  * It can fold `ptrtoint` and `inttoptr` casts when it knows the exact pointer size of the CPU.

### `ConstantFoldLoadFromConstPtr(Constant *C, Type *Ty, const DataLayout &DL)`
This function is fascinating. If you have a global variable that is marked as `constant` (e.g., `const int my_array[] = {1, 2, 3};`), and the code does a `load` from `my_array[1]`, this function can evaluate that load at compile-time!
It looks at the memory address, looks up the Global Variable, checks its initializer, and returns the constant `2`.

---

## How they work together in the Pipeline

When a pass like `InstCombine` runs:
1. It looks at an instruction, say `%ptr = getelementptr %struct, ptr %base, i32 0, i32 1`.
2. It calls the Analysis layer's `ConstantFoldInstruction(..., DataLayout)`.
3. The Analysis layer realizes it needs the `DataLayout` to resolve the GEP. It calculates the byte offset and returns a constant.
4. Later, `InstCombine` sees `%val = add i32 5, 5`.
5. It calls `ConstantFoldInstruction`.
6. The Analysis layer tries the `IR` layer first. The `IR` layer says "I don't need DataLayout for basic math!", evaluates it to `10`, and returns it.

**Takeaway:** The `IR` directory handles universal math. The `Analysis` directory wraps the `IR` directory and adds intelligence for target-specific memory layouts and global variables.
