# Day 32 Extra: TBAA — Deep Dive

Source: [TypeBasedAliasAnalysis.cpp](https://github.com/llvm/llvm-project/blob/main/llvm/lib/Analysis/TypeBasedAliasAnalysis.cpp). Overview: [Day-32-0-TBAA.md](Day-32-0-TBAA.md).

## Diagram

```text
   struct S { int a; float b; };

   store i32 %x, ptr %s            !tbaa !A   (the int field)
   %f = load float, ptr %sb        !tbaa !B   (the float field)

   TBAA tree:

        omnipotent char
         /          \
       int         float

   !A and !B have no common child
   path that says they may overlap,
   so this load and store are NoAlias
   -- unless one of them is char,
   which aliases everything
```

TBAA is metadata on loads and stores naming a node in a type tree. Two accesses alias only if the tree says their types are allowed to overlap. A `char*` (or an access tagged as omnipotent char) aliases every type, which is why a byte-wise copy blocks TBAA. The metadata is not a proof if the frontend lied; LLVM treats a violated TBAA tag as undefined behavior.

## File orientation

The file documents metadata shapes in comments (omnipotent char, scalar type nodes, struct path tags). Read the comment block near the top before the code—it is the best short spec.

Key methods:

- `TypeBasedAAResult::alias`
- `TypeBasedAAResult::Aliases`
- `getModRefInfo` overloads
- `MDNode::isTBAAVtableAccess`
- `shouldUseTBAA`

## Struct-path access tags

A modern tag conceptually says: “this access is to type AccessTy at offset Off within BaseTy.” Two tags NoAlias when the type DAG says their access types cannot refer to the same memory—**unless** one path involves the char root exception.

Field-sensitive example: `struct S { int a; float b; }` — stores to `a` and `b` get tags with different offsets; TBAA can prove NoAlias between those fields even though both live in one object (for type-based rules; path TBAA encodes this).

## Old scalar TBAA

Older tags were a tree of type names without offsets. Compatibility code still recognizes them. When grepping lit tests, you will see both styles; prefer understanding struct-path.

## `Aliases` logic (how to study it)

1. Fast reject if either node is null.
2. Walk / compare type paths according to the metadata grammar.
3. Treat root/char specially.
4. Return whether overlap is possible.

Do not memorize every MDNode operand index—follow the parser helpers in the same file.

## Interaction with AAResults

Order in the aa-pipeline matters for evaluation experiments, but AAResults combines answers according to AA rules: any NoAlias from a layer that is allowed to speak can refine MayAlias from another. Typically you run `basic-aa,tbaa` together.

TBAA never turns MustAlias into NoAlias incorrectly: if BasicAA already proved MustAlias (same address), type metadata must not “disprove” that—the combiner logic keeps soundness. (If you ever see a conflict, it is a bug.)

## `!tbaa.struct` and memcpy

MemCpyOpt and friends use `!tbaa.struct` on memcpy intrinsics to describe which relative byte ranges correspond to which field tags. That allows further AA after a memcpy of a struct without treating the whole aggregate as omnipotent char.

## Clang emission notes (behavioral)

- Strict aliasing on → tags on most loads/stores of typed C/C++ memory.
- `char*` / `unsigned char*` → tags that participate in the omnipotent rule.
- Unions are carefully modeled; union TBAA is a frequent source of subtle bugs—treat as advanced.
- `-fno-strict-aliasing` → no (or empty) TBAA reliance.

## TypeSanitizer

`TypeBasedAA::run` inspects whether the function uses type sanitization and constructs `TypeBasedAAResult` accordingly. Under sanitizers, TBAA-based opts may be restricted so instrumentation remains meaningful.

## Vtable helper

`MDNode::isTBAAVtableAccess` identifies vtable-related TBAA nodes for special cases in the optimizer (devirtualization adjacency). Mention only as a landmark when grepping.

## Common failure modes in teaching IR

1. Hand-written stores without `!tbaa` — TBAA silent.
2. Mixing `-fno-strict-aliasing` bitcode with mental strict-aliasing models.
3. Expecting TBAA to distinguish two `int*` arguments — same type **may** alias; TBAA does not help; need `restrict`/noalias.
4. Confusing “same TBAA type ⇒ MustAlias” — false; same type only means they *may* alias.

## Suggested exercises

1. Dump MDNodes for `int` vs `float` store tags; draw the tree including the root.
2. Same for two fields of a struct.
3. Compare `aa-eval` pipelines.
4. Rebuild with `-fno-strict-aliasing` and show tags vanish / AA worsens.

## Frontend contract wording

TBAA metadata means: “the frontend asserts these accesses follow the language aliasing rules encoded in the type tree.” LLVM optimizers may assume that assertion when returning NoAlias. They should not delete a store solely because “UB means we can do anything”—correct uses of TBAA are NoAlias proofs between typed ops, enabling CSE/DSE/LICM across those ops.

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

## Source navigation checklist (Day 32 DeepDive)

1. Open the primary .cpp linked from the main note.
2. Jump to DEBUG_TYPE / cl::opt block.
3. Find the public `run` method or analysis entry.
4. Find the core legality helper and the core transform helper.
5. Grep remark strings / LLVM_DEBUG strings you saw experimentally.
6. Note which analyses are required vs preserved.
7. Skim one Failure path and one Success path to completion.

## Frontend contract wording

TBAA metadata means: “the frontend asserts these accesses follow the language aliasing rules encoded in the type tree.” LLVM optimizers may assume that assertion when returning NoAlias. They should not delete a store solely because “UB means we can do anything”—correct uses of TBAA are NoAlias proofs between typed ops, enabling CSE/DSE/LICM across those ops.

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

## Source navigation checklist (Day 32 DeepDive)

1. Open the primary .cpp linked from the main note.
2. Jump to DEBUG_TYPE / cl::opt block.
3. Find the public `run` method or analysis entry.
4. Find the core legality helper and the core transform helper.
5. Grep remark strings / LLVM_DEBUG strings you saw experimentally.
6. Note which analyses are required vs preserved.
7. Skim one Failure path and one Success path to completion.

## Frontend contract wording

TBAA metadata means: “the frontend asserts these accesses follow the language aliasing rules encoded in the type tree.” LLVM optimizers may assume that assertion when returning NoAlias. They should not delete a store solely because “UB means we can do anything”—correct uses of TBAA are NoAlias proofs between typed ops, enabling CSE/DSE/LICM across those ops.

## IR hygiene for experiments

Always generate IR with `clang -S -emit-llvm -O1 -Xclang -disable-llvm-passes` so functions are not marked `optnone`. Opaque pointers (`ptr`) are mandatory in modern IR examples; do not revive typed pointer syntax in new notes.

When comparing before/after, prefer `diff -u` on files produced by the same opt pipeline with one pass removed. That isolates the transform better than comparing against clang -O3, which runs dozens of passes.

## Verifier and analysis invalidation

After a transform, `opt -passes='verify'` (or the verifier embedded in pipelines) should still succeed. Loop passes must preserve LCSSA and simplified form when they advertise those preservations. Analysis managers invalidate SCEV, AA, and MemorySSA according to the preserved-analyses set returned by the pass—reading `PreservedAnalyses` in the pass’s `run` method teaches what clients may keep.

## Reading upstream tests

For any pass, open `llvm/test/Transforms/<PassName>/` or `llvm/test/Analysis/<Name>/` and skim the smallest lit test. The `RUN:` lines show the canonical `opt` invocation, including required simplify/lcssa partners. Copying a RUN line into your shell is the fastest way to get a known-good baseline before editing IR by hand.
