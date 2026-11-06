# Day 27 Extra: Loop Idiom — Implementation Deep Dive

Companion to [Day-27-0-LoopIdiom.md](Day-27-0-LoopIdiom.md). Source of truth: [LoopIdiomRecognize.cpp](https://github.com/llvm/llvm-project/blob/main/llvm/lib/Transforms/Scalar/LoopIdiomRecognize.cpp).

## Diagram

```text
   for (i = 0; i < n; ++i)
     a[i] = 0;

   loop-idiom recognizes a store of a
   loop-invariant byte across a unit stride

              |
              v
   memset(a, 0, n * sizeof(a[0]))

   the loop is gone; the library call remains
```

Loop idiom recognition looks for a loop that is really a memory primitive: a zeroing store becomes `llvm.memset`, a copy becomes `llvm.memcpy`, and a popcount-style shift loop can become a bit-count intrinsic. The replacement is profitable because the library call lowers to a tuned sequence. A loop that writes a non-invariant value, or that has a carried dependence, is not a memset.

## Pass structure

`DEBUG_TYPE` is `"loop-idiom"`. Hidden cl::opt flags disable subsets (`disable-loop-idiom-all`, `disable-loop-idiom-memset`, `disable-loop-idiom-memcpy`, and others for hash/shift idioms depending on revision).

The pass runs per loop (loop pass infrastructure). It collects analyses: ScalarEvolution, AliasAnalysis, DominatorTree, LoopInfo, TargetLibraryInfo, TargetTransformInfo, and often MemorySSA for precise dependence.

`LoopIdiomRecognize` (the worker) scans the loop and dispatches to idiom-specific routines. Debug lines of the form `loop-idiom Scanning: F[...]` mark entry.

## Memset recognition walk

### Finding the store

The pass looks for a simple store in the loop body. The stored value should be a constant that memset can materialize (commonly zero, or a byte-splat pattern). The pointer operand is run through SCEV:

- Expect an AddRec with loop = current loop.
- Step size should match the store’s memory size (or a legally coalesced larger store pattern when the implementation supports wider idiom stores).

### Contiguous coverage

From the AddRec start, step, and backedge-taken count, the total byte length is derived. If SCEV cannot compute a usable count, recognition fails. Gaps (stride larger than store size without a fill explanation) fail.

### Safety

- **Volatile** → reject.
- Other memory instructions in the loop are checked so they do not read the region being zeroed in a way that would observe intermediate states the memset would collapse, and so they do not write overlapping locations that would change semantics.
- AA answers MayAlias queries; MayAlias with an unmodeled op typically blocks the transform.
- Atomic stores are outside the simple memset happy path.

### Emission

On success, IRBuilder creates `llvm.memset` with alignment from the original stores when provable, length from SCEV (zext/trunc to the intrinsic’s length type), and `isVolatile = false`. The loop is deleted or gutted; dead phis/IVs cleaned up.

Optimization remarks (`OptimizationRemark` / `OptimizationRemarkMissed`) explain success (`ProcessLoopStridedStore`) or failure (`SizeStrideUnequal`, `LoopMayAccessStore`, …)—useful when teaching with `-pass-remarks=loop-idiom` / missed remarks.

## Memcpy / memmove recognition walk

### Pattern

A load of size S from `SrcAddRec` and a store of size S to `DstAddRec` in lockstep.

### Overlap

For memcpy, the pass must prove the source and destination objects/regions do not overlap for any iteration. That usually means NoAlias between the underlying objects or SCEV-based disjointness of the iterated ranges. If overlap cannot be disproved, either reject or, when the code path supports it, emit `memmove` only when that is semantically required and safe.

### Alias failures

Debug/remarks such as `LoopMayAccessLoad` appear when another memory op may touch the load’s region. Same idea as memset safety.

## Popcount idiom

Search the file for popcount comments: loops that iterate by clearing the lowest set bit (`x = x & (x - 1)` style) and increment a counter until zero map to `llvm.ctpop`. Preconditions typically include:

- Finite loop / analyzable trip related to bit width.
- No extra side effects in the body.
- Profitability: recognizing tiny popcount on targets where a hardware popcount exists; avoid expanding into worse code when the “loop” is already trivial.

The matcher is stricter than memset; small IR perturbations break it—good example fodder.

## Shift-until-zero / bit-scan idioms

Later sections detect loops that shift a value each iteration until it becomes zero, or shift until a bit test succeeds. DEBUG strings include `shift-until-zero idiom detected!` and `shift-until-bittest idiom detected!`. These may lower to `llvm.cttz` / `llvm.ctlz` or related expansions after cost checks (`Shift is too costly, not beneficial`).

Read these after memset/memcpy; they show how LIR shares SCEV loop finiteness checks (`Bad block/backedge count`, `Can not prove the loop is finite`).

## Role of TTI and TLI

- **TLI:** `getLibFunc` / `has` for `memset`, `memcpy`, `memmove`. Fortified variants (`__memset_chk`) complicate matching elsewhere (SimplifyLibCalls); LIR mainly cares that *some* usable API exists when emitting libcalls.
- **TTI:** cost of alternatives; whether a memset intrinsic is preferable to a scalar loop on this CPU; vector-friendly decisions for some idioms.

Always pass a realistic `-mtriple` when experimenting with libcall emission:

```bash
opt -mtriple=x86_64-unknown-linux-gnu -passes='loop-idiom' -S in.ll
```

## Interaction with nearby passes

1. **Loop simplify / rotate / indvars** — cleaner IVs make AddRec matching easier.
2. **MemCpyOpt** ([MemCpyOptimizer.cpp](https://github.com/llvm/llvm-project/blob/main/llvm/lib/Transforms/Scalar/MemCpyOptimizer.cpp)) — after LIR emits memset/memcpy, MemCpyOpt may merge adjacent memsets or forward stores into memcpy.
3. **InstCombine** — folds alignment, length casts, memcpy of zero length, etc.
4. **Loop vectorize** — if LIR fails, LV might still vectorize the store loop; LIR is the better outcome when it hits.

## Failure mode catalog (for debugging)

| Symptom | Likely cause |
|---------|----------------|
| No change on obvious zeroing loop | volatile, non-affine index, TLI disabled, idiom flag disabled |
| Copy loop survives | src/dst MayAlias, size mismatch, struct padding holes |
| Popcount not recognized | extra body instructions, wrong bit-clear pattern |
| Works in clang -O3, not in opt | missing triple/datalayout/TLI or different pipeline order |

## Reading strategy

1. Skim class fields / constructor for analysis handles.
2. Read memset from “find store” through IR emission only.
3. Diff against memcpy—the alias/overlap logic is the delta.
4. Spot-read popcount + one shift idiom.
5. Run Examples file cases with `-debug-only=loop-idiom` and match log lines to source.

## Memset emission details

When forming `llvm.memset`, alignment is taken from the original stores when a common alignment is proven; otherwise a conservative alignment is used. The length type follows the intrinsic overload (`i32`/`i64`) and may require casts from SCEV-expanded counts. The volatile bit must stay false for ordinary idiom lowering—volatile loops are rejected earlier.

After emission, the old loop blocks are deleted or emptied; DT/LI updates fall out of the loop-pass infrastructure. Dead PHIs are cleaned so later passes do not see a zombie IV.

## Alias set thinking

The safety checks are essentially: “is there any MayAlias memory op in the loop that is not the idiom store/load itself?” A debug print of other memory ops in the body is the first thing to look at when a “perfect” memset loop fails. Lifetime intrinsics and assumes are usually ignored as non-clobbering; ordinary calls are fatal unless proven readnone/inaccessiblememonly in ways the recognizer accepts.

## Popcount finiteness

Popcount recognition needs a proof the loop is finite—for `unsigned x` with `x &= x-1` that is natural (at most bitwidth iterations). If the IR uses a wider type or a weird update, finiteness proofs fail (`Can not prove the loop is finite` style debug lines). Always reconnect these failure strings to the source.

## Cost checks on shift idioms

Even after pattern match, TTI may say materializing `cttz`/`ctlz` or an equivalent is not profitable relative to a tiny loop (`Shift is too costly`). That is a profitability no, not a legality no—useful when comparing `-mcpu` effects (Day 40).

## Relation to MemCpyOpt

LoopIdiom creates mem* intrinsics from loops. MemCpyOpt merges and forwards among mem* and stores. They share vocabulary but different entry patterns: loops vs straight-line chains. See Day 33.

### Study note addendum 1


After emission, the old loop blocks are deleted or emptied; DT/LI updates fall out of the loop-pass infrastructure. Dead PHIs are cleaned so later passes do not see a zombie IV.

## Alias set thinking

The safety checks are essentially: “is there any MayAlias memory op in the loop that is not the idiom store/load itself?” A debug print of other memory ops in the body is the first thing to look at when a “perfect” memset loop fails. Lifetime intrinsics and assumes are usually ignored as non-clobbering; ordinary calls are fatal unless proven readnone/inaccessiblememonly in ways the recognizer accepts.

## Popcount finiteness

Popcount recognition needs a proof the loop is finite—for `unsigned x` with `x &= x-1` that is natural (at most bitwidth iterations). If the IR uses a wider type or a weird update, finiteness proofs fail (`Can not prove the loop is finite` style debug lines). Always reconnect these failure strings to the source.

## Cost checks on shift idioms

Even after pattern match, TTI may say materializing `cttz`/`ctlz` or an equivalent is not profitable relative to a tiny loop (`Shift is too costly`). That is a profitability no, not a legality no—useful when comparing `-mcpu` effects (Day 40).

## Relation to MemCpyOpt

LoopIdiom creates mem* intrinsics from loops. MemCpyOpt merges and forwards among mem* and stores. They share vocabulary but different entry patterns: loops vs straight-line chains. See Day 33.
## Memset emission details


### Study note addendum 2

## Alias set thinking

The safety checks are essentially: “is there any MayAlias memory op in the loop that is not the idiom store/load itself?” A debug print of other memory ops in the body is the first thing to look at when a “perfect” memset loop fails. Lifetime intrinsics and assumes are usually ignored as non-clobbering; ordinary calls are fatal unless proven readnone/inaccessiblememonly in ways the recognizer accepts.

## Popcount finiteness

Popcount recognition needs a proof the loop is finite—for `unsigned x` with `x &= x-1` that is natural (at most bitwidth iterations). If the IR uses a wider type or a weird update, finiteness proofs fail (`Can not prove the loop is finite` style debug lines). Always reconnect these failure strings to the source.

## Cost checks on shift idioms

Even after pattern match, TTI may say materializing `cttz`/`ctlz` or an equivalent is not profitable relative to a tiny loop (`Shift is too costly`). That is a profitability no, not a legality no—useful when comparing `-mcpu` effects (Day 40).

## Relation to MemCpyOpt

LoopIdiom creates mem* intrinsics from loops. MemCpyOpt merges and forwards among mem* and stores. They share vocabulary but different entry patterns: loops vs straight-line chains. See Day 33.
## Memset emission details

When forming `llvm.memset`, alignment is taken from the original stores when a common alignment is proven; otherwise a conservative alignment is used. The length type follows the intrinsic overload (`i32`/`i64`) and may require casts from SCEV-expanded counts. The volatile bit must stay false for ordinary idiom lowering—volatile loops are rejected earlier.

After emission, the old loop blocks are deleted or emptied; DT/LI updates fall out of the loop-pass infrastructure. Dead PHIs are cleaned so later passes do not see a zombie IV.

