# Day 37: MemorySSA

MemorySSA overlays a sparse SSA form on memory operations: `MemoryDef`, `MemoryUse`, and `MemoryPhi`, enabling fast “who clobbers this load?” queries. It replaces many MemDep use cases (Day 33). Deep dive: [Day-37-1-Extra-MemorySSA-DeepDive.md](Day-37-1-Extra-MemorySSA-DeepDive.md). Examples: [Day-37-2-Extra-MemorySSA-Examples.md](Day-37-2-Extra-MemorySSA-Examples.md).

## Why this exists

Walking the CFG for every load (MemDep) does not scale inside iterative loop passes. MemorySSA builds an explicit def/use graph of memory states once; the **walker** answers clobber queries by climbing that graph, disambiguating with AA along the way.

## Diagram

```text
   MemorySSA

   entry:
     liveOnEntry
   block:
     1 = MemoryDef(liveOnEntry)    store %p
     MemoryUse(1)                  load  %p   -- uses def 1
     2 = MemoryDef(1)              call @g    -- may clobber
     MemoryUse(2)                  load  %p   -- uses def 2, not def 1
```

MemorySSA is a small SSA form for memory. Stores and calls that may write are `MemoryDef`s. Loads are `MemoryUse`s that point at the def that reaches them. A clobber is the nearest def, not a search backward through the block. Passes such as DSE, LICM, and EarlyCSE-with-MemorySSA walk this graph instead of asking MemDep from scratch.

## The idea

- Every store/call that may write is a **MemoryDef**, numbered, chaining to a previous def.
- Every load (and read-only use) is a **MemoryUse** optimized to point at the nearest relevant def.
- At CFG merge points, **MemoryPhi** merges memory versions—like a PHI of “memory clocks.”
- A distinguished **liveOnEntry** def represents the state on function entry.

`ValueToMemoryAccess` maps IR instructions to these nodes.

`getClobberingMemoryAccess` walks from a use/def to the nearest dominating MemoryDef that **MayAlias** the query location (conservative: MayAlias counts as a clobber).

## Where it sits

