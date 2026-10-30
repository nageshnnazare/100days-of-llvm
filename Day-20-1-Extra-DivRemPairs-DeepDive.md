# Day 20 Extra: DivRemPairs — Deep Dive

Companion to [Day-20-0-DivRemPairs.md](Day-20-0-DivRemPairs.md). Examples: [Day-20-2-Extra-DivRemPairs-Examples.md](Day-20-2-Extra-DivRemPairs-Examples.md).

Source: [DivRemPairs.cpp](https://github.com/llvm/llvm-project/blob/main/llvm/lib/Transforms/Scalar/DivRemPairs.cpp).

## Diagram

```text
   block A:  %q = udiv i32 %a, %b
   block B:  %r = urem i32 %a, %b

   TTI.hasDivRemOp is true
              |
              v
   both operations moved into one block

   SelectionDAG then builds one UDIVREM
   node: quotient and remainder from a
   single machine divide
```

DivRemPairs runs when the target can compute quotient and remainder together. It moves a matching `udiv` and `urem` into the same block so instruction selection sees one divide. The query is `hasDivRemOp` on the target transform info. If the target has no such instruction, the pair stays split.

## 1. File orientation

| Symbol | Role |
|--------|------|
| `DEBUG_TYPE "div-rem-pairs"` | `-debug-only=div-rem-pairs` |
| `NumPairs`, `NumHoisted`, `NumDecomposed`, `NumRecomposed` | Statistics |
| `DRPCounter` / `div-rem-pairs-transform` | DebugCounter |
| `matchExpandedRem` | Recognize `X - ((X/Y)*Y)` |
| `getWorklist` | Collect pairs with TrackingVH |
| `optimizeDivRem` | Main transform driver |
| Pass `run` | Fetch TTI + DT; call `optimizeDivRem` |

## 2. Worklist construction

`getWorklist` scans the function for integer div and rem ops (and expanded rem shapes). It maps keys roughly by `(dividend, divisor, signedness)` so `sdiv` meets `srem` on the same operands.

Tracking value handles matter: when one rem is rewritten, another pair’s operand might be that instruction—raw `Instruction*` can dangle across RAUW.

## 3. Expanded rem matching

`matchExpandedRem` looks for the tree:

```
%q   = {s,u}div %X, %Y
%m   = mul %q, %Y
%rem = sub %X, %m
```

Operand identity must line up (same `%X`, same `%Y`, mul/sub in the right places). NSW/NUW flags may appear; the matcher focuses on value identity first—read the helper for flag sensitivity.

If the target **has** a div-rem op, the pass can replace this tree with a real `{s,u}rem` (`NumRecomposed`) so ISel sees a pair of IR opcodes it knows how to fuse.

If the target **lacks** a div-rem op and the rem is already expanded, optimizeDivRem continues (nothing to decompose).

## 4. Branch on `hasDivRemOp`

```text
HasDivRemOp = TTI.hasDivRemOp(Ty, IsSigned)

if HasDivRemOp:
  prefer real rem
  co-locate div and rem in one block when apart
  recompose expanded rem → rem
else:
  prefer expanded rem
  decompose real rem → mul+sub using existing div
```

X86 TTI historically returns true for common integer types; many RISC targets return false. Always confirm the override in the target’s `*TargetTransformInfo.cpp`.

## 5. Hoisting / co-location logic

When both ops exist as real instructions:

1. If already same block and HasDivRemOp → done for that pair.
2. If one dominates the other → `moveAfter` to sit next to the dominator.
3. Additional CFG patterns: shared unique predecessor that always transfers execution to both successors’ blocks, etc. (see the PredBB logic in `optimizeDivRem`). Comments note FIXMEs for more hoist cases.

Safety helpers refuse moves that would execute a div/rem on a path that did not already “pay” for the sibling’s exceptional behavior—pairing is special because the first op already implies much of the cost/poison of the second when operands match.

## 6. Decomposition mechanics

Without HasDivRemOp:

```llvm
%q   = sdiv i32 %x, %y
%r   = srem i32 %x, %y
; becomes conceptually:
%q   = sdiv i32 %x, %y
%m   = mul  i32 %q, %y
%r   = sub  i32 %x, %m
```

Placement: if rem dominated div, hoist div toward rem (or vice versa) so the mul/sub can use `%q` without breaking SSA. Exact move order is in the `#else` branch of the HasDivRemOp split—read carefully.

## 7. Poison and speculation notes

IR semantics:

- `div`/`rem` by zero → poison (and historically UB discussions; treat as must-not-speculate-onto-new-paths carelessly).
- Signed min / -1 → poison for `sdiv`/`srem`.

DivRemPairs asserts that once one of the pair executes on a path, introducing the sibling’s computation on that same path is acceptable in the common matched case. Moving into a **new** block that did not execute either needs the explicit safety predicates in the hoist paths.

## 8. What about `isDivRemCheap`?

Study prompt folklore sometimes mentions `isDivRemCheap`. In current LLVM TTI, the DivRemPairs-relevant boolean is **`hasDivRemOp`**. General “is this div cheap?” questions use `getArithmeticInstrCost` and friends. If your tree has an additional helper, cite that file; do not assume the name without grepping.

SimplifyCFG has separate heuristics about sinking div/rem when replacing constant denominators with variables—that is unrelated to this pass.

## 9. DebugCounter bisection

```bash
opt -passes=div-rem-pairs -mtriple=x86_64-unknown-linux-gnu \
  -debug-counter=div-rem-pairs-transform-skip=1,div-rem-pairs-transform-count=1 \
  -S in.ll
```

Useful when multiple pairs exist and only one miscompiles.

## 10. Analyses

- **TargetTransformInfo** — policy.
- **DominatorTree** — hoist legality and move direction.

No LoopInfo requirement in the core signature of `optimizeDivRem` (confirm includes if used indirectly).

## 11. Not handled / limitations

- Floating-point `fdiv`/`frem` (integer-focused pass).
- Div and rem with mismatched widths without prior canonicalization.
- Inventing a div when only rem exists and HasDivRemOp is false—you still need a div to expand rem optimally; a lone rem may remain for the backend.
- Full global reordering beyond implemented CFG patterns (see FIXME comments).
- Replacing InstCombine’s strength reduction for `n % 8` → `n & 7`.

## 12. ISel connection (why same block matters)

SelectionDAG patterns and X86 isel match div/rem pairs more reliably when both IR ops are in the same basic block without intervening control. Hoisting is explicitly “make the matched pair visible to the backend.”

After DRP, `llc -O2` on X86 should show a single `idiv`/`div` feeding both results for simple functions.

## 13. Reading checklist

1. Trace one same-block signed pair on X86 (no change beyond maybe ordering).
2. Trace rem in a dominated child block → hoist.
3. Trace the same IR with a triple whose TTI returns false → decompose.
4. Trace expanded mul/sub → recompose on X86.
5. Confirm non-matching signedness ignored.

## 14. Takeaway

DivRemPairs is a **TTI-directed div/rem normalizer**: fuse visibility for targets with joint ops; expand rem for targets without. Everything else is domination plumbing and poison-aware motion.

## 15. Worked TTI lookup

In your checkout:

```bash
rg -n "hasDivRemOp" llvm/lib/Target -g '*TargetTransformInfo*'
```

Read the X86 override and one RISC override side by side. That ten-minute exercise explains more experimental `opt -mtriple` differences than any prose summary.

## 16. Statistics to watch

After a transform run with stats enabled (`-stats` on an asserts build):

- `NumPairs` — how many matches were considered
- `NumHoisted` — co-location moves
- `NumDecomposed` — rem → mul+sub
- `NumRecomposed` — mul+sub → rem

If `NumPairs` is zero, your IR does not contain matching operands/signedness—fix the pass, not TTI.

Keep a personal note of which triples you verified; TTI defaults change more often than the identity algebra.
