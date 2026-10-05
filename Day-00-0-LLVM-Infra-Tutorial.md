# Day 0: LLVM Infrastructure & The Pass Manager

Before we jump into specific optimization passes like Constant Folding or InstCombine, we need to understand **how LLVM actually runs these passes**. If you have an LLVM IR module, how does LLVM know which optimizations to run, in what order, and how do they share information?

The answer is the **LLVM Pass Manager**.

---

## Diagram

```text
                    opt / clang -O2
                          |
                          v
                    +-----------+
                    |PassBuilder|  parses "default<O2>" or "-passes=..."
                    +-----------+
                          |
          +---------------+---------------+
          v               v               v
   ModulePassManager  CGSCCPassManager  FunctionPassManager
   GlobalDCE, IPSCCP   Inliner           InstCombine, SROA, LICM, ...
          |               |               |
          +---------------+---------------+
                          |
                          v
                  AnalysisManager
            DT, AA, SCEV, LoopInfo, ...
                          |
              PreservedAnalyses says which
              of those caches are still valid
```

Read top to bottom. A pipeline string is not a list of free functions: PassBuilder turns it into nested pass managers. A function pass only sees one function. A CGSCC pass sees a strongly connected component of the call graph, which is why the inliner lives there. After a pass returns, the analysis manager throws away any analysis the pass did not preserve.

## 1. The Big Picture: How Passes Come Together

When you run a frontend compiler like `clang -O3 main.c`, the following high-level steps happen:
1. **Frontend (Clang):** Parses `main.c` into an Abstract Syntax Tree (AST).
2. **CodeGen:** Clang converts the AST into unoptimized LLVM IR.
3. **The Pass Pipeline (Optimization):** Clang hands the unoptimized IR over to LLVM's **Pass Manager**. The Pass Manager runs a carefully curated list (pipeline) of optimization and analysis passes.
4. **Backend:** The optimized IR is lowered to machine code (e.g., x86 assembly).

The "Pass Pipeline" is where 90% of the compiler's magic happens.

---

## 2. The New Pass Manager

LLVM recently completed a massive migration from the "Legacy Pass Manager" to the "New Pass Manager" (NPM). If you are looking at LLVM code today, you should focus entirely on the New Pass Manager.

### The Core Components
The New Pass Manager infrastructure revolves around a few key classes located in `llvm/include/llvm/IR/PassManager.h`:

* **`PassManager<IRUnit>`**: This is the engine that actually loops over a list of passes and runs them. Passes can operate on different units of IR:
  * `ModulePassManager`: Runs passes that look at the entire file/module (e.g., Global Dead Code Elimination).
  * `FunctionPassManager`: Runs passes that look at a single function at a time (e.g., InstCombine, Mem2Reg).
  * `LoopPassManager`: Runs passes that look at a single loop at a time (e.g., Loop Unrolling).

* **`AnalysisManager<IRUnit>`**: Optimizations often need complex data (like Dominator Trees or Alias Analysis). Instead of recomputing this data every time, the `AnalysisManager` computes it once and caches it. Passes can request analyses from this manager.

* **`PreservedAnalyses`**: When a pass finishes running, it must return a `PreservedAnalyses` object. If the pass didn't change the IR at all, it returns `PreservedAnalyses::all()`. If it completely destroyed the IR structure, it returns `PreservedAnalyses::none()`, telling the `AnalysisManager` to throw away its cached data.

---

## 3. What Does a Pass Look Like?

A modern LLVM pass is just a C++ struct or class with a `run` method. It doesn't need to inherit from a massive base class, it just needs to follow a specific signature.

Here is a conceptual example of what an optimization pass looks like:

```cpp
#include "llvm/IR/PassManager.h"

// Your pass inherits from PassInfoMixin for some boilerplate helpers.
struct MyAwesomePass : public llvm::PassInfoMixin<MyAwesomePass> {
  
  // The 'run' method is the entry point.
  // It takes the Function to optimize and the AnalysisManager.
  llvm::PreservedAnalyses run(llvm::Function &F, llvm::FunctionAnalysisManager &FAM) {
    bool Changed = false;
    
    // ... do some optimizations on the Function F ...
    
    if (!Changed) {
      // We didn't change anything, so all cached analyses are still valid!
      return llvm::PreservedAnalyses::all();
    }
    
    // We changed the IR, so tell the manager that older analyses might be invalid.
    return llvm::PreservedAnalyses::none();
  }
};
```

---

## 4. Where is the Pipeline Defined? (How do passes get called?)

So, who decides the order in which passes run? Does Constant Folding run before or after Inlining?

This heavily curated ordering is called the **Pass Pipeline**. 
The logic for building these pipelines lives in the **`PassBuilder`**.

**Where to read the code:**
* `llvm-project/llvm/lib/Passes/PassBuilderPipelines.cpp`

