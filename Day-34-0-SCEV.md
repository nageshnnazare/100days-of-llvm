# Day 34: Scalar Evolution (`scalar-evolution`)

ScalarEvolution (SCEV) is LLVM’s algebra for integer and pointer values in loops: it maps IR values to expressions, especially AddRec inductions `{start,+,step}<loop>`. Almost every serious loop pass depends on it. Deep dive: [Day-34-1-Extra-SCEV-DeepDive.md](Day-34-1-Extra-SCEV-DeepDive.md). Examples: [Day-34-2-Extra-SCEV-Examples.md](Day-34-2-Extra-SCEV-Examples.md). Trip counts specifically: Day 35.

## Why this exists

Pattern matching IR PHIs by hand does not scale. SCEV answers:

- Is this value an induction of loop L?
- Is it loop-invariant?
- What is its closed form at an exit?
- What is the backedge-taken count?

Indvars, unrolling, vectorization legality helpers, loop idiom, interchange, and more all call `getSCEV`.

## Diagram

```text
   %i = phi i32 [ 0, %pre ], [ %i.next, %latch ]
   %i.next = add nsw i32 %i, 1

   SCEV expression for %i:

        {0,+,1}<header>

   start 0, step 1, in this loop

   %p = getelementptr i32, ptr %base, i32 %i
   SCEV of the pointer:
        {%base,+,4}<header>
```

SCEV is a normal form for integer recurrences. An induction variable becomes an add recurrence `{start,+,step}`. A GEP becomes a recurrence in bytes. Two IVs that look different in the IR often become the same SCEV, which is how trip counts, dependence distances, and legality checks compare them. A value with a possible overflow that SCEV cannot treat as `nsw` stays a more pessimistic expression.

## The idea

`getSCEV(Value *)` returns a uniqued `SCEV*` node. Important kinds (from `SCEVTypes` in ScalarEvolutionExpressions.h)—mention only real ones:

| Kind | Role |
|------|------|
| `scConstant` | Constant integer |
| `scUnknown` | Opaque value (args, loads, …) |
| `scTruncate` / `scZeroExtend` / `scSignExtend` | Casts |
| `scAddExpr` / `scMulExpr` / `scUDivExpr` | Arithmetic |
| `scAddRecExpr` | **AddRec** `{start,+,step}<L>` |
| `scUMax` / `scSMax` / `scUMin` / `scSMin` | Min/max |
| `scPtrToInt` | Pointer-to-int modeling |
| `scSequentialUMinExpr` | Sequential UMin (poison-aware min sequences) |

**SCEV does not model floats.** Floating inductions are out of scope.

### AddRec

`{0,+,1}<%loop>` means: at iteration 0 the value is 0; each iteration adds 1. Pointers use byte steps: `{%p,+,4}<%loop>` for an `i32*` walk (opaque ptr + GEP size).

### Loop disposition

- **Invariant:** no dependence on the loop’s iteration
- **Computable evolution:** AddRec (or similar) in that loop
- **Varying** but not a nice AddRec: unknown evolution

APIs: `isLoopInvariant`, `hasComputableLoopEvolution`, `getSCEVAtScope`.

### Flags and poison

SCEV tracks `nsw`/`nuw`-like no-wrap facts. Misstating wrap can invent poison; SCEV is conservative when flags are absent.

## Where it sits in the pipeline

