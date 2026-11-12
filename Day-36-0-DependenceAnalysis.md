# Day 36: Dependence Analysis and Loop Access Analysis

Classical loop-nest dependence analysis (`DependenceInfo`) computes direction/distance vectors between memory references using SCEV. In practice, **LoopAccessAnalysis (LAA)** is what LoopVectorize relies on. Learn both and keep the split straight. Deep dive: [Day-36-1-Extra-DependenceAnalysis-DeepDive.md](Day-36-1-Extra-DependenceAnalysis-DeepDive.md). Examples: [Day-36-2-Extra-DependenceAnalysis-Examples.md](Day-36-2-Extra-DependenceAnalysis-Examples.md).

## Why this exists

Parallelizing or reordering loops (interchange, vectorization, distribution) is illegal when iterations communicate through memory in the wrong order. Dependence analysis decides whether iteration \(i\) must precede \(j\).

## Diagram

```text
   for i:
     A[i+1] = A[i] + 1

   dependence from the load A[i]
   to the store A[i+1]:

     direction:  forward  (<)
     distance:   1 iteration

   that is a loop-carried flow dependence.
   Interchange and vectorization must
   respect it.

   A[i] = B[i] + 1
   has no carried dependence on A if B
   does not alias A.
```

Dependence analysis asks whether two memory accesses in a loop nest touch the same location, and if so in which direction and at what distance. A distance of zero is loop-independent. A positive distance is carried by that loop. The vectorizer needs "no carried dependence, or a dependence it can honor with a vector width." The answer is conservative: if the subscripts are not affine, the result is "do not know."

## The idea

### Classical DependenceAnalysis

