# Day 35: SCEV Trip Counts

Trip-count queries are the most-used “product feature” of ScalarEvolution: how many times will this loop run? Deep dive: [Day-35-1-Extra-TripCount-DeepDive.md](Day-35-1-Extra-TripCount-DeepDive.md). Examples: [Day-35-2-Extra-TripCount-Examples.md](Day-35-2-Extra-TripCount-Examples.md). Background: Day 34.

## Why this exists

Unrolling, vectorization, peeling, runtime-check budgeting, and LFTR all need iteration counts. Counting by staring at IR branches fails as soon as there are multiple exits, wrapping math, or symbolic bounds. SCEV centralizes that math.

## Diagram

```text
   for (i = 0; i < n; ++i)        trip count = n
                                  backedge taken count = n-1
                                  (the latch runs n-1 times
                                   if n >= 1; 0 times if n == 0)

   SCEV:
     backedge-taken count = umax(n, 1) - 1
     or, with a guard:      n - 1

   a data-dependent exit
     if (a[i] == 0) break;
   has no computable trip count
```

The trip count is how many times the header executes. The backedge-taken count is how many times the latch branches back, which is one less when the loop runs at least once. ScalarEvolution computes the backedge count from the exit predicate. A loop with a break that depends on loaded memory has an unknown count, and the vectorizer must then version or give up.

## The idea: two different counts

| Concept | Meaning |
|---------|---------|
| **Backedge-taken count** | How many times the latch→header edge executes |
| **Trip count** | How many times the **header** executes |

For a typical loop that enters and then takes the backedge until done:

\[
\text{trip count} = \text{backedge-taken count} + 1
\]

when the count is exact and the loop is entered. If the loop may be skipped entirely (zero trip), the relationship needs a guard—this is why rotation + guards matter (Day 26).

Example: `for (i = 0; i < n; ++i)` with `n > 0`:

- IV AddRec `{0,+,1}<L>`
- Backedge taken **`n - 1`** times (exact when `n` is positive and no wrap)
- Header executes **`n`** times

When `n` may be zero, the backedge-taken count is often expressed in a form that is only valid under predicates, or uses umax/umin to be safe.

## Where it sits

