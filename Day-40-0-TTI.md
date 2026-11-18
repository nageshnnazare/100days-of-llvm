# Day 40: Target Transform Info (TTI)

TTI is the optimizer’s query API for **target-specific costs and capabilities**: how expensive is a shuffle on this CPU? what unroll factor is good? are masked loads legal? Without a real target, you get the generic “everything costs 1” model—and `opt` disagrees with `clang -O3`. Deep dive: [Day-40-1-Extra-TTI-DeepDive.md](Day-40-1-Extra-TTI-DeepDive.md). Examples: [Day-40-2-Extra-TTI-Examples.md](Day-40-2-Extra-TTI-Examples.md).

## Why this exists

Mid-end passes are mostly target-independent, but profitability is not. X86 loves certain shuffles; AArch64 has different memory op costs; some CPUs lack masked load ISAs. TTI funnels those facts into LoopVectorize, SLP, Unroll, SimplifyCFG, and parts of the inliner.

## Diagram

```text
   LoopVectorize
        |
        |  "what does this target want?"
        v
   +------------------+
   | TargetTransform  |---- X86TTI    (AVX widths, costs)
   | Info             |---- AArch64TTI
   +------------------+---- RISCVTTI
        |
        +-- getRegisterBitWidth(Fixed)
        +-- getArithmeticInstrCost(Add, v4i32)
        +-- getNumberOfRegisters
        +-- isLegalMaskedLoad
```

TTI is the question interface between target-independent passes and the backend. The vectorizer does not include `X86ISelLowering.h`. It asks TTI how wide a vector register is and what an add costs. Each target subclasses `BasicTTIImpl` and overrides the answers. A cost of 1 means "as cheap as a typical instruction," not "one cycle."

## The idea