| Item | Value |
|------|--------|
| Analysis | `memoryssa` |
| File | [llvm/lib/Analysis/MemorySSA.cpp](https://github.com/llvm/llvm-project/blob/main/llvm/lib/Analysis/MemorySSA.cpp) |
| Header | [MemorySSA.h](https://github.com/llvm/llvm-project/blob/main/llvm/include/llvm/Analysis/MemorySSA.h) |
| Updater | MemorySSAUpdater |
| Print | `opt -passes='print<memoryssa>'` |
| Debug | `-debug-only=memoryssa` |

Built from AA + DominatorTree.

Clients: LICM, GVN (memssa variants), EarlyCSE (`early-cse` memssa mode), MemCpyOpt, SimpleLoopUnswitch, DSE ([DeadStoreElimination.cpp](https://github.com/llvm/llvm-project/blob/main/llvm/lib/Transforms/Scalar/DeadStoreElimination.cpp)).

```bash
clang -S -emit-llvm -O1 -Xclang -disable-llvm-passes f.c -o f.ll
opt -passes='print<memoryssa>' -disable-output f.ll
```

## Core concepts

### Optimizing uses

`optimizeUses` (and related) tighten MemoryUse edges past defs that NoAlias the load. Without optimization, uses may point at looser defs—still correct, less precise.

### Conservativeness

If AA says MayAlias, the walker stops—even if a human thinks NoAlias. Better AA (TBAA, noalias) ⇒ better MSSA clobbers ⇒ better LICM/DSE.

### Incremental updates

Passes that insert/delete stores must use `MemorySSAUpdater` instead of discarding the whole analysis when possible—critical inside loop pipelines.

## The algorithm (build sketch)

1. RPO walk; create MemoryDefs for writing instructions; MemoryUses for reads.
2. Insert MemoryPhis where memory SSA would need merges (IDF-like on memory defs).
3. Optionally optimize uses with AA.
4. Walker answers queries by following defining accesses + phis.

## Where to read the code

1. Conceptual comment at the top of MemorySSA.h (excellent ASCII example).
2. Walker `getClobberingMemoryAccess`.
3. MemorySSAUpdater usage in LICM or SimpleLoopUnswitch.
4. `print<memoryssa>` on a diamond CFG (Day 33 Example 4).

## Worked printer mental model

```llvm
; 1 = MemoryDef(liveOnEntry)
  store i32 1, ptr %p
; MemoryUse(1)
  %v = load i32, ptr %p
```

After an intervening MayAlias store, the load’s use points at that store’s def.

## Reading order

1. MemoryDef/Use/Phi vocabulary.
2. Header comment example.
3. Walker + AA conservatism.
4. DeepDive + Examples.
5. Revisit Day 33 as legacy contrast.

## Hands-on

```bash
cat > mssa.c <<'EOF'
int f(int *p, int *q) {
  *p = 1;
  *q = 2;
  return *p;
}
EOF
clang -S -emit-llvm -O1 -Xclang -disable-llvm-passes mssa.c -o mssa.ll
opt -passes='print<memoryssa>' -disable-output mssa.ll
```

With and without `restrict` on `p` and `q`.

## Pitfalls

- Editing memory IR without updater.
- Expecting NoAlias precision without AA facts.
- Confusing MemoryPhi with value PHI—related idea, different universe.

## Neighbors

- MemDep (Day 33), AA (Days 31–32), LICM, DSE, SimpleLoopUnswitch (Day 28).

## Check yourself

1. **Q:** Three MemorySSA node kinds?  
   **A:** MemoryDef, MemoryUse, MemoryPhi.

2. **Q:** What is liveOnEntry?  
   **A:** The implicit memory def at function entry.

3. **Q:** Print pass?  
   **A:** `print<memoryssa>`.

4. **Q:** Why MayAlias hurts MSSA clients?  
   **A:** Walker treats MayAlias as clobber, blocking LICM/DSE.

5. **Q:** How do passes keep MSSA fresh?  
   **A:** `MemorySSAUpdater`.

## Walker caching pitfalls

Optimized MemoryUses cache clobber answers. If a pass updates AA-relevant IR but forgets to invalidate MSSA caches/updater state, walkers can return stale defs. That is a classic heisenbug. After manual IR edits in a pass, prefer updater APIs that clear optimized flags on affected uses.

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

## Extended hands-on checklist (Day 37)

1. Build IR with the disable-llvm-passes recipe.
2. Run the minimal pass pipeline named in this day’s main note.
3. Diff against a pipeline that adds one neighbor pass.
4. Re-run with the day’s `-debug-only` flag.
5. Skim one upstream lit test RUN line and reproduce it.
6. Write one counterexample IR that must not transform.
7. Cross-check a fact against the linked github source file.

## Walker caching pitfalls

Optimized MemoryUses cache clobber answers. If a pass updates AA-relevant IR but forgets to invalidate MSSA caches/updater state, walkers can return stale defs. That is a classic heisenbug. After manual IR edits in a pass, prefer updater APIs that clear optimized flags on affected uses.

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

## Extended hands-on checklist (Day 37)

1. Build IR with the disable-llvm-passes recipe.
2. Run the minimal pass pipeline named in this day’s main note.
3. Diff against a pipeline that adds one neighbor pass.
4. Re-run with the day’s `-debug-only` flag.
5. Skim one upstream lit test RUN line and reproduce it.
6. Write one counterexample IR that must not transform.
7. Cross-check a fact against the linked github source file.

## Walker caching pitfalls

Optimized MemoryUses cache clobber answers. If a pass updates AA-relevant IR but forgets to invalidate MSSA caches/updater state, walkers can return stale defs. That is a classic heisenbug. After manual IR edits in a pass, prefer updater APIs that clear optimized flags on affected uses.

## IR hygiene for experiments

Always generate IR with `clang -S -emit-llvm -O1 -Xclang -disable-llvm-passes` so functions are not marked `optnone`. Opaque pointers (`ptr`) are mandatory in modern IR examples; do not revive typed pointer syntax in new notes.

When comparing before/after, prefer `diff -u` on files produced by the same opt pipeline with one pass removed. That isolates the transform better than comparing against clang -O3, which runs dozens of passes.

## Verifier and analysis invalidation

After a transform, `opt -passes='verify'` (or the verifier embedded in pipelines) should still succeed. Loop passes must preserve LCSSA and simplified form when they advertise those preservations. Analysis managers invalidate SCEV, AA, and MemorySSA according to the preserved-analyses set returned by the pass—reading `PreservedAnalyses` in the pass’s `run` method teaches what clients may keep.

## Reading upstream tests

For any pass, open `llvm/test/Transforms/<PassName>/` or `llvm/test/Analysis/<Name>/` and skim the smallest lit test. The `RUN:` lines show the canonical `opt` invocation, including required simplify/lcssa partners. Copying a RUN line into your shell is the fastest way to get a known-good baseline before editing IR by hand.

## Debug and remarks together

Combine `-debug-only=<DEBUG_TYPE>` with `-pass-remarks` / `-pass-remarks-missed` for the same pass. Debug logs show internal decisions; remarks show the user-facing summary. When they disagree with your mental model, trust the code path that emits the remark—jump to that string in the .cpp file.
