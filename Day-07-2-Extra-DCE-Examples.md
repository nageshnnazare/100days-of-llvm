# Day 7 Extra: DCE and ADCE Examples

Produce input IR with:

```bash
clang -S -emit-llvm -O1 -Xclang -disable-llvm-passes ex.c -o ex.ll
opt -passes=mem2reg -S ex.ll -o ex.m2r.ll
```

Then compare `dce` and `adce` on `ex.m2r.ll`.

---

## Diagram

```text
   DCE (no CFG needed)              ADCE (needs the CFG)

   %a = add ...   <--- no uses       entry
   delete %a                          |
                                      br
                                     / \
                                 live   dead block
                                 ret    (no path to a
                                         live instruction)
                                         |
                                         delete the block
                                         once its control
                                         dependence is dead
```

DCE deletes an instruction whose result nobody reads and that has no side effect. It will not delete a store, and it will not delete a branch. ADCE starts from instructions that are obviously live (stores, returns, calls) and marks everything they are control-dependent on. Code that cannot reach a live instruction goes away.

## Example 1: Trivial arithmetic

```llvm
define i32 @f(i32 %x) {
  %dead = mul i32 %x, 500
  %live = add i32 %x, 1
  ret i32 %live
}
```

```bash
opt -passes=dce -S trivial.ll
opt -passes=adce -S trivial.ll
```

**Expected:** both outputs are `add` + `ret`. No multiply.

**Why:** `%dead` has no uses and `mul` has no side effects.

**Counterexample:**

```llvm
define i32 @keep(i32 %x) {
  %live = mul i32 %x, 500
  ret i32 %live
}
```

The multiply has a use. Neither pass deletes it. (InstCombine might rewrite `* 500` into shifts and adds. That is not DCE.)

---

## Example 2: Dead arm

```c
int g(int c, int x) {
  int y = x + 1;
  if (c) {
    int z = y * 3;
    (void)z;
  }
  return y;
}
```

```bash
opt -passes=dce -S ex.m2r.ll -o dce.ll
opt -passes=adce -S ex.m2r.ll -o adce.ll
opt -passes=adce,simplifycfg -S ex.m2r.ll -o adce.scfg.ll
```

**Expected:**

- `dce.ll` has no multiply, and still has a conditional branch.
- `adce.ll` rewrites or drops the dead control. Combined with SimplifyCFG, the function is an `add` and a `ret`.

**Why:** DCE does not mark branches. ADCE proves the branch controls no live instruction.

**Counterexample:**

```c
int obs;
int h(int c, int x) {
  if (c)
    obs = x;
  return x;
}
```

The store to `obs` is live. ADCE must keep the branch and the store. If your IR shows the store deleted, you accidentally ran a later pass, or `obs` was optimized as a dead global by `globaldce` because it had internal linkage and no reader. Give `obs` `extern` linkage in the `.ll` (`@obs = external global i32`) when you want the store to be obviously observable.

---

## Example 3: Unused call

```c
int ext(int);
int caller(int x) {
  ext(x);
  return x;
}
```

**Expected:** the call remains under both `dce` and `adce`.

**Why:** `ext` is an unknown declaration. It might store to a global.

**Counterexample that becomes dead:**

```c
int ext(int) __attribute__((const));
int caller(int x) {
  ext(x);
  return x;
}
```

`const` means readnone and no side effects, but check the IR: Clang may still not mark `willreturn`. If the call survives, look at the attributes in `ex.ll`. Add `willreturn` in the `.ll` by hand:

```llvm
declare i32 @ext(i32) readnone willreturn
```

Rerun `opt -passes=dce`. The call should disappear. This is the cleanest way to see that "unused" is not a property of the return value alone.

---

## Example 4: Dead phi cycle

```llvm
define i32 @cycle(i32 %x) {
entry:
  br label %loop
loop:
  %p = phi i32 [ %x, %entry ], [ %q, %loop ]
  %q = phi i32 [ %x, %entry ], [ %p, %loop ]
  br label %loop
dead:
  ret i32 %p
}
```

This function loops forever and the return is unreachable. Think before you run ADCE.

**What is legal:** the phis only feed each other. The return does not execute. Whether the infinite loop may be deleted depends on `mustprogress` (see the main notes and Day 25).

```bash
opt -passes=dce -S cycle.ll
opt -passes=adce -S cycle.ll
```

**Expected:** DCE does not remove the phis, because each has a use. ADCE's result depends on the function attributes. If the loop is still there, that is ADCE refusing to change termination behavior. If you add `mustprogress` to the function and the loop disappears, that is LoopDeletion's territory more than DCE's; try `opt -passes=loop-deletion` on a version that is a real finite dead loop:

```c
int finite(int n) {
  int s = 0;
  for (int i = 0; i < n; ++i)
    s += i;
  return 0;   /* s is unused */
}
```

After mem2reg, the loop still has a phi cycle that *is* used inside the loop but not outside. DCE will not delete the loop. LoopDeletion (Day 25) will, once it proves the loop is finite and side-effect free. Run:

```bash
opt -passes=mem2reg,loop-deletion -S finite.ll
```

and compare with `mem2reg,dce`. Knowing which pass is allowed to do this is the point.

---

## Example 5: Volatile load

```llvm
define i32 @v(ptr %p) {
  %a = load volatile i32, ptr %p
  ret i32 0
}
```

**Expected:** the load remains under `dce` and `adce`.

**Why:** volatile is observable.

**Counterexample:** remove `volatile`. The unused load is trivially dead. `opt -passes=dce` deletes it. A non-volatile load does not "count" as a side effect in IR.

---

## Example 6: Store is not DCE's job

```llvm
@g = external global i32
define void @st(i32 %x) {
  %d = mul i32 %x, 3
  store i32 %d, ptr @g
  ret void
}
```

**Expected:** multiply and store both remain. The multiply has a use (the store). The store is a side effect.

```llvm
define void @deadst(i32 %x) {
  %a = alloca i32
  store i32 %x, ptr %a
  ret void
}
```

**Expected:** DCE does **not** remove this store. The store is a side effect as far as `mayHaveSideEffects` is concerned. SROA/mem2reg remove it because they understand the alloca is never read. Dead store elimination (`opt -passes=dse`) can remove a store to an alloca or to a local that is overwritten. Try:

```bash
opt -passes=dse -S deadst.ll
opt -passes=mem2reg -S deadst.ll
```

`mem2reg` deletes both the store and the alloca. `dse` may delete the store and leave, or not leave, the alloca depending on other uses. `dce` changes nothing. Three passes, three different definitions of "dead".

---

## What to write down

For one function from a real codebase, compiled with `-O1 -Xclang -disable-llvm-passes` and then `mem2reg`:

1. Which instructions does `dce` remove?
2. Which branches does `adce` remove that `dce` keeps?
3. Which calls are keeping a large live set? (`-debug-only=adce` on an asserts build, or just look at the remaining calls.)

That third list is the usual answer to "why didn't DCE shrink this function?".
