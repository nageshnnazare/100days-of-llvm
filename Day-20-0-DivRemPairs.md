# Day 20: DivRemPairs

Division and remainder of the same operands often share one hardware operation (x86 `DIV`/`IDIV` produces quotient and remainder together). The `div-rem-pairs` pass prepares IR so the backend can see that pair—or, when the target has **no** joint op, expands `rem` into `sub`/`mul` using the sibling `div`. Code: [DivRemPairs.cpp](https://github.com/llvm/llvm-project/blob/main/llvm/lib/Transforms/Scalar/DivRemPairs.cpp). Deep dive: [Day-20-1-Extra-DivRemPairs-DeepDive.md](Day-20-1-Extra-DivRemPairs-DeepDive.md). Examples: [Day-20-2-Extra-DivRemPairs-Examples.md](Day-20-2-Extra-DivRemPairs-Examples.md).

## 1. Why this exists

Frontends and mid-end opts freely emit `sdiv`/`srem` or `udiv`/`urem` in different blocks. On X86, two separate divisions are painfully expensive; one `DIV` that feeds both quotient and remainder is the natural lowering. Conversely, on targets without a fused div/rem, keeping a real `rem` instruction may be worse than `X - (X/Y)*Y` once a `div` already exists.

DivRemPairs is the target-aware cleanup that **matches** div/rem pairs (same operands, same signedness) and either:

1. **Decomposes** rem into mul+sub using the div, or  
2. **Hoists/sinks** so div and rem sit in the same block (or otherwise become visible together) for ISel when `TTI.hasDivRemOp` is true.

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

## 2. The idea

Identity (when defined):

```
X % Y  ==  X - (X / Y) * Y
```

If the target reports a unified div-rem operation (`TargetTransformInfo::hasDivRemOp`), keep real rem instructions and move the pair together so the DAG/GlobalISel matcher can emit one machine op.

If the target does **not** have such an op, expand rem in terms of div (when a matching div exists) so you do not pay a second expensive division at ISel.

Poison / UB corners still apply: division or remainder by zero; `sdiv`/`srem` of `INT_MIN` by `-1`. Pairing does not “fix” those; it preserves the same poison rules while rearranging ops that already execute.

## 3. Where it sits in the -O2/-O3 pipeline

| Item | Value |
|------|--------|
| Pass name | `div-rem-pairs` |
| File | [DivRemPairs.cpp](https://github.com/llvm/llvm-project/blob/main/llvm/lib/Transforms/Scalar/DivRemPairs.cpp) |
| TTI hook | `TargetTransformInfo::hasDivRemOp(Type *, bool IsSigned)` |
| Debug | `-debug-only=div-rem-pairs` |
| Debug counter | `div-rem-pairs-transform` |

Typically a late scalar pass before CodeGen prepare-ish territory (exact PassBuilder slot varies). Always pass a triple when experimenting so TTI answers match the CPU you care about:

```bash
clang -S -emit-llvm -O1 -Xclang -disable-llvm-passes file.c -o file.ll
opt -passes=div-rem-pairs -mtriple=x86_64-unknown-linux-gnu -S file.ll -o file.drp.ll
```

## 4. Core concepts

### Matching pairs

A pair shares dividend, divisor, and signedness (`sdiv` with `srem`, `udiv` with `urem`). The pass builds a worklist of such matches (`getWorklist`), using tracking handles because RAUW can invalidate raw pointers.

### Expanded rem form

`matchExpandedRem` recognizes the already-expanded shape `X - ((X / Y) * Y)` so the pass can **recompose** into a real rem when `hasDivRemOp` is true—or avoid re-expanding.

### `hasDivRemOp`

TTI query: unified opcode available for that type and signedness. X86 generally wants pairing/hoisting. Many targets return false → prefer decomposition.

### Hoist / same-block placement

When `hasDivRemOp` and the ops live in different blocks, the pass tries to move one next to the other when domination and safety allow (including some predecessor-hoist patterns). Same-block pairs with a real rem are left for the backend.

### Decomposition

When no unified op: replace rem with mul of the div’s quotient by Y, then sub from X—placed so the existing div is reused.

### Poison checklist

| Case | Issue |
|------|--------|
| `*/ % 0` | UB / poison |
| `sdiv/srem INT_MIN, -1` | Overflow poison on many targets’ IR semantics |
| Speculating div into a block | May introduce poison on a path that previously avoided the op—pass has safety checks when moving |

## 5. The algorithm

From DivRemPairs.cpp (conceptual):

1. `getWorklist(F)` — find matching div/rem (or expanded rem) pairs.
2. For each entry, consult `TTI.hasDivRemOp`.
3. If has joint op and rem is expanded → recompose to rem instruction when profitable.
4. If has joint op and ops are apart → hoist/sink to co-locate (`NumHoisted`).
5. If no joint op and rem is real → expand to mul+sub (`NumDecomposed`).
6. If no joint op and already expanded → nothing.
7. Respect `DEBUG_COUNTER` for bisection; update statistics (`NumPairs`, `NumRecomposed`, …).

Driver: optimize function with DominatorTree + TTI (`optimizeDivRem`), called from the new-PM pass `run`.

## 6. Where to read the code

1. [DivRemPairs.cpp](https://github.com/llvm/llvm-project/blob/main/llvm/lib/Transforms/Scalar/DivRemPairs.cpp) — all logic.
2. [TargetTransformInfo.h](https://github.com/llvm/llvm-project/blob/main/llvm/include/llvm/Analysis/TargetTransformInfo.h) — `hasDivRemOp` documentation.
3. Target impls, e.g. X86 TTI overrides of `hasDivRemOp`.
4. Tests under `llvm/test/Transforms/DivRemPairs/`.

Note: a TTI API named `isDivRemCheap` does **not** appear in current `TargetTransformInfo` (verify on your tree). Cost of div/rem otherwise flows through general arithmetic cost hooks; this pass keys off `hasDivRemOp` plus local safety/domination.

## 7. Worked before/after

```c
int both(int x, int y) {
  int q = x / y;
  int r = x % y;
  return q + r;
}
```

```bash
clang -S -emit-llvm -O1 -Xclang -disable-llvm-passes both.c -o both.ll
opt -passes=div-rem-pairs -mtriple=x86_64-unknown-linux-gnu -S both.ll -o both.x86.ll
opt -passes=div-rem-pairs -mtriple=riscv64-unknown-elf -S both.ll -o both.rv.ll
```

**X86-ish expectation:** `sdiv` and `srem` remain and sit together for ISel to emit one `IDIV`.

**Target without div-rem op:** `srem` may become `mul` + `sub` using the `sdiv` result.

Split-block variant: rem in a dominated successor → hoisting may pull them into one block when `hasDivRemOp`.

## 8. Reading order for today

1. Identity `X%Y = X-(X/Y)*Y` and poison cases.
2. Read `hasDivRemOp` comments in TTI.
3. Read `getWorklist`, `matchExpandedRem`, `optimizeDivRem` in DivRemPairs.cpp.
4. Run the same IR under two triples.
5. [Day-20-1-Extra-DivRemPairs-DeepDive.md](Day-20-1-Extra-DivRemPairs-DeepDive.md).
6. [Day-20-2-Extra-DivRemPairs-Examples.md](Day-20-2-Extra-DivRemPairs-Examples.md).

## 9. Hands-on commands

```bash
clang -S -emit-llvm -O1 -Xclang -disable-llvm-passes dr.c -o dr.ll
opt -passes=div-rem-pairs -mtriple=x86_64-unknown-linux-gnu -debug-only=div-rem-pairs -S dr.ll 2>dr.log
opt -passes=div-rem-pairs -mtriple=aarch64-linux-gnu -S dr.ll -o dr.a64.ll
```

Compare signed vs unsigned pairs; try rem without a matching div (pass should leave it); try expanded form recomposition on X86.

## 10. Pitfalls and invariants

- **Always set `-mtriple`** in experiments; default TTI may not match your intuition.
- **Matching signedness:** `udiv` does not pair with `srem`.
- **Different operands:** no pair.
- **Hoisting safety:** moving div can introduce division on paths that did not execute it—implementation checks domination / guaranteed execution.
- **Poison:** co-locating does not remove INT_MIN/-1 issues; it only helps instruction selection.
- **Expanded form recognition:** hand-written mul/sub must match what `matchExpandedRem` expects (same `%div` value, etc.).
- **Not InstCombine:** general rem strength reduction for power-of-two remains InstCombine’s job.

### Same operands, different blocks, always executed

A reliable hoist demo (both ops always run):

```llvm
define i32 @fallthrough(i32 %x, i32 %y) {
entry:
  %q = sdiv i32 %x, %y
  br label %bb
bb:
  %r = srem i32 %x, %y
  %t = add i32 %q, %r
  ret i32 %t
}
```

```bash
opt -passes=div-rem-pairs -mtriple=x86_64-unknown-linux-gnu -S fallthrough.ll
```

Expect `%q` and `%r` adjacent after the pass. Then `llc` and confirm a single `idiv`.

### Interaction with BypassSlowDivision

Some targets expand wide divisions into libcalls or narrower sequences in a dedicated bypass pass. DivRemPairs still wants a coherent IR-level pair *before* that expansion when both quotient and remainder are required. Order experiments as `div-rem-pairs` then codegen; if you insert bypass first, pairs may already be gone.

### Flag hygiene on recomposed rem

When recomposing mul/sub into rem, check whether nsw/nuw on the mul/sub survive or are dropped. ISel cares more about opcode pairing than those flags on the rem; mid-end users of exact flags should re-read the recomposed instruction’s attributes after DRP.

## 11. Neighbors

- **InstCombine** — folds rem/div by constants; creates/cleans patterns DRP sees.
- **CodeGenPrepare / ISel** — consume same-block pairs into machine DIV.
- **BypassSlowDivision** — related late expansion for slow width divisions (different pass).
- **AggressiveInstCombine** — more div/rem patterns.
- **TTI** — policy oracle for this pass.

## 12. Check yourself

**Q1.** Pass name and debug type?  
**A1.** `div-rem-pairs` / `div-rem-pairs`.

**Q2.** Primary TTI hook?  
**A2.** `hasDivRemOp(Type *, bool IsSigned)`.

**Q3.** What algebraic rewrite expands rem?  
**A3.** `X % Y → X - (X/Y)*Y`.

**Q4.** Why hoist on X86?  
**A4.** So ISel sees div and rem together and emits one `DIV`/`IDIV`.

**Q5.** Name two poison cases for signed div/rem.  
**A5.** Divisor zero; `INT_MIN / -1` (and corresponding rem).

**Q6.** Does `udiv` pair with `srem`?  
**A6.** No—signedness must match.

Tip: implementation details in [Day-20-1-Extra-DivRemPairs-DeepDive.md](Day-20-1-Extra-DivRemPairs-DeepDive.md); IR drills in [Day-20-2-Extra-DivRemPairs-Examples.md](Day-20-2-Extra-DivRemPairs-Examples.md).

### Quick triple matrix

| Triple (typical) | Expectation for `sdiv`+`srem` |
|------------------|-------------------------------|
| `x86_64-*-*` | Keep rem; co-locate; ISel → one `idiv` |
| Many RISC without joint op | Expand rem to mul+sub |
| Same IR, no `-mtriple` | Ambiguous—do not draw conclusions |

Always record the triple next to any DivRemPairs lit-style reduction you save.

### Rem-only hot path

If a profile shows remainder is hot but quotient is cold/unused, expanding rem into mul+sub of a newly introduced div can still be correct but may not be profitable on targets without joint ops—DRP’s decomposition assumes a matching div already exists for reuse, which is the common “both results used” pattern from languages’ `/` and `%` together.
