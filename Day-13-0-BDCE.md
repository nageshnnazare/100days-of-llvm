# Day 13: Bit-Tracking Dead Code Elimination (BDCE)

BDCE deletes or trivializes integer computations whose **demanded bits** are empty (or whose dead bits make an operation a no-op). It is driven by the `DemandedBits` analysis. Pass name: `bdce` (`BDCEPass`). Integer-only; it does not remove dead control flow.

Deep dive: [Day-13-1-Extra-BDCE-DeepDive.md](Day-13-1-Extra-BDCE-DeepDive.md). Examples: [Day-13-2-Extra-BDCE-Examples.md](Day-13-2-Extra-BDCE-Examples.md).

## 1. Why this exists

Classic DCE / ADCE remove instructions with no uses. Many instructions still have uses, but those uses only read a subset of bits—or no bits at all after masks and truncations. Example: compute a wide multiply, then `and` with `255` and `trunc` to `i8`. High bits of the multiply never affect the program. Ordinary DCE keeps the full multiply chain because SSA uses exist.

Bit-tracking DCE answers: for each integer SSA value, which bits are demanded by any root (returns, stores, side-effecting calls, branches, etc.)? If an instruction demands **no** bits and would be trivially dead, delete it. If an operand use demands no bits, replace that use with zero. If a mask/`or`/`xor` does not touch demanded bits, simplify.

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

## 2. The idea

`DemandedBits` performs a backward analysis from sinks that demand bits. Shifts, `and`/`or`/`xor`, extensions, and truncations kill or transfer subsets of bits. BDCE consumes that map:

- `DB.isInstructionDead(&I)` or `getDemandedBits(&I).isZero()` with `wouldInstructionBeTriviallyDead` → delete.
- Per-use: `DB.isUseDead(&U)` → set operand to `0`.
- Special cases: `sext` → `zext` when extension bits undemanded; simplify `and`/`or`/`xor` with constant masks that do not affect demanded bits.

Side-effecting instructions and assumptions that demand their operands stay. BDCE does **not** delete dead branches or unreachable blocks—that is CFG / ADCE territory.

## 3. Where it sits in the -O2/-O3 pipeline

In the default function simplification pipeline, `BDCEPass` runs **immediately after** `SCCPPass` and **before** InstCombine (see `PassBuilderPipelines.cpp`). Comment in-tree: delete dead bit computations; InstCombine folds further; ADCE later catches new DCE opportunities.

| Item | Value |
|------|--------|
| Pass name | `bdce` |
| Class | `BDCEPass` |
| Analysis | `DemandedBitsAnalysis` → `DemandedBits` |
| Debug | `-debug-only=bdce` |

```bash
clang -S -emit-llvm -O1 -Xclang -disable-llvm-passes file.c -o file.ll
opt -passes=bdce -S file.ll -o file.bdce.ll
```

## 4. Core concepts

### Demanded bits

For an integer-typed instruction, an `APInt` mask of bits that somehow affect observable behavior. Empty mask ⇒ candidate for deletion if no side effects.

### Dead uses of live instructions

An instruction can be live (some uses demand bits) while **one** use demands nothing. BDCE can replace that use with `0` (`isUseDead`), enabling later simplifies (e.g. rotate patterns → shifts).

### Trivialization vs deletion

“Trivialized” means replacing with a simpler operand or zeroing a use (`NumSimplified`). “Removed” means erase (`NumRemoved`). `clearAssumptionsOfUsers` drops poison-generating annotations on users when bits change underneath them.

### Integer only

`isIntOrIntVectorTy` gates queries. Pointer / FP / aggregate bit-tracking is out of scope for this pass.

## 5. The algorithm

