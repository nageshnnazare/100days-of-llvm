# Day 37 Extra: MemorySSA — Deep Dive

Sources: [MemorySSA.h](https://github.com/llvm/llvm-project/blob/main/llvm/include/llvm/Analysis/MemorySSA.h), [MemorySSA.cpp](https://github.com/llvm/llvm-project/blob/main/llvm/lib/Analysis/MemorySSA.cpp). Parent: [Day-37-0-MemorySSA.md](Day-37-0-MemorySSA.md).

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

## Mental model from the header

The file header shows stores creating MemoryDefs `(ID) = MemoryDef(prev)` and loads as `MemoryUse(ID)`. Importantly:

- **MemoryUses are disambiguated** when optimized: a use lists only defs that may alias it.
- **MemoryDefs’ RHS chains are not fully disambiguated** the same way—def-def edges may connect through MayAlias/NoAlias mixes; walker + AA refine at query time.
- Each def also has a “defining access” used when iterating users.

LiveOnEntry is def ID 0 conceptually—the memory state before any function-local write.

## Building MemoryPhis

Similar to ordinary SSA construction: whenever multiple memory-def reaching definitions arrive at a block, insert a MemoryPhi. IDF (iterated dominance frontier) style placement keeps phis sparse. Without MemoryPhis, diamond CFGs could not name a single reaching memory version for a load at the join.

## The walker

`getClobberingMemoryAccess(MemoryAccess *, MemoryLocation)` (and instruction overloads):

1. Start from the access linked to the query instruction.
2. Walk upward through defining accesses.
3. At each MemoryDef, ask AA whether it aliases the location; if Yes/May → that def is the clobber.
4. At MemoryPhi, examine arguments (with visited/caching to handle loops).
5. Stop at liveOnEntry if nothing aliases.

Caching on optimized uses makes repeated LICM queries cheap.

## `optimizeUses`

After initial construction, MemoryUses may point at coarse defs. Optimization walks and rewires uses past NoAlias defs. Passes can call ensure-optimized APIs before querying. Conservative mode skips this—correct but fewer opts.

## MemorySSAUpdater

API themes (names in the Updater header):

- Create new MemoryAccess when inserting a store/load
- Remove access when deleting an instruction
- Move instructions between blocks
- Fix MemoryPhis when CFG edges change

SimpleLoopUnswitch (Day 28) and LICM are good call-site readers: they show real updater sequences after cloning or hoisting.

## Clients and what they ask

| Pass | Typical query |
|------|----------------|
| LICM | Clobber of invariant load in loop? |
| DSE | Is this store killed before any intervening use? |
| EarlyCSE-memssa | Same memory version + same address → CSE |
| MemCpyOpt | Nearest defs for merging/forwarding |
| SimpleLoopUnswitch | Update after clone |

DSE file: [DeadStoreElimination.cpp](https://github.com/llvm/llvm-project/blob/main/llvm/lib/Transforms/Scalar/DeadStoreElimination.cpp).

## Verification

`-verify-memoryssa` (when enabled in the build/pipeline) checks def-use well-formedness after updates. Failures usually mean a pass forgot to update MSSA after editing memory IR.

## Debug

```bash
opt -debug-only=memoryssa -passes='print<memoryssa>' -disable-output f.ll
```

## Relationship to MemDep

MemDep: demand-driven CFG walk + cache.  
MSSA: eager sparse graph + walker.  
Same AA backend limits both. Teaching MSSA after MemDep (Day 33) should feel like indexing the same search.

## Reading path

1. Header ASCII example twice.
2. Walker function.
3. optimizeUses overview.
4. One updater call sequence in LICM.
5. Examples with `print<memoryssa>`.

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

## Source navigation checklist (Day 37 DeepDive)

1. Open the primary .cpp linked from the main note.
2. Jump to DEBUG_TYPE / cl::opt block.
3. Find the public `run` method or analysis entry.
4. Find the core legality helper and the core transform helper.
5. Grep remark strings / LLVM_DEBUG strings you saw experimentally.
6. Note which analyses are required vs preserved.
7. Skim one Failure path and one Success path to completion.

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

## Source navigation checklist (Day 37 DeepDive)

1. Open the primary .cpp linked from the main note.
2. Jump to DEBUG_TYPE / cl::opt block.
3. Find the public `run` method or analysis entry.
4. Find the core legality helper and the core transform helper.
5. Grep remark strings / LLVM_DEBUG strings you saw experimentally.
6. Note which analyses are required vs preserved.
7. Skim one Failure path and one Success path to completion.

## Walker caching pitfalls

Optimized MemoryUses cache clobber answers. If a pass updates AA-relevant IR but forgets to invalidate MSSA caches/updater state, walkers can return stale defs. That is a classic heisenbug. After manual IR edits in a pass, prefer updater APIs that clear optimized flags on affected uses.

## IR hygiene for experiments

Always generate IR with `clang -S -emit-llvm -O1 -Xclang -disable-llvm-passes` so functions are not marked `optnone`. Opaque pointers (`ptr`) are mandatory in modern IR examples; do not revive typed pointer syntax in new notes.

When comparing before/after, prefer `diff -u` on files produced by the same opt pipeline with one pass removed. That isolates the transform better than comparing against clang -O3, which runs dozens of passes.
