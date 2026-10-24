# Day 8 Extra: EarlyCSE Examples

IR that still has duplicates is easier to write by hand than to coax out of Clang, because Clang's own pipeline (and even `-disable-llvm-passes` plus a later `instcombine`) may have folded them. Use the `.ll` files below directly.

```bash
opt -passes=early-cse -S ex.ll
opt -passes=early-cse-memssa -S ex.ll
opt -passes=gvn -S ex.ll
```

---

## Diagram

```text
   entry
     %x = add %a, %b          hash(%a,%b,add) -> %x
     |
     +-- then --+
     |  %y = add %a, %b       same hash -> reuse %x
     |                        delete %y
     +-- else --+
        %z = add %a, %b       NOT the same scope if this
                              block is not dominated by
                              the first add... it is, via
                              entry, so %z folds too

   a store to %p followed by a load of %p, with no
   intervening write, forwards the stored value
```

EarlyCSE keeps a scoped hash table that pushes a scope when it enters a block and pops when it leaves. A value computed in a block is available in blocks it dominates. The MemorySSA mode can forward a load across a call that does not alias; the plain mode is more conservative.

## Example 1: Commutative add

```llvm
define i32 @add(i32 %x, i32 %y) {
  %a = add i32 %x, %y
  %b = add i32 %y, %x
  %c = mul i32 %a, %b
  ret i32 %c
}
```

**Expected after `early-cse`:** one `add`, `mul i32 %a, %a`.

**Why:** `SimpleValue` sorts commutative operands.

**Counterexample:**

```llvm
define i32 @sub(i32 %x, i32 %y) {
  %a = sub i32 %x, %y
  %b = sub i32 %y, %x
  %c = add i32 %a, %b
  ret i32 %c
}
```

`sub` is not commutative. EarlyCSE must keep both subtracts. (InstCombine knows `%a + %b` is zero here. That is a different pass. Confirm EarlyCSE alone does not fold it.)

---

## Example 2: Not dominated, no phi insertion

```llvm
define i32 @partial(i1 %c, i32 %x, i32 %y) {
entry:
  br i1 %c, label %L, label %R
L:
  %a = add i32 %x, %y
  br label %J
R:
  br label %J
J:
  %b = add i32 %x, %y
  ret i32 %b
}
```

**Expected after `early-cse`:** both adds remain. `%a` does not dominate `%b`.

**Expected after `gvn`:** GVN may still leave both, or it may PRE the add into `entry` so both sides reuse it. PRE of *arithmetic* is limited; GVN is aggressive about *loads*. Do not write a test that requires arithmetic PRE. The lesson is the negative one: EarlyCSE does not move `%a` into `entry`.

**Counterexample that EarlyCSE does get:** move the first add into `entry`. Then it dominates the add in `J`, the scoped table still contains it when the walk reaches `J` (because `entry` dominates `J`), and the second add disappears. Try that variant.

---

## Example 3: Store-to-load forward

```llvm
define i32 @fwd(ptr %p, i32 %v) {
  store i32 %v, ptr %p, align 4
  %a = load i32, ptr %p, align 4
  ret i32 %a
}
```

**Expected:** `ret i32 %v` under `early-cse`.

**Counterexample:**

```llvm
define i32 @vol(ptr %p, i32 %v) {
  store volatile i32 %v, ptr %p, align 4
  %a = load volatile i32, ptr %p, align 4
  ret i32 %a
}
```

Both operations remain. Volatile memory is not a CSE candidate.

---

## Example 4: Alias-blind generation counter

```llvm
define i32 @alias(ptr noalias %p, ptr noalias %q, i32 %v) {
  %a = load i32, ptr %p, align 4
  store i32 %v, ptr %q, align 4
  %b = load i32, ptr %p, align 4
  %s = add i32 %a, %b
  ret i32 %s
}
```

**Expected with `early-cse`:** both loads often remain, because the store bumps the memory generation.

**Expected with `early-cse-memssa` or `gvn`:** one load. `noalias` on the arguments is a promise that `%p` and `%q` do not alias. MemorySSA's walker, using BasicAA, steps over the store.

**Counterexample:** remove `noalias` and pass the same pointer as `%p` and `%q` at runtime. The IR without `noalias` must keep both loads, because the store may change `%p`. Run `early-cse-memssa` on the version without `noalias` and confirm both loads survive. If they do not, the pass is wrong; file a bug, do not "fix" your reading.

---

## Example 5: Dead store of the same value

```llvm
define void @ds(ptr %p, i32 %v) {
  store i32 %v, ptr %p, align 4
  store i32 %v, ptr %p, align 4
  ret void
}
```

**Expected:** one store after `early-cse`.

**Why:** the second store does not change memory.

**Counterexample:**

```llvm
define void @ds2(ptr %p, i32 %v, i32 %w) {
  store i32 %v, ptr %p, align 4
  store i32 %w, ptr %p, align 4
  ret void
}
```

The second store is the one that remains, if anything is deleted. EarlyCSE's trivial DSE removes a store that is overwritten with no intervening read. Full DSE (`opt -passes=dse`) handles stores separated by non-aliasing operations. Compare:

```bash
opt -passes=early-cse -S ds2.ll
opt -passes=dse -S ds2.ll
```

On `ds2` both passes should leave a single store of `%w`. Now insert a store to a different pointer between them and compare again. DSE plus AA is the one that should cope.

---

## Example 6: Flags block CSE

```llvm
define i32 @flags(i32 %x, i32 %y) {
  %a = add nsw i32 %x, %y
  %b = add i32 %x, %y
  %c = add i32 %a, %b
  ret i32 %c
}
```

**Expected:** both adds remain under EarlyCSE. `nsw` means "signed overflow is poison". The two adds are not interchangeable: replacing `%b` with `%a` would make the second use poison in cases where the original `%b` was a defined wrap. Replacing `%a` with `%b` would drop `nsw` on the first use and might hide poison from a later transform that was relying on it. The conservative choice is "not the same value".

**Counterexample:** make the flags match (`add nsw` on both). EarlyCSE keeps one `add nsw`.

---

## Pipeline order

On a real C file:

```bash
clang -S -emit-llvm -O1 -Xclang -disable-llvm-passes ex.c -o ex.ll
opt -passes=sroa,early-cse,instcombine,early-cse -S ex.ll -o out.ll
```

The second EarlyCSE is not redundant. InstCombine creates new identical expressions (the same `and` mask applied twice, the same GEP rebuilt) that the first EarlyCSE never saw. When you read `-print-pipeline-passes` output and see `early-cse` more than once, this is why.
