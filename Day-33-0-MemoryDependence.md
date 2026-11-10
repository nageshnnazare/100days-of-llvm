# Day 33: Memory Dependence Analysis (`memdep`)

MemoryDependenceAnalysis answers “what is the nearest memory instruction this load/store depends on?” It is powerful, expensive, and largely **legacy** relative to MemorySSA—but still appears in some clients (historically GVN; still relevant teaching for MemCpyOpt). Deep dive: [Day-33-1-Extra-MemoryDependence-DeepDive.md](Day-33-1-Extra-MemoryDependence-DeepDive.md). Examples: [Day-33-2-Extra-MemoryDependence-Examples.md](Day-33-2-Extra-MemoryDependence-Examples.md).

## Why this exists

CSE, memcpy formation, and store forwarding need more than pairwise AA: they need the **closest** clobbering or defining memory op along CFG paths. MemDep walks predecessors (locally and non-locally) using AA to classify dependencies.

MemorySSA (Day 37) rephrases the same idea as an explicit SSA graph of MemoryDefs/Uses with nearly O(1) clobber queries after construction—so new code should prefer MSSA. MemDep remains important historically and for understanding why MSSA exists.

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

## The idea

For an instruction Q (usually load/store/call):

- **Local dependency:** scan backward in Q’s block until a memory op that MayAlias/MustAlias Q’s location (or a barrier).
- **Non-local dependency:** if the scan hits the top of the block, follow CFG predecessors, caching results—can be costly.
- Results classify as clobber, def, non-local, non-function-local, etc. (`MemDepResult`).

APIs:

- `MemoryDependenceResults::getDependency(Instruction *)`
- `getNonLocalPointerDependency(...)`

## Where it sits in the pipeline

