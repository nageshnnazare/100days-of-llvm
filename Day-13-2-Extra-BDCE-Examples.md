# Day 13 Extra: BDCE Examples

```bash
clang -S -emit-llvm -O1 -Xclang -disable-llvm-passes FILE.c -o FILE.ll
opt -passes=bdce -S FILE.ll -o FILE.bdce.ll
opt -passes=bdce -debug-only=bdce -S FILE.ll
```

Opaque pointers where pointers appear; BDCE itself targets integers.

---

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

## Example 1: Instruction with zero demanded bits deleted

### Goal
Delete a multiply that only feeds a fully cleared mask result that is itself unused… clearer: delete a value whose only use demands no bits and leaves the producer trivially dead.

### Input (IR)

```llvm
define i32 @kill_mul(i32 %x) {
entry:
  %m = mul i32 %x, %x
  %z = and i32 %m, 0
  ret i32 %z
}
```

### Commands

```bash
opt -passes=bdce -S kill_mul.ll -o kill_mul.bdce.ll
```

### Expected
`%m` becomes dead after `%z` trivialization / demand empty; BDCE removes the mul (and may simplify the `and`). Final return is `0`.

### Why it fires
Demanded bits of `%m` are empty once the `and` with 0 is understood; `wouldInstructionBeTriviallyDead` allows erase.

### Counterexample
`ret i32 %m` instead of masking → all bits demanded → mul stays.

---

## Example 2: Dead use of a still-live instruction

### Goal
Zero one dead use while another use keeps the producer live.

### Input (IR)

```llvm
declare void @use(i32)

define void @dead_use(i32 %x) {
entry:
  %m = mul i32 %x, 7
  %masked = and i32 %m, 0
  call void @use(i32 %m)
  call void @use(i32 %masked)
  ret void
}
```

### Commands

```bash
opt -passes=bdce -debug-only=bdce -S dead_use.ll
```

### Expected
The operand use of `%m` inside `%masked = and i32 %m, 0` may be reported dead and replaced with `0`; `%m` itself stays for `@use`. Log: `BDCE: Trivializing`.

### Why it fires
`isUseDead` on that operand use; per-use demand is empty even though instruction demand is not.

### Counterexample
Both uses observe `%m` without a killing mask → no trivializing of uses to zero.

---

## Example 3: `or` with mask disjoint from demanded bits

### Goal
Simplify `or` when the constant mask bits are outside the demanded set.

### Input (IR)

```llvm
define i8 @or_high(i8 %x) {
entry:
  %w = zext i8 %x to i32
  %o = or i32 %w, 65280          ; set bits 8..15
  %t = trunc i32 %o to i8        ; only low 8 demanded
  ret i8 %t
}
```

### Commands

```bash
opt -passes=bdce -S or_high.ll -o or_high.bdce.ll
opt -passes='bdce,instcombine' -S or_high.ll -o or_high.both.ll
```

### Expected
For the `or`, demanded bits (through trunc) are low 8; mask `65280` does not intersect → BDCE can replace `%o` uses with `%w`. InstCombine may finish further cleanup.

### Why it fires
`!Demanded.intersects(Mask)` for `or`.

### Counterexample
`trunc` to `i16` instead of `i8` → demanded bits intersect the mask → `or` stays.

---

## Example 4: `sext` to `zext` when extension bits unused

### Goal
Convert `sext` to `zext` when none of the extended bits are demanded.

### Input (IR)

```llvm
define i16 @sext_low(i8 %x) {
entry:
  %s = sext i8 %x to i32
  %a = and i32 %s, 65535         ; only low 16 bits
  %t = trunc i32 %a to i16
  ret i16 %t
}
```

### Commands

```bash
opt -passes=bdce -S sext_low.ll -o sext_low.bdce.ll
```

### Expected
If demanded bits of `%s` have enough leading zeros relative to the extension width, BDCE rewrites the sext to a zext (statistic `NumSExt2ZExt`).

### Why it fires
`Demanded.countl_zero() >= (DestBitSize - SrcBitSize)`.

### Counterexample
Use arithmetic that demands sign bits (e.g. compare `%s < 0` on the wide value) → sext stays.

---

## Example 5 (bonus): Side-effecting call survives

### Goal
Prove BDCE will not delete a call with side effects.

### Input (IR)

```llvm
declare void @foo()

define i32 @keep_side_effect(i32 %x) {
entry:
  call void @foo()
  %y = mul i32 %x, 0
  ret i32 %y
}
```

### Commands

```bash
opt -passes=bdce -S keep_se.ll -o keep_se.bdce.ll
```

### Expected
`@foo` remains. The `mul` may simplify/delete depending on demand of `%y` (return demands bits—here `%y` is 0; mul may go away). Control flow unchanged.

### Why it fires / stays
`mayHaveSideEffects` protects the call; BDCE is not SimplifyCFG.

### Counterexample
Pure `call i32 @pure(i32 %x)` declared `readnone` returning unused value — other DCE passes may delete; BDCE’s integer demand path is separate from “unused call returns.”

---

## Hands-on checklist

1. Zero-demand mul deleted.
2. Dead use trivialized to 0.
3. Disjoint `or` mask simplified.
4. `sext`→`zext`.
5. Confirm branches/blocks untouched by `bdce` alone.
