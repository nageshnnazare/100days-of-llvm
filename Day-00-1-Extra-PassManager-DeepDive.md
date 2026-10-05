# Day 0 Extra Notes: A Deep Dive into `PassManager.h`

This document serves as an in-depth companion to Day 0, breaking down the exact classes and functions inside `llvm/include/llvm/IR/PassManager.h`. If you want to write your own passes or understand how LLVM's internal engine works, you must understand the classes defined in this header.

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

## 1. `PassInfoMixin<DerivedT>` (The Pass Boilerplate)

**What it is:** 
A CRTP (Curiously Recurring Template Pattern) mixin that every optimization pass in LLVM inherits from. 

**How it works:**
In C++, CRTP is when a class inherits from a template base class, passing *itself* as the template argument. 
```cpp
class MyPass : public PassInfoMixin<MyPass> { ... }
```
If you look at lines 81-89 in `PassManager.h`, you'll see this provides basic boilerplate like `name()`. It automatically extracts the name of your pass from the class type (e.g., returning the string `"MyPass"`). This is why you don't have to write a `getName()` function for every pass you create—the mixin handles it, which is crucial for debugging and pass pipeline printing.

---

## 2. `PreservedAnalyses` (The "What Changed?" Class)

**What it is:** 
A class representing which analyses are still valid after a pass runs.

**Why we need it:**
Analyses (like the Dominator Tree or Alias Analysis) take a long time to compute. We don't want to recompute them after every single optimization pass. 

**Key Methods:**
* **`PreservedAnalyses::all()`**: A pass returns this if it didn't change the IR at all (or only made changes that don't invalidate any analyses).
* **`PreservedAnalyses::none()`**: A pass returns this if it heavily modified the IR (e.g., deleted blocks, changed control flow) and destroyed all cached analyses.
* **`preserve<AnalysisT>()`**: If your pass modified the IR but you specifically carefully updated the Dominator Tree yourself, you can return a `PreservedAnalyses` object calling `.preserve<DominatorTreeAnalysis>()`. The `AnalysisManager` will know *not* to throw away the Dominator Tree.

---

## 3. `PassManager<IRUnit, AnalysisManager>` (The Runner)

**What it is:** 
The actual engine that loops over a list of passes and executes them on a specific piece of IR.

**Template Parameters:**
* `IRUnit`: The level of IR being optimized (e.g., `llvm::Module`, `llvm::Function`, or `llvm::Loop`).
* `AnalysisManager`: The corresponding manager for that IR unit (e.g., `FunctionAnalysisManager`).

**How it gets used:**
Inside the `PassManager` class, there is a `std::vector<std::unique_ptr<PassConcept>> Passes;` (a list of passes). 
When you call `PassManager::run(IRUnit &IR, AnalysisManager &AM)`, the `run` method iterates over this vector:
1. It extracts the next pass.
2. It calls `Pass->run(IR, AM)`.
3. It takes the `PreservedAnalyses` returned by the pass.
4. It tells the `AnalysisManager` to invalidate any analyses that were *not* preserved.
5. It repeats for the next pass.

---

## 4. `AnalysisKey` and `AnalysisInfoMixin` (The Analysis Boilerplate)

**What it is:**
Just like optimizations inherit from `PassInfoMixin`, analysis passes inherit from `AnalysisInfoMixin`.

**How it gets used:**
To cache an analysis result, the `AnalysisManager` needs a unique identifier (a key) for that specific analysis. `AnalysisKey` provides a unique memory address to act as this identifier. `AnalysisInfoMixin` injects this `Key` into your analysis class. 

When a pass asks for an analysis (`AM.getResult<DominatorTreeAnalysis>(F)`), the manager uses `DominatorTreeAnalysis::Key` to look up the cached result in a DenseMap.

---

## 5. `AnalysisManager<IRUnit>` (The Cache)

**What it is:**
A registry and cache for analysis results. 

**Key Methods:**
* **`getResult<AnalysisT>(IRUnit &IR)`**: This is the most used function by pass writers. If your pass needs the Dominator Tree, it calls `AM.getResult<DominatorTreeAnalysis>(F)`. 
  * If the tree is already computed and cached, the manager returns it instantly (O(1)).
  * If the tree is *not* cached (or was invalidated by a previous pass), the manager runs the `DominatorTreeAnalysis` pass on the spot, caches the result, and then returns it.
* **`invalidate(IRUnit &IR, PreservedAnalyses PA)`**: The `PassManager` calls this after every optimization pass. The `AnalysisManager` looks at the `PA` object, compares it to its internal cache, and deletes any cached results that were not preserved.

---

## 6. Proxies: `InnerAnalysisManagerProxy` & `OuterAnalysisManagerProxy`

**What they are:**
Adapters that allow passes on one level of IR to access analyses from a different level.

**Why we need them:**
Imagine you are running a `ModulePassManager` (looking at the whole file). Inside this module pass, you want to get the Dominator Tree for a specific Function. But you only have a `ModuleAnalysisManager`, and the Dominator Tree lives in the `FunctionAnalysisManager`!

**How they work:**
* **`FunctionAnalysisManagerModuleProxy`**: This is an analysis pass on a Module that actually just contains a `FunctionAnalysisManager`. It allows a Module pass to query Function-level analyses.
* The proxies also handle complex invalidation rules. For example, if a Module pass deletes a Function, the proxy ensures that all cached analyses for that specific deleted function are wiped from the `FunctionAnalysisManager`.

---

## Summary of the Flow

1. You create an optimization class inheriting from `PassInfoMixin`.
2. You implement `run(Function &F, FunctionAnalysisManager &AM)`.
3. Inside `run`, you ask for data: `auto &DT = AM.getResult<DominatorTreeAnalysis>(F);`.
4. The `PassBuilder` adds your pass to a `FunctionPassManager`.
5. The `FunctionPassManager::run` method executes your pass, takes the `PreservedAnalyses` you returned, and tells the `FunctionAnalysisManager` to invalidate stale data.