| Item | Value |
|------|--------|
| Analysis | `scalar-evolution` |
| Files | [ScalarEvolution.cpp](https://github.com/llvm/llvm-project/blob/main/llvm/lib/Analysis/ScalarEvolution.cpp), [ScalarEvolutionExpander.cpp](https://github.com/llvm/llvm-project/blob/main/llvm/lib/Analysis/ScalarEvolutionExpander.cpp) |
| Headers | ScalarEvolution.h, ScalarEvolutionExpressions.h |
| Debug | `-debug-only=scalar-evolution` |
| Print (new PM) | `opt -passes='print<scalar-evolution>'` |
| Print (legacy) | `opt -analyze -scalar-evolution` — **legacy**; prefer the new PM printer |

```bash
clang -S -emit-llvm -O1 -Xclang -disable-llvm-passes f.c -o f.ll
opt -passes='print<scalar-evolution>' -disable-output f.ll
```

The analysis manager invalidates SCEV when IR that might affect expressions changes; passes must not keep raw `SCEV*` across invalidation without care.

## Core concepts

### `getSCEV` vs `getSCEVAtScope`

`getSCEV` builds the expression at the value’s definition. `getSCEVAtScope` evaluates what the value means in another loop scope (e.g., exit value relative to an outer loop).

### Expander

`SCEVExpander` turns a SCEV back into IR instructions (for trip counts, exit values, indvars). Expansion must respect dominance and cost.

### Caching

SCEV uniquifies nodes in a folding set and caches Value→SCEV maps. Wrong invalidation ⇒ miscompile risk.

## The algorithm (building an AddRec)

For a loop PHI:

1. Identify start value from preheader edge.
2. Identify add/sub update from latch edge.
3. Recursively `getSCEV` on operands.
4. If update is invariant step, form `{start,+,step}<L>`.
5. Attach no-wrap flags when proven.

## Where to read the code

1. `SCEVTypes` enum in ScalarEvolutionExpressions.h.
2. `ScalarEvolution::getSCEV` and AddRec creation in ScalarEvolution.cpp.
3. SCEVExpander for materialization.
4. Day 35 for trip counts.

## Worked example

```c
void f(int *p, int n) {
  for (int i = 0; i < n; ++i)
    p[i] = i;
}
```

Typical SCEV:

- `%i` → `{0,+,1}<%loop>`
- Store pointer → `{%p,+,4}<%loop>` (if `sizeof(int)==4`)
- Trip-count related SCEVs → Day 35

## Reading order

1. AddRec notation.
2. Enum of SCEV kinds.
3. `print<scalar-evolution>` on a for-loop.
4. DeepDive; then Day 35.

## Hands-on

```bash
cat > scev.c <<'EOF'
int sum(int *a, int n) {
  int s = 0;
  for (int i = 0; i < n; ++i) s += a[i];
  return s;
}
EOF
clang -S -emit-llvm -O1 -Xclang -disable-llvm-passes scev.c -o scev.ll
opt -passes='print<scalar-evolution>' -disable-output scev.ll 2>&1 | head -80
```

## Pitfalls

- Using legacy `-analyze` without knowing it is old PM.
- Expecting float SCEVs.
- Holding SCEV pointers after invalidation.
- Confusing trip count with backedge-taken count (Day 35).

## Neighbors

- Indvars (Day 29), trip counts (Day 35), LAA/DA (Day 36), LoopIdiom (Day 27).

## Check yourself

1. **Q:** Write AddRec for `i` starting 0 step 1.  
   **A:** `{0,+,1}<loop>`.

2. **Q:** New PM print pass?  
   **A:** `print<scalar-evolution>`.

3. **Q:** Does SCEV model floats?  
   **A:** No.

4. **Q:** What expands SCEV to IR?  
   **A:** `SCEVExpander`.

5. **Q:** Name three SCEV kinds beyond AddRec.  
   **A:** e.g. Constant, Unknown, Add, Mul, SignExtend, UMax, …

## Pointer SCEV and DataLayout

Pointer AddRec steps are in bytes according to DataLayout. An `i32` GEP step is not abstractly “+1 element” inside SCEV—it is `+4` (or whatever DL says). Wrong DataLayout in `opt` experiments produces wrong steps and broken idiom/vectorize decisions. Copy the `target datalayout` line from clang’s IR into hand-written tests.

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

## Extended hands-on checklist (Day 34)

1. Build IR with the disable-llvm-passes recipe.
2. Run the minimal pass pipeline named in this day’s main note.
3. Diff against a pipeline that adds one neighbor pass.
4. Re-run with the day’s `-debug-only` flag.
5. Skim one upstream lit test RUN line and reproduce it.
6. Write one counterexample IR that must not transform.
7. Cross-check a fact against the linked github source file.

## Pointer SCEV and DataLayout

Pointer AddRec steps are in bytes according to DataLayout. An `i32` GEP step is not abstractly “+1 element” inside SCEV—it is `+4` (or whatever DL says). Wrong DataLayout in `opt` experiments produces wrong steps and broken idiom/vectorize decisions. Copy the `target datalayout` line from clang’s IR into hand-written tests.

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
