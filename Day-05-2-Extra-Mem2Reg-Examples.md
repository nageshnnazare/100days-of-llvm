# Day 5 Extra: Mem2Reg Examples

Commands used throughout:

```bash
clang -S -emit-llvm -O1 -Xclang -disable-llvm-passes ex.c -o ex.ll
opt -passes=mem2reg -S ex.ll -o ex.m2r.ll
```

`mem2reg` leaves the CFG alone. If your diff contains new block labels, you ran a different pass.

---

## Diagram

```text
   entry:  %y = alloca i32
           /            \
        then            else
     store 1, %y     store 2, %y
           \            /
            join:  %t = load i32, ptr %y
                         |
                         v
   after mem2reg:

   entry
     /        \
   then       else
     \        /
      join:  %y = phi i32 [ 1, %then ], [ 2, %else ]
```

The alloca is a stack slot only because the frontend did not want to place phis. Mem2Reg puts a phi at the join, which is the iterated dominance frontier of the stores, then deletes the alloca. A load in a block dominated by exactly one store becomes that stored value and needs no phi.

## Example 1: Diamond

```c
int pick(int c, int x, int y) {
  int r;
  if (c)
    r = x;
  else
    r = y;
  return r + 1;
}
```

**Expected:** one phi in the join, no alloca, the add uses the phi.

```llvm
join:
  %r = phi i32 [ %x, %then ], [ %y, %else ]
  %sum = add i32 %r, 1
  ret i32 %sum
```

Block names will differ. The structure will not.

**Why:** two defining blocks, frontier is the join, both predecessors define the variable so the phi has two real incoming values.

**Counterexample:**

```c
int capture(int c) {
  int r = 0;
  if (c)
    r = 1;
  sink(&r);
  return r;
}
```

with `void sink(int *);`. The address escapes. `ex.m2r.ll` still contains `alloca` and `call void @sink`. Promotion would be a miscompile if `sink` stored `7` into `r`.

---

## Example 2: Loop-carried variables

```c
int sum(int n) {
  int s = 0;
  for (int i = 0; i < n; ++i)
    s += i;
  return s;
}
```

**Expected:** in the header,

```llvm
%i = phi i32 [ 0, %entry ], [ %i.next, %latch ]
%s = phi i32 [ 0, %entry ], [ %s.next, %latch ]
```

and in the latch an add that feeds `%s.next`. There is no `alloca`, no `load`, and no `store` of `s` or `i`.

**Why:** each variable is defined in the latch (the update) and in the entry (the initial value). The header is on the iterated dominance frontier because of the backedge.

**Counterexample:** a loop that passes `&s` to a function each iteration does not promote `s`. `i` still promotes if its address is never taken. Try it: promotion is per alloca, not per function.

---

## Example 3: Single store, many branches

```c
int once(int x, int c) {
  int y = x * 3;
  if (c)
    return y;
  return y + 1;
}
```

**Expected:** no phi. Both returns use `%y`'s multiply directly. The single-store fast path applies because the store in the entry dominates both loads.

**Counterexample:**

```c
int maybe(int x, int c) {
  int y;
  if (c)
    y = x;
  return y;
}
```

On the path where `c` is false, `y` is uninitialized. Mem2Reg still removes the alloca, and the phi (or the uninit incoming value) becomes `undef`. Do not "fix" this by expecting a zero. If you need a defined value, the C source has to store one.

---

## Example 4: Aggregate, the pass that should refuse

```c
typedef struct { int x; int y; } P;
int sumxy(int a, int b) {
  P p;
  p.x = a;
  p.y = b;
  return p.x + p.y;
}
```

**Expected after `mem2reg` only:** the alloca of `%struct.P` remains, with GEPs, stores, and loads. `isAllocaPromotable` rejects GEPs.

```bash
opt -passes=mem2reg -S sumxy.ll | grep alloca
opt -passes=sroa -S sumxy.ll
```

The second command should scalarize it (Day 6) and you will see two independent values, often with no phi at all.

**Counterexample that *does* promote as a blob:** a memcpy of the whole struct into another whole-struct local can be promotable when the only operations are loads and stores of the entire aggregate type. That is rare in Clang's output, which prefers GEPs. Do not write a test that depends on whole-struct promotion unless you have looked at the IR.

---

## Example 5: A phi of phis (nested branches)

```c
int nest(int a, int b, int c) {
  int y = 0;
  if (a) {
    if (b)
      y = 1;
    else
      y = c;
  }
  return y;
}
```

**Expected:** two phis.

- Inner join merges `1` and `c`.
- Outer join merges that result with `0`.

**Why:** the inner phi is a new definition. The iterated frontier includes both joins. See Day 4 Example 4.

**Counterexample:** change the function so both inner arms `return` directly and the outer `else` returns 0. The CFG has no joins. Mem2Reg has nothing to place. Each return uses a constant or `c` directly.

---

## Example 6: Volatile stays in memory

```llvm
define i32 @vol(i1 %c) {
entry:
  %a = alloca i32, align 4
  store volatile i32 1, ptr %a, align 4
  br i1 %c, label %t, label %f
t:
  store volatile i32 2, ptr %a, align 4
  br label %j
f:
  br label %j
j:
  %v = load volatile i32, ptr %a, align 4
  ret i32 %v
}
```

```bash
opt -passes=mem2reg -S vol.ll
```

**Expected:** identical memory operations. Volatile loads and stores are observable side effects. Promoting them into a phi would delete those side effects.

**Counterexample / contrast:** delete the three `volatile` keywords and rerun. You get a phi of `2` and `1` and no alloca. The CFG did not change between the two runs; only the side-effect flags did.

---

## What to compare

Put `run.ll` and `run.m2r.ll` in a diff tool and count:

- `alloca` before and after,
- `load` / `store` before and after,
- `phi` after.

Then run `opt -passes=mem2reg,instcombine,simplifycfg -S` and see how many of those phis the later passes immediately fold. A lot of "mem2reg phis" in real programs are temporary scaffolding. That is fine. InstCombine is supposed to clean them.
