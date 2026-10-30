# Day 16: Branch Weights and `llvm.expect`

Branch weight metadata tells LLVM which successors of a `br` or `switch` are likely. Frontends emit it from `__builtin_expect`, `__builtin_expect_with_probability`, and C++ `[[likely]]` / `[[unlikely]]`. The mid-end pass `lower-expect` turns `llvm.expect` / `llvm.expect.with.probability` intrinsics into `!prof` `branch_weights`. Consumers include **BranchProbabilityInfo** and **BlockFrequencyInfo**. This day is **metadata + expect**, not instrumentation PGO.

Deep dive: [Day-16-1-Extra-BranchWeights-DeepDive.md](Day-16-1-Extra-BranchWeights-DeepDive.md). Examples: [Day-16-2-Extra-BranchWeights-Examples.md](Day-16-2-Extra-BranchWeights-Examples.md).

## 1. Why this exists

Layout, speculation, and many heuristics need a hot/cold story. Without weights, BranchProbabilityInfo falls back to static rules (loop backedges likely, null checks unlikely, …). Expect builtins and likely attributes give the programmer / FE a way to attach the same `MD_prof` channel that PGO uses—without requiring a profile training run.

Passes that read BPI/BFI: HotColdSplitting, inlining heuristics, codegen layout, some vectorization/unroll cost models, cold-call recognition in other transforms.

## Diagram

```text
   source                         IR

   if (__builtin_expect(c, 0))   br i1 %c, label %cold, label %hot
     cold();                      !prof !{!"branch_weights", i32 1, i32 2000}
   else
     hot();                       lower-expect turns llvm.expect
                                  into these weights

   MachineBlockPlacement later
   lays the heavy edge out as fall-through
```

`__builtin_expect` is not a branch hint the CPU sees directly. Clang emits `llvm.expect`, the `lower-expect` pass turns that into `!prof` branch weights, and the weights survive until block placement. A weight is a ratio, not a cycle count.

## 2. The idea

### Metadata shape

On a terminator:

```llvm
br i1 %c, label %hot, label %cold, !prof !0
!0 = !{!"branch_weights", i32 2000, i32 1}
```

Kind is `MD_prof`. First operand is the string `branch_weights`; following `i32` weights match successors (true/false for conditional `br`; default + cases for `switch`—see LangRef / BranchWeightMetadata docs). An optional `!"expected"` marker can indicate expect-derived weights.

### Expect intrinsics

Clang lowers builtins to:

- `llvm.expect.*`
- `llvm.expect.with.probability.*`

`lower-expect` (`LowerExpectIntrinsic`) replaces uses, attaches `branch_weights` on controlling terminators, and erases the intrinsic.

### `llvm.assume` ≠ expect

`llvm.assume` is a **correctness / fact** barrier for optimizers (value is true). Expect / branch weights are **performance hints** that must not change semantics. Never teach them as interchangeable.

### ProfileSummary vs raw weights

Raw `branch_weights` are local relative counts on an edge. **ProfileSummary** metadata summarizes module-wide profile shape (hot/cold thresholds) used by PGO-aware decisions. Expect-only builds may have weights without a full ProfileSummary from instrumentation.

## 3. Where it sits in the -O2/-O3 pipeline

`LowerExpectIntrinsicPass` runs early enough that later analyses see weights—check `PassBuilderPipelines.cpp` for `LowerExpectIntrinsicPass` placement in your revision (often among early simplify / late-ish FE cleanup). BPI/BFI are **analyses**, recomputed when needed by clients—not a single “branch weights pass” in the scalar sense.

