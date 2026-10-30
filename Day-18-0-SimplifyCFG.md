# Day 18: SimplifyCFG

SimplifyCFG is LLVM’s workhorse control-flow cleaner: fold trivial branches, merge blocks, delete unreachable code, turn two-entry PHIs into selects, reshape switches, and optionally hoist/sink common instructions. Implementation lives mainly in [SimplifyCFG.cpp](https://github.com/llvm/llvm-project/blob/main/llvm/lib/Transforms/Utils/SimplifyCFG.cpp) (`SimplifyCFGOpt`); the pass wrapper is [SimplifyCFGPass.cpp](https://github.com/llvm/llvm-project/blob/main/llvm/lib/Transforms/Scalar/SimplifyCFGPass.cpp). Options: [SimplifyCFGOptions.h](https://github.com/llvm/llvm-project/blob/main/llvm/include/llvm/Transforms/Utils/SimplifyCFGOptions.h). Deep dive: [Day-18-1-Extra-SimplifyCFG-DeepDive.md](Day-18-1-Extra-SimplifyCFG-DeepDive.md). Examples: [Day-18-2-Extra-SimplifyCFG-Examples.md](Day-18-2-Extra-SimplifyCFG-Examples.md).

## 1. Why this exists

Frontends and earlier opts produce legal but ugly CFGs: empty blocks, `br label %next` chains, diamonds that are really selects, switches that are dense integer maps, duplicated ops on both arms. Every later analysis pays for that mess (dominators, loop info, vectorization legality).

SimplifyCFG is the shared utility that repeatedly applies local CFG rewrites until a fixed point (within budgets). It runs **early and late** with *different* `SimplifyCFGOptions`: early passes stay conservative about destroying LoopSimplify form; late passes enable switch tables, more speculation, and common-code sinking.

## Diagram

```text
   before                         after simplifycfg

   entry                          entry
     br label %mid                  %x = add %a, 1
   mid:                             ret %x
     %x = add %a, 1
     br label %end               the empty mid block is gone;
   end:                          its instructions were folded
     ret %x                      into the predecessor

   br i1 %c, label %t, label %t   --->   br label %t
   (both edges the same)                 (unconditional)
```

SimplifyCFG folds blocks that have one predecessor and one successor, turns a branch with identical successors into a jump, and folds a branch on a constant. It is also what deletes a block after a threading or DCE pass has made it empty. Loop passes ask it not to destroy the canonical loop shape.

## 2. The idea

Treat each basic block’s terminator and its neighborhood as a rewrite system:

- Unconditional branch to a block with a single predecessor → merge.
- Branch to a block that only contains an identical branch → thread / fold.
- Unreachable blocks → delete.
- Two-predecessor join with a PHI of values computed on both sides → often `select`.
- Switch with dense cases → lookup table or icmp/bit-test forms when TTI agrees.
- Identical instructions in multiple predecessors → hoist to common dominator or sink to common successor when options allow.

All rewrites must preserve SSA, dominance, and observable behavior (with explicit speculation rules when folding blocks into predecessors).

## 3. Where it sits in the -O2/-O3 pipeline

| Item | Value |
|------|--------|
| Pass name | `simplifycfg` |
| Util core | [SimplifyCFG.cpp](https://github.com/llvm/llvm-project/blob/main/llvm/lib/Transforms/Utils/SimplifyCFG.cpp) — `SimplifyCFGOpt` |
| Pass | [SimplifyCFGPass.cpp](https://github.com/llvm/llvm-project/blob/main/llvm/lib/Transforms/Scalar/SimplifyCFGPass.cpp) |
| Options | [SimplifyCFGOptions.h](https://github.com/llvm/llvm-project/blob/main/llvm/include/llvm/Transforms/Utils/SimplifyCFGOptions.h) |
| Debug | `-debug-only=simplifycfg` |

PassBuilder schedules multiple SimplifyCFG runs. Early: often `NeedCanonicalLoop = true`, limited switch transforms. Late: `ForwardSwitchCondToPhi`, `ConvertSwitchRangeToICmp`, lookup tables, `SinkCommonInsts` / `HoistCommonInsts` as configured.

```bash
clang -S -emit-llvm -O1 -Xclang -disable-llvm-passes file.c -o file.ll
opt -passes=simplifycfg -S file.ll -o file.cfg.ll
```

Pipeline strings can pass options, e.g. `simplifycfg<switch-to-lookup>` (syntax evolves—check `opt --help` / PassBuilder parsing for your version).

## 4. Core concepts

### Fold uncond br / merge blocks

If `A` ends with `br label %B` and `B` has only `A` as predecessor, splice `B` into `A` (PHIs in `B` become direct values). Chains of empty blocks collapse.

### Remove unreachable

Blocks with no predecessors (after folding) or dominated-unreachable regions are erased; critical for cleaning `unreachable` terminators after constant folding of conditions.

### Branch to common destination

`br i1 %c, label %X, label %X` becomes unconditional. More generally, threading through blocks that only re-branch can create common destinations that then fold.

### Two-entry PHI → select

Classic diamond:

```llvm
br i1 %c, label %T, label %F
T: br label %J
F: br label %J
J: %v = phi i32 [ %a, %T ], [ %b, %F ]
```

can become `select i1 %c, i32 %a, i32 %b` when side effects / costs allow folding `T`/`F` away (`phi-node-folding-threshold`, `bonus-inst-threshold`).

### Switch transforms

- **Lookup table:** dense case values → global table + GEP/load (`ConvertSwitchToLookupTable`, gated by TTI `shouldBuildLookupTables`).
- **Range icmp:** contiguous cases → `icmp` range checks (`ConvertSwitchRangeToICmp`).
- **Bit test:** small maskable sets → and/icmp patterns.
- **Forward switch cond to PHI:** replace PHI inputs with the switch condition value when cases are identity-like (`ForwardSwitchCondToPhi`).

### Hoist / sink common instructions

With `HoistCommonInsts` / `SinkCommonInsts`, identical ops in multiple predecessors/successors move to a common point—related to Day 17’s Sink, but driven by CFG identity matching inside SimplifyCFG.

### Speculation knobs

- `SpeculateBlocks` / `-simplifycfg-speculate-blocks`
- `phi-node-folding-threshold`
- `bonus-inst-threshold` (`BonusInstThreshold` in options)

These control how much code may be speculated into a predecessor to eliminate a branch.

### `NeedCanonicalLoop`

When true, transforms that would destroy LoopSimplify form (unique preheader, dedicated exits, single latch) are suppressed. Early pipeline needs this so LICM / IndVars still see canonical loops. Late CFG simplification may set it false (`-keep-loops` CLI maps to this option when overridden).

## 5. The algorithm

Conceptual fixed point inside `SimplifyCFGOpt::run` / per-block simplify:

1. Iterate blocks (worklist); for each, attempt terminator-focused simplifications.
2. Try fold-branch, merge, unreachable removal, switch rewrite, select formation, common hoist/sink as options allow.
3. Use DominatorTree (and often AssumptionCache, TTI, TargetLibraryInfo) for legality and profitability.
4. Reschedule / revisit predecessors when a change opens a new fold.
5. Stop when no rewrite applies or iteration caps hit.

Exact helper names are numerous (`simplifyOnce`, `FoldBranchToCommonDest`, switch conversion helpers, etc.). Prefer navigating from `SimplifyCFGOpt` methods and the options they consult rather than memorizing every static function.

## 6. Where to read the code

1. [SimplifyCFGOptions.h](https://github.com/llvm/llvm-project/blob/main/llvm/include/llvm/Transforms/Utils/SimplifyCFGOptions.h) — flag meanings and defaults.
2. [SimplifyCFGPass.cpp](https://github.com/llvm/llvm-project/blob/main/llvm/lib/Transforms/Scalar/SimplifyCFGPass.cpp) — how PassBuilder sets early vs late options; CLI overrides (`-forward-switch-cond`, `-switch-range-to-icmp`, `-keep-loops`, …).
3. [SimplifyCFG.cpp](https://github.com/llvm/llvm-project/blob/main/llvm/lib/Transforms/Utils/SimplifyCFG.cpp) — `SimplifyCFGOpt`, folding thresholds, switch and select transforms.
4. TTI hooks for lookup tables (`shouldBuildLookupTables`) in target TTI files.
5. Tests under `llvm/test/Transforms/SimplifyCFG/`.

## 7. Worked before/after

```c
int abs_like(int c, int a, int b) {
  if (c)
    return a;
  return b;
}
```

```bash
clang -S -emit-llvm -O1 -Xclang -disable-llvm-passes abs_like.c -o abs_like.ll
opt -passes=simplifycfg -S abs_like.ll -o abs_like.cfg.ll
```

**Before:** diamond with PHI in the join (or separate rets).

**After:** often a single `select` (or direct `ret` of a select), fewer blocks.

Switch example shape:

```llvm
switch i32 %x, label %def [
  i32 0, label %c0
  i32 1, label %c1
  i32 2, label %c2
]
```

Late SimplifyCFG with lookup enabled and cooperative TTI may replace the switch with a table load of result values when each case is a simple constant return.

## 8. Reading order for today

1. Skim `SimplifyCFGOptions` field list end-to-end.
2. Read how `SimplifyCFGPass` applies early vs late defaults and CLI overrides.
3. Pick three transforms in SimplifyCFG.cpp: merge/uncond fold, PHI→select, one switch path.
4. Run examples with `-debug-only=simplifycfg`.
5. [Day-18-1-Extra-SimplifyCFG-DeepDive.md](Day-18-1-Extra-SimplifyCFG-DeepDive.md) then [Day-18-2-Extra-SimplifyCFG-Examples.md](Day-18-2-Extra-SimplifyCFG-Examples.md).
6. Note interaction with LoopSimplify (`NeedCanonicalLoop`) before Day 21.

## 9. Hands-on commands

```bash
clang -S -emit-llvm -O1 -Xclang -disable-llvm-passes t.c -o t.ll
opt -passes=simplifycfg -S t.ll -o t.out.ll
opt -passes=simplifycfg -debug-only=simplifycfg -S t.ll -o /dev/null 2>cfg.log

# Explore knobs (names stable enough to try; confirm with --help)
opt -passes=simplifycfg -phi-node-folding-threshold=1 -S t.ll
opt -passes=simplifycfg -bonus-inst-threshold=2 -S t.ll
```

Compare IR before/after with a dense `switch` at `-O2` vs isolated `simplifycfg` on unoptimized IR.

## 10. Pitfalls and invariants

- **Early vs late options differ.** A transform that “always” happens at -O3 may be disabled early to protect loops.
- **`NeedCanonicalLoop`:** folding a latch edge wrongly can break LoopSimplify; respect the flag when reading failures.
- **Speculation safety:** folding a block into a predecessor can execute side effects that used to be conditional—thresholds and safety checks matter.
- **Lookup tables need TTI approval** (`shouldBuildLookupTables`); bare `opt` without a triple may behave differently than `llc`-bound pipelines.
- **Infinite-looking churn:** worklists can revisit blocks; bugs show up as huge compile times—note iteration caps when reading.
- **Opaque pointers:** switch lookup GEPs use `ptr` to tables; do not expect typed pointer element types in modern IR.
- **Not a replacement for InstCombine:** arithmetic folds still belong there; SimplifyCFG focuses on CFG + select/switch shapes.

### Profile metadata and branch folding

When `!prof` branch weights exist, some folds become profitability-sensitive: eliminating a cold edge is different from flattening a 50/50 diamond into a select on a target where select is expensive. Read transform sites that consult `extractBranchWeights` / BPI-adjacent helpers when your IR carries PGO metadata (Days 84–85). Without metadata, thresholds alone drive speculation.

### Verifier-driven experiments

```bash
opt -passes='simplifycfg,verify' -S t.ll -o t.out.ll
```

Illegal select formation (speculating a trapping load) should refuse the fold rather than produce IR that fails verification. If you ever see verify failures after SimplifyCFG, file them as bugs—the pass is expected to maintain well-formed SSA and dominance.

## 11. Neighbors

- **Sink (Day 17)** — instruction motion without CFG rewrite; SimplifyCFG’s common sink is CFG-driven.
- **LoopSimplify (Day 21)** — produces form SimplifyCFG must sometimes preserve.
- **InstCombine** — creates constants that let SimplifyCFG delete edges.
- **JumpThreading** — related CFG threading with different heuristics.
- **LowerSwitch** — late alternative lowering when switches remain.
- **CGP / CodeGenPrepare** — further select/branch preferences toward ISel.

## 12. Check yourself

**Q1.** Pass name? Primary class in the util file?  
**A1.** `simplifycfg`; `SimplifyCFGOpt`.

**Q2.** Which options header configures behavior?  
**A2.** [SimplifyCFGOptions.h](https://github.com/llvm/llvm-project/blob/main/llvm/include/llvm/Transforms/Utils/SimplifyCFGOptions.h).

**Q3.** What does `NeedCanonicalLoop` protect?  
**A3.** LoopSimplify-style canonical loop shape (preheader / latch / exits) from aggressive CFG edits.

**Q4.** Name three switch-related transforms.  
**A4.** Lookup table, range-to-icmp, bit-test / arithmetic forms; also forward-switch-cond-to-phi.

**Q5.** Debug type string?  
**A5.** `simplifycfg` → `-debug-only=simplifycfg`.

**Q6.** Why might PHI→select not fire?  
**A6.** Cost above `phi-node-folding-threshold` / bonus thresholds, unsafe side effects, or options disabling speculation.

Tip: implementation and option wiring in [Day-18-1-Extra-SimplifyCFG-DeepDive.md](Day-18-1-Extra-SimplifyCFG-DeepDive.md); worked cases in [Day-18-2-Extra-SimplifyCFG-Examples.md](Day-18-2-Extra-SimplifyCFG-Examples.md).

### One-line regression habit

When a CFG-related miscompile appears at `-O2`, reduce with `llvm-reduce` guided by `opt -passes=simplifycfg` alone before blaming later loop passes—many “loop” bugs are actually simplifycfg destroying shape with `NeedCanonicalLoop` off.
