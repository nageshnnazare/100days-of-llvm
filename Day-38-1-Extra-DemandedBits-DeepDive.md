# Day 38 Extra: Demanded Bits — Deep Dive

Source: [DemandedBits.cpp](https://github.com/llvm/llvm-project/blob/main/llvm/lib/Analysis/DemandedBits.cpp), [DemandedBits.h](https://github.com/llvm/llvm-project/blob/main/llvm/include/llvm/Analysis/DemandedBits.h). Parent: [Day-38-0-DemandedBits.md](Day-38-0-DemandedBits.md).

## Diagram

```text
   %a = or  i32 %x, 1        bit 0 of %a is 1; other bits are %x
   %b = and i32 %a, 254      bit 0 is cleared
                             so bit 0 of %x was never demanded

   demanded bits of %x:  1111 1110
                         (bit 0 is dead)

   InstCombine can then drop
   the `or` with 1 if no other
   use wants bit 0
```

DemandedBits computes, from the roots (returns, stores, branch conditions), which bits of each value can affect the result. A bit that no root reads is dead even if the instruction has users, as long as those users also ignore the bit. BDCE deletes instructions on this information; InstCombine uses it to shrink masks and constants.

## Analysis API

`DemandedBitsAnalysis` produces a `DemandedBits` object. Queries:

- `getDemandedBits(Instruction *I)` — mask of bits demanded from `I`’s result
- `getDemandedBits(Use *U)` — demand through a specific use edge (can differ per user)

`DEBUG_TYPE` is `"demanded-bits"`. Printer: `DemandedBitsPrinterPass` / `print<demanded-bits>`.

## Propagation engine

The .cpp maintains a map from instructions to APInt demand masks. A worklist processes nodes whose demand grew. For each user edge, `determineLiveOperandBits`-style logic (names vary) applies the transfer from the user’s opcode to the operand’s demanded mask, then unions into the operand instruction’s mask.

Roots: terminators returning values, stores (value operand), call operands that are observed, atomic ops, etc. Undemanded dead code never seeds—so truly dead instructions may have empty demand (BDCE fodder).

## Transfer function highlights

Read the switch carefully for:

### Bitwise

- `and`/`or`/`xor` with constants: mask filtering
- `shl`/`lshr`/`ashr` with constant amounts: shift the demand mask; bits shifted out become undemanded on the input; new bits may be known zero (not demand)

### Casts

- `trunc`: demand low `dstWidth` bits of input
- `zext`: demand low `srcWidth` bits; high bits of result are zero (not “demanded from input”)
- `sext`: demand low bits; sign bit of input may be demanded if any high result bit is demanded

### Arithmetic

Adds demand bits that can affect demanded result bits including carries—often wider than a naive “same mask on both operands.” Multiplies similarly. This is why demanded-bits on add is less surgical than on `and`.

### Phi / select

Union demands from the selected/phi result onto each incoming value (select condition has its own demand—usually the low bit).

## Relation to `SimplifyDemandedBits`

Located in ValueTracking / used heavily from InstCombine:

- Invoked on a single instruction with a demand mask from a local context
- May rewrite the instruction immediately (replace with constant, narrow, etc.)
- Does not compute a global fixed-point map for the whole function

DemandedBits **analysis** answers whole-function questions for BDCE; SimplifyDemandedBits is the peephole hammer.

## Relation to KnownBits

`computeKnownBits` propagates known zeros/ones **down** the def-use chains (forward). Example synergy:

- KnownBits: high bits of `%x` are zero
- DemandedBits: only low 8 bits demanded
- Combined: many ops become no-ops or narrower

They share bit-APInt vocabulary but opposite dataflow directions.

## BDCE consumption

BDCE walks instructions, asks DemandedBits, and:

- Clears undemanded bits by inserting masks or rewriting
- Deletes instructions with no demanded bits
- Avoids changing observable behavior (side-effecting ops always “live”)

## Performance

Bit masks are cheap; fixed-point on large functions with dense integer IR is usually fine. Exotic vector demanded-bits are more limited—focus teaching on scalars.

## Reading path

1. Header class comments.
2. Opcode transfer section in the .cpp.
3. Grep InstCombine for `SimplifyDemandedBits` call—one example.
4. Grep BDCE for `DemandedBits`.
5. Examples file.

## Demanded bits on phis

Phi demand is the union of demands on the phi result, pushed to each incoming value. In loops, that fixed point can mark an IV’s high bits undemanded if only a truncating use escapes—enabling BDCE to rewrite the IV update. Print demanded-bits on a looping trunc use to see the lattice settle.

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

## Source navigation checklist (Day 38 DeepDive)

1. Open the primary .cpp linked from the main note.
2. Jump to DEBUG_TYPE / cl::opt block.
3. Find the public `run` method or analysis entry.
4. Find the core legality helper and the core transform helper.
5. Grep remark strings / LLVM_DEBUG strings you saw experimentally.
6. Note which analyses are required vs preserved.
7. Skim one Failure path and one Success path to completion.

## Demanded bits on phis

Phi demand is the union of demands on the phi result, pushed to each incoming value. In loops, that fixed point can mark an IV’s high bits undemanded if only a truncating use escapes—enabling BDCE to rewrite the IV update. Print demanded-bits on a looping trunc use to see the lattice settle.

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

## Source navigation checklist (Day 38 DeepDive)

1. Open the primary .cpp linked from the main note.
2. Jump to DEBUG_TYPE / cl::opt block.
3. Find the public `run` method or analysis entry.
4. Find the core legality helper and the core transform helper.
5. Grep remark strings / LLVM_DEBUG strings you saw experimentally.
6. Note which analyses are required vs preserved.
7. Skim one Failure path and one Success path to completion.

## Demanded bits on phis

Phi demand is the union of demands on the phi result, pushed to each incoming value. In loops, that fixed point can mark an IV’s high bits undemanded if only a truncating use escapes—enabling BDCE to rewrite the IV update. Print demanded-bits on a looping trunc use to see the lattice settle.

## IR hygiene for experiments

Always generate IR with `clang -S -emit-llvm -O1 -Xclang -disable-llvm-passes` so functions are not marked `optnone`. Opaque pointers (`ptr`) are mandatory in modern IR examples; do not revive typed pointer syntax in new notes.

When comparing before/after, prefer `diff -u` on files produced by the same opt pipeline with one pass removed. That isolates the transform better than comparing against clang -O3, which runs dozens of passes.
