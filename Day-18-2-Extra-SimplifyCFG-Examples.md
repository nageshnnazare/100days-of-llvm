# Day 18 Extra: SimplifyCFG — Examples

Overview: [Day-18-0-SimplifyCFG.md](Day-18-0-SimplifyCFG.md). Deep dive: [Day-18-1-Extra-SimplifyCFG-DeepDive.md](Day-18-1-Extra-SimplifyCFG-DeepDive.md).

```bash
clang -S -emit-llvm -O1 -Xclang -disable-llvm-passes file.c -o file.ll
opt -passes=simplifycfg -S file.ll -o file.out.ll
opt -passes=simplifycfg -debug-only=simplifycfg -S file.ll
```

---

## Diagram

```text
   before                         after simplifycfg

   entry                          entry
     br label %mid                  %x = add %a, 1
   mid:                             ret %x
     %x = add %a, 1
     br label %end               the empty mid block is gone;
   end:                          its instructions were folded
     ret %x                      into the predecessor

   br i1 %c, label %t, label %t   --->   br label %t
   (both edges the same)                 (unconditional)
```

SimplifyCFG folds blocks that have one predecessor and one successor, turns a branch with identical successors into a jump, and folds a branch on a constant. It is also what deletes a block after a threading or DCE pass has made it empty. Loop passes ask it not to destroy the canonical loop shape.

## Example 1: Fold empty block / uncond branch chain

### Goal

Collapse `br label` through an empty block.

### Input

```llvm
define i32 @chain(i32 %x) {
entry:
  br label %mid
mid:
  br label %end
end:
  ret i32 %x
}
```

### Commands

```bash
opt -passes=simplifycfg -S chain.ll -o chain.out.ll
```

### Expected

Single block function: `entry` with `ret i32 %x` (or equivalent merged form). Intermediate labels gone.

### Why it fires

Unconditional branches to single-pred blocks merge; empty blocks disappear.

### Counterexample

Give `mid` a second predecessor. Merge is blocked; you may still simplify other ways but the chain cannot fully collapse into one edge identity.

---

## Example 2: Two-entry PHI becomes select

### Goal

Classic diamond → `select`.

### Input

```llvm
define i32 @diamond(i1 %c, i32 %a, i32 %b) {
entry:
  br i1 %c, label %T, label %F
T:
  br label %J
F:
  br label %J
J:
  %v = phi i32 [ %a, %T ], [ %b, %F ]
  ret i32 %v
}
```

### Commands

```bash
opt -passes=simplifycfg -S diamond.ll -o diamond.out.ll
```

### Expected

```llvm
define i32 @diamond(i1 %c, i32 %a, i32 %b) {
entry:
  %v = select i1 %c, i32 %a, i32 %b
  ret i32 %v
}
```

(Exact names may differ.)

### Why it fires

Empty arms, two-entry PHI, speculation cost under threshold.

### Counterexample

Put a `call void @side()` in `T` before the branch to `J`. Folding into a select would speculate the call → blocked or requires different structure (call stays conditional).

---

## Example 3: Branch to common destination

### Goal

Both arms of a conditional branch go to the same block.

### Input

```llvm
define void @same_dest(i1 %c) {
entry:
  br i1 %c, label %X, label %X
X:
  ret void
}
```

### Commands

```bash
opt -passes=simplifycfg -S same_dest.ll -o same_dest.out.ll
```

### Expected

Unconditional `br label %X` or direct merge to `ret void` in `entry`.

### Why it fires

Condition does not matter; FoldBranchToCommonDest-style logic deletes the useless compare use for control (the `i1` may become dead and later DCE).

### Counterexample

Different destinations with distinct side effects—no common-dest fold.

---

## Example 4: Switch range toward icmp (option-gated)

### Goal

See `ConvertSwitchRangeToICmp` / `-switch-range-to-icmp` influence.

### Input

```llvm
define i32 @range_sw(i32 %x) {
entry:
  switch i32 %x, label %def [
    i32 0, label %on
    i32 1, label %on
    i32 2, label %on
  ]
on:
  ret i32 1
def:
  ret i32 0
}
```

### Commands

```bash
opt -passes=simplifycfg -S range_sw.ll -o range_sw.default.ll
opt -passes=simplifycfg -switch-range-to-icmp -S range_sw.ll -o range_sw.icmp.ll
```

### Expected

Default options may leave a switch (depending on pass defaults for your opt invocation). With range-to-icmp enabled (CLI override or late options), expect something like an unsigned compare on `%x` against an upper bound after a bias, branching to `on` vs `def`.

### Why it fires

Contiguous case set sharing one destination is exactly the range pattern.

### Counterexample

Non-contiguous cases (`0, 1, 100`) — range icmp alone does not encode the set; bit-test or lookup or remaining switch handle it instead.

---

## Example 5: Lookup table needs TTI + option

### Goal

Understand gating for switch-to-lookup.

### Input

```llvm
define i32 @lut(i32 %x) {
entry:
  switch i32 %x, label %def [
    i32 0, label %c0
    i32 1, label %c1
    i32 2, label %c2
    i32 3, label %c3
  ]
c0: ret i32 10
c1: ret i32 20
c2: ret i32 30
c3: ret i32 40
def: ret i32 -1
}
```

### Commands

```bash
opt -passes=simplifycfg -S -mtriple=x86_64-unknown-linux-gnu lut.ll -o lut.out.ll
# Also try enabling lookup via CLI if available, e.g. switch-to-lookup style flags / pass params
```

### Expected

On triples where TTI `shouldBuildLookupTables` is true and options enable lookup, cases become a load from a constant table (plus default check). Without TTI approval or with lookup disabled, switch remains.

### Why it fires

Dense cases mapping to constants + cooperative target + `ConvertSwitchToLookupTable`.

### Counterexample

Huge sparse case list, or each arm with distinct heavy side effects—not a constant table of results.

---

## Example 6: `NeedCanonicalLoop` protecting a latch

### Goal

See why early SimplifyCFG refuses some folds around loops.

### Input

Hand-reduce a loop with a dedicated preheader and latch; attempt a CFG fold that would merge an external edge into the latch (construct from Clang `for` IR after `loop-simplify`).

```bash
clang -S -emit-llvm -O1 -Xclang -disable-llvm-passes loop.c -o loop.ll
opt -passes=loop-simplify -S loop.ll -o loop.ls.ll
opt -passes=simplifycfg -S loop.ls.ll -o loop.cfg.ll
```

### Expected

With default `NeedCanonicalLoop` true, canonical preheader/latch/exit shape remains recognizable (`opt -passes='print<loops>'`).

### Why it (conservatively) fires / refuses

Options block transforms that would destroy LoopSimplify form.

### Counterexample

Force late-style options that set `needCanonicalLoops(false)` (PassBuilder late pipelines or `-keep-loops` override behavior—confirm polarity on your tree). Re-run and observe more aggressive merging; you may need `loop-simplify` again afterward.

---

## Verification

After every example:

```bash
opt -passes=verify -disable-output file.out.ll
```