[TargetTransformInfo.h](https://github.com/llvm/llvm-project/blob/main/llvm/include/llvm/Analysis/TargetTransformInfo.h) declares a façade; [TargetTransformInfo.cpp](https://github.com/llvm/llvm-project/blob/main/llvm/lib/Analysis/TargetTransformInfo.cpp) implements dispatch. Per-target overrides live under `llvm/lib/Target/*/`:

- [X86TargetTransformInfo.cpp](https://github.com/llvm/llvm-project/blob/main/llvm/lib/Target/X86/X86TargetTransformInfo.cpp)
- [AArch64TargetTransformInfo.cpp](https://github.com/llvm/llvm-project/blob/main/llvm/lib/Target/AArch64/AArch64TargetTransformInfo.cpp)

### Cost kinds

```text
TCK_RecipThroughput
TCK_Latency
TCK_CodeSize
TCK_SizeAndLatency
```

Queries often take a `TargetCostKind`. Throughput costs dominate vectorization; code size matters for `-Os`/`-Oz`.

### Important APIs

- `getInstructionCost`
- `getArithmeticInstrCost`
- `getMemoryOpCost`
- `getCFInstrCost`
- `getUnrollingPreferences`
- `getMaxInterleaveFactor`
- `isLegalMaskedLoad` (and friends)

### Default baseline

`TargetTransformInfoImplBase` provides generic costs (commonly treating many ops as cost 1). When `opt` runs without a TargetMachine/`-mcpu`, passes see this bland model—**different unroll/vectorize decisions than clang -O3** with `-march=native`.

## Where it sits

| Item | Value |
|------|--------|
| Analysis | Provided via `TargetIRAnalysis` / TTI from the TargetMachine |
| Printer | `print<cost-model>` (`CostModelPrinterPass` in PassRegistry.def) |
| Users | LoopVectorize, SLPVectorizer, LoopUnroll, SimplifyCFG (lookup tables, etc.), inliner heuristics (some) |

```bash
opt -passes='print<cost-model>' -disable-output f.ll
# Better: use a target
# clang -O3 -emit-llvm ... already embeds target-aware decisions in the pipeline
```

When experimenting with `opt`, pass target flags your build supports (e.g. `-mtriple=...` `-mcpu=...` with a configured LLVM build that includes that backend).

## Core concepts

### Cost model ≠ correctness

TTI never decides legality alone; it decides **whether a legal transform is worth it**.

### Unrolling preferences

`getUnrollingPreferences` fills a struct (partial/full/runtime thresholds, force flags). This is why AArch64 and X86 unroll differently for the same IR.

### Vector legality helpers

`isLegalMaskedLoad`, gather/scatter legality, interleave factor caps—all feed LV’s plan.

### Recip throughput vs latency

Vectorizers mostly minimize reciprocal throughput cost of the loop body; latency models matter for critical paths / scheduling-minded heuristics.

## The algorithm (client view)

LoopVectorize pseudocode:

1. Build candidate VF/UF plans that are **legal** (LAA, TTI legality).
2. For each plan, sum `getInstructionCost` / memory / shuffle costs under `TCK_RecipThroughput`.
3. Pick the minimum cost plan that beats scalar.

## Where to read the code

1. Cost kind enum + `getInstructionCost` in TargetTransformInfo.h.
2. `TargetTransformInfoImplBase` defaults.
3. One concrete override in X86TargetTransformInfo.cpp (e.g. shuffle or memory cost).
4. `CostModelPrinterPass` in CostModel.cpp.
5. LoopVectorize cost-model call sites (optional).

## Worked example

The same reduction loop may:

- Vectorize at VF=8 on one `-mcpu`
- Stay scalar on generic TTI because “vector ops cost the same as scalar” removes the incentive, or legality fails for masked ops

Printing the cost model on a function shows per-instruction costs under the active TTI.

## Reading order

1. Cost kinds.
2. Why bare `opt` differs from clang.
3. Unrolling preferences + masked load legality.
4. DeepDive + Examples.

## Hands-on

```bash
cat > cost.c <<'EOF'
int sum(int *a, int n) {
  int s = 0;
  for (int i = 0; i < n; ++i) s += a[i];
  return s;
}
EOF
clang -S -emit-llvm -O1 -Xclang -disable-llvm-passes cost.c -o cost.ll
opt -passes='print<cost-model>' -disable-output cost.ll
# Compare pipelines:
opt -passes='loop-vectorize' -S cost.ll -o cost.lv.ll
clang -O3 -Rpass=loop-vectorize cost.c  # remarks when vectorizing
```

## Pitfalls

- Expecting `opt` without target info to match clang -O3 vectorization.
- Treating cost 1 as “cycles.”
- Confusing TLI (library availability, Day 39) with TTI (hardware cost/legality).

## Neighbors

- TLI (Day 39), LoopVectorize, SLP, Unroll, LoopInterchange profitability (Day 30).

## Check yourself

1. **Q:** Name four `TargetCostKind`s.  
   **A:** RecipThroughput, Latency, CodeSize, SizeAndLatency.

2. **Q:** Default impl when no target?  
   **A:** `TargetTransformInfoImplBase` generic costs.

3. **Q:** Printer pass?  
   **A:** `print<cost-model>`.

4. **Q:** Name three TTI APIs.  
   **A:** e.g. `getInstructionCost`, `getUnrollingPreferences`, `isLegalMaskedLoad`.

5. **Q:** TLI vs TTI?  
   **A:** TLI = which libfuncs exist; TTI = what ops cost / what ISA features exist.

## InstructionCost type

Modern TTI returns `InstructionCost`, which can represent valid numeric costs or invalid/impossible costs when an op cannot be lowered cheaply. Clients must check validity, not assume every cost is a plain integer. That API evolution matters when reading older blog posts that treat costs as raw `unsigned`.

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

## Extended hands-on checklist (Day 40)

1. Build IR with the disable-llvm-passes recipe.
2. Run the minimal pass pipeline named in this day’s main note.
3. Diff against a pipeline that adds one neighbor pass.
4. Re-run with the day’s `-debug-only` flag.
5. Skim one upstream lit test RUN line and reproduce it.
6. Write one counterexample IR that must not transform.
7. Cross-check a fact against the linked github source file.

## InstructionCost type

Modern TTI returns `InstructionCost`, which can represent valid numeric costs or invalid/impossible costs when an op cannot be lowered cheaply. Clients must check validity, not assume every cost is a plain integer. That API evolution matters when reading older blog posts that treat costs as raw `unsigned`.

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

## Extended hands-on checklist (Day 40)

1. Build IR with the disable-llvm-passes recipe.
2. Run the minimal pass pipeline named in this day’s main note.
3. Diff against a pipeline that adds one neighbor pass.
4. Re-run with the day’s `-debug-only` flag.
5. Skim one upstream lit test RUN line and reproduce it.
6. Write one counterexample IR that must not transform.
7. Cross-check a fact against the linked github source file.
