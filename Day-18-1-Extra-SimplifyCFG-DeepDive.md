# Day 18 Extra: SimplifyCFG — Deep Dive

Companion to [Day-18-0-SimplifyCFG.md](Day-18-0-SimplifyCFG.md). Examples: [Day-18-2-Extra-SimplifyCFG-Examples.md](Day-18-2-Extra-SimplifyCFG-Examples.md).

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

## 1. Two layers: pass vs util

| Layer | Path | Job |
|-------|------|-----|
| Pass | [SimplifyCFGPass.cpp](https://github.com/llvm/llvm-project/blob/main/llvm/lib/Transforms/Scalar/SimplifyCFGPass.cpp) | New PM adaptation, acquire analyses, apply `SimplifyCFGOptions`, CLI overrides |
| Util | [SimplifyCFG.cpp](https://github.com/llvm/llvm-project/blob/main/llvm/lib/Transforms/Utils/SimplifyCFG.cpp) | `SimplifyCFGOpt` rewrite engine |
| Options | [SimplifyCFGOptions.h](https://github.com/llvm/llvm-project/blob/main/llvm/include/llvm/Transforms/Utils/SimplifyCFGOptions.h) | Struct of knobs with builder setters |

Other passes also call into SimplifyCFG utilities for one-off cleanups; the dedicated pass is the bulk user.

## 2. `SimplifyCFGOptions` field tour

Defaults (as in the header; verify on your tree):

| Field | Default (typical) | Meaning |
|-------|-------------------|---------|
| `BonusInstThreshold` | 1 | Extra budget when folding/speculating |
| `ForwardSwitchCondToPhi` | false | Switch cond forwarding into PHIs |
| `ConvertSwitchRangeToICmp` | false | Contiguous cases → icmp |
| `ConvertSwitchToArithmetic` | false | Arithmetic switch lowering |
| `ConvertSwitchToLookupTable` | false | Table-driven switch |
| `NeedCanonicalLoop` | true | Preserve canonical loops |
| `HoistCommonInsts` | false | Hoist identical ops |
| `SinkCommonInsts` | false | Sink identical ops |
| `SimplifyCondBranch` | true | Cond branch folds |
| `SpeculateBlocks` | true | Allow block speculation |
| `SpeculateUnpredictables` | false | Speculate past `unpredictable` metadata |
| `AC` | nullptr | Optional AssumptionCache |

Builder style: `SimplifyCFGOptions().forwardSwitchCondToPhi(true).needCanonicalLoops(false)`.

PassBuilder sets different combinations for early vs late instrumentation. CLI flags in SimplifyCFGPass (e.g. `-forward-switch-cond`, `-switch-range-to-icmp`, `-keep-loops`) **override** when explicitly passed (`applyCommandLineOverridesToOptions`).

## 3. Run path

Approximate new-PM flow:

1. `SimplifyCFGPass::run(Function &F, FunctionAnalysisManager &AM)`.
2. Fetch DominatorTree, TTI, TLI, AssumptionCache as required by current options.
3. Construct / adjust `SimplifyCFGOptions`.
4. Invoke simplification entry that constructs `SimplifyCFGOpt` and runs until stable.
5. Return preserved analyses (CFG often changed → DT/LI invalidated unless carefully updated—read the pass’s preservation logic for your revision).

Many transforms update DT in place; still treat “CFG changed” as invalidating loop structures unless the pass claims preservation.

## 4. PHI → select path

Key knobs:

- `-phi-node-folding-threshold` (cl::opt in SimplifyCFG.cpp, historically default 2)
- `BonusInstThreshold`

The engine estimates the cost of speculating predecessor instructions into a select form. Cheap arithmetic folds; calls with side effects block; thresholds cut off large blocks.

Two-entry PHIs are the bread-and-butter case. More predecessors need different strategies (nested selects or leave alone).

## 5. Switch lowering helpers

Study in this order inside SimplifyCFG.cpp (search symbol names):

1. Conditions for converting to lookup tables (density, result uniformity, TTI `shouldBuildLookupTables`).
2. Range-to-icmp conversion gated by `ConvertSwitchRangeToICmp`.
3. Bit-test patterns for sparse small sets.
4. `forwardSwitchConditionToPHI` when `ForwardSwitchCondToPhi` is set.

Lookup tables usually create a private global constant array and replace the switch with index arithmetic + load. That interacts with later ConstantMerge / data layout.

## 6. Hoist / sink common instructions

When `HoistCommonInsts` is enabled, identical instructions at the start of multiple successors of a block can move to the block itself (dominated execution always happens).

When `SinkCommonInsts` is enabled, identical instructions at the end of multiple predecessors can move to the common successor (often creating a PHI for differing operands—or using matching operands directly).

This is **not** the Day 17 `sink` pass. Same English verb; different code path and triggering conditions (identity across preds/succs vs use-dominated single instruction motion).

## 7. Speculative block folding

`SpeculateBlocks` / `-simplifycfg-speculate-blocks` controls whether a successor’s instructions may be pulled into the predecessor, eliminating a branch. Safety: no trapping ops unless proven safe; respect `unpredictable` metadata unless `SpeculateUnpredictables`.

Bonus thresholds allow slightly more speculation when it removes a branch on a hot pattern.

## 8. `NeedCanonicalLoop` in practice

Canonical loop form (Day 21): preheader, dedicated exits, single latch/backedge.

SimplifyCFG transforms that merge the preheader into something else, create extra backedges, or share exit blocks with external preds break later loop passes. With `NeedCanonicalLoop == true`, those edits are skipped.

Late pipelines often relax this after major loop opts finished—or accept re-running LoopSimplify.

## 9. Analyses consulted

- **DominatorTree** — merge legality, hoist points.
- **TargetTransformInfo** — lookup tables, sometimes cost of selects vs branches.
- **TargetLibraryInfo** — recognized libcalls when folding.
- **AssumptionCache** — `@llvm.assume` facts for condition folding.
- Optionally profile metadata (`!prof`) for branch profitability when present.

## 10. Debugging

```bash
opt -passes=simplifycfg -debug-only=simplifycfg -S in.ll -o out.ll 2>cfg.log
```

Also useful:

```bash
opt -passes='simplifycfg,print<domtree>' -S in.ll
```

Bisect options by constructing explicit `SimplifyCFGOptions` in a unit-test-like opt pipeline string when available, or by flipping CLI overrides one at a time.

## 11. What SimplifyCFG does **not** own

- Full jump threading across non-trivial blocks (JumpThreading pass).
- Loop-rotate / loop-unswitch (dedicated loop passes).
- General instruction combine on non-CFG patterns (InstCombine).
- IR sinking solely by use domination without common-identity (Day 17 Sink).
- Ensuring LCSSA (LCSSA pass / LoopSimplify options).

## 12. Reading strategy for the huge .cpp

SimplifyCFG.cpp is large. Do **not** read linearly end-to-end on day one.

1. Grep for `SimplifyCFGOpt::`.
2. Grep for option field uses (`Options.NeedCanonicalLoop`, `SinkCommonInsts`, …).
3. Open one transform you care about (select, switch lookup) and read only that helper cluster.
4. Use lit tests as an index: each test file name maps to a transform family.

## 13. Early vs late checklist

When something “should simplify” but does not:

1. Are you on an early options set with lookup/forward/range disabled?
2. Is `NeedCanonicalLoop` blocking the fold?
3. Is the triple/TTI disabling lookup tables?
4. Is the cost above phi/bonus thresholds?
5. Is side effect / volatile blocking speculation?

## 14. Common transform name → grep map

| Goal | Search strings in SimplifyCFG.cpp |
|------|-----------------------------------|
| Uncond / merge | `MergeBlockIntoPredecessor`, empty-block folds |
| Common dest | `FoldBranchToCommonDest` |
| PHI → select | `FoldTwoEntryPHINode`, phi folding threshold uses |
| Switch → LUT | `SwitchToLookupTable`, `shouldBuildLookupTables` |
| Switch range | `ConvertSwitchRangeToICmp`, range helpers |
| Forward cond | `forwardSwitchConditionToPHI` |
| Hoist/sink common | `HoistCommon*`, `SinkCommon*` / Options fields |
| Speculate block | `SpeculateBlocks`, `bonus-inst-threshold` |

Use this table as an index; function names shift slightly across releases, but the grep stems stay stable enough to land you in the right cluster.

## 15. PassBuilder wiring note

When reading how `-O2` differs from `-O3` for CFG, open PassBuilder’s pipeline construction (search `SimplifyCFGOptions` in [PassBuilderPipelines.cpp](https://github.com/llvm/llvm-project/blob/main/llvm/lib/Passes/PassBuilderPipelines.cpp)). Note which stage enables `convertSwitchToLookupTable`, `sinkCommonInsts`, and `needCanonicalLoops(false)`. That file is the ground truth for “early vs late” beyond the options header defaults.

## 16. Takeaway

SimplifyCFG is a **parameterized CFG rewrite engine**. Memorizing every transform is impossible; memorizing the options struct and where PassBuilder sets it pays off every time you debug -O2 IR shape.