[DependenceAnalysis.cpp](https://github.com/llvm/llvm-project/blob/main/llvm/lib/Analysis/DependenceAnalysis.cpp), class `DependenceInfo`:

- For two memory ops in a loop nest, build SCEV subscripts.
- Run classical tests: **strong SIV**, **weak-crossing**, **GCD**, **Banerjee** (Banerjee may be gated/disabled by default in modern trees due to correctness history—check the file’s test-type options).
- Produce a dependence with per-level `DVEntry` directions: `LT`, `EQ`, `GT`, `ALL` (and combinations).

Used notably by **LoopInterchange** (Day 30).

### LoopAccessAnalysis (the practical vectorizer tool)

[LoopAccessAnalysis.cpp](https://github.com/llvm/llvm-project/blob/main/llvm/lib/Analysis/LoopAccessAnalysis.cpp):

- Analyzes a single loop for vectorization/runtime checks.
- Uses SCEV + AA to find dependences that prevent vectorization, and emits **memory runtime checks** / transformation scaffolding when static proofs fail but checks can guard a vector path.
- This is **more important for LoopVectorize** than classical DA.

Do **not** claim the vectorizer primarily uses `DependenceAnalysis`/`da`—it primarily uses LAA.

## Where it sits

| Analysis | Pass / print name | Role |
|----------|-------------------|------|
| DependenceAnalysis | `da`, `print<da>` | Classical direction vectors |
| LoopAccessAnalysis | `print<access-info>` (loop access info printer) | Vectorization legality + runtime checks |

```bash
opt -passes='print<da>' -disable-output nest.ll
opt -passes='print<access-info>' -disable-output loop.ll
```

## Core concepts

### Direction vectors

For nest `(i,j)`, a dependence might be `(=, <)` meaning same outer iteration, inner source before sink. Interchange permutes these vectors; illegal permutations are rejected (Day 30).

### Distance

Sometimes a constant distance (e.g., `A[i+1]` vs `A[i]` → distance 1) is known; vectorization may need distance ≥ VF.

### LAA vs DA

| | DA | LAA |
|--|----|-----|
| Nest-aware classical tests | Yes | Focused on vectorizable inner loops |
| Runtime checks | Not the main product | Central |
| Primary client | Interchange (etc.) | LoopVectorize |

## The algorithm (DA sketch)

1. Collect pair of load/store in common loops.
2. Classify subscripts (SIV = single index variable, MIV = multiple).
3. Apply strong SIV test; if inconclusive, GCD/Banerjee/MIV tests as enabled.
4. Fill `DVEntry` directions/distances.
5. Query API returns a `Dependence` object (or none if independent).

LAA algorithm (sketch): gather all memory ops in the loop; build dependence / alias sets; decide if vectorization needs runtime pointer checks; record max dependency distance.

## Where to read the code

1. `DependenceInfo::strongSIVtest` and GCD/Banerjee sections in DependenceAnalysis.cpp.
2. LoopAccessAnalysis.cpp — entry analysis for a loop (used by LV).
3. LoopInterchange dependence checks (Day 30).
4. LoopVectorize — search for `LoopAccessInfo` usage, not `DependenceInfo`, for the main path.

## Worked example

```c
for (int i = 1; i < n; ++i)
  a[i] = a[i-1] + 1;
```

DA/LAA: loop-carried dependence distance 1 on `a` → cannot freely vectorize without special handling; interchange with an outer independent loop may still be constrained.

```c
for (int i = 0; i < n; ++i)
  a[i] = b[i] + 1;
```

Independent (given NoAlias `a`,`b`) → vectorizable; LAA may still emit checks if AA is MayAlias.

## Reading order

1. Direction vector vocabulary.
2. LAA vs DA split (memorize).
3. strong SIV in DA.
4. `print<access-info>` output.
5. Extras.

## Hands-on

```bash
cat > dep.c <<'EOF'
void carried(int *a, int n) {
  for (int i = 1; i < n; ++i) a[i] = a[i-1] + 1;
}
void parallel(int *restrict a, int *restrict b, int n) {
  for (int i = 0; i < n; ++i) a[i] = b[i] + 1;
}
EOF
clang -S -emit-llvm -O1 -Xclang -disable-llvm-passes dep.c -o dep.ll
opt -passes='print<access-info>' -disable-output dep.ll
opt -passes='print<da>' -disable-output dep.ll
```

## Pitfalls

- Saying “the vectorizer uses DependenceAnalysis” as the primary story.
- Ignoring that Banerjee may be off by default.
- Forgetting runtime checks are a *feature* of LAA, not a failure.

## Neighbors

- LoopInterchange (Day 30), LoopVectorize, SCEV (Days 34–35), BasicAA (Day 31).

## Check yourself

1. **Q:** What is `DVEntry`?  
   **A:** Per-loop-level direction entry (LT/EQ/GT/ALL) in classical DA.

2. **Q:** Primary analysis for LoopVectorize memory legality?  
   **A:** LoopAccessAnalysis / LoopAccessInfo.

3. **Q:** Print passes?  
   **A:** `print<da>`, `print<access-info>`.

4. **Q:** Name two classical tests.  
   **A:** Strong SIV, GCD; also weak-crossing, Banerjee.

5. **Q:** Who uses classical DA among earlier days?  
   **A:** Loop interchange (notably).

## Runtime checks vs static DA

LAA’s runtime checks typically compare base pointers and region sizes so vector code runs only when objects do not overlap. Classical DA trying to prove static independence cannot invent those checks. When `print<access-info>` says checks are needed, LoopVectorize may still vectorize behind a guard—successful vectorization ≠ static NoAlias.

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

## Extended hands-on checklist (Day 36)

1. Build IR with the disable-llvm-passes recipe.
2. Run the minimal pass pipeline named in this day’s main note.
3. Diff against a pipeline that adds one neighbor pass.
4. Re-run with the day’s `-debug-only` flag.
5. Skim one upstream lit test RUN line and reproduce it.
6. Write one counterexample IR that must not transform.
7. Cross-check a fact against the linked github source file.

## Runtime checks vs static DA

LAA’s runtime checks typically compare base pointers and region sizes so vector code runs only when objects do not overlap. Classical DA trying to prove static independence cannot invent those checks. When `print<access-info>` says checks are needed, LoopVectorize may still vectorize behind a guard—successful vectorization ≠ static NoAlias.

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

## Extended hands-on checklist (Day 36)

1. Build IR with the disable-llvm-passes recipe.
2. Run the minimal pass pipeline named in this day’s main note.
3. Diff against a pipeline that adds one neighbor pass.
4. Re-run with the day’s `-debug-only` flag.
5. Skim one upstream lit test RUN line and reproduce it.
6. Write one counterexample IR that must not transform.
7. Cross-check a fact against the linked github source file.

## Runtime checks vs static DA

LAA’s runtime checks typically compare base pointers and region sizes so vector code runs only when objects do not overlap. Classical DA trying to prove static independence cannot invent those checks. When `print<access-info>` says checks are needed, LoopVectorize may still vectorize behind a guard—successful vectorization ≠ static NoAlias.

## IR hygiene for experiments

Always generate IR with `clang -S -emit-llvm -O1 -Xclang -disable-llvm-passes` so functions are not marked `optnone`. Opaque pointers (`ptr`) are mandatory in modern IR examples; do not revive typed pointer syntax in new notes.

When comparing before/after, prefer `diff -u` on files produced by the same opt pipeline with one pass removed. That isolates the transform better than comparing against clang -O3, which runs dozens of passes.
