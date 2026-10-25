# Day 9 Extra: Reassociate Examples

Run:

```bash
opt -passes=reassociate -S ex.ll -o step1.ll
opt -passes=reassociate,instcombine -S ex.ll -o step2.ll
```

Always look at `step1.ll`. The pass's own effect is the shape change. `step2.ll` is the payoff.

---

## Diagram

```text
   before                         after reassociate

   (%a + 1) + %b                  (%a + %b) + 1
        ^                              ^
   rank(const) is lowest          constants are pulled
   so they sink to the right      to the right end

   (%x * 2) * 3                   %x * 6
```

Reassociate ranks operands (constants lowest, arguments next, instructions by their order) and rewrites a chain into one canonical order. Two expressions that were algebraically the same but parenthesized differently then hash to the same thing for CSE. Integer reassociation drops `nsw`: the rewritten form must not invent overflow poison.

## Example 1: Constants that were not adjacent

```llvm
define i32 @c(i32 %x) {
  %a = add i32 %x, 1
  %b = add i32 %a, 2
  ret i32 %b
}
```

**After reassociate + instcombine:** `add i32 %x, 3`.

**Why:** linearize to `{x:1, const 1, const 2}`, sum the constants, rebuild.

**Counterexample:**

```llvm
define i32 @nsw(i32 %x) {
  %a = add nsw i32 %x, 1
  %b = add nsw i32 %a, 2
  ret i32 %b
}
```

The fold to `+ 3` may still happen, but the result should not carry `nsw` unless the pass proved it. `x = INT_MAX - 1` makes `%a` poison under `nsw` already; the interesting case is a chain where the original wraps only at the second add. Dropping `nsw` is the safe output. Check the textual IR for the flag instead of assuming it survived.

---

## Example 2: The same expression, two spellings

```llvm
define i32 @same(i32 %x, i32 %y) {
  %a1 = add i32 %x, 1
  %a  = add i32 %a1, %y
  %b1 = add i32 %y, %x
  %b  = add i32 %b1, 1
  %d  = sub i32 %a, %b
  ret i32 %d
}
```

**After the pair of passes:** `ret i32 0`.

**After reassociate only:** `%a` and `%b` should be the same chain. They might still be two instructions until EarlyCSE or InstCombine notices. If `step2.ll` is already `ret i32 0`, InstCombine's own CSE did it.

**Counterexample:** change the second constant to `2`. The result is `ret i32 -1` or `sub i32 1, 2`, not zero. Reassociate did not get "confused"; the expressions are no longer equal. If you still see a long subtract chain, run `instcombine`.

---

## Example 3: XOR cancellation

```llvm
define i32 @xor(i32 %x, i32 %y) {
  %a = xor i32 %x, %y
  %b = xor i32 %a, %x
  ret i32 %b
}
```

**Expected:** `ret i32 %y` after `reassociate` or after `reassociate,instcombine`.

**Why:** weight of `%x` mod 2 is 0.

**Counterexample:**

```llvm
define i32 @or(i32 %x, i32 %y) {
  %a = or i32 %x, %y
  %b = or i32 %a, %x
  ret i32 %b
}
```

The result is `or i32 %x, %y`, not `%y`. OR is idempotent, not cancelling. If the output is `ret i32 %y`, that is a miscompile.

---

## Example 4: Subtraction as a negative weight

```llvm
define i32 @sub(i32 %x, i32 %y) {
  %a = add i32 %x, %y
  %b = sub i32 %a, %x
  ret i32 %b
}
```

**Expected:** `ret i32 %y`.

**Why:** `%x` appears with weights `+1` and `-1`.

**Counterexample:**

```llvm
define i32 @notsub(i32 %x, i32 %y) {
  %a = sub i32 %x, %y
  %b = sub i32 %y, %x
  %c = add i32 %a, %b
  ret i32 %c
}
```

This *should* become 0, because it is `(x - y) + (y - x)`. It may require both reassociate and instcombine. If it does not fold, look at whether the `sub`s were converted into adds of negations. They should be. A remaining `sub` pair is a missed canonicalization worth understanding, not a reason to hand-rewrite it in your head and call it done: the exercise is to see which pass fires.

---

## Example 5: Floating point stays put

```llvm
define float @fp(float %x, float %y) {
  %a = fadd float %x, 1.0
  %b = fadd float %a, %y
  ret float %b
}
```

**Expected after `reassociate`:** unchanged. No `reassoc` flag.

```llvm
define float @fpfast(float %x, float %y) {
  %a = fadd reassoc float %x, 1.0
  %b = fadd reassoc float %a, %y
  ret float %b
}
```

**Expected:** the operands may be reordered so the constant is grouped, or the two `fadd`s swapped in rank order. The `reassoc` flag remains.

**Counterexample:** add `nsz` or `fast` and confirm the pass still refuses the version that lacks `reassoc`. `fast` implies `reassoc`. A bare `nsz` does not. Try `fadd nsz` without `reassoc`; it must not move.

---

## Example 6: Multi-use stops the walk

```llvm
define i32 @multi(i32 %x, i32 %y, i32 %z) {
  %a = add i32 %x, %y
  %b = add i32 %a, %z
  %c = add i32 %a, 1
  %d = add i32 %b, %c
  ret i32 %d
}
```

`%a` has two users. Reassociate may canonicalize `%b` and `%c` locally and still not flatten everything into one chain of `%x, %y, %z, 1`, because pulling `%a` apart would duplicate `%x + %y` or rewrite both users together.

**What to check:** count instructions before and after. Some folding of the `+ 1` is fine. A full flatten into a single rank-sorted chain of all four leaves is not required, and on this input is limited by the use count.

**Counterexample that does fully flatten:** give `%a` one use by deleting `%c`'s dependence, i.e. change the body so `%a` feeds only `%b` and `%c = add i32 %z, 1` is independent. Then the `%x + %y + %z` chain has single-use links and flattens.

---

## Reading the dump

In `step1.ll`, circle every `add` chain and write the leaves left to right. Two chains that you believe are algebraically equal should list the same leaves in the same order. If they do not, either rank is doing something you have not accounted for (an argument versus an instruction) or the use limit split the chain. Those are the only two reasons.
