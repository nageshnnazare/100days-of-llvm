# Day 40 Extra: TTI — Deep Dive

Sources: [TargetTransformInfo.h](https://github.com/llvm/llvm-project/blob/main/llvm/include/llvm/Analysis/TargetTransformInfo.h), [TargetTransformInfo.cpp](https://github.com/llvm/llvm-project/blob/main/llvm/lib/Analysis/TargetTransformInfo.cpp), [X86TargetTransformInfo.cpp](https://github.com/llvm/llvm-project/blob/main/llvm/lib/Target/X86/X86TargetTransformInfo.cpp), [AArch64TargetTransformInfo.cpp](https://github.com/llvm/llvm-project/blob/main/llvm/lib/Target/AArch64/AArch64TargetTransformInfo.cpp), printer in [CostModel.cpp](https://github.com/llvm/llvm-project/blob/main/llvm/lib/Analysis/CostModel.cpp). Parent: [Day-40-0-TTI.md](Day-40-0-TTI.md).

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

## Façade pattern

`TargetTransformInfo` is a value type holding a concept/vtable-style pointer to a target impl. Passes ask the analysis manager for TTI; under a TargetMachine, `TargetIRAnalysis` returns a TTI bound to `X86TTIImpl` / `AArch64TTIImpl` / etc. Without TM, you get the base stub.

This design keeps the mid-end free of `#include "X86.h"` while still specializing costs.

## CostKind semantics

| Kind | Typical user intent |
|------|---------------------|
| `TCK_RecipThroughput` | How many ops / cycle approx when pipelined — LV/SLP |
| `TCK_Latency` | Critical-path delay |
| `TCK_CodeSize` | Bytes / uops size for `-Os` |
| `TCK_SizeAndLatency` | Blended metric; some APIs default here |

When reading a cost hook, always note which CostKind the client passed—comparing a throughput number to a size threshold is meaningless.

## Key hooks to skim in the header

1. **`getArithmeticInstrCost`** — add/mul/fadd by type and operand kinds (uniform constants cheaper, etc.).
2. **`getMemoryOpCost`** — load/store alignment, address space, interleaving.
3. **`getShuffleCost`** (nearby) — often the make-or-break for SLP.
4. **`getCFInstrCost`** — branches/switches; SimplifyCFG uses related cost ideas for lookup tables.
5. **`getUnrollingPreferences`** — fills `UnrollingPreferences` (thresholds, partial thresholds, runtime unroll).
6. **`getMaxInterleaveFactor`** — caps LV interleave.
7. **`isLegalMaskedLoad` / store / gather / scatter** — legality gates.

## X86 vs AArch64 reading strategy

Do not read entire TTI files (they are huge). Instead:

1. Jump to `getShuffleCost` or memory cost overrides.
2. Note CPU feature checks (`AVX2`, `AVX512`, SVE predicates, etc.).
3. Compare one opcode’s cost on a small type vs a wide vector type.

The lesson: **same IR, different numbers**.

## Cost model printer

`CostModelPrinterPass` registered as `print<cost-model>` walks instructions and prints TTI costs. Use it to see what the active TTI believes before blaming LoopVectorize heuristics.

```bash
opt -passes='print<cost-model>' -disable-output f.ll
```

## Interaction with TLI

- TLI: “May I call `__svml_sin4`?” / “Does memset exist?”
- TTI: “Does a masked load cost 2 or 40?” / “Is masked load legal?”

LV needs both for math-heavy loops: TTI for SIMD ops, TLI for vector libcalls.

## Why clang -O3 ≠ bare opt

clang constructs a full TargetMachine from `-march`/`-mcpu`/triple, registers the right TTI, sets TLI, and runs a pass pipeline with target cpu features in the IR module flags / attributes. Bare `opt -passes=loop-vectorize` on IR without target attrs uses generic TTI → fewer or different vector plans.

When writing lit tests for LV, note how many specify `-mtriple` and `-mattr=+avx2` style flags.

## Inliner / SimplifyCFG (secondary users)

Some heuristics consult TTI for call costs or switch lowering thresholds. Not the main teaching path—but explains why TTI sits in Analysis rather than only in Vectorize.

## Reading path

1. CostKind enum + `getInstructionCost` docs.
2. `TargetTransformInfoImplBase` default returns.
3. One X86 override function.
4. `getUnrollingPreferences` struct fields.
5. Cost model printer on a tiny function.
6. Examples.

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

## Source navigation checklist (Day 40 DeepDive)

1. Open the primary .cpp linked from the main note.
2. Jump to DEBUG_TYPE / cl::opt block.
3. Find the public `run` method or analysis entry.
4. Find the core legality helper and the core transform helper.
5. Grep remark strings / LLVM_DEBUG strings you saw experimentally.
6. Note which analyses are required vs preserved.
7. Skim one Failure path and one Success path to completion.

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

## Source navigation checklist (Day 40 DeepDive)

1. Open the primary .cpp linked from the main note.
2. Jump to DEBUG_TYPE / cl::opt block.
3. Find the public `run` method or analysis entry.
4. Find the core legality helper and the core transform helper.
5. Grep remark strings / LLVM_DEBUG strings you saw experimentally.
6. Note which analyses are required vs preserved.
7. Skim one Failure path and one Success path to completion.

## InstructionCost type

Modern TTI returns `InstructionCost`, which can represent valid numeric costs or invalid/impossible costs when an op cannot be lowered cheaply. Clients must check validity, not assume every cost is a plain integer. That API evolution matters when reading older blog posts that treat costs as raw `unsigned`.

## IR hygiene for experiments

Always generate IR with `clang -S -emit-llvm -O1 -Xclang -disable-llvm-passes` so functions are not marked `optnone`. Opaque pointers (`ptr`) are mandatory in modern IR examples; do not revive typed pointer syntax in new notes.

When comparing before/after, prefer `diff -u` on files produced by the same opt pipeline with one pass removed. That isolates the transform better than comparing against clang -O3, which runs dozens of passes.

## Verifier and analysis invalidation

After a transform, `opt -passes='verify'` (or the verifier embedded in pipelines) should still succeed. Loop passes must preserve LCSSA and simplified form when they advertise those preservations. Analysis managers invalidate SCEV, AA, and MemorySSA according to the preserved-analyses set returned by the pass—reading `PreservedAnalyses` in the pass’s `run` method teaches what clients may keep.
