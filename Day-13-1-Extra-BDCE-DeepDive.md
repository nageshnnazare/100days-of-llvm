# Day 13 Extra: BDCE Deep Dive

Companion to [Day-13-0-BDCE.md](Day-13-0-BDCE.md). Examples: [Day-13-2-Extra-BDCE-Examples.md](Day-13-2-Extra-BDCE-Examples.md).

## Diagram

```text
   %a = and i32 %x, 1        demanded bits of %a: ---- ---1
   %b = and i32 %a, 1        demanded bits of the
                             second and: only bit 0

   so the first `and` is redundant:
   bit 0 of %x already flows to %b

   %dead = or i32 %x, 0      if no demanded bit reaches a
                             root (store, return, branch),
                             the instruction is dead
```

BDCE asks, for each bit of each value, whether any use actually reads that bit. A use of only the low bit of an `and` means the high bits of the operand were never needed. An instruction whose bits are demanded by nobody is deleted, including some instructions DCE would keep because they have a use that itself turns out to be dead.

## 1. Files and entry points

- [llvm/lib/Transforms/Scalar/BDCE.cpp](https://github.com/llvm/llvm-project/blob/main/llvm/lib/Transforms/Scalar/BDCE.cpp)
- [llvm/include/llvm/Transforms/Scalar/BDCE.h](https://github.com/llvm/llvm-project/blob/main/llvm/include/llvm/Transforms/Scalar/BDCE.h)
- [llvm/lib/Analysis/DemandedBits.cpp](https://github.com/llvm/llvm-project/blob/main/llvm/lib/Analysis/DemandedBits.cpp)
- [llvm/include/llvm/Analysis/DemandedBits.h](https://github.com/llvm/llvm-project/blob/main/llvm/include/llvm/Analysis/DemandedBits.h)

`DEBUG_TYPE` is `"bdce"`. Statistics: `NumRemoved`, `NumSimplified`, `NumSExt2ZExt`.

`BDCEPass::run` fetches `DemandedBitsAnalysis`, runs `bitTrackingDCE`, preserves `CFGAnalyses` on change.

Legacy PM wrappers for BDCE/DemandedBits were removed; use the new PM (`-passes=bdce`).

## 2. `DemandedBits` API (what BDCE calls)

From the header (conceptual):

- `APInt getDemandedBits(Instruction *I)` — union of bits demanded of `I`’s value.
- `APInt getDemandedBits(Use *U)` — per-use demand (finer).
- `bool isInstructionDead(Instruction *I)` — never reached / no demand in analysis.
- `bool isUseDead(Use *U)` — this use demands no bits.

Analysis is lazy / computed on demand for the function. Known-bits style transfer functions walk uses backward. Visitation avoids asking demanded bits of non-integer results (void calls, etc.).

## 3. `bitTrackingDCE` walk

Order inside the main loop over `instructions(F)`:

1. **Skip** `mayHaveSideEffects && use_empty` early (no benefit computing demand).
2. **Delete candidates:** `isInstructionDead` OR (integer type AND demanded bits all zero AND `wouldInstructionBeTriviallyDead`).
3. **SExt → ZExt:** if `countl_zero(Demanded) >= DestWidth - SrcWidth`, high bits of the sext are unused → replace with `zext`, salvage dbg, erase later.
4. **And/Or/Xor simplify:** if operand 1 is constant mask:
   - `or`/`xor`: simplify when mask disjoint from demanded bits.
   - `and`: simplify when demanded bits ⊆ mask (mask does not clear any demanded bit).
   Replace uses with operand 0.
5. **Dead uses:** for each integer operand that is an instruction or argument, if `isUseDead`, `U.set(ConstantInt::get(…, 0))`.

After the scan: reverse worklist `salvageDebugInfo` + `dropAllReferences`, then erase.

## 4. `clearAssumptionsOfUsers`

When trivializing, NSW/NUW/`exact` on *users* may be invalid because the operand changed. The helper DFS-walks integer users, calling `dropPoisonGeneratingAnnotations`, stopping when a user still demands all bits (nothing below can have been affected by *this* trivialization’s under-demand).

Comment in source: no need to special-case `llvm.assume` here—the assume demands its operand, so BDCE will not trivialize that operand away casually through the dead-bits path the same way.

## 5. What is NOT handled

- **Dead control flow** — no branch deletion, no block removal.
- **Non-integer** computations as primary targets.
- **Full multiply narrowing** to i8 just because low 8 bits demanded — often InstCombine’s job after masks are canonical; BDCE’s contract is dead-bit DCE / trivialization.
- **Side-effecting** calls with empty ordinary uses — still effects.
- Replacing dead uses with `freeze poison` — code comments prefer `0` as the substitute.

## 6. Interaction with SCCP and InstCombine

Pipeline comment (PassBuilder): SCCP → BDCE → InstCombine.

- SCCP creates constants and unreachable edges → fewer live sinks → more zero-demand regions.
- BDCE clears dead bit ops and dead uses.
- InstCombine pattern-matches the simplified masks/xors and may narrow types.

Running `bdce` alone on IR that still has many uses of high bits shows little change; invent IR where a value is fully masked out or only feeds undemanded sext bits.

## 7. Debugging

```bash
opt -passes=bdce -debug-only=bdce -S in.ll 2>bdce.log
```

Expect lines like `BDCE: Trivializing: … (all bits dead)` for dead-use replacement.

Also dump demand:

```bash
opt -passes='print<demanded-bits>' -disable-output in.ll
```

(Exact printer pass name follows the analysis’s print pipeline registration—if unavailable in your build, use `-debug-only` inside tests or `llvm-assert` builds with DemandedBits debug.)

## 8. Correctness invariants

- Never remove side effects.
- Never claim demand on non-int types through the integer API.
- After changing bits, drop poison-generating flags on affected user chains.
- CFG shape preserved (`CFGAnalyses`).

## 9. Per-use deadness motivation

Multi-use values: one use may be a `trunc` to i8 (low bits) while another returns the full width. The instruction stays live; the trunc use still demands low bits. Conversely, a use that feeds an `and` with `0` can be dead even if other uses live—zeroing that use can unlock InstCombine on the sibling pattern (historical motivation for `isUseDead`).

## 10. Source reading checklist

1. `BDCEPass::run`.
2. `bitTrackingDCE` cases in order.
3. `clearAssumptionsOfUsers`.
4. `DemandedBits::getDemandedBits` for instructions and uses.
5. One opcode transfer in `DemandedBits.cpp` (shift or bitwise).

## 11. Vectors

Integer vectors are included (`isIntOrIntVectorTy`). Demand masks are per-element-width APInts as implemented by the analysis—read vector paths carefully before assuming scalar-only examples.

## 12. DemandedBits transfer intuition

For an `and` with constant mask `C`, bits demanded of the non-mask operand are roughly `Demanded(and) & C` (analysis details vary with known bits). For `lshr` by constant `K`, demand shifts left by `K` when propagating to the LHS. For `trunc` to width `W`, only the low `W` bits of the source are demanded. Reading one of these transfers in `DemandedBits.cpp` end-to-end is worth more than memorizing BDCE’s rewrite list.

Analysis also tracks “dead instructions” never reached from sinks—`isInstructionDead`—distinct from “reached but zero mask.”

## 13. Erase protocol

BDCE collects erase candidates, then:

1. Reverse order: `salvageDebugInfo`, `dropAllReferences` (breaks cycles among doomed instrs).
2. Forward order: `eraseFromParent`, bump `NumRemoved`.

Skipping salvage would drop location info for survivors that referenced deleted values. When reducing crashes, preserve this two-phase delete.

## 14. And/or/xor simplify conditions (restated)

Given demanded mask `D` and constant operand `M`:

- `or`/`xor`: simplify if `(D & M) == 0` (no demanded bit is affected by the op).
- `and`: simplify if `D` is a subset of `M` (every demanded bit is already guaranteed by the mask; the `and` is redundant).

Self-referential IR (`%x = or i32 %x, 1` style) is explicitly guarded against.

## 15. What “low 8 bits” looks like in demand

If the only sink is `trunc i32 %v to i8` or `and i32 %v, 255` feeding a return of i8/i32 with high bits unused, `getDemandedBits(%v)` should show only low bits set. Producers of bit 31 of `%v` can die. Confirm with the demanded-bits printer when available; otherwise infer from which instructions BDCE deletes under `-debug-only=bdce`.

## 16. Non-goals versus ADCE

ADCE removes instructions not contributing to *any* live root along control-dependent paths, including some CFG-aware liveness. BDCE assumes control flow as given and asks a bit-mask question. A dead basic block still “demands” nothing useful once SCCP/SimplifyCFG kill it—but BDCE will not delete the block.

## 17. Takeaway

BDCE is a DemandedBits consumer that deletes zero-demand integer ops, zeros dead uses, and simplifies masks/sexts—leaving CFG and side effects alone for other passes.

## 18. Known interaction with `wouldInstructionBeTriviallyDead`

Zero demanded bits alone is not enough if the instruction has side effects or other liveness the helper cares about. The conjunction with `wouldInstructionBeTriviallyDead` prevents deleting volatile ops, language side effects, etc. Always read both conditions in the delete predicate when debugging a “why wasn’t this deleted?” question.

## 19. Debug-info salvage

`salvageDebugInfo` before erase tries to keep `dbg.value` meaningful when the deleted instruction was an intermediate. BDCE intentionally replaces non-dbg uses first in some paths (`replaceNonMetadataUsesWith` for sext/zext) so dbg users can still be salvaged pointing at the old instruction briefly. Treat debug preservation as part of the algorithm, not an afterthought.

## 20. Testing tips

LLVM’s `llvm/test/Transforms/BDCE/` tests are the ground truth for patterns that must fire. Copy a failing personal example into that style (`; RUN: opt -passes=bdce` with FileCheck). Prefer checking deleted opcodes and remaining masks over exact value names.

## 23. Preserve set

On change, BDCE preserves `CFGAnalyses` only—memory analyses that keyed off deleted values must recompute. That matches “CFG untouched, instructions deleted.”

## 24. Final BDCE motto

No bits demanded + no side effects → delete; dead use → zero; mask/`sext` cleanup when demand allows.