`bitTrackingDCE(Function &F, DemandedBits &DB)` in [BDCE.cpp](https://github.com/llvm/llvm-project/blob/main/llvm/lib/Transforms/Scalar/BDCE.cpp):

1. Get `DemandedBits` from the analysis manager.
2. For each instruction:
   - Skip useless analysis on side-effecting unused instrs early.
   - If dead / zero demanded bits and trivially dead → push to erase worklist.
   - Else try `sext`→`zext` when high bits undemanded.
   - Else try simplify `and`/`or`/`xor` with constant masks vs demanded mask.
   - Else for each integer operand use, if `isUseDead`, set to zero constant.
3. Salvage debug info; drop references; erase collected instructions.
4. Preserve CFG analyses (`CFGAnalyses`).

## 6. Where to read the code

- [llvm/lib/Transforms/Scalar/BDCE.cpp](https://github.com/llvm/llvm-project/blob/main/llvm/lib/Transforms/Scalar/BDCE.cpp) — `BDCEPass::run`, `bitTrackingDCE`, `clearAssumptionsOfUsers`
- [llvm/include/llvm/Transforms/Scalar/BDCE.h](https://github.com/llvm/llvm-project/blob/main/llvm/include/llvm/Transforms/Scalar/BDCE.h)
- [llvm/lib/Analysis/DemandedBits.cpp](https://github.com/llvm/llvm-project/blob/main/llvm/lib/Analysis/DemandedBits.cpp) — analysis implementation
- [llvm/include/llvm/Analysis/DemandedBits.h](https://github.com/llvm/llvm-project/blob/main/llvm/include/llvm/Analysis/DemandedBits.h) — `getDemandedBits`, `isInstructionDead`, `isUseDead`

## 7. Worked before/after

**C** (`bdce_low8.c`):

```c
unsigned low8(unsigned x, unsigned y) {
  unsigned wide = x * y;      /* many bits computed */
  return wide & 0xffu;        /* only low 8 demanded */
}
```

**Commands:**

```bash
clang -S -emit-llvm -O1 -Xclang -disable-llvm-passes bdce_low8.c -o bdce_low8.ll
opt -passes=bdce -S bdce_low8.ll -o bdce_low8.out.ll
```

**IR idea before:**

```llvm
define i32 @low8(i32 %x, i32 %y) {
  %wide = mul i32 %x, %y
  %masked = and i32 %wide, 255
  ret i32 %masked
}
```

**After BDCE / with DemandedBits insight:** the multiply’s high bits are undemanded for the `and`/`ret` pattern; BDCE may not rewrite the mul width by itself the way InstCombine would narrow, but it **will** delete pure dead bit producers and zero dead uses. A clearer BDCE-only win:

```llvm
define i32 @dead_or(i32 %x) {
  %y = mul i32 %x, %x          ; suppose only used in undemandeds
  %z = and i32 %y, 0           ; demands no bits of %y meaningfully
  ret i32 %z
}
```

If `%y`’s bits are undemanded and the mul is trivially dead after trivialization, BDCE erases it. Prefer the examples file for precise patterns that fire under `bdce` alone.

**Shrink intuition:** “only low 8 bits demanded” is the teaching slogan—masks and truncs are how demand shrinks; InstCombine often finishes width changes after BDCE clears dead bit ops.

## 8. Reading order for today

1. File header in `BDCE.cpp`.
2. `bitTrackingDCE` loop order (delete → sext → and/or/xor → dead uses).
3. `DemandedBits.h` public API.
4. One transfer function in `DemandedBits.cpp` (e.g. `and` or `lshr`).
5. Deep dive + examples.

## 9. Hands-on commands

```bash
clang -S -emit-llvm -O1 -Xclang -disable-llvm-passes bdce_low8.c -o bdce_low8.ll
opt -passes=bdce -S bdce_low8.ll -o out.ll
opt -passes=bdce -debug-only=bdce -S bdce_low8.ll 2>&1 | less
opt -passes='sccp,bdce,instcombine' -S bdce_low8.ll -o out.pipeline.ll
opt -passes='print<demanded-bits>' -disable-output bdce_low8.ll
```

## 10. Pitfalls and invariants

- Integer / int-vector only; do not expect FP BDCE.
- Side-effecting calls stay even if return bits undemanded in some sense—effects matter.
- `llvm.assume` demands its condition; trivializing must not casually break assume contracts (`clearAssumptionsOfUsers` documents the NSW/NUW/`exact` drop case).
- BDCE does **not** remove dead control flow.
- After trivializing, poison flags on users may be dropped—necessary for correctness.
- Opaque pointers: demand is on integer payloads, not on `ptr` bits as a BDCE target.

## 11. Neighbors

- **Before:** `sccp` exposes constants and dead paths that shrink demand.
- **After:** InstCombine folds simplified masks; ADCE removes newly dead code.
- **Analysis:** `DemandedBits` also used elsewhere; InstCombine has its own demanded-bits logic for combines.
- **Not the same as:** ADCE (control-dependent dead code), DCE (use-empty only).

## 12. Check yourself

**Q1.** Pass name?
**A1.** `bdce`.

**Q2.** Which analysis?
**A2.** `DemandedBits` (`DemandedBits.cpp`).

**Q3.** What if no bits demanded and no side effects?
**A3.** Instruction can be deleted (trivially dead path).

**Q4.** Does BDCE delete dead branches?
**A4.** No.

**Q5.** Debug flag?
**A5.** `-debug-only=bdce`.

**Q6.** Why drop NSW/NUW on users after trivializing?
**A6.** Operand bits changed; old poison-generating assumptions may no longer hold.

> Tip: Implementation details are in `Day-13-1-Extra-BDCE-DeepDive.md`. Runnable cases are in `Day-13-2-Extra-BDCE-Examples.md`.

## Appendix A: Demand sinks

Roots that demand bits include: return values, store stored-values, branch/switch conditions, call arguments (subject to ABI / attrs), atomic ops, and other observable uses. A value that only feeds a chain into a fully masked-out path can end with zero demand.

## Appendix B: BDCE vs InstCombine demanded bits

InstCombine uses demanded bits locally to rewrite patterns (narrow ops, kill masks). BDCE is a dedicated sweep that deletes/trivializes based on a whole-function `DemandedBits` result. They reinforce each other in the pipeline: BDCE after SCCP, InstCombine right after BDCE.

## Appendix C: Worked mask table (teaching)

| Pattern | Demanded effect | BDCE angle |
|---------|-----------------|------------|
| `and x, C` | kills bits outside C going upward | dead producers of killed bits |
| `or x, C` / `xor x, C` | forces bits; may be removable if C∩demand=∅ | simplify to `x` |
| `shl` / `lshr` / `ashr` | shifts demand toward other bit positions | upstream bits may die |
| `trunc` | demands only low dest-width bits | high bits undemand |
| `zext` / `sext` | may demand (or not) high bits of result | sext→zext case |

Walk one toy IR through this table on paper before trusting `-debug-only=bdce`.

## Appendix D: Interaction with assumes and poison

If BDCE zeros an operand that fed a chain carrying `nsw`, users lose `nsw` via `dropPoisonGeneratingAnnotations`. That can disable later folds that needed nsw—but keeping a stale nsw would be a miscompile. Prefer correctness over preserving flags.

`llvm.assume` demands its condition bits; treating assume as “free to drop” is wrong. BDCE’s comments call this out when discussing trivialization.

## Appendix E: Pipeline experiment

```bash
opt -passes=bdce -S t.ll -o t1.ll
opt -passes='bdce,instcombine' -S t.ll -o t2.ll
opt -passes='sccp,bdce,instcombine' -S t.ll -o t3.ll
```

Diff `t1`/`t2`/`t3`. SCCP often creates the zero masks and dead paths that make DemandedBits empty; InstCombine finishes width changes BDCE only hints at by deleting dead bit math.

## Appendix F: One-line summary

BDCE = delete/trivialize integer ops whose DemandedBits mask is empty (or unused), without touching dead CFG.

## Appendix G: DemandedBits vs KnownBits

KnownBits (ValueTracking / InstCombine) answers “which bits are 0 or 1.” DemandedBits answers “which bits are observed.” A bit can be known one without being demanded, and demanded without being known. BDCE only needs demand. InstCombine often consults both. When a multiply’s high bits are unknown but undemanded, BDCE can still delete producers of those bits if no sink reads them.

## Appendix H: One more command

```bash
opt -passes='sccp,bdce' -debug-only=bdce -S t.ll 2>&1 | head
```
