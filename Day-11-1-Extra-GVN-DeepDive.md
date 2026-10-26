# Day 11 Extra: GVN Deep Dive

This note walks the real `GVNPass` implementation: types inside `ValueTable`, the `run` path, load elimination, PRE, debug flags, and what GVN deliberately refuses to do. Pair with `Day-11-0-GVN.md`.

## Diagram

```text
   %a = add %x, 1
   ...
   %b = add %x, 1          GVN: %b and %a are the same
                           value number -> replace %b with %a

   store 7, ptr %p
   %v = load ptr %p        MemoryDependence says the store
                           is the clobber -> replace %v with 7
```

GVN gives two instructions the same value number when their operands have the same numbers and the opcode matches. Loads are included only when a memory dependence query says the loaded address was written by a known store and not clobbered in between. A load is not "the same" as another load just because the address expression matches.

## 1. Files and entry points

Primary implementation:

- [llvm/lib/Transforms/Scalar/GVN.cpp](https://github.com/llvm/llvm-project/blob/main/llvm/lib/Transforms/Scalar/GVN.cpp)
- [llvm/include/llvm/Transforms/Scalar/GVN.h](https://github.com/llvm/llvm-project/blob/main/llvm/include/llvm/Transforms/Scalar/GVN.h)

New Pass Manager entry: `GVNPass::run(Function &F, FunctionAnalysisManager &AM)`. It pulls analyses, configures `ValueTable`, then iterates blocks/instructions until a fixed point or a single transformative sweep (depending on the current structure of the pass — read `runImpl`-style helpers next to `run`).

Legacy PM still registers the pass name `gvn` for `opt` compatibility; prefer `-passes=gvn`.

## 2. `ValueTable` and `Expression`

`GVNPass::ValueTable` is the heart of classic LLVM GVN.

- **`Expression`:** opcode, type, and a list of operand value numbers (`VarArgs`). For some ops (ICmp/FCmp), predicate bits are part of the key. GEP and extractvalue have dedicated `createGEPExpr` / `createExtractvalueExpr` helpers so structurally equal addressing folds together.
- **`lookupOrAdd(Value*)`:** returns the integer value number for an SSA value, creating one if needed. Non-instructions (arguments, constants) get fresh or canonical numbers.
- **`createExpr(Instruction*)`:** builds the expression from opcode + numbered operands.
- **`addMemoryStateToExp`:** when MemorySSA is enabled for numbering, appends a number representing the clobbering memory access (LiveOnEntry, MemoryPhi’s block, or defining memory instruction). Two loads with the same pointer operands but different clobber keys do **not** share a number.

Leader maps associate a value number with the instruction instances that currently define it. Replacement only proceeds when a candidate leader dominates the instruction being removed.

## 3. The `run` path (conceptual walk)

1. **Options.** `GVNOptions` (from the header) gate scalar PRE, load PRE, MemDep, and MemorySSA. Command-line mirrors include `-enable-load-pre`, `-enable-gvn-memdep`, `-enable-gvn-memoryssa`.
2. **Analyses.** DominatorTree, AAResults, AssumptionCache; then MemDep and/or MemorySSA per options. When MemorySSA is used for rewrites, construct a `MemorySSAUpdater`.
3. **Initialize `ValueTable`.** Point it at AA / MemDep / MSSA as required (`set*` style methods on `ValueTable`).
4. **Process the function.** For each block in a dominance-friendly order, visit instructions. Dead instructions may be deleted early. For each live instruction, attempt value-number CSE, then specialized handlers (loads, PHIs, etc.).
5. **Load path.** Specialized analysis finds reaching defs (`findReachingValuesForLoad`-style logic on the MSSA path, or MemDep dependencies on the classic path). `AnalyzeLoadAvailability` (name historically used in GVN) classifies whether a replacement value exists locally or must be PHI-merged / PRE-inserted.
6. **PRE.** If partial availability and load PRE enabled, split critical edges as needed, insert loads, create PHIs, update MSSA.
7. **Cleanup.** Erase replaced instructions; verify removed numbers (`verifyRemoved` exists for debug builds).

## 4. Important helpers

| Helper area | Role |
| --- | --- |
| `lookupOrAddCall` | CSE only safe calls; else unique VN |
| `computeLoadStoreVN` | Memory-keyed VN for load/store when MSSA numbering on |
| `processBlock` / per-inst process | Drive replacements |
| Load availability / PRE helpers | Decide CSE vs insert-on-edge |
| `MemorySSAUpdater` hooks | Keep MSSA valid after inserts/deletes |

Also watch statistics (`STATISTIC`) at the top of `GVN.cpp` — redundant loads eliminated, PRE loads inserted, etc. Useful when comparing flags.

## 5. Debug flags and experiments

```bash
opt -passes=gvn -debug-only=gvn -S in.ll
```

`DEBUG_TYPE` is `"gvn"`. Under `#ifndef NDEBUG`, expect dumps of value numbers, replaced instructions, and PRE decisions.

Useful knobs:

- `-enable-load-pre` / `-enable-load-pre=false`
- `-enable-scalar-pre`
- `-enable-gvn-memoryssa` / `-enable-gvn-memdep`

When comparing MSSA vs MemDep paths, keep the rest of the pipeline identical and diff IR plus `-debug-only=gvn` traces.

## 6. Analyses required and preserved

**Required (typical):** DominatorTree, AAResults, AssumptionCache; MemoryDependenceResults and/or MemorySSA.

**Related:** OptimizationRemarkEmitter for missed/success remarks; LoopInfo sometimes consulted for PRE heuristics.

**Preserved:** CFG often mostly preserved (except critical-edge splits for PRE). MemorySSA preserved when updated correctly. AA is not “recomputed” by GVN but results remain usable if IR types/attrs are intact.

## 7. Call CSE policy (read this carefully)

GVN does **not** generally CSE arbitrary calls. `lookupOrAddCall` checks memory effects. A call that may write memory, throw, or diverge gets a distinct number every time even if the callee and arguments look identical. `readonly` / `readnone` / `memory(...)` attributes that prove no conflicting write allow congruent calls to collapse. Intrinsic handling may special-case known identities, but the default mental model is: **calls are unique unless proven pure enough**.

## 8. What GVN deliberately does NOT handle

- **Full predicate awareness** like NewGVN’s congruence with predicates — classic GVN is weaker on branch-correlated equivalences.
- **Arbitrary cross-thread memory** — assumes LLVM IR memory model / AA contract.
- **Speculative loads across EH edges** without safety checks — PRE and availability logic bail out.
- **Replacing volatile loads** — volatiles stay.
- **Being a general PRE for every opcode** — scalar PRE exists but is limited compared to a textbook lazy code motion pass.
- **NewGVN’s algorithm** — separate pass; do not edit one expecting the other to match.

## 9. NewGVN in one paragraph

[NewGVN.cpp](https://github.com/llvm/llvm-project/blob/main/llvm/lib/Transforms/Scalar/NewGVN.cpp) maintains congruence classes and can exploit predicates. It was an attempt at a more principled GVN/PRE. Production `-O2` still centers on classic `GVNPass`. Read NewGVN when you care about the theory; debug production issues in `GVN.cpp` first.

## 10. Suggested breakpoints

If you build a debug `opt`:

1. Break on `GVNPass::run`.
2. Step into first `ValueTable::lookupOrAdd` for an `add`.
3. Break on load availability when stepping through `load_cse`.
4. Force a may-alias store and confirm availability fails.

## 11. Mental model checklist

When GVN “misses”:

1. Is the function `optnone`?
2. Do operands really share numbers (bitcasts / nsw differences / extra uses)?
3. Does AA say may-alias on the clobber?
4. Is the call impure?
5. Is load PRE disabled or blocked by a critical edge / EH pad?

## 12. Tie-back

Return to the main tutorial for pipeline placement and the quiz. Use `Day-11-2-Extra-GVN-Examples.md` to exercise CSE, blocked aliasing, call non-CSE, and load PRE shapes.


## 13. MemorySSAUpdater responsibilities

When GVN inserts a load for PRE or deletes a load after CSE, MemorySSA’s use-def graph would rot if left untouched. `MemorySSAUpdater` provides APIs to create memory accesses for new instructions, remove accesses for deleted ones, and fix MemoryPhis when edges change. If you hack GVN for learning, never RAUW a load out of the IR without updating MSSA when that mode is on — later LICM/DSE will mis-compile or assert in expensive checks (`-enable-memoryssa-check` / verifier builds).

## 14. AssumptionCache touchpoints

`llvm.assume` can prove non-null pointers or mask properties that AA and value tracking consult. GVN itself is not a full assume propagator (see CVP / ConstraintElimination on later days), but AssumptionCache is in the analysis set so dependent helpers see assumes. When a load CSE mysteriously fails, check whether an assume only holds on one path — GVN will not invent that correlation unless the dominating leader already sits on that path.

## 15. Reading `lookupOrAdd` with a debugger

Set a breakpoint on `GVNPass::ValueTable::lookupOrAdd`. Step through numbering of:

1. an argument,
2. an `add` of two arguments,
3. a second congruent `add`,
4. a `store` then a `load`.

Record the integer numbers printed under `-debug-only=gvn`. You should see the second `add` reuse the first’s number and then be deleted. The load’s number should incorporate memory state when MSSA numbering is enabled.

## 16. Failure modes worth memorizing

| Symptom | Likely cause |
| --- | --- |
| No changes at all | `optnone`, or wrong pass name |
| Scalar CSE works, loads do not | Alias clobber / volatile / ordered atomics |
| PRE inserts then CFG looks odd | Critical edge splits — expected |
| Calls not removed | Missing readnone/readonly |
| Crash in MSSA | Updater not run after IR edit |
| Differs from NewGVN | Different algorithm — expected |

## 17. Minimal curated reading list inside GVN.cpp

1. File-level comments and `cl::opt` flags near `DEBUG_TYPE`.
2. `ValueTable::createExpr` and `lookupOrAdd`.
3. `lookupOrAddCall`.
4. Load elimination / availability section (search for `LoadInst`).
5. Any function mentioning `LoadPRE` or `enable-load-pre`.
6. `GVNPass::run` for analysis wiring.

## 18. Scalar PRE vs load PRE

Scalar PRE (`-enable-scalar-pre`) tries to hoist/insert arithmetic similarly to load PRE when a congruent computation is missing on a predecessor. It is easier to reason about than load PRE because it does not fault. Still, code-size and register-pressure heuristics matter — PRE is not always a win on cold paths.

Load PRE is the headline feature for memory. When studying, toggle the two flags independently and keep a spreadsheet of IR size and load counts on a microbenchmark.

## 19. Closing map of neighbors in code

From `GVNPass::run`, note which `AM.getResult` / `getCachedResult` calls appear. That list is the ground truth for “analyses required” on your LLVM revision. Pipeline builders in `PassBuilderPipelines.cpp` show whether MemorySSA is cached before GVN in `-O2`/`-O3`. Re-check after major merges — MSSA defaulting has been in flux.
