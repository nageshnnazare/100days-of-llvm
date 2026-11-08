# Day 29: Induction Variable Simplify (`indvars`)

IndVarSimplify canonicalizes and strengthens induction variables using ScalarEvolution: widen narrow IVs, delete redundant compares, rewrite exit tests (LFTR), strength-reduce IV users, and replace exit values with closed forms. Deep dive: [Day-29-1-Extra-IndVarSimplify-DeepDive.md](Day-29-1-Extra-IndVarSimplify-DeepDive.md). Examples: [Day-29-2-Extra-IndVarSimplify-Examples.md](Day-29-2-Extra-IndVarSimplify-Examples.md).

## Why this exists

Loops are full of related integers: the canonical counter, a byte pointer offset, a compare against `n`, a `sext` of an `i32` IV used in a 64-bit GEP, and an LCSSA phi of the final IV at the exit. Each is mathematically linked. Leaving them all in IR:

- Blocks strength reduction and LSR (later).
- Keeps extend ops on every iteration.
- Makes trip-count oriented passes harder.

`indvars` rewrites the IR toward a small set of canonical IVs that SCEV understands cleanly.

## Diagram

```text
   %i = phi i32 [ 0, %pre ], [ %i.next, %latch ]
   %i.next = add nsw i32 %i, 1
   %cmp = icmp slt i32 %i, %n

   indvars rewrites users of %i onto a
   canonical induction variable, and
   widens a narrow IV when the arithmetic
   is legal:

   %i64 = phi i64 [ 0, %pre ], [ %next, %latch ]
   %next = add nuw nsw i64 %i64, 1
```

IndVarSimplify picks one induction variable as canonical and rewrites the others in terms of it. Strength reduction turns `i * scale` into a recurrence that only adds. Trip-count compares become compares against the canonical IV. Widening a 32-bit IV to 64-bit is done only when the extra bits cannot change the defined result; a wrapping IV is left narrow.

## The idea

Using SCEV’s `{start,+,step}<loop>` AddRecs:

1. **Canonicalize** IVs and exit conditions toward simple forms.
2. **Widen** IVs to a wider legal type so in-loop `sext`/`zext` disappear when safe.
3. **LFTR (linear function test replace):** change the exit test to compare a canonical IV against a trip-count-derived limit so other IVs become dead.
4. **Eliminate redundant IV comparisons** proved equivalent by SCEV.
5. **Rewrite exit values:** replace LCSSA phis of IVs with closed-form SCEVs expanded by `SCEVExpander`.
6. **Strengthen IV users** (e.g., replace expensive recurrences when SCEV + expander can materialize cheaper code).

## Where it sits in the pipeline

