# Day 26 Extra: Loop Rotation — Implementation Deep Dive

This note walks the code path from `LoopRotatePass` into `LoopRotationUtils`. Pair with [Day-26-0-LoopRotate.md](Day-26-0-LoopRotate.md) and the examples in [Day-26-2-Extra-LoopRotate-Examples.md](Day-26-2-Extra-LoopRotate-Examples.md).

## Diagram

```text
   before rotate                         after loop-rotate

   header:                               preheader / guard
     br i1 %cond, %body, %exit              br i1 %cond, %body, %exit
   body:                                 body:          (the old header
     ... work ...                          ... work ...   test is now
     br label %header                      %cond2 = ...   at the latch)
   latch is the header                     br i1 %cond2, %body, %exit

   the first iteration's test is hoisted,
   so the body is a do-while
```

Rotation turns a while-loop into a do-while by peeling the condition into a guard and putting a copy of the test at the latch. The latch then branches on a value computed in the body, which is the shape IndVarSimplify and the vectorizer want. A loop that is already a do-while, or whose header is too profitable to duplicate, stays put.

## Pass shell vs util core

[LoopRotation.cpp](https://github.com/llvm/llvm-project/blob/main/llvm/lib/Transforms/Scalar/LoopRotation.cpp) is intentionally thin. `LoopRotatePass::run` receives a `Loop &`, `LoopAnalysisManager`, and `LoopStandardAnalysisResults` (dominator tree, SCEV, TTI, AssumptionCache, optional MemorySSA, LoopInfo).

It computes a **threshold**:

- If header duplication is enabled (and the function is not `minsize`, unless overridden, or the loop is forced for vectorization), use `DefaultRotationThreshold` from `-rotation-max-header-size`.
- Otherwise threshold can be `0`, which effectively disables automatic duplication-heavy rotation while still allowing the util to run in constrained modes.

It optionally wraps MemorySSA in a `MemorySSAUpdater`, builds a `SimplifyQuery`, and calls:

```text
LoopRotation(&L, &AR.LI, &AR.TTI, &AR.AC, &AR.DT, &AR.SE,
             MSSAU ? &*MSSAU : nullptr, SQ, /*RotationOnly=*/false,
             Threshold, /*IsUtilMode=*/false, PrepareForLTO, CheckExitCount);
```

On change it preserves the standard loop analyses and MemorySSA when present. `printPipeline` exposes pipeline options such as `header-duplication`, `prepare-for-lto`, and `check-exit-count`.

## The `LoopRotate` class

Defined in [LoopRotationUtils.cpp](https://github.com/llvm/llvm-project/blob/main/llvm/lib/Transforms/Utils/LoopRotationUtils.cpp) (anonymous namespace). Constructor stores MaxHeaderSize, analysis pointers, `RotationOnly`, `IsUtilMode`, `PrepareForLTO`, and `CheckExitCount`.

Public util API in [LoopRotationUtils.h](https://github.com/llvm/llvm-project/blob/main/llvm/include/llvm/Transforms/Utils/LoopRotationUtils.h):

```cpp
bool LoopRotation(Loop *L, LoopInfo *LI, const TargetTransformInfo *TTI,
                  AssumptionCache *AC, DominatorTree *DT, ScalarEvolution *SE,
                  MemorySSAUpdater *MSSAU, const SimplifyQuery &SQ,
                  bool RotationOnly, unsigned Threshold, bool IsUtilMode,
                  bool PrepareForLTO = false, bool CheckExitCount = false);
```

That function constructs `LoopRotate` and calls `processLoop`.

## `processLoop` responsibilities

While reading `processLoop`, keep these phases in mind:

### 1. Form and early exits

The loop should have the usual simplified structure (preheader, latch). If the loop is already a bottom-tested form and `profitableToRotateLoopExitingLatch` says there is little to gain, return false. That helper uses SCEV exit-count information when `CheckExitCount` is enabled—avoiding churn when the latch already exits cleanly.

### 2. Header size / metrics

CodeMetrics-style counting (instruction costs, calls, etc.) decides whether the header fits under `MaxHeaderSize`. Oversized headers increment `NumNotRotatedDueToHeaderSize` and bail. This is the mechanical meaning of `-rotation-max-header-size`.

### 3. Folding and hoisting into the preheader

Instructions in the header that are loop-invariant or otherwise foldable may be simplified with the `SimplifyQuery` and moved to the preheader (`NumInstrsHoisted`). Anything that remains and is needed for the exit test may be **cloned** into the preheader for the guard and into the latch for the bottom test (`NumInstrsDuplicated`).

Cloning uses the usual `CloneBasicBlock` / value-mapping infrastructure from Transforms/Utils, with DominatorTree updates via `DomTreeUpdater` patterns used elsewhere in the file.

### 4. Rewriting control flow

The classic rewrite:

- Split or reuse the preheader so a **guard branch** can skip the loop when the initial condition is false.
- Make the old body (or a suitable block) the new header.
- Redirect the latch to perform the exit compare and branch either to the header or to the exit.
- Fix PHI nodes: preheader predecessors supply start values; latch predecessors supply inductions.

Zero-trip edges can receive branch-weight metadata biased toward “usually enters” (`ZeroTripCountWeights`).

### 5. SSA / LCSSA / MemorySSA

After CFG edits:

- SSA form for non-memory values is repaired (SSAUpdater / local utilities as needed).
- LCSSA phis at exits are preserved or recreated so outer uses still see single-block definitions.
- If `MSSAU` is non-null, MemorySSA is updated incrementally instead of rebuilt from scratch—important because LICM and other clients may run next in the same loop pipeline.

### 6. `RotationOnly` vs full mode

When `RotationOnly` is true, the util sticks to rotation and skips related latch simplifications. The pass typically passes `false` so latch cleanup can accompany rotation. `IsUtilMode` changes profitability heuristics when other transforms call the util as a helper rather than as the dedicated pass.

## Interaction with LTO and vectorize metadata

`PrepareForLTO` (pass option / `-rotation-prepare-for-lto`) adjusts behavior for the prepare-for-LTO pipeline stage—useful when you want rotation decisions consistent with later IPO. Separately, `hasVectorizeTransformation` (from LoopUtils) can keep a non-zero threshold even when header duplication is generally disabled, because the vectorizer expects rotated loops.

## Debug tracing

```bash
opt -debug-only=loop-rotate -passes='loop-simplify,lcssa,loop-rotate' -S in.ll
```

`DEBUG_TYPE` is `"loop-rotate"` in both the pass and the utils file, so one debug flag covers both. Look for messages about header size failures, profitability skips, and successful rotations.

## What “preserving LoopSimplify/LCSSA” means in practice

Callers in the loop pass manager assume:

- Unique preheader remains unique (or is recreated).
- Dedicated exits and latch structure stay analyzable.
- Values defined in the loop and used outside still flow through LCSSA phis.

If a bug in rotation broke LCSSA, IndVarSimplify and LoopUnroll would miscompile or assert. When reading patches to LoopRotationUtils, treat LCSSA/DT/MSSA updates as first-class correctness, not bookkeeping.

## How LICM consumes the result

LICM (separate pass) asks: “is this instruction invariant, and is it safe to hoist to the preheader?” For a top-tested loop, hoisting a load from the header into the preheader can execute the load even when `n == 0`. After rotation, the load lives in a block that only runs when the guard passed (or it can be hoisted to the *new* preheader *after* the guard). That is why rotation is scheduled before LICM in standard pipelines.

## Common failure modes when reading IR

- **Multiple exiting blocks:** rotation still applies in many cases, but the latch-exit ideal is strongest for single-exit loops; extra exits complicate profitability.
- **Indirectbr / unusual terminators:** rarely appear in C; generally unsupported for this reshape.
- **Switches in the header:** treated carefully; large or side-effecting headers fail the size/safety checks.
- **Exception pads:** landing pads and cleanup pads are outside the happy path for this transform.

## Suggested source reading path

1. `LoopRotatePass::run` — threshold policy only.
2. `LoopRotation` → `LoopRotate::processLoop` — main algorithm.
3. `profitableToRotateLoopExitingLatch` — when *not* to rotate.
4. Instruction cloning / PHI rewrite sections — where most bugs hide.
5. One failing and one succeeding test in `llvm/test/Transforms/LoopRotate/`.

## Tie-in to later days

- Day 27 Loop Idiom often runs on loops that are already simplified/rotated.
- Day 28 SimpleLoopUnswitch also duplicates loop regions; compare cost models (`unswitch-threshold` vs header size).
- Day 34–35 SCEV trip counts explain when the zero-trip guard is statically removable (`n` known positive).

## Cloning and PHI rewrite details

When the header is cloned for the guard and for the latch test, SSA uses must be remapped carefully:

- Values defined in the header and used in the body need consistent clones or shared defs after the reshape.
- Loop-carried PHIs change which predecessor provides which value: the preheader edge carries the original start, the latch edge carries the updated IV.
- LCSSA phis outside must be updated if exit edges move.

Read the cloning portion of LoopRotationUtils.cpp alongside `llvm/Transforms/Utils/Cloning.h` helpers. Bugs here show up as dominator verifier failures or wrong exit values—not as silent performance issues.

## Profitability of already-exiting latches

`profitableToRotateLoopExitingLatch` encodes the “already a do-while” case. If the latch already exits and SCEV exit counts (when CheckExitCount is on) suggest no further benefit, the pass returns early. This prevents churn that would only rename blocks. When teaching, take a do-while C loop, emit IR, and confirm rotation is a no-op or near-no-op.

## Prepare-for-LTO mode

`PrepareForLTO` / `-rotation-prepare-for-lto` adjusts rotation for the prepare-for-thin/full-LTO pipeline. The idea is to make loop shapes consistent with what later IPO-time loop passes expect, without over-duplicating in the early compile phase. Treat it as a pipeline-policy switch rather than a different algorithm.

## Metrics and statistics

Useful statistics when experimenting:

- `NumNotRotatedDueToHeaderSize`
- `NumInstrsHoisted`
- `NumInstrsDuplicated`

They tell you whether the pass considered the loop and which strategy (hoist vs duplicate) dominated. Combine with `-stats` on `opt`.

## Connection to Day 35 trip counts

Zero-trip guards are the CFG manifestation of “BTC formulas that carefully handle `n==0`.” After rotation, body execution count matches the intuitive trip count more cleanly for LICM. Reading Day 35 right after this DeepDive reinforces why the guard exists.

### Study note addendum 1


- Values defined in the header and used in the body need consistent clones or shared defs after the reshape.
- Loop-carried PHIs change which predecessor provides which value: the preheader edge carries the original start, the latch edge carries the updated IV.
- LCSSA phis outside must be updated if exit edges move.

Read the cloning portion of LoopRotationUtils.cpp alongside `llvm/Transforms/Utils/Cloning.h` helpers. Bugs here show up as dominator verifier failures or wrong exit values—not as silent performance issues.

## Profitability of already-exiting latches

`profitableToRotateLoopExitingLatch` encodes the “already a do-while” case. If the latch already exits and SCEV exit counts (when CheckExitCount is on) suggest no further benefit, the pass returns early. This prevents churn that would only rename blocks. When teaching, take a do-while C loop, emit IR, and confirm rotation is a no-op or near-no-op.

## Prepare-for-LTO mode

`PrepareForLTO` / `-rotation-prepare-for-lto` adjusts rotation for the prepare-for-thin/full-LTO pipeline. The idea is to make loop shapes consistent with what later IPO-time loop passes expect, without over-duplicating in the early compile phase. Treat it as a pipeline-policy switch rather than a different algorithm.

## Metrics and statistics

Useful statistics when experimenting:

- `NumNotRotatedDueToHeaderSize`