| Item | Value |
|------|--------|
| Lowering pass | `lower-expect` |
| Metadata | `MD_prof` / `branch_weights` |
| Analyses | `BranchProbabilityInfo`, `BlockFrequencyInfo` |
| Docs | [BranchWeightMetadata](https://llvm.org/docs/BranchWeightMetadata.html) |

```bash
clang -S -emit-llvm -O1 -Xclang -disable-llvm-passes file.c -o file.ll
opt -passes=lower-expect -S file.ll -o file.exp.ll
```

## 4. Core concepts

### Weight meaning

Weights are relative. Probability of successor i ≈ `Wi / sum(W)`. Absolute scale differs between expect defaults and real PGO counters.

### MDBuilder helpers

[llvm/include/llvm/IR/MDBuilder.h](https://github.com/llvm/llvm-project/blob/main/llvm/include/llvm/IR/MDBuilder.h): `createBranchWeights`, `createLikelyBranchWeights`, `createUnlikelyBranchWeights`. Likely/unlikely use a large bias (historically ~2000:1; modern trees may use `(1<<20)-1 : 1` via `kLikelyBranchWeight`).

### BPI heuristics without metadata

When `!prof` is missing, BranchProbabilityInfo applies static heuristics: loop backedges taken; pointer null unlikely; invoke unwind unlikely; etc. (read `BranchProbabilityInfo.cpp`).

### Clang surface

- `__builtin_expect(expr, c)`
- `__builtin_expect_with_probability(expr, c, p)` with `p ∈ [0.0, 1.0]`
- `[[likely]]` / `[[unlikely]]` on labels (C++20) → expect-like metadata

## 5. The algorithm (lower-expect)

In [LowerExpectIntrinsic.cpp](https://github.com/llvm/llvm-project/blob/main/llvm/lib/Transforms/Scalar/LowerExpectIntrinsic.cpp):

1. Find `llvm.expect` / `llvm.expect.with.probability` calls.
2. Walk to the branch/switch (or PHI-fed conditions) controlled by the expected value.
3. Compute likely/unlikely weights (`getBranchWeight`); for probability form, scale by the given double and number of successors.
4. `setMetadata(MD_prof, …)` / `setBranchWeights` with `IsExpected` as appropriate.
5. Replace expect with its first operand; erase intrinsic.

misexpect checking may validate expect vs real profiles when both exist.

## 6. Where to read the code

- [llvm/lib/Transforms/Scalar/LowerExpectIntrinsic.cpp](https://github.com/llvm/llvm-project/blob/main/llvm/lib/Transforms/Scalar/LowerExpectIntrinsic.cpp)
- [llvm/lib/Analysis/BranchProbabilityInfo.cpp](https://github.com/llvm/llvm-project/blob/main/llvm/lib/Analysis/BranchProbabilityInfo.cpp)
- [llvm/lib/Analysis/BlockFrequencyInfo.cpp](https://github.com/llvm/llvm-project/blob/main/llvm/lib/Analysis/BlockFrequencyInfo.cpp)
- [llvm/lib/IR/MDBuilder.cpp](https://github.com/llvm/llvm-project/blob/main/llvm/lib/IR/MDBuilder.cpp)
- LangRef / [BranchWeightMetadata](https://llvm.org/docs/BranchWeightMetadata.html)

## 7. Worked before/after

**C** (`expect.c`):

```c
int f(int x) {
  if (__builtin_expect(x == 0, 0))
    return -1; /* unlikely */
  return x;
}
```

**Commands:**

```bash
clang -S -emit-llvm -O1 -Xclang -disable-llvm-passes expect.c -o expect.ll
opt -passes=lower-expect -S expect.ll -o expect.out.ll
```

**Before:** `call i64 @llvm.expect.i64(…)` feeding an icmp/br.

**After:**

```llvm
br i1 %cmp, label %unlikely, label %likely, !prof !{!"branch_weights", i32 1, i32 /*large*/}
```

(Exact weight numbers follow `MDBuilder` / lower-expect constants; true/false order matches successor order.)

## 8. Reading order for today

1. LangRef / BranchWeightMetadata page (operand layout).
2. `LowerExpectIntrinsic.cpp` handlers for `br` and `switch`.
3. `MDBuilder::createBranchWeights`.
4. BPI: metadata calc vs heuristic fallback.
5. Skim BFI as consumer of BPI.
6. Deep dive + examples.

## 9. Hands-on commands

```bash
clang -S -emit-llvm -O1 -Xclang -disable-llvm-passes expect.c -o expect.ll
opt -passes=lower-expect -S expect.ll -o out.ll
opt -passes='print<branch-prob>' -disable-output out.ll
opt -passes='print<block-freq>' -disable-output out.ll
# C++ likely:
clang++ -S -emit-llvm -O1 -Xclang -disable-llvm-passes likely.cpp -o likely.ll
```

## 10. Pitfalls and invariants

- Weights are hints; mis-predict costs performance, not correctness (unless you confuse them with assume).
- Successor order matters when reading `i32` lists.
- Instrumentation PGO also uses `branch_weights`—same MD channel, different provenance; today’s focus is expect/FE.
- `ProfileSummary` may be absent in expect-only IR.
- Switch weights include **default** as an entry—count carefully.
- Opaque pointers irrelevant to terminator metadata; keep examples on `i1` / integer conditions.

## 11. Neighbors

- **FE:** Clang codegen of expect / likely.
- **Pass:** `lower-expect`.
- **Analyses:** BPI → BFI → many opts.
- **Later days:** instrumentation PGO (separate)—shares metadata vocabulary.

## 12. Check yourself

**Q1.** Metadata string for weights?
**A1.** `!"branch_weights"` under `MD_prof`.

**Q2.** Pass that lowers expect intrinsics?
**A2.** `lower-expect`.

**Q3.** Name two Clang entry points that create expect-like info.
**A3.** `__builtin_expect`, `__builtin_expect_with_probability` (also `[[likely]]`/`[[unlikely]]`).

**Q4.** Does missing metadata mean 50/50 always?
**A4.** No—BPI applies static heuristics (e.g. loop backedges).

**Q5.** Is `llvm.assume` the same as expect?
**A5.** No—assume is a semantic fact; expect is a hint.

**Q6.** ProfileSummary vs branch_weights?
**A6.** Summary is module-level hotness context; weights are per-terminator relative edge counts.

> Tip: Implementation details are in `Day-16-1-Extra-BranchWeights-DeepDive.md`. Runnable cases are in `Day-16-2-Extra-BranchWeights-Examples.md`.

## Appendix A: Reading switch weights

For `switch`, operands after the string are weights for **default**, then each case in the IR’s case order. LowerExpect fills unlikely weights for non-expected cases and a large weight for the expected case index.

## Appendix B: Block frequency

BFI solves a system over BPI edge probabilities and loop scale to estimate relative execution counts of blocks. HotColdSplitting and similar passes threshold on BFI, not raw weights alone.

## Appendix C: Weight arithmetic example

True weight 2000, false weight 1 → P(true) = 2000/2001 ≈ 99.95%. BPI stores this as a `BranchProbability` rational. BFI then scales frequencies through loops: a header with frequency `F` and backedge probability `p` gets a trip-count-related boost of about `1/(1-p)` in the classic model (implementation uses fixed-point / block mass algorithms—read BFI for exactness).

Expect’s large bias is intentionally extreme so layout treats the path like “almost always.”

## Appendix D: Same MD, different producers

| Producer | How weights appear |
|----------|--------------------|
| `lower-expect` | From llvm.expect* |
| Clang likely/unlikely | FE → expect or MD |
| Instrumentation PGO use | From counter profiles |
| Sample PGO | From sample maps |
| Manual IR | Hand-written `!prof` |

Consumers rarely care which producer wrote the MD—BPI mostly sees numbers. Provenance matters for diagnostics (misexpect) and for knowing whether ProfileSummary exists.

## Appendix E: Printing and verifying

```bash
opt -passes=lower-expect,verify -S t.ll -o t2.ll
opt -passes='print<branch-prob>,print<block-freq>' -disable-output t2.ll
```

If verify fails after hand-editing weights, you likely mismatched successor arity. Fix MD before blaming BPI.

## Appendix F: One-line summary

Expect/likely → `lower-expect` → `!prof branch_weights` → BPI/BFI; heuristics fill gaps; assume ≠ expect; not today’s PGO instrumentation story.