| Item | Value |
|------|--------|
| Pass name | `indvars` |
| File | [llvm/lib/Transforms/Scalar/IndVarSimplify.cpp](https://github.com/llvm/llvm-project/blob/main/llvm/lib/Transforms/Scalar/IndVarSimplify.cpp) |
| Expander | [ScalarEvolutionExpander.cpp](https://github.com/llvm/llvm-project/blob/main/llvm/lib/Analysis/ScalarEvolutionExpander.cpp) |
| Debug | `-debug-only=indvars` |
| Needs | SCEV, LoopSimplify/LCSSA friendly loops |

```bash
clang -S -emit-llvm -O1 -Xclang -disable-llvm-passes iv.c -o iv.ll
opt -passes='loop-simplify,lcssa,indvars' -S iv.ll
```

### Flag naming trap

`-indvars-predicate-loops` exists inside IndVarSimplify and relates to predication-style rewrites in *this* file. It is **not** the same as the separate Loop Predication pass (`loop-predication`). Do not conflate them when reading docs or bugs.

## Core concepts

### Canonical IV

Prefer a simple integer counter AddRec `{0,+,1}` (or similar) as the primary IV when possible.

### Widening

An `i32` IV used only after `sext` to `i64` for GEPs can become an `i64` IV when the wider arithmetic is legal (no extra poison from `nsw` mismatches). Goal: remove per-iteration extends.

### LFTR

Before: exit on `icmp eq i32 %i, %n` while also maintaining a pointer IV.  
After: exit on compare of the canonical counter against a computed trip boundary; pointer IV may be strength-reduced or deleted if unused.

**Poison:** LFTR must not introduce new `nsw`/`nuw` poison. If replacing a test would assume non-wrapping that the original IR did not guarantee, LFTR must refrain or use wrapping-safe forms. Comments around LFTR in IndVarSimplify.cpp discuss pre-inc vs post-inc and poison carefully.

### Exit value rewriting

After the loop, `%.lcssa = phi i32 [ %i.next, %latch ]` might equal `n` (or another closed form). `rewriteLoopExitValues` substitutes that closed form so the phi dies.

## The algorithm (pipeline inside the pass)

Approximate order in `IndVarSimplifyPass` / simplify path:

1. Collect SCEV info for the loop; require a single backedge for some LFTR cases.
2. Widen IVs where profitable/legal.
3. Eliminate congruent IVs / redundant compares.
4. `rewriteLoopExitValues` via SCEVExpander.
5. LFTR on exiting blocks when `needsLFTR` says so and `-disable-lftr` is false.
6. Clean up dead code.

Statistics include `NumLFTR` for how many exit tests were replaced.

## Where to read the code

1. [IndVarSimplify.cpp](https://github.com/llvm/llvm-project/blob/main/llvm/lib/Transforms/Scalar/IndVarSimplify.cpp) — `needsLFTR`, LFTR implementation, `rewriteLoopExitValues` call site, widen IV logic, `LoopPredication` / `indvars-predicate-loops` option (know it is local).
2. SCEV days 34–35 for AddRec and trip counts.
3. Tests in `llvm/test/Transforms/IndVarSimplify/`.

## Worked before / after

```c
int sum(int *a, int n) {
  int s = 0;
  for (int i = 0; i < n; ++i)
    s += a[i];
  return s;
}
```

After indvars (typical themes, not a guarantee of exact IR):

- Exit test may compare a canonical IV to a trip count.
- GEP indices may use a widened IV without a separate `sext` each iteration.
- Exit LCSSA of `i` may become unused if rewritten.

## Reading order

1. LFTR poison rules in this note.
2. `needsLFTR` + LFTR functions in the .cpp.
3. Exit value rewrite + expander.
4. Day 29 extras; then SCEV (Day 34) if AddRecs feel shaky.

## Hands-on

```bash
cat > iv.c <<'EOF'
long gather(int *a, int n) {
  long s = 0;
  for (int i = 0; i < n; ++i)
    s += a[i];
  return s;
}
EOF
clang -S -emit-llvm -O1 -Xclang -disable-llvm-passes iv.c -o iv.ll
opt -passes='loop-simplify,lcssa,indvars' -S iv.ll -o iv.out.ll
opt -debug-only=indvars -passes='loop-simplify,lcssa,indvars' -S iv.ll -o /dev/null
```

Compare extends and exit icmp.

## Pitfalls

- **Blaming indvars for LoopPredication bugs** — different pass (`loop-predication`).
- **Expecting float IVs** — SCEV is integer/pointer oriented; float induction is out of scope.
- **Ignoring poison** when hand-writing LFTR-like transforms.
- Running without LCSSA/simplify reduces effectiveness.

## Neighbors

- SCEV / trip counts (Days 34–35).
- LSR (loop-reduce) — further strength reduction after indvars.
- Loop vectorize — likes clean IVs.

## Check yourself

1. **Q:** What does LFTR stand for and do?  
   **A:** Linear Function Test Replace — rewrite the exit compare to use a canonical IV vs trip-derived bound.

2. **Q:** Why widen IVs?  
   **A:** So arithmetic happens in the wide type and in-loop sext/zext can die.

3. **Q:** Pass name?  
   **A:** `indvars`.

4. **Q:** Is `-indvars-predicate-loops` the LoopPredication pass?  
   **A:** No. Loop Predication is `loop-predication`; the flag only toggles related logic inside IndVarSimplify.

5. **Q:** What expands closed-form exit SCEVs to IR?  
   **A:** `SCEVExpander` in ScalarEvolutionExpander.cpp.

## Congruent IVs in practice

```c
for (int i = 0, j = 0; i < n; ++i, ++j)
  use(i, j);
```

SCEV sees the same AddRec for `i` and `j`. Indvars can replace one with the other and delete redundant compares like `i==j`. Afterward LSR/vectorize see fewer live ranges.

## Pre-inc vs post-inc LFTR

LFTR may switch an exit test from “compare current IV” to “compare next IV” (or reverse) depending on which counter is canonical and where the latch sits. Poison rules differ because a dead `add nsw` becoming live as the exit driver is dangerous. Read the comments around LFTR carefully before assuming post-inc is always better.

## Interaction with LSR

Indvars prepares IR; LoopStrengthReduce (later) rewrites addressing. If indvars leaves a pointer IV solely for addressing while the exit uses a counter, LSR can focus on address math. If both remain, LSR has more candidates. The handoff is intentional in the pipeline design.

## Debugging widen failures

When a narrow IV refuses to widen, usual causes are:

- A use that truncates back and depends on narrow wrap.
- Missing no-wrap flags so wide add would change semantics.
- Legalizer/type concerns on weird integer widths.

Use `-debug-only=indvars` and SCEV prints before/after.

## Predication flag reminder

`-indvars-predicate-loops` toggles logic *inside* IndVarSimplify. The separate pass remains `loop-predication`. Say both names aloud when filing bugs.

### Additional study points (1)

```c
- for (int i = 0, j = 0; i < n; ++i, ++j)
- use(i, j);
```
SCEV sees the same AddRec for `i` and `j`. Indvars can replace one with the other and delete redundant compares like `i==j`. Afterward LSR/vectorize see fewer live ranges.
LFTR may switch an exit test from “compare current IV” to “compare next IV” (or reverse) depending on which counter is canonical and where the latch sits. Poison rules differ because a dead `add nsw` becoming live as the exit driver is dangerous. Read the comments around LFTR carefully before assuming post-inc is always better.
Indvars prepares IR; LoopStrengthReduce (later) rewrites addressing. If indvars leaves a pointer IV solely for addressing while the exit uses a counter, LSR can focus on address math. If both remain, LSR has more candidates. The handoff is intentional in the pipeline design.
- When a narrow IV refuses to widen, usual causes are:
- A use that truncates back and depends on narrow wrap.
- Missing no-wrap flags so wide add would change semantics.
- Legalizer/type concerns on weird integer widths.
- Use `-debug-only=indvars` and SCEV prints before/after.
`-indvars-predicate-loops` toggles logic *inside* IndVarSimplify. The separate pass remains `loop-predication`. Say both names aloud when filing bugs.

```bash
# revisit with debug (session 1)
opt -passes='loop-simplify,lcssa' -S in.ll -o /tmp/in.simp.ll
```

## LFTR and poison checklist

Before trusting an LFTR rewrite in a patch:

1. Does the new compare’s IV already execute on all paths to the latch?
2. Are nsw/nuw flags on the IV update justified?
3. Could poison of the old condition differ from poison of the new one?
4. Is the limit expansion itself poison-free under the loop predicates?

If any answer is cloudy, SCEV/indvars should leave the test alone. That conservatism is intentional.

## IR hygiene for experiments

Always generate IR with `clang -S -emit-llvm -O1 -Xclang -disable-llvm-passes` so functions are not marked `optnone`. Opaque pointers (`ptr`) are mandatory in modern IR examples; do not revive typed pointer syntax in new notes.

When comparing before/after, prefer `diff -u` on files produced by the same opt pipeline with one pass removed. That isolates the transform better than comparing against clang -O3, which runs dozens of passes.
