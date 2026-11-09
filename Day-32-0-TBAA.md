# Day 32: Type-Based Alias Analysis (TBAA)

TBAA encodes C/C++ strict-aliasing rules as metadata on memory ops. LLVM uses it only to prove **NoAlias**, never to “delete UB.” Deep dive: [Day-32-1-Extra-TBAA-DeepDive.md](Day-32-1-Extra-TBAA-DeepDive.md). Examples: [Day-32-2-Extra-TBAA-Examples.md](Day-32-2-Extra-TBAA-Examples.md).

## Why this exists

In C, an `int*` and a `float*` are not supposed to address the same object (with the omnipotent exception of character types). Frontends know the language types; the mid-end IR has erased them into `ptr`. `!tbaa` metadata reattaches just enough type hierarchy for alias analysis.

Without TBAA, BasicAA often returns MayAlias for:

```c
void f(int *i, float *f) {
  *i = 1;
  *f = 2.0f;
}
```

even though strict aliasing says these do not alias.

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

## The idea

### Access tags

On a load/store, `!tbaa !N` points at an MDNode **access tag** describing:

- Base type
- Access type
- Offset within the base (struct-path TBAA)

Modern LLVM uses **struct-path TBAA**. Older scalar TBAA used a simpler type DAG; you will still see comments contrasting “new” vs “old” formats in [TypeBasedAliasAnalysis.cpp](https://github.com/llvm/llvm-project/blob/main/llvm/lib/Analysis/TypeBasedAliasAnalysis.cpp).

### Omnipotent char

The root of the type tree includes **omnipotent char**—anything may alias `char` / `unsigned char` accesses, matching C’s exception that character pointers can address any object.

### Frontend contract

Clang emits TBAA when strict aliasing is enabled (default for C/C++). `-fno-strict-aliasing` disables emission. Violating strict aliasing in source is **undefined behavior**; the metadata assumes the program follows the rules. LLVM does **not** need TBAA to “exploit UB” for deleting code—it uses tags to prove NoAlias between well-typed accesses.

### `!tbaa.struct`

Used for memcpy-like transfers of aggregates: describes which relative offsets carry which TBAA tags so MemCpyOpt / AA can be precise about partial field copies.

## Where it sits in the pipeline

| Item | Value |
|------|--------|
| Analysis | `tbaa` (`TypeBasedAA` / `TypeBasedAAResult`) |
| File | [llvm/lib/Analysis/TypeBasedAliasAnalysis.cpp](https://github.com/llvm/llvm-project/blob/main/llvm/lib/Analysis/TypeBasedAliasAnalysis.cpp) |
| Stack | Part of `AAResults` alongside BasicAA |
| Helper | `MDNode::isTBAAVtableAccess` (vtable-related) |

```bash
clang -S -emit-llvm -O1 -Xclang -disable-llvm-passes mix.c -o mix.ll  # has !tbaa
clang -S -emit-llvm -O1 -fno-strict-aliasing -Xclang -disable-llvm-passes mix.c -o mix.nos.ll
opt -passes='aa-eval' -aa-pipeline='basic-aa,tbaa' -print-all-alias-modref-info -disable-output mix.ll
```

## Core concepts

### `Aliases(tagA, tagB)`

`TypeBasedAAResult::Aliases` decides whether two tags may refer to overlapping objects. If metadata is **missing** on either side, TBAA conservatively allows MayAlias (falls through).

### `isTBAA` / sanitizer interaction

TypeBasedAA’s `run` can note TypeSanitizer attributes; sanitizers change how aggressively type rules apply. For study, focus on ordinary `!tbaa` paths.

### New struct-path vs old scalar trees

Struct-path tags include an offset; two accesses to different fields of a struct can be NoAlias even under the same base type when offsets prove disjoint. Scalar TBAA was coarser. When reading IR from very old bitcode, tags may look different—match the parser in TypeBasedAliasAnalysis.cpp.

## The algorithm (alias)

1. Look at `MemoryLocation` AATags.TBAA for both locations.
2. If either lacks TBAA → do not claim NoAlias from TBAA.
3. If `Aliases(A,B)` is false → `NoAlias`.
4. Otherwise defer (MayAlias from TBAA’s perspective; other AA layers may still prove NoAlias).

ModRef for calls similarly consults TBAA tags on argued memory.

## Where to read the code

1. `TypeBasedAAResult::alias` and `Aliases` in TypeBasedAliasAnalysis.cpp.
2. Clang TBAA emission (optional): clang CodeGen TBAA—only if you need to see how tags are built.
3. LangRef section on TBAA metadata.

## Worked example

C:

```c
void mix(int *i, float *f) {
  *i = 1;
  *f = 2.0f;
}
```

IR theme (opaque ptr):

```llvm
store i32 1, ptr %i, !tbaa !0
store float 2.0, ptr %f, !tbaa !1
; !0 and !1 are different scalar types under a root that is not char
```

BasicAA alone: MayAlias. BasicAA+TBAA: NoAlias.

## Reading order

1. Omnipotent char exception.
2. `Aliases` implementation.
3. Compare IR with and without `-fno-strict-aliasing`.
4. Extras.

## Hands-on

```bash
cat > mix.c <<'EOF'
void mix(int *i, float *f) { *i = 1; *f = 2.0f; }
EOF
clang -S -emit-llvm -O1 -Xclang -disable-llvm-passes mix.c -o mix.ll
grep tbaa mix.ll
opt -passes='aa-eval' -aa-pipeline='basic-aa' -print-all-alias-modref-info -disable-output mix.ll
opt -passes='aa-eval' -aa-pipeline='basic-aa,tbaa' -print-all-alias-modref-info -disable-output mix.ll
```

## Pitfalls

- Thinking LLVM “weaponizes UB” via TBAA — it proves NoAlias under a frontend contract.
- Forgetting char aliases everything under the root.
- Expecting TBAA without clang metadata (`opt` on hand-written IR without tags does nothing).
- Confusing `!tbaa.struct` with access tags.

## Neighbors

- BasicAA (Day 31), ScopedNoAlias, MemDep/MSSA.

## Check yourself

1. **Q:** What does omnipotent char mean?  
   **A:** Char-typed accesses may alias any other type in the TBAA tree.

2. **Q:** Flag to disable clang TBAA emission?  
   **A:** `-fno-strict-aliasing`.

3. **Q:** Missing metadata ⇒ ?  
   **A:** TBAA cannot prove NoAlias (MayAlias from TBAA).

4. **Q:** Analysis name?  
   **A:** `tbaa`.

5. **Q:** int* vs float* — who proves NoAlias?  
   **A:** TypeBasedAA using `!tbaa`, not BasicAA alone.

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

## Extended hands-on checklist (Day 32)

1. Build IR with the disable-llvm-passes recipe.
2. Run the minimal pass pipeline named in this day’s main note.
3. Diff against a pipeline that adds one neighbor pass.
4. Re-run with the day’s `-debug-only` flag.
5. Skim one upstream lit test RUN line and reproduce it.
6. Write one counterexample IR that must not transform.
7. Cross-check a fact against the linked github source file.

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

## Extended hands-on checklist (Day 32)

1. Build IR with the disable-llvm-passes recipe.
2. Run the minimal pass pipeline named in this day’s main note.
3. Diff against a pipeline that adds one neighbor pass.
4. Re-run with the day’s `-debug-only` flag.
5. Skim one upstream lit test RUN line and reproduce it.
6. Write one counterexample IR that must not transform.
7. Cross-check a fact against the linked github source file.