Same analysis as Day 34: [ScalarEvolution.cpp](https://github.com/llvm/llvm-project/blob/main/llvm/lib/Analysis/ScalarEvolution.cpp). Key APIs on `ScalarEvolution`:

- `getBackedgeTakenCount`
- `getPredicatedBackedgeTakenCount` — also returns assumptions/predicates under which the count is valid
- `getExitCount` — count for a specific exiting block
- `getSmallConstantTripCount` / `getSmallConstantMaxTripCount`
- `getSymbolicMaxBackedgeTakenCount`

Helpers used internally:

- `computeExitLimit`
- `howManyLessThans` / `howManyGreaterThans` — convert icmp predicates into how-far-to-exit math

Debug: `-debug-only=scalar-evolution`. Print: `print<scalar-evolution>`.

## Core concepts

### Exact vs max vs symbolic

- Exact constant (e.g. 100) → full unroll possible
- Max constant → upper bound when exact unknown
- Symbolic (`n-1`) → runtime unroll / vectorize with runtime trip

`CouldNotCompute` appears when wrapping or unstructured exits defeat the math.

### Multiple exits

Each exit has its own `getExitCount`. The loop’s overall backedge-taken count is essentially the **minimum** of the exit-limited counts (first exit wins). A `break` on a rare condition can make the symbolic count complex or unknown.

### Predicated counts

`getPredicatedBackedgeTakenCount` returns a count plus predicates (e.g. assumptions that `n` is nonnegative / no wrap). Clients that can emit runtime checks may use predicated counts; others require exact unconditional counts.

### Wrapping and poison

An `i8` counter that can wrap through all 256 values does not have a simple trip count of “until icmp fails” in the same way a non-wrapping `i32` does. Missing `nsw` on IV updates often blocks clean counts.

### How-far-to-zero / icmp logic

For `icmp slt %iv, %n` with AddRec IV, `howManyLessThans` computes how many times the predicate stays true. Similar for greater-than forms. This is the heart of `computeExitLimit`.

## The algorithm (single-exit sketch)

1. Find the exiting block and its `icmp` (or switch).
2. Get SCEVs for both sides; identify AddRec vs invariant limit.
3. Dispatch to `howManyLessThans` / `howManyGreaterThans` / equality cases.
4. Apply no-wrap flags to choose exact vs max.
5. Combine multiple exits via minimum.
6. Cache the result on the Loop.

## Where to read the code

1. Declarations of the get*Trip/Backedge APIs in ScalarEvolution.h.
2. `computeExitLimit` and `howManyLessThans` in ScalarEvolution.cpp.
3. A simple `for (i=0;i<n;++i)` under `print<scalar-evolution>`.

## Worked IR story

```c
void f(int *p, int n) {
  for (int i = 0; i < n; ++i)
    p[i] = 0;
}
```

SCEV themes:

- `{0,+,1}<L>` for `i`
- Backedge-taken count ≈ `(-1 + n)` or `n-1` style, often with `umax` when `n` may be 0 (`max(n,1)-1` patterns appear in some formulations)
- Zero-trip complication: without a guard, “execute body `n` times” is wrong for `n==0`

Rotated loops (Day 26) make the “entered ⇒ body runs” story cleaner for later passes.

## Reading order

1. Backedge vs trip definition until automatic.
2. `getBackedgeTakenCount` vs predicated vs exit count.
3. DeepDive + Examples.
4. Revisit indvars LFTR (Day 29).

## Hands-on

```bash
cat > tc.c <<'EOF'
void f(int n) { for (int i = 0; i < n; ++i) { } }
void g(void) { for (int i = 0; i < 100; ++i) { } }
EOF
clang -S -emit-llvm -O1 -Xclang -disable-llvm-passes tc.c -o tc.ll
opt -passes='print<scalar-evolution>' -disable-output tc.ll
```

Compare symbolic `n` vs constant 100.

## Pitfalls

- Calling trip count when you needed backedge-taken (off-by-one).
- Ignoring multi-exit minima.
- Assuming `i8` loops always have small constant counts when they wrap.
- Using legacy `-analyze` only.

## Neighbors

- LoopUnroll, LoopVectorize, IndVars, LoopRotate.

## Check yourself

1. **Q:** Relation between exact backedge-taken and trip count when entered?  
   **A:** trip ≈ backedge-taken + 1.

2. **Q:** For `i=0; i<n; ++i` with `n>0`, backedge-taken?  
   **A:** `n-1`.

3. **Q:** Multiple exits ⇒ loop count?  
   **A:** Minimum of per-exit counts (first exit).

4. **Q:** API for counts valid under assumptions?  
   **A:** `getPredicatedBackedgeTakenCount`.

5. **Q:** Name the icmp helpers.  
   **A:** `howManyLessThans` / `howManyGreaterThans`.

## Off-by-one drill

Say out loud for `for (i=0;i<n;++i)` with `n=5`: header runs 5 times; backedge taken 4 times; last IV value before exit depends on pre/post-inc form. Then write the SCEV BTC and trip count. Repeat for `i<=n` and for countdown loops. Most LFTR bugs are off-by-one bugs.

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

## Extended hands-on checklist (Day 35)

1. Build IR with the disable-llvm-passes recipe.
2. Run the minimal pass pipeline named in this day’s main note.
3. Diff against a pipeline that adds one neighbor pass.
4. Re-run with the day’s `-debug-only` flag.
5. Skim one upstream lit test RUN line and reproduce it.
6. Write one counterexample IR that must not transform.
7. Cross-check a fact against the linked github source file.

## Off-by-one drill

Say out loud for `for (i=0;i<n;++i)` with `n=5`: header runs 5 times; backedge taken 4 times; last IV value before exit depends on pre/post-inc form. Then write the SCEV BTC and trip count. Repeat for `i<=n` and for countdown loops. Most LFTR bugs are off-by-one bugs.

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

## Extended hands-on checklist (Day 35)

1. Build IR with the disable-llvm-passes recipe.
2. Run the minimal pass pipeline named in this day’s main note.
3. Diff against a pipeline that adds one neighbor pass.
4. Re-run with the day’s `-debug-only` flag.
5. Skim one upstream lit test RUN line and reproduce it.
6. Write one counterexample IR that must not transform.
7. Cross-check a fact against the linked github source file.
