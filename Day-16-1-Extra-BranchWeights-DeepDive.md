# Day 16 Extra: Branch Weights Deep Dive

Companion to [Day-16-0-BranchWeights.md](Day-16-0-BranchWeights.md). Examples: [Day-16-2-Extra-BranchWeights-Examples.md](Day-16-2-Extra-BranchWeights-Examples.md).

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

## 1. Files and entry points

- [llvm/lib/Transforms/Scalar/LowerExpectIntrinsic.cpp](https://github.com/llvm/llvm-project/blob/main/llvm/lib/Transforms/Scalar/LowerExpectIntrinsic.cpp)
- [llvm/lib/Analysis/BranchProbabilityInfo.cpp](https://github.com/llvm/llvm-project/blob/main/llvm/lib/Analysis/BranchProbabilityInfo.cpp)
- [llvm/lib/Analysis/BlockFrequencyInfo.cpp](https://github.com/llvm/llvm-project/blob/main/llvm/lib/Analysis/BlockFrequencyInfo.cpp) (+ `BlockFrequencyInfoImpl` details)
- [llvm/lib/IR/MDBuilder.cpp](https://github.com/llvm/llvm-project/blob/main/llvm/lib/IR/MDBuilder.cpp)
- [llvm/include/llvm/IR/MDBuilder.h](https://github.com/llvm/llvm-project/blob/main/llvm/include/llvm/IR/MDBuilder.h)

Documentation: [https://llvm.org/docs/BranchWeightMetadata.html](https://llvm.org/docs/BranchWeightMetadata.html)

## 2. Metadata layout

`MDNode` attached via `Instruction::setMetadata(LLVMContext::MD_prof, …)`:

1. `MDString "branch_weights"`
2. One `ConstantInt` weight per successor (as specified for that terminator kind)
3. Optionally expect provenance tagging (`!"expected"`) depending on helper APIs

`MDBuilder::createBranchWeights(True, False, IsExpected)` and the `ArrayRef` overload are the usual constructors. `createLikelyBranchWeights` / `createUnlikelyBranchWeights` bake in the strong bias constants (`kLikelyBranchWeight` / `kUnlikelyBranchWeight` when present).

## 3. `lower-expect` path

Handlers include:

- Conditional branches whose condition is (derived from) an expect intrinsic
- Switches on expect (`handleSwitchExpect`) — builds a weight vector sized `numCases+1`, sets the expected case’s weight high, others low
- PHI-defined conditions (`handlePhiDef`) — walk copy-like ops, find dominating conditional branches, annotate them; for `expect.with.probability`, if probability ≤ 0.5 the “expected” value is treated as **unlikely** and weights swap

`getBranchWeight` distinguishes plain expect vs probability form: probability form converts the double into integer weights using a scaled `INT32_MAX`-based scheme and spreads remaining probability across other successors.

After annotation, the expect call is replaced by its value operand and removed.

## 4. BranchProbabilityInfo

BPI computation sketch:

1. If terminator has usable `branch_weights`, convert to `BranchProbability` fractions.
2. Else apply heuristic classification (loop, compare vs null, invoke edges, …).
3. Expose probabilities to clients via `getEdgeProbability`.

Invalid or malformed metadata should fall back safely—read the validation paths when debugging odd IR.

## 5. BlockFrequencyInfo

BFI takes BPI + LoopInfo-style structure and computes relative block frequencies (and sometimes profile counts when absolute profile data exists). Without ProfileSummary / entry counts, frequencies are still useful relatively inside a function.

Consumers threshold “hot” vs “cold” using either relative BFI or summary-based hotness when PGO summaries exist.

## 6. What this day does NOT cover

- **Instrumentation PGO** pipeline (`pgo-instr-gen` / `pgo-instr-use`) — later days; it *writes* the same `branch_weights` kind from counters.
- Sample PGO specifics.
- Assuming expect changes program semantics (it must not).

## 7. Clang mapping (conceptual)

| Source | Rough IR |
|--------|----------|
| `__builtin_expect(c, 1)` | expect intrinsic; true path likely |
| `__builtin_expect(c, 0)` | false path likely |
| `__builtin_expect_with_probability` | expect.with.probability + double |
| `[[likely]]` / `[[unlikely]]` | FE emits expect-like weights on labels |

Exact Clang codegen evolves—inspect Clang’s CodeGen for `EmitBuiltinExpr` / likely attributes when you need FE line-level truth; mid-end truth starts at the intrinsics and `!prof`.

## 8. Heuristics worth memorizing

Without metadata, BPI typically treats:

- Loop **backedges** as taken
- Comparisons against **null** / zero as unlikely equal (pattern-dependent)
- Exception / unwind edges as cold

These exist so unprofiled builds still layout loops sanely.

## 9. Debugging

```bash
opt -passes=lower-expect -S in.ll -o out.ll
opt -passes='print<branch-prob>' -disable-output out.ll
opt -passes='print<block-freq>' -disable-output out.ll
```

Dump `!prof` with `llvm-dis` / plain read of the `.ll`. Compare expect-derived weights to `MDBuilder` likely constants.

## 10. Interaction with misexpect

When real profiles disagree with expect annotations, LLVM’s misexpect diagnostics can warn. Useful when a codebase’s `__builtin_expect` bitrots. Still a performance issue, not a correctness one.

## 11. Source reading checklist

1. BranchWeightMetadata docs (operand counts).
2. `handleSwitchExpect` + branch expect handler in LowerExpectIntrinsic.cpp.
3. `MDBuilder` weight helpers.
4. BPI: metadata path then heuristic path.
5. One BFI client call site (e.g. hot/cold split) for motivation.

## 12. BranchProbability type

Clients rarely divide integers by hand. `BranchProbability` wraps a numerator/denominator with saturating helpers. BPI returns these per edge. When writing a new pass, prefer BPI APIs over parsing `MD_prof` yourself—metadata shapes evolve (optional expect tags, unpredictable MD).

`MD_prof` also hosts other strings (`function_entry_count`, `VP`, etc.). Always check the `MDString` tag before interpreting operands as branch weights.

## 13. Unpredictable branches

`MDBuilder::createUnpredictable` marks branches that should not be statically predicted aggressively. Distinct from weights. Useful when a branch is known chaotic; layout should avoid assuming a side.

## 14. Invoke and exception edges

BPI heuristics treat unwind edges as cold. Expect lowering focuses on ordinary `br`/`switch`; exception edges usually get cold probabilities without expect. When reading EH-heavy IR, do not expect `__builtin_expect` on landing pads.

## 15. Updating weights when CFG changes

Passes that duplicate or redirect terminators must update or drop `!prof` (Scale utilities exist under profile helper headers). Stale weights after JumpThreading-like edits cause BPI confusion. When your transform clones a `br`, clone or recompute its metadata intentionally.

## 16. Teaching drill

1. Write IR with expect intrinsic by hand.
2. Run `lower-expect`.
3. Print BPI.
4. Strip `!prof` and print BPI again—observe heuristic differences.
5. Add `llvm.assume` on the same condition in a twin function and show InstCombine can delete a side that weights never could.

## 17. Takeaway

Expect lowers to the same `branch_weights` MD that profiles use; BPI/BFI consume it; heuristics cover the gaps; keep assume and expect in separate mental registers.

## 18. Default expect weight magnitude

Older teaching materials cite 2000:1. Current `MDBuilder` likely uses `(1<<20)-1 : 1` (`kLikelyBranchWeight`). Always print the actual MD in your tree rather than memorizing a single pair. The educational point is “extremely biased,” not the exact integers.

## 19. `setBranchWeights` helper

LowerExpect uses shared helpers to attach weights and optional expected tags. Profile helper utilities elsewhere in LLVM also mutate weights when scaling CFG. When writing tests, prefer `MDBuilder` over hand-rolled MDNodes once you leave toy examples.

## 20. BPI and LoopInfo

Loop-aware heuristics need loop structure: backedges vs exits. BPI therefore depends on LoopInfo (or computes loop-ish structure—read the analysis dependencies in the BPI pass header). A malformed loop simplify form can change heuristic outcomes even with identical user code.

## 21. Cold blocks vs cold edges

BFI marks blocks cold; BPI marks edges. HotColdSplitting thresholds often use block frequency. A single cold edge into an otherwise warm diamond can still leave a block warm if another edge is hot. Draw the CFG when surprised by splitting decisions.

## 22. Connection to later PGO days

Instrumentation PGO will generate the same `branch_weights` MD from counters and may add ProfileSummary. Your mental model from this day transfers directly—only the producer changes. Keep expect/FE notes separate so PGO days can focus on counters, linking, and merging.

## 23. Verifier and arity

Hand-edited `branch_weights` with the wrong operand count are ignored or rejected depending on path—validate with `verify` and BPI print. Prefer `MDBuilder` once past the first toy.

## 24. Final branch-weight motto

Expect → `lower-expect` → `!prof`; BPI/BFI consume; heuristics if missing; assume still means truth.

## 25. See also

Main note and examples companions for this day.
