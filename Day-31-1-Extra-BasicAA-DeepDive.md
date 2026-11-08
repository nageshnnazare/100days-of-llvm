# Day 31 Extra: BasicAA — Deep Dive

Source: [BasicAliasAnalysis.cpp](https://github.com/llvm/llvm-project/blob/main/llvm/lib/Analysis/BasicAliasAnalysis.cpp), [BasicAliasAnalysis.h](https://github.com/llvm/llvm-project/blob/main/llvm/include/llvm/Analysis/BasicAliasAnalysis.h). Parent: [Day-31-0-BasicAA.md](Day-31-0-BasicAA.md).

## Diagram

```text
   %p = alloca i32
   %q = alloca i32
   store 1, ptr %p
   %v = load ptr %q          BasicAA: different allocas
                             never alias -> load is not
                             clobbered by the store

   %g = getelementptr i32, ptr %base, i64 %i
   %h = getelementptr i32, ptr %base, i64 %j
   if %i and %j are not known equal,
   BasicAA says MayAlias
```

BasicAA answers "must", "no", or "may" from the IR shape: distinct allocas do not alias, a GEP into an object does not alias a different object, and two GEPs of the same base alias unless the offsets are provably different. It does not look at types. TBAA is a different analysis that adds type-based facts on top.

## `BasicAAResult` in the AA stack

`BasicAAResult` extends `AAResultBase`. The analysis `BasicAA` is registered as `FUNCTION_ALIAS_ANALYSIS("basic-aa", ...)`. `AAManager` / `AAResults` chains several AA results; a query returns the **most precise** answer the stack can agree on (with rules for combining). Clients like GVN, LICM, DSE, MemCpyOpt, LoopIdiom call `AAResults::alias`.

Teaching rule: **debug with `aa-pipeline=basic-aa` alone** to see BasicAA’s contribution, then add `tbaa` to see the stack improve.

## `DecomposedGEP`

Inside `BasicAAResult`, `DecomposedGEP` holds:

- Base `Value *`
- Constant `Offset`
- Variable index scaled contributions

`DecomposeGEPExpression` walks GEP operators (and some folds), accumulating constants through the DataLayout’s type sizes. Opaque pointers mean indices are interpreted via the GEP’s source element types still carried on the GEP instruction.

`subtractDecomposedGEPs` forms a difference used by `aliasGEP`:

- If the difference proves the accessed byte ranges cannot overlap → `NoAlias`.
- If bases MustAlias and offsets match → toward `MustAlias` / `PartialAlias` depending on sizes.
- Leftover symbolic indices without constraints → `MayAlias`.

## Path through `alias` / `aliasGEP`

1. Trivial equal pointers → MustAlias (modulo sizes → Partial).
2. `getUnderlyingObject` style identity: different allocas/globals → NoAlias.
3. Argument/global special cases with `noalias`.
4. GEP vs GEP or GEP vs object via `aliasGEP`.
5. Phi/select: optionally recurse (`basic-aa-recphi`) with depth caps.
6. Cseq / errno and call ModRef helpers in separate methods.

Read `aliasGEP` carefully—it is the mathematical heart.

## ModRef

`getModRefInfo` for calls uses:

- Function attributes (`readnone`, `readonly`, `writeonly`, memory effects)
- Argument memory locations vs `nocapture`/`readonly` args
- Whether the call can touch errno (special aliasErrno paths)

This is how LICM decides if a call clobbers a load.

## Captures

Capture tracking (ValueTracking / CaptureTracking family) informs whether a local alloca address could have been stored into a global and thus alias an unknown pointer. If the alloca did not escape, unknown parameters typically NoAlias with it.

## Scoped noalias and alias.scopes

BasicAA understands `!alias.scope` / `!noalias` metadata pairs (Clang emits them for some restrict cases / OpenMP). This is **not** TBAA; it is explicitly scoped disjointness.

## `invariant.group`

Used for C++ `const` / vtable-like invariance. Rules are subtle—when reading, treat as “same invariant.group load can CSE” more than as a general alias hammer. Do not overclaim in notes beyond “BasicAA has explicit handling.”

## Recursion limits and compile time

Phi cycles (`%p = phi [%p2, %bb], ...`) can explode. Depth limits and the `basic-aa-recphi` flag exist so BasicAA stays linear-ish. If you see MayAlias that a human would call NoAlias through a shallow phi, try enabling recursive phi analysis and re-running `aa-eval`.

## What BasicAA will never do

- Type-based disambiguation of `int*` vs `float*` (needs TBAA).
- Full Andersen/Steensgaard whole-program points-to.
- Trusting UB from strict aliasing without metadata.
- Modeling atomic concurrency beyond conservative ModRef.

## Provenance caution

After `ptrtoint` / `inttoptr`, or `getelementptr` with undef indices, BasicAA should stay conservative. When teaching, emphasize: **MayAlias is not failure; it is honesty.**

## Experiment design

1. Two allocas — expect NoAlias.
2. Two plain pointer args — MayAlias.
3. `noalias` args — NoAlias.
4. GEP `p` vs `p+1` with i32 access — NoAlias if sizes fit.
5. Same with overlapping i64 access at `p` and `p` — Must/Partial.
6. Add `!tbaa` and compare `basic-aa` alone vs `basic-aa,tbaa` pipelines.

## PartialAlias and DSE

DSE and related passes must treat PartialAlias as “do not assume disjoint.” A store that PartialAlias a later store may still kill some bytes but not others—byte-level reasoning is specialized. BasicAA reporting PartialAlias is already a strong signal compared to MayAlias; clients still need size/offset math.

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

## Source navigation checklist (Day 31 DeepDive)

1. Open the primary .cpp linked from the main note.
2. Jump to DEBUG_TYPE / cl::opt block.
3. Find the public `run` method or analysis entry.
4. Find the core legality helper and the core transform helper.
5. Grep remark strings / LLVM_DEBUG strings you saw experimentally.
6. Note which analyses are required vs preserved.
7. Skim one Failure path and one Success path to completion.

## PartialAlias and DSE

DSE and related passes must treat PartialAlias as “do not assume disjoint.” A store that PartialAlias a later store may still kill some bytes but not others—byte-level reasoning is specialized. BasicAA reporting PartialAlias is already a strong signal compared to MayAlias; clients still need size/offset math.

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

## Source navigation checklist (Day 31 DeepDive)

1. Open the primary .cpp linked from the main note.
2. Jump to DEBUG_TYPE / cl::opt block.
3. Find the public `run` method or analysis entry.
4. Find the core legality helper and the core transform helper.
5. Grep remark strings / LLVM_DEBUG strings you saw experimentally.
6. Note which analyses are required vs preserved.
7. Skim one Failure path and one Success path to completion.

## PartialAlias and DSE

DSE and related passes must treat PartialAlias as “do not assume disjoint.” A store that PartialAlias a later store may still kill some bytes but not others—byte-level reasoning is specialized. BasicAA reporting PartialAlias is already a strong signal compared to MayAlias; clients still need size/offset math.

## IR hygiene for experiments

Always generate IR with `clang -S -emit-llvm -O1 -Xclang -disable-llvm-passes` so functions are not marked `optnone`. Opaque pointers (`ptr`) are mandatory in modern IR examples; do not revive typed pointer syntax in new notes.

When comparing before/after, prefer `diff -u` on files produced by the same opt pipeline with one pass removed. That isolates the transform better than comparing against clang -O3, which runs dozens of passes.