| Item | Value |
|------|--------|
| Analysis name | `memdep` |
| File | [llvm/lib/Analysis/MemoryDependenceAnalysis.cpp](https://github.com/llvm/llvm-project/blob/main/llvm/lib/Analysis/MemoryDependenceAnalysis.cpp) |
| Class | `MemoryDependenceResults` |
| Debug | `-debug-only=memdep` |
| Famous client | [MemCpyOptimizer.cpp](https://github.com/llvm/llvm-project/blob/main/llvm/lib/Transforms/Scalar/MemCpyOptimizer.cpp) (pass often still called MemCpyOpt) |

```bash
# MemDep is an analysis; exercise via clients, e.g.:
opt -passes='memcpyopt' -S f.ll
```

## Core concepts

### Why expensive

Non-local queries crawl the CFG with AA at each memory instruction. Without tight caching, compile time blows up on large functions. The cache is **invalidated aggressively** when IR changes—another reason passes migrated away.

### Why MemorySSA replaced it for many clients

MSSA builds a sparse memory SSA form once; `getClobberingMemoryAccess` walks a short def chain. Incremental `MemorySSAUpdater` keeps it fresh. LICM, EarlyCSE-memssa, DSE, SimpleLoopUnswitch, and modern MemCpyOpt paths lean on MSSA.

### MemCpyOpt as the teaching client

MemCpyOpt uses dependence information to:

- Turn load→store loops / pairs into `memcpy`
- Forward stores into later loads / memcpys
- Merge adjacent memsets
- Call-slot optimizations

Whether a given build uses MemDep, MSSA, or both is version-dependent—read the includes/analysis acquisition in MemCpyOptimizer.cpp for your tree. The **dependence idea** is the same.

## The algorithm (local scan)

1. Start at Q; walk previous instructions in-block.
2. Skip non-memory IR.
3. For each memory op, query AA vs Q’s location.
4. On MustAlias store to same loc → def dependency for a load.
5. On MayAlias store → clobber.
6. On load–load: typically not a dependency for MemDep’s store-forwarding purposes (details in code).
7. If nothing found, mark need for non-local query.

Non-local: iterate predecessor blocks with a visited set; memoize.

## Where to read the code

1. `getDependency` / `getNonLocalPointerDependency` in MemoryDependenceAnalysis.cpp.
2. `MemDepResult` in the header.
3. MemCpyOptimizer.cpp — how a pass consumes dependence.
4. MemorySSA.cpp (Day 37) — the replacement model.

## Worked example (MemCpyOpt angle)

```c
void f(char *p) {
  p[0] = 1; p[1] = 2; p[2] = 3; p[3] = 4;
}
```

MemCpyOpt / store merging may collapse to a wider store or memset-like pattern when dependences show the stores are consecutive and unclobbered.

```c
void g(int *d, int *s) {
  int tmp = *s;
  *d = tmp;
}
```

May become memcpy of 4 bytes when profitable and dependence/AA allow.

## Reading order

1. Local vs non-local cost model.
2. `getDependency` implementation.
3. One MemCpyOpt transform.
4. Day 37 for MSSA contrast.

## Hands-on

```bash
cat > mc.c <<'EOF'
void copy4(char *d, char *s) {
  d[0]=s[0]; d[1]=s[1]; d[2]=s[2]; d[3]=s[3];
}
EOF
clang -S -emit-llvm -O1 -Xclang -disable-llvm-passes mc.c -o mc.ll
opt -passes='memcpyopt' -S mc.ll -o mc.out.ll
```

## Pitfalls

- Writing new passes on MemDep instead of MemorySSA.
- Assuming MemDep is gone—it still exists as `memdep`.
- Ignoring cache invalidation when manually editing IR in a pass.

## Neighbors

- BasicAA/TBAA (Days 31–32), MemorySSA (Day 37), LoopIdiom (Day 27).

## Check yourself

1. **Q:** Two primary query APIs?  
   **A:** `getDependency`, `getNonLocalPointerDependency`.

2. **Q:** Why is non-local MemDep slow?  
   **A:** CFG predecessor walks with AA at each memory instruction.

3. **Q:** Modern replacement for many clients?  
   **A:** MemorySSA.

4. **Q:** Analysis pass name?  
   **A:** `memdep`.

5. **Q:** Name a client transform file.  
   **A:** `MemCpyOptimizer.cpp` (MemCpyOpt).

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

## Extended hands-on checklist (Day 33)

1. Build IR with the disable-llvm-passes recipe.
2. Run the minimal pass pipeline named in this day’s main note.
3. Diff against a pipeline that adds one neighbor pass.
4. Re-run with the day’s `-debug-only` flag.
5. Skim one upstream lit test RUN line and reproduce it.
6. Write one counterexample IR that must not transform.
7. Cross-check a fact against the linked github source file.

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

## Extended hands-on checklist (Day 33)

1. Build IR with the disable-llvm-passes recipe.
2. Run the minimal pass pipeline named in this day’s main note.
3. Diff against a pipeline that adds one neighbor pass.
4. Re-run with the day’s `-debug-only` flag.
5. Skim one upstream lit test RUN line and reproduce it.
6. Write one counterexample IR that must not transform.
7. Cross-check a fact against the linked github source file.

## MemCpyOpt transform catalog (study list)

While reading MemCpyOptimizer.cpp, make a checklist of transforms: load/store pair → memcpy; store sequence → memset; memcpy/memmove folding; byval/call slot; stack-move opts. For each, write the MemDep/MSSA query it needs (“nearest clobber of this load”). That catalog cements why Day 33 exists even as MSSA dominates.

## IR hygiene for experiments

Always generate IR with `clang -S -emit-llvm -O1 -Xclang -disable-llvm-passes` so functions are not marked `optnone`. Opaque pointers (`ptr`) are mandatory in modern IR examples; do not revive typed pointer syntax in new notes.

When comparing before/after, prefer `diff -u` on files produced by the same opt pipeline with one pass removed. That isolates the transform better than comparing against clang -O3, which runs dozens of passes.
