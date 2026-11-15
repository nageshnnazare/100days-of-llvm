# Day 38 Extra: Demanded Bits — Examples

```bash
opt -passes='print<demanded-bits>' -disable-output f.ll
opt -passes='bdce' -S f.ll
```

Parent: [Day-38-0-DemandedBits.md](Day-38-0-DemandedBits.md).

---

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

## Example 1: Trunc demands low bits only

### Input

```llvm
define i8 @trunc(i32 %x) {
  %t = trunc i32 %x to i8
  ret i8 %t
}
```

### Commands

`print<demanded-bits>`.

### Expected result

`%x` demanded mask is `0xff` (low 8 bits), not all 32.

### Why it fires

Trunc transfer keeps low bits only.

### Counterexample

```llvm
define i32 @ret_all(i32 %x) {
  ret i32 %x
}
```

All 32 bits demanded.

---

## Example 2: `and` with mask

### Input

```llvm
define i32 @mask(i32 %x) {
  %a = and i32 %x, 255
  ret i32 %a
}
```

### Commands

Printer + `bdce` / `instcombine`.

### Expected result

Only low 8 bits of `%x` demanded. High bits of `%x` undemanded.

### Why it fires

`and` with constant intersects the demand mask.

### Counterexample

```llvm
define i32 @mask_or(i32 %x) {
  %a = or i32 %x, 255
  ret i32 %a
}
```

Low 8 bits of result are always 1; demand on `%x` for those bits may drop, but **higher** bits of `%x` remain demanded for the result’s high bits—different pattern than `and`.

---

## Example 3: Shift moves demand

### Input

```llvm
define i32 @shl(i32 %x) {
  %s = shl i32 %x, 8
  %a = and i32 %s, 65280   ; 0xFF00
  ret i32 %a
}
```

### Commands

`print<demanded-bits>`.

### Expected result

Demand on `%x` concentrates on its low 8 bits (those shifted into `0xFF00`), not on bits that shift into undemanded positions.

### Why it fires

`shl` transfers demand toward lower input bits by the shift amount; `and` kills other result bits.

### Counterexample

Variable shift amount → transfer cannot be a simple constant-mask reshape; demands often become conservative (all bits).

---

## Example 4: InstCombine local vs analysis

### Input

```llvm
define i32 @local(i32 %x) {
  %s = lshr i32 %x, 24
  ret i32 %s
}
```

### Commands

```bash
opt -passes='instcombine' -S local.ll
opt -passes='print<demanded-bits>' -disable-output local.ll
```

### Expected result

InstCombine/`SimplifyDemandedBits` may rewrite based on only 8 bits surviving the lshr. The analysis printer shows demand on `%x`’s high byte. Both agree in spirit; mechanisms differ (rewrite now vs map for BDCE).

### Why it fires

Only high bits of `%x` affect the result.

### Counterexample

```llvm
define i32 @uses_both(i32 %x) {
  %hi = lshr i32 %x, 24
  %lo = and i32 %x, 255
  %s = add i32 %hi, %lo
  ret i32 %s
}
```

Nearly all bits of `%x` demanded via some path—cannot drop `%x`’s middle bits carelessly for both uses’ union.

---

## Example 5 (bonus): KnownBits contrast (forward)

### Input

```llvm
define i32 @known(i8 %x) {
  %z = zext i8 %x to i32
  ret i32 %z
}
```

### Commands

Mentally: KnownBits says bits 8–31 of `%z` are zero. DemandedBits (from ret) demands all bits of `%z`, but high bits are known zero—not “undemanded.” Different lattice.

### Expected result

DemandedBits still demands the high bits of the **value** `%z` for the ret (they are part of the returned i32); KnownBits proves they are zero. BDCE won’t delete the zext because the result is used; InstCombine may still simplify users that only demand low bits.

### Why the distinction matters

Students often conflate “known zero” with “undemanded.” They are not the same.

### Twin demanded-bits case

If the only user is `trunc i32 %z to i8`, high bits of `%z` become undemanded—and KnownBits also says zero—double win for simplification.

---

## Table

| Pattern | Demanded on input |
|---------|-------------------|
| trunc → i8 | low 8 |
| and 255 | low 8 |
| shl+mask | shifted region |
| lshr 24 | high byte |
| ret full i32 | all bits |
| known zero high | still demanded if returned |
