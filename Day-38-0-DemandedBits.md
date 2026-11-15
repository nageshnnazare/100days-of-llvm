# Day 38: Demanded Bits

DemandedBits is a **backward** bit-lattice analysis: for each integer instruction, which bits of its result are actually needed by users? That drives BDCE and related cleanups. Distinguish it from InstCombine’s local `SimplifyDemandedBits` and from forward `computeKnownBits`. Deep dive: [Day-38-1-Extra-DemandedBits-DeepDive.md](Day-38-1-Extra-DemandedBits-DeepDive.md). Examples: [Day-38-2-Extra-DemandedBits-Examples.md](Day-38-2-Extra-DemandedBits-Examples.md).

## Why this exists

```llvm
%z = and i32 %x, 255
%t = trunc i32 %z to i8
```

Only the low 8 bits of `%x` ever matter. Demanded bits can prove high bits of `%x` are irrelevant, enabling BDCE to delete dead extensions or replace ops with narrower equivalents.

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

## The idea

A bit-vector (APInt mask) per instruction: bit `i` set ⇒ that bit of the result is demanded. Analysis starts from roots (returns, stores, calls, volatile, …) demanding all bits, then propagates **backward** through defs using opcode-specific transfer functions, taking the **union** (meet of demands) across all users.

Examples of transfers:

| Opcode | Backward effect (intuition) |
|--------|-----------------------------|
| `and` with const | Demand &= const mask on the variable operand |
| `shl` by const | Demand shifts toward higher bits on the input |
| `lshr`/`ashr` | Demand shifts toward lower bits |
| `trunc` | Only low bits of input demanded |
| `zext`/`sext` | Demand low bits; high bits of result come from extension rules |
| `add`/`sub`/`mul` | Generally demand all bits that users need on **both** operands (with refinements) |
| `or`/`xor` | Similar bit-wise demand push |

## Where it sits

| Item | Value |
|------|--------|
| Analysis | `demanded-bits` |
| File | [llvm/lib/Analysis/DemandedBits.cpp](https://github.com/llvm/llvm-project/blob/main/llvm/lib/Analysis/DemandedBits.cpp) |
| Class | `DemandedBits` |
| Print | `print<demanded-bits>` |
| Debug | `-debug-only=demanded-bits` |
| Client | BDCE (bit-tracking DCE), others |

```bash
opt -passes='print<demanded-bits>' -disable-output f.ll
```

### Not the same as…

1. **`SimplifyDemandedBits` in ValueTracking / InstCombine** — local, opportunistic simplification during InstCombine, not the whole-function analysis pass.
2. **`computeKnownBits`** — **forward** analysis of which bits are known 0/1; orthogonal. Known bits + demanded bits together are especially powerful.

## Core concepts

### Meet over users

If one user demands only bit 0 and another demands bits 0–7, the def must provide bits 0–7.

### Fixed-point

Because of loops/phis, DemandedBits iterates to a fixed point (bits only add demands—monotonic).

### Alive bits for BDCE

BDCE replaces instructions with equivalents that only produce demanded bits (e.g., mask constants) or deletes ops whose results are entirely undemanded.

## The algorithm

1. Seed: instructions with observable effects demand all bits of their inputs as required.
2. Worklist propagate to defining instructions.
3. Apply opcode transfer to convert output demand → input demand.
4. Union with existing demand; revisit if changed.
5. Query API: `getDemandedBits(Instruction *)`.

## Where to read the code

1. DemandedBits.cpp — transfer switch on opcodes.
2. BDCE pass — how masks are consumed.
3. ValueTracking `SimplifyDemandedBits` — contrast local vs analysis.
4. KnownBits in ValueTracking — forward cousin.

## Worked example

```llvm
define i8 @f(i32 %x) {
  %a = mul i32 %x, 3
  %b = and i32 %a, 255
  %c = trunc i32 %b to i8
  ret i8 %c
}
```

Demanded bits of `%a` collapse toward the low 8 bits (with care for mul’s bit influence). BDCE/InstCombine may simplify the mul/and/trunc chain.

## Reading order

1. Backward vs KnownBits forward.
2. Transfer for `and`/`shl`/`trunc`.
3. DeepDive + Examples.

## Hands-on

```bash
cat > db.ll <<'EOF'
define i8 @f(i32 %x) {
  %a = or i32 %x, 0
  %b = and i32 %a, 255
  %c = trunc i32 %b to i8
  ret i8 %c
}
EOF
opt -passes='print<demanded-bits>' -disable-output db.ll
opt -passes='bdce' -S db.ll
```

## Pitfalls

- Calling InstCombine’s SimplifyDemandedBits “the DemandedBits analysis.”
- Forgetting add/mul can demand more bits than the user mask (carry/partial products)—transfers are carefully written for a reason.
- Expecting FP demanded bits—focus is integers.

## Neighbors

- BDCE, InstCombine, KnownBits, AggressiveInstCombine.

## Check yourself

1. **Q:** Direction of DemandedBits?  
   **A:** Backward from uses to defs.

2. **Q:** Print pass?  
   **A:** `print<demanded-bits>`.

3. **Q:** How does `trunc i32→i8` demand the input?  
   **A:** Only the low 8 bits.

4. **Q:** Name the forward cousin.  
   **A:** `computeKnownBits`.

5. **Q:** Local InstCombine cousin?  
   **A:** `SimplifyDemandedBits` in ValueTracking.

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

## Extended hands-on checklist (Day 38)

1. Build IR with the disable-llvm-passes recipe.
2. Run the minimal pass pipeline named in this day’s main note.
3. Diff against a pipeline that adds one neighbor pass.
4. Re-run with the day’s `-debug-only` flag.
5. Skim one upstream lit test RUN line and reproduce it.
6. Write one counterexample IR that must not transform.
7. Cross-check a fact against the linked github source file.

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

## Extended hands-on checklist (Day 38)

1. Build IR with the disable-llvm-passes recipe.
2. Run the minimal pass pipeline named in this day’s main note.
3. Diff against a pipeline that adds one neighbor pass.
4. Re-run with the day’s `-debug-only` flag.
5. Skim one upstream lit test RUN line and reproduce it.
6. Write one counterexample IR that must not transform.
7. Cross-check a fact against the linked github source file.

## Demanded bits on phis

Phi demand is the union of demands on the phi result, pushed to each incoming value. In loops, that fixed point can mark an IV’s high bits undemanded if only a truncating use escapes—enabling BDCE to rewrite the IV update. Print demanded-bits on a looping trunc use to see the lattice settle.

## IR hygiene for experiments

Always generate IR with `clang -S -emit-llvm -O1 -Xclang -disable-llvm-passes` so functions are not marked `optnone`. Opaque pointers (`ptr`) are mandatory in modern IR examples; do not revive typed pointer syntax in new notes.

When comparing before/after, prefer `diff -u` on files produced by the same opt pipeline with one pass removed. That isolates the transform better than comparing against clang -O3, which runs dozens of passes.
