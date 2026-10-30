# Day 20 Extra: DivRemPairs — Examples

Overview: [Day-20-0-DivRemPairs.md](Day-20-0-DivRemPairs.md). Deep dive: [Day-20-1-Extra-DivRemPairs-DeepDive.md](Day-20-1-Extra-DivRemPairs-DeepDive.md).

```bash
clang -S -emit-llvm -O1 -Xclang -disable-llvm-passes file.c -o file.ll
opt -passes=div-rem-pairs -mtriple=x86_64-unknown-linux-gnu -S file.ll -o file.x86.ll
opt -passes=div-rem-pairs -debug-only=div-rem-pairs -mtriple=x86_64-unknown-linux-gnu -S file.ll
```

Pick a second triple whose TTI returns false for `hasDivRemOp` when testing decomposition (confirm for your LLVM version).

---

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

## Example 1: Same-block signed pair on X86

### Goal

See a co-located `sdiv`/`srem` left for ISel (or lightly reordered).

### Input

```llvm
define i32 @pair(i32 %x, i32 %y) {
entry:
  %q = sdiv i32 %x, %y
  %r = srem i32 %x, %y
  %s = add i32 %q, %r
  ret i32 %s
}
```

### Commands

```bash
opt -passes=div-rem-pairs -mtriple=x86_64-unknown-linux-gnu -S pair.ll -o pair.out.ll
llc -O2 -mtriple=x86_64-unknown-linux-gnu pair.out.ll -o pair.s
```

### Expected

IR still has both `sdiv` and `srem` (same block). Assembly shows a single `idiv` (or equivalent) producing both quotient and remainder uses.

### Why it fires / applies

`hasDivRemOp` true; already same block → nothing to hoist; backend fuses.

### Counterexample

```llvm
%q = sdiv i32 %x, %y
%r = urem i32 %x, %y   ; unsigned rem — not a signed pair
```

No match; no pairing transform.

---

## Example 2: Decomposition when no joint op

### Goal

Expand `urem` using an existing `udiv`.

### Input

```llvm
define i32 @decompose(i32 %x, i32 %y) {
entry:
  %q = udiv i32 %x, %y
  %r = urem i32 %x, %y
  %s = add i32 %q, %r
  ret i32 %s
}
```

### Commands

```bash
# Choose a triple where hasDivRemOp is false for i32 — verify in that target's TTI.
opt -passes=div-rem-pairs -mtriple=<no-divrem-triple> -S decompose.ll -o decompose.out.ll
```

### Expected

```llvm
%q = udiv i32 %x, %y
%m = mul i32 %q, %y
%r = sub i32 %x, %m
```

(Names may differ.)

### Why it fires

Without a unified op, rem expands to reuse `%q`.

### Counterexample

Same IR with `-mtriple=x86_64-unknown-linux-gnu` — keep real `urem` next to `udiv` instead of expanding.

---

## Example 3: Hoist rem to div’s block

### Goal

Co-locate ops across a dominated successor when `hasDivRemOp`.

### Input

```llvm
define i32 @split(i32 %x, i32 %y, i1 %c) {
entry:
  %q = sdiv i32 %x, %y
  br i1 %c, label %use.rem, label %only.q
use.rem:
  %r = srem i32 %x, %y
  %s = add i32 %q, %r
  ret i32 %s
only.q:
  ret i32 %q
}
```

### Commands

```bash
opt -passes=div-rem-pairs -mtriple=x86_64-unknown-linux-gnu -S split.ll -o split.out.ll
```

### Expected

When legal, `%r` moves next to `%q` in `entry` (or `%q` moves toward `%r`—domination dictates direction). On the `only.q` path, watch poison/safety: if rem is hoisted into `entry`, rem executes even when `%c` is false—**only legal if the pass’s safety predicates allow**. If your output keeps rem in `use.rem`, the hoist was rejected for safety; try a CFG where both ops always execute (both in blocks that post-dominate each other or share a common always-executed region).

Safer variant for a guaranteed hoist demo:

```llvm
define i32 @split_always(i32 %x, i32 %y) {
entry:
  %q = sdiv i32 %x, %y
  br label %next
next:
  %r = srem i32 %x, %y
  %s = add i32 %q, %r
  ret i32 %s
}
```

Here rem’s block is a fallthrough; co-location should succeed on X86.

### Why it fires

Matched signed pair; DT allows move; HasDivRemOp true.

### Counterexample

Different divisors (`srem i32 %x, %z`) → no pair.

---

## Example 4: Recompose expanded rem on X86

### Goal

Turn mul/sub tree back into `srem` when the target has div-rem.

### Input

```llvm
define i32 @expanded(i32 %x, i32 %y) {
entry:
  %q = sdiv i32 %x, %y
  %m = mul i32 %q, %y
  %r = sub i32 %x, %m
  %s = add i32 %q, %r
  ret i32 %s
}
```

### Commands

```bash
opt -passes=div-rem-pairs -mtriple=x86_64-unknown-linux-gnu -S expanded.ll -o expanded.out.ll
```

### Expected

`%m`/`%r` recomposed into `srem i32 %x, %y` sitting with `%q`.

### Why it fires

`matchExpandedRem` succeeds; `hasDivRemOp` true → `NumRecomposed`.

### Counterexample

Break the pattern (`sub i32 %m, %x` flipped, or mul by a different value). Matcher fails; tree stays.

---

## Example 5: Lone rem without matching div

### Goal

Show the pass does not invent work without a pair.

### Input

```llvm
define i32 @only_rem(i32 %x, i32 %y) {
entry:
  %r = srem i32 %x, %y
  ret i32 %r
}
```

### Commands

```bash
opt -passes=div-rem-pairs -mtriple=x86_64-unknown-linux-gnu -S only_rem.ll -o only_rem.out.ll
opt -passes=div-rem-pairs -mtriple=<no-divrem-triple> -S only_rem.ll -o only_rem2.out.ll
```

### Expected

No change (or only trivial). Without a sibling div, decomposition-to-reuse cannot run; recomposition does not apply.

### Why

Worklist needs a pair (or expanded form tied to a div).

### Counterexample

Add `%q = sdiv i32 %x, %y` used elsewhere → pair appears and transforms apply.

---

## Verification

```bash
opt -passes=verify -disable-output file.out.ll
grep -E 's(div|rem)|u(div|rem)|mul|sub' file.out.ll
```
