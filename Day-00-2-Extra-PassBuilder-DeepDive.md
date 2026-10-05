# Day 0 Extra Notes: A Deep Dive into `PassBuilderPipelines.cpp`

If `PassManager.h` is the engine that runs passes, then `llvm/lib/Passes/PassBuilderPipelines.cpp` is the instruction manual that tells the engine *what* passes to run and in *what order*.

This is a massive file (over 2500 lines), but you don't need to read it line-by-line. Instead, you need to understand the macro-structure of how LLVM organizes its optimization pipeline.

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

## 1. The Pass Hierarchy (The 4 Pass Managers)

LLVM organizes optimizations from the "widest" scope down to the "narrowest" scope. `PassBuilderPipelines.cpp` nests these managers inside each other:

1. **`ModulePassManager` (MPM)**: Operates on the entire file. (e.g., Global Dead Code Elimination).
2. **`CGSCCPassManager` (CGPM)**: Operates on strongly connected components of the Call Graph. Crucial for Inlining.
3. **`FunctionPassManager` (FPM)**: Operates on a single function. (e.g., InstCombine, Mem2Reg).
4. **`LoopPassManager` (LPM)**: Operates on a single loop. (e.g., Loop Unrolling, LICM).

You will frequently see `PassBuilder` adding a `FunctionPassManager` *into* a `ModulePassManager`. This tells LLVM: "Run this chunk of function-level optimizations on every function in the module."

---

## 2. The Big Three Functions

If you want to read `PassBuilderPipelines.cpp`, focus your attention on these three core functions that define the default `-O2`/`-O3` pipelines.

### A. `buildModuleSimplificationPipeline`
**Goal:** Clean up the IR, do early inlining, and get the code into a canonical state.
* It starts with "Early" passes like `EarlyCSEPass` to eliminate obvious redundancies.
* It runs `InlinerPass` (part of the CGSCC pipeline) to inline small functions into their callers early on, which exposes more optimization opportunities.
* It runs the `buildFunctionSimplificationPipeline` on all functions.

### B. `buildFunctionSimplificationPipeline`
**Goal:** The workhorse of LLVM optimizations. This is where the heavy lifting happens for a single function.
If you read this function, you will see a massive list of additions:
* `SROAPass`: Eliminates `alloca` memory access.
* `EarlyCSEPass`: Clears up more redundancies.
* `InstCombinePass`: Folds instructions.
* `SimplifyCFGPass`: Cleans up branches and basic blocks.
* `LoopPassManager`: It nests a loop pipeline here to run LICM (Loop Invariant Code Motion) and Loop Rotation.
* `CorrelatedValuePropagationPass`, `InstCombine` (again!), `SimplifyCFG` (again!). 

*(Notice how passes like InstCombine and SimplifyCFG are run multiple times! This is because other passes often leave behind messy IR that needs to be cleaned up again.)*

### C. `buildModuleOptimizationPipeline`
**Goal:** The final polish before Code Generation.
* Runs advanced scalar optimizations like `GVNPass` (Global Value Numbering) or `SCCPPass`.
* Runs Vectorization (if enabled): `LoopVectorizePass` and `SLPVectorizerPass`.
* Runs unrolling: `LoopUnrollPass`.
* Finishes with passes that prepare the IR for the backend (like instruction sinking).

---

## 3. Optimization Levels (`OptimizationLevel`)

The functions above take an `OptimizationLevel` parameter (e.g., `O1`, `O2`, `O3`, `Os`, `Oz`). 
Throughout `PassBuilderPipelines.cpp`, you will see `if` statements checking the level:

```cpp
if (Level == OptimizationLevel::O3)
  FPM.addPass(AggressiveInstCombinePass());
```

* **`O1`**: Focuses on quick compilation. Only runs the most profitable and fast passes.
* **`O2`**: The standard balance. Enables vectorization and heavy inlining.
* **`O3`**: Maximizes performance at the cost of long compile times. Enables aggressive loop unrolling and expensive analyses.
* **`Os` / `Oz`**: Optimizes for size (shrinking the binary) rather than execution speed. You will see checks like `Level.isOptimizingForSize()` that disable passes like unrolling (which make the binary bigger).

---

## 4. Extension Points (For custom passes)

One of the best features of `PassBuilder` is its Extension Points. If you write your own custom optimization pass, you don't need to modify `PassBuilderPipelines.cpp` directly. 

Instead, the `PassBuilder` has "Extension Point Callbacks." 
If you search the file for `invokePeepholeEPCallbacks`, `invokeScalarOptimizerLateEPCallbacks`, or `invokePipelineStartEPCallbacks`, you will see where the builder pauses to let external plugins inject their own passes into the pipeline.

## Summary

When reading `PassBuilderPipelines.cpp`, don't get bogged down in the specific order of all 100+ passes. Instead, look at:
1. How it structures the pipeline (Module -> CGSCC -> Function -> Loop).
2. How the optimization level (`Level`) toggles expensive passes on and off.
3. How `buildFunctionSimplificationPipeline` acts as the core loop of the optimizer.
