# Day 33 Extra: Memory Dependence — Deep Dive

Sources: [MemoryDependenceAnalysis.cpp](https://github.com/llvm/llvm-project/blob/main/llvm/lib/Analysis/MemoryDependenceAnalysis.cpp), [MemoryDependenceAnalysis.h](https://github.com/llvm/llvm-project/blob/main/llvm/include/llvm/Analysis/MemoryDependenceAnalysis.h), client [MemCpyOptimizer.cpp](https://github.com/llvm/llvm-project/blob/main/llvm/lib/Transforms/Scalar/MemCpyOptimizer.cpp). Parent: [Day-33-0-MemoryDependence.md](Day-33-0-MemoryDependence.md).

## Diagram

```text
   store 1, ptr %p          <-- clobber of a load of %p
   call void @g()           <-- may clobber, unless AA says no
   store 2, ptr %q          <-- not a clobber of %p if NoAlias
   %v = load i32, ptr %p    <-- query starts here and walks up

   MemDep reports:
     Def    a store that produces the loaded value
     Clobber a write that may overlap
     NonFuncLocal  the answer is outside this block
```

MemoryDependenceAnalysis walks backward from a load or store until it finds the nearest memory instruction that must define the value or might overlap it. EarlyCSE and GVN use that answer to forward a store into a load. The walk is local and stops at calls it cannot see through, which is why MemorySSA later replaced it for many passes: one upfront graph instead of a walk per query.

## Class layout

`MemoryDependenceResults` holds:

- AA, DT, and related handles
- Dense caches from instructions to `MemDepResult`
- Non-local pointer dependency caches
- Invalidation hooks responding to deletion/RAUW of instructions

`MemoryDependenceAnalysis` is the new-PM analysis (`FUNCTION_ANALYSIS("memdep", ...)`). A legacy `MemoryDependenceWrapperPass` may still exist for old-PM bridges.

`DEBUG_TYPE` is `"memdep"`.

## `MemDepResult` vocabulary

Results distinguish (names from the header):

- Local definitive dependencies (clobber vs def)
- Non-local (need inter-block query)
- Non-function-local (depends on something outside the function—e.g., unknown through a call)
- Other sentinel states for “unknown” / “none”

When reading client code, always check *which* state you got before assuming a specific `Instruction *` clobber.

## `getDependency` walk

Implementation sketch:

1. Look up cache; return if fresh.
2. If instruction has no memory effect, trivial result.
3. Scan the basic block backward from the instruction.
4. Use AA + instruction kind to decide def vs clobber vs skip.
5. Volatile/atomic get special conservative treatment.
6. If the scan exhausts the block, return a non-local marker so the client/API continues with `getNonLocalPointerDependency`.

Comments in the .cpp note that some clients always follow up with the non-local API when marked non-local—and that caching interacts carefully with that.

## Non-local pointer dependency

`getNonLocalPointerDependency` fills a vector of per-predecessor results:

- Starts from blocks that reach the query instruction
- For each predecessor path, finds the nearest dependence
- Uses a visited set / worklist to avoid cycles
- Caches aggressively but must invalidate when CFG or memory IR changes

This is the compile-time cliff: a load in a block with many predecessors in a mega-function triggers wide walks.

## Invalidation

Any pass that inserts/deletes stores or changes CFG must invalidate MemDep. Aggressive invalidation is safer but causes recomputation—another reason MSSA’s updater model won for iterative loop passes.

## MemCpyOpt: how dependence feeds transforms

Open MemCpyOptimizer.cpp and locate analysis acquisition (MemDep and/or MSSA depending on version). Typical dependences used:

1. **Load–store to memcpy:** a sequence of loads from `s+i` and stores to `d+i` with clean dependences and no intervening clobbers → form `llvm.memcpy`.
2. **Store–store forwarding / merging:** consecutive stores with known bytes and no reads between → wider store or memset.
3. **Memset merging:** two memsets to adjacent regions with same value → one memset.
4. **Call slot opt:** store to alloca then call by pointer → sometimes rewrite to use the call’s memory directly when dependences prove safety.

Even when MSSA is the engine, the *reasoning* matches MemDep’s “nearest clobber” mental model—MSSA just indexes it differently.

## Historical clients

- Older GVN used MemDep heavily; modern GVN/MemorySSA variants differ.
- EarlyCSE has a MemorySSA mode (`early-cse<memssa>` / similar).
- When reading ancient tutorials naming MemDep+GVN, translate to today’s MSSA-aware passes.

## Honest status in the new PM

- Analysis still registered and usable.
- New transforms should not start from MemDep without a strong reason.
- Understanding MemDep remains valuable for reading old patches and for grasping why MSSA’s walker looks the way it does.

## Debugging tips

```bash
opt -debug-only=memdep -passes='memcpyopt' -S in.ll
```

Also compare:

```bash
opt -passes='print<memoryssa>' -S in.ll
```

Same function, two views of memory dependences.

## Reading sequence

1. Header `MemDepResult` + public methods.
2. `getDependency` body only.
3. Skim non-local (appreciate complexity; no need to memorize).
4. One MemCpyOpt transform end-to-end.
5. Day 37 MemorySSA intro.

## MemCpyOpt transform catalog (study list)

While reading MemCpyOptimizer.cpp, make a checklist of transforms: load/store pair → memcpy; store sequence → memset; memcpy/memmove folding; byval/call slot; stack-move opts. For each, write the MemDep/MSSA query it needs (“nearest clobber of this load”). That catalog cements why Day 33 exists even as MSSA dominates.

## IR hygiene for experiments

Always generate IR with `clang -S -emit-llvm -O1 -Xclang -disable-llvm-passes` so functions are not marked `optnone`. Opaque pointers (`ptr`) are mandatory in modern IR examples; do not revive typed pointer syntax in new notes.

When comparing before/after, prefer `diff -u` on files produced by the same opt pipeline with one pass removed. That isolates the transform better than comparing against clang -O3, which runs dozens of passes.

## Verifier and analysis invalidation

After a transform, `opt -passes='verify'` (or the verifier embedded in pipelines) should still succeed. Loop passes must preserve LCSSA and simplified form when they advertise those preservations. Analysis managers invalidate SCEV, AA, and MemorySSA according to the preserved-analyses set returned by the pass—reading `PreservedAnalyses` in the pass’s `run` method teaches what clients may keep.

## Reading upstream tests

For any pass, open `llvm/test/Transforms/<PassName>/` or `llvm/test/Analysis/<Name>/` and skim the smallest lit test. The `RUN:` lines show the canonical `opt` invocation, including required simplify/lcssa partners. Copying a RUN line into your shell is the fastest way to get a known-good baseline before editing IR by hand.

## Debug and remarks together

Combine `-debug-only=<DEBUG_TYPE>` with `-pass-remarks` / `-pass-remarks-missed` for the same pass. Debug logs show internal decisions; remarks show the user-facing summary. When they disagree with your mental model, trust the code path that emits the remark—jump to that string in the .cpp file.

## How this day connects forward and back

Loop and AA topics form a dependency chain: simplify/LCSSA → rotate/unswitch → SCEV/indvars → idiom/interchange → vectorize. Alias analyses (BasicAA, TBAA) and MemorySSA sit underneath almost every memory transform. TLI/TTI decide availability and profitability at the edges of the mid-end. Re-read neighboring day notes when a failure looks like “wrong pass” but is actually a missing analysis fact.

## Source navigation checklist (Day 33 DeepDive)

1. Open the primary .cpp linked from the main note.
2. Jump to DEBUG_TYPE / cl::opt block.
3. Find the public `run` method or analysis entry.
4. Find the core legality helper and the core transform helper.
5. Grep remark strings / LLVM_DEBUG strings you saw experimentally.
6. Note which analyses are required vs preserved.
7. Skim one Failure path and one Success path to completion.

## MemCpyOpt transform catalog (study list)

While reading MemCpyOptimizer.cpp, make a checklist of transforms: load/store pair → memcpy; store sequence → memset; memcpy/memmove folding; byval/call slot; stack-move opts. For each, write the MemDep/MSSA query it needs (“nearest clobber of this load”). That catalog cements why Day 33 exists even as MSSA dominates.

## IR hygiene for experiments

Always generate IR with `clang -S -emit-llvm -O1 -Xclang -disable-llvm-passes` so functions are not marked `optnone`. Opaque pointers (`ptr`) are mandatory in modern IR examples; do not revive typed pointer syntax in new notes.

When comparing before/after, prefer `diff -u` on files produced by the same opt pipeline with one pass removed. That isolates the transform better than comparing against clang -O3, which runs dozens of passes.

## Verifier and analysis invalidation

After a transform, `opt -passes='verify'` (or the verifier embedded in pipelines) should still succeed. Loop passes must preserve LCSSA and simplified form when they advertise those preservations. Analysis managers invalidate SCEV, AA, and MemorySSA according to the preserved-analyses set returned by the pass—reading `PreservedAnalyses` in the pass’s `run` method teaches what clients may keep.

## Reading upstream tests

For any pass, open `llvm/test/Transforms/<PassName>/` or `llvm/test/Analysis/<Name>/` and skim the smallest lit test. The `RUN:` lines show the canonical `opt` invocation, including required simplify/lcssa partners. Copying a RUN line into your shell is the fastest way to get a known-good baseline before editing IR by hand.

## Debug and remarks together

Combine `-debug-only=<DEBUG_TYPE>` with `-pass-remarks` / `-pass-remarks-missed` for the same pass. Debug logs show internal decisions; remarks show the user-facing summary. When they disagree with your mental model, trust the code path that emits the remark—jump to that string in the .cpp file.

## How this day connects forward and back

Loop and AA topics form a dependency chain: simplify/LCSSA → rotate/unswitch → SCEV/indvars → idiom/interchange → vectorize. Alias analyses (BasicAA, TBAA) and MemorySSA sit underneath almost every memory transform. TLI/TTI decide availability and profitability at the edges of the mid-end. Re-read neighboring day notes when a failure looks like “wrong pass” but is actually a missing analysis fact.