If you open `PassBuilderPipelines.cpp` and look for a function called `buildModuleOptimizationPipeline` or `buildFunctionSimplificationPipeline`, you will literally see LLVM adding passes to a list:

```cpp
// Conceptual snippet from PassBuilderPipelines.cpp
FunctionPassManager FPM;
FPM.addPass(InstCombinePass());      // Run InstCombine!
FPM.addPass(SimplifyCFGPass());      // Then simplify the Control Flow Graph!
FPM.addPass(SROAPass());             // Then run Scalar Replacement of Aggregates!
```

When you pass `-O2` or `-O3` to Clang, Clang calls into the `PassBuilder`, asks it to construct the `-O3` pipeline, and then tells the `PassManager` to execute that pipeline on the IR module.

---

## 5. How to Play with the Pass Manager Manually

You don't need to write C++ to see how passes are scheduled. LLVM provides a command-line tool called `opt` (The LLVM Optimizer) that lets you run passes manually on an IR file.

1. Create a file `test.ll` containing unoptimized LLVM IR.
2. Run a specific pass manually using the New Pass Manager syntax (`-passes=...`):
   ```bash
   # Run just the instcombine pass on the IR and print the result
   opt -passes=instcombine -S test.ll 
   ```
3. Run an entire pipeline:
   ```bash
   # Run the exact pipeline that -O2 would run
   opt -passes='default<O2>' -S test.ll
   ```
4. Ask `opt` to print out which passes are running and in what order:
   ```bash
   opt -passes='default<O2>' -debug-pass-manager -disable-output test.ll
   ```
   *(This `-debug-pass-manager` flag is incredibly useful for seeing exactly how the PassManager orchestrates passes, analyses, and invalidations in real time!)*

---

## 6. Tracing the Call Stack: Where does the execution actually start?

If you want to read exactly how this infrastructure is booted up, here are the two main call stacks you should trace through the LLVM source code:

### Path A: How `clang` runs the Pass Manager
When you run `clang -O3`, the compilation goes through the frontend (Lexing, Parsing, AST generation) and eventually hits the **CodeGen** phase where it lowers the AST to LLVM IR and optimizes it.
1. **`clang/tools/driver/driver.cpp`**: The main entry point for the `clang` executable. It sets up jobs for the compiler.
2. **`clang/lib/CodeGen/CodeGenAction.cpp`**: This bridges the frontend AST with the LLVM backend.
3. **`clang/lib/CodeGen/BackendUtil.cpp`**: **(CRITICAL FILE)** Look for the function `EmitAssemblyHelper::CreatePasses()`. This is exactly where Clang instantiates the LLVM `PassBuilder`, populates it with target-specific information, and asks it to build the `-O3` pipeline.
4. **`llvm/lib/Passes/PassBuilderPipelines.cpp`**: `PassBuilder` receives the request from Clang and constructs the massive list of passes.
5. **`llvm/include/llvm/IR/PassManager.h`**: Finally, `BackendUtil.cpp` calls `PM.run(Module)` to start executing the passes.

### Path B: How `opt` runs the Pass Manager
When you run `opt -passes='default<O2>'`, the frontend is completely bypassed. `opt` just parses a `.ll` (or `.bc`) file directly and runs passes on it.
1. **`llvm/tools/opt/opt.cpp`**: The main entry point for the `opt` tool. It parses command-line arguments.
2. **`llvm/tools/opt/NewPMDriver.cpp`**: **(CRITICAL FILE)** Look for `llvm::runPassPipeline(...)`. This is where `opt` initializes the `PassBuilder`, parses your `-passes=...` string, builds the requested pipeline, and calls `PM.run()`.

---

## Summary of your reading for Day 0:
If you want to trace the infrastructure from top to bottom, read these files in order:
1. **The Entry Point:** `clang/lib/CodeGen/BackendUtil.cpp` (How Clang starts the optimizer) OR `llvm/tools/opt/NewPMDriver.cpp` (How `opt` starts it).
2. **The Pipeline Builder:** `llvm/lib/Passes/PassBuilderPipelines.cpp` (Look for `buildFunctionSimplificationPipeline`).
3. **The Engine:** `llvm/include/llvm/IR/PassManager.h` (Look at how the `PassManager::run` method loops through the passes).

> [!TIP]
> **Want to dive deeper into the code?** 
> * [Day 0 Extra Notes: A Deep Dive into PassManager.h](Day-00-1-Extra-PassManager-DeepDive.md) breaks down the engine (`PassInfoMixin`, `PreservedAnalyses`, `AnalysisManager`).
> * [Day 0 Extra Notes: A Deep Dive into PassBuilderPipelines.cpp](Day-00-2-Extra-PassBuilder-DeepDive.md) breaks down how the `-O3` pipeline is actually constructed and ordered.
